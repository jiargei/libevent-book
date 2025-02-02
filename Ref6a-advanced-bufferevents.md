# Bufferevents: advanced topics

This chapter describes some advanced features of Libevent's bufferevent implementation that aren't necessary for typical uses.  If you're just learning how to use bufferevents, you should skip this chapter for now and go on to read l[the evbuffer chapter](Ref7-evbuffer.md).

## Paired bufferevents

Sometimes you have a networking program that needs to talk to itself. For example, you could have a program written to tunnel user connections over some protocol that sometimes also wants to tunnel connections _of its own_ over that protocol.  You could achieve this by opening a connection to your own listening port and having your program use itself, of course, but that would waste resources by having your program talk to itself via the network stack.

Instead, you can create a pair of _paired_ bufferevents such that all bytes written on one are received on the other (and vice versa), but no actual platform sockets are used.

Interface

```c
int bufferevent_pair_new(struct event_base *base, int options,
    struct bufferevent *pair[2]);
```

Calling bufferevent_pair_new() sets pair[0] and pair[1] to a pair of bufferevents, each connected to the other.  All the usual options are supported, except for BEV_OPT_CLOSE_ON_FREE, which has no effect, and BEV_OPT_DEFER_CALLBACKS, which is always on.

Why do bufferevent pairs need to run with callbacks deferred?  It's pretty common for an operation on one element of the pair to invoke a callback that alters the bufferevent, thus invoking the other bufferevent's callbacks, and so on through many steps.  When the callbacks were not deferred, this chain of calls would pretty frequently overflow the stack, starve other connections, and require all the callbacks to be reentrant.

Paired bufferevents support flushing; setting the mode argument to either either BEV_NORMAL or BEV_FLUSH forces all the relevant data to get transferred from one bufferevent in the pair to the other, ignoring the watermarks that would otherwise restrict it.  Setting mode to BEV_FINISHED additionally generates an EOF event on the opposite bufferevent.

Freeing either member of the pair _does not_ automatically free the other or generate an EOF event; it just makes the other member of the pair become unlinked.  Once the bufferevent is unlinked, it will no longer successfully read or write data or generate any events.

Interface

```c
struct bufferevent *bufferevent_pair_get_partner(struct bufferevent *bev)
```

Sometimes you may need to get the other member of a bufferevent pair given only one member.  To do this, you can invoke the bufferevent_pair_get_partner() function.  It will return the other member of the pair if 'bev' is a member of a pair, and the other member still exists. Otherwise, it returns NULL.

Bufferevent pairs were new in Libevent 2.0.1-alpha; the bufferevent_pair_get_partner() function was introduced in  libevent 2.0.6.

## Filtering bufferevents

Sometimes you want to transform all the data passing through a bufferevent object.  You could do this to add a compression layer, or wrap a protocol in another protocol for transport.

Interface

```c
enum bufferevent_filter_result {
    BEV_OK = 0,
    BEV_NEED_MORE = 1,
    BEV_ERROR = 2
};
typedef enum bufferevent_filter_result (*bufferevent_filter_cb)(
    struct evbuffer *source, struct evbuffer *destination, ev_ssize_t dst_limit,
    enum bufferevent_flush_mode mode, void *ctx);


struct bufferevent *bufferevent_filter_new(struct bufferevent *underlying,
    bufferevent_filter_cb input_filter,
        bufferevent_filter_cb output_filter,
    int options,
    void (*free_context)(void *),
    void *ctx);
```

The bufferevent_filter_new() function creates a new filtering bufferevent, wrapped around an existing "underlying" bufferevent.  All data received via the underlying bufferevent is transformed with the "input" filter before arriving at the filtering bufferevent, and all data sent via the filtering bufferevent is transformed with an "output" filter before being sent out to the underlying bufferevent.

Adding a filter to an underlying bufferevent replaces the callbacks on the underlying bufferevent.  You can still add callbacks to the underlying bufferevent's evbuffers, but you can't set the callbacks on the bufferevent itself if you want the filter to still work.

The 'input_filter' and 'output_filter' functions are described below. All the usual options are supported in 'options'.  If BEV_OPT_CLOSE_ON_FREE is set, then freeing the filtering bufferevent also frees the underlying bufferevent.  The 'ctx' field is an arbitrary pointer passed to the filter functions; if a 'free_context' function is provided, it is called on 'ctx' just before the filtering bufferevent is closed.

The input filter function will be called whenever there is new readable data on the underlying input buffer.  The output filter function is called whenever there is new writable data on the filter's output buffer.  Each one receives a pair of evbuffers: a 'source' evbuffer to read data from, and a 'destination' evbuffer to write data to.  The 'dst_limit' argument describes the upper bound of bytes to add to 'destination'.  The filter function is allowed to ignore this value, but doing so might violate high-water marks or rate limits.  If 'dst_limit' is -1, there is no limit.  The 'mode' parameter tells the filter how aggressive to be in writing.  If it is BEV_NORMAL, then it should write as much as can be conveniently transformed. The BEV_FLUSH value means to write as much as possible, and BEV_FINISHED means that the filtering function should additionally do any cleanup necessary at the end of the stream.  Finally, the filter function's 'ctx' argument is a void pointer as provided to the bufferevent_filter_new() constructor.

Filter functions must return BEV_OK if any data was successfully written to the destination buffer, BEV_NEED_MORE if no more data can be written to the destination buffer without getting more input or using a different flush mode, and BEV_ERROR if there is a non-recoverable error on the filter.

Creating the filter enables both reading and writing on the underlying bufferevent.  You do not need to manage reads/writes on your own: the filter will suspend reading on the underlying bufferevent for you whenever it doesn't want to read.  For 2.0.8-rc and later, it is permissible to enable/disable reading and writing on the underlying bufferevent independently from the filter.  If you do this, though, you may keep the filter from successfully getting the data it wants.

You don't need to specify both an input filter and an output filter: any filter you omit is replaced with one that passes data on without transforming it.

// TODO: Write a couple of examples.

## Limiting maximum single read/write size

By default, bufferevents won't read or write the maximum possible amount of bytes on each invocation of the event loop; doing so can lead to weird unfair behaviors and resource starvation.  On the other hand, the defaults might not be reasonable for all situations.

Interface

```c
int bufferevent_set_max_single_read(struct bufferevent *bev, size_t size);
int bufferevent_set_max_single_write(struct bufferevent *bev, size_t size);

ev_ssize_t bufferevent_get_max_single_read(struct bufferevent *bev);
ev_ssize_t bufferevent_get_max_single_write(struct bufferevent *bev);
``` 

The two "set" functions replace the current read and write maxima respectively.  If the 'size' value is 0 or above EV_SSIZE_MAX, they instead set the maxima to the default value.  These functions return 0 on success and -1 on failure.

The two "get" functions return the current per-loop read and write maxima respectively.

These functions were added in 2.1.1-alpha.

## Bufferevents and Rate-limiting

Some programs want to limit the amount of bandwidth used for any single bufferevent, or for a group of bufferevents.  Libevent 2.0.4-alpha and Libevent 2.0.5-alpha added a basic facility to put caps on individual bufferevents, or to assign bufferevents to a rate-limited group.

### The rate-limiting model

Libevent's rate-limiting uses a _token bucket_ algorithm to decide how many bytes to read or write at a time.  Every rate-limited object, at any given time, has a "read bucket" and a "write bucket", the sizes of which determine how many bytes the object is allowed to read or write immediately.  Each bucket has a refill rate, a maximum burst size, and a timing unit or "tick".  Whenever the timing unit elapses, the bucket is refilled proportionally to the refill rate--but if would become fuller than its burst size, any excess bytes are lost.

Thus, the refill rate determines the maximum average rate at which the object will send or receive bytes, and the burst size determines the largest number of bytes that will be sent or received in a single burst.  The timing unit determines the smoothness of the traffic.

### Setting a rate limit on a bufferevent

Interface

```c
#define EV_RATE_LIMIT_MAX EV_SSIZE_MAX
struct ev_token_bucket_cfg;
struct ev_token_bucket_cfg *ev_token_bucket_cfg_new(
    size_t read_rate, size_t read_burst,
    size_t write_rate, size_t write_burst,
    const struct timeval *tick_len);
void ev_token_bucket_cfg_free(struct ev_token_bucket_cfg *cfg);
int bufferevent_set_rate_limit(struct bufferevent *bev,
    struct ev_token_bucket_cfg *cfg);
```

An 'ev_token_bucket_cfg' structure represents the configuration values for a pair of token buckets used to limit reading and writing on a single bufferevent or group of bufferevents.  To create one, call the ev_token_bucket_cfg_new function and provide the maximum average read rate, the maximum read burst, the maximum write rate, the maximum write burst, and the length of a tick.  If the 'tick_len' argument is NULL, the length of a tick defaults to one second.  The function may return NULL on error.

Note that the 'read_rate' and 'write_rate' arguments are scaled in units of bytes per tick.  That is, if the tick is one tenth of a second, and 'read_rate' is 300, then the maximum average read rate is 3000 bytes per second.  Rate and burst values over EV_RATE_LIMIT_MAX are not supported.

To limit a bufferevent's transfer rate, call bufferevent_set_rate_limit() on it with an ev_token_bucket_cfg.  The function returns 0 on success, and -1 on failure.  You can give any number of bufferevents the same ev_token_bucket_cfg.  To remove a bufferevent's rate limits, call bufferevent_set_rate_limit(), passing NULL for the 'cfg' parameter.

To free an ev_token_bucket_cfg, call ev_token_bucket_cfg_free().  Note that it is NOT currently safe to do this until no bufferevents are using the ev_token_bucket_cfg.

### Setting a rate limit on a group of bufferevents

You can assign bufferevents to a 'rate limiting group' if you want to limit their total bandwidth usage.

Interface

```c
struct bufferevent_rate_limit_group;

struct bufferevent_rate_limit_group *bufferevent_rate_limit_group_new(
    struct event_base *base,
    const struct ev_token_bucket_cfg *cfg);
int bufferevent_rate_limit_group_set_cfg(
    struct bufferevent_rate_limit_group *group,
    const struct ev_token_bucket_cfg *cfg);
void bufferevent_rate_limit_group_free(struct bufferevent_rate_limit_group *);
int bufferevent_add_to_rate_limit_group(struct bufferevent *bev,
    struct bufferevent_rate_limit_group *g);
int bufferevent_remove_from_rate_limit_group(struct bufferevent *bev);
```

To construct a rate limiting group, call bufferevent_rate_limit_group() with an event_base and an initial ev_token_bucket_cfg.  You can add bufferevents to the group with bufferevent_add_to_rate_limit_group() and bufferevent_remove_from_rate_limit_group(); these functions return 0 on success and -1 on error.

A single bufferevent can be a member of no more than one rate limiting group at a time.  A bufferevent can have both an individual rate limit (as set with bufferevent_set_rate_limit()) and a group rate limit.  When both limits are set, the lower limit for each bufferevent applies.

You can change the rate limit for an existing group by calling bufferevent_rate_limit_group_set_cfg().  It returns 0 on success and -1 on failure.  The bufferevent_rate_limit_group_free() function frees a rate limit group and removes all of its members.

As of version 2.0, Libevent's group rate limiting tries to be fair on aggregate, but the implementation can be unfair on very small timescales.  If you care strongly about scheduling fairness, please help out
with patches for future versions.

### Inspecting current rate-limit values

Sometimes your code may want to inspect the current rate limits that apply for a given bufferevent or group.  Libevent provides some functions to do so.

Interface

```c
ev_ssize_t bufferevent_get_read_limit(struct bufferevent *bev);
ev_ssize_t bufferevent_get_write_limit(struct bufferevent *bev);
ev_ssize_t bufferevent_rate_limit_group_get_read_limit(
    struct bufferevent_rate_limit_group *);
ev_ssize_t bufferevent_rate_limit_group_get_write_limit(
    struct bufferevent_rate_limit_group *);
```

The above functions return the current size, in bytes, of a bufferevent's or a group's read or write token buckets.  Note that these values can be negative if a bufferevent has been forced to exceed its allocations. (Flushing the bufferevent can do this.)

Interface

```c
ev_ssize_t bufferevent_get_max_to_read(struct bufferevent *bev);
ev_ssize_t bufferevent_get_max_to_write(struct bufferevent *bev);
```

These functions return the number of bytes that a bufferevent would be willing to read or write right now, taking into account any rate limits that apply to the bufferevent, its rate limiting group (if any), and any maximum-to-read/write-at-a-time values imposed by Libevent as a whole.

Interface

```c
void bufferevent_rate_limit_group_get_totals(
    struct bufferevent_rate_limit_group *grp,
    ev_uint64_t *total_read_out, ev_uint64_t *total_written_out);
void bufferevent_rate_limit_group_reset_totals(
    struct bufferevent_rate_limit_group *grp);
```

Each bufferevent_rate_limit_group tracks the total number of bytes sent over it, in total.  You can use this to track total usage by a number of bufferevents in the group.   Calling bufferevent_rate_limit_group_get_totals() on a group sets *'total_read_out' and \*'total_written_out' to the total number of bytes read and written on a bufferevent group respectively.  These totals start at 0 when the group is created, and reset to 0 whenever bufferevent_rate_limit_group_reset_totals() is called on a group.

### Manually adjusting rate limits

For programs with really complex needs, you might want to adjust the current values of a token bucket.  You might want to do this, for example, if your program is generating traffic in some way that isn't via a bufferevent.

Interface

```c
int bufferevent_decrement_read_limit(struct bufferevent *bev, ev_ssize_t decr);
int bufferevent_decrement_write_limit(struct bufferevent *bev, ev_ssize_t decr);
int bufferevent_rate_limit_group_decrement_read(
    struct bufferevent_rate_limit_group *grp, ev_ssize_t decr);
int bufferevent_rate_limit_group_decrement_write(
    struct bufferevent_rate_limit_group *grp, ev_ssize_t decr);
```

These functions decrement a current read or write bucket in a bufferevent or rate limiting group.  Note that the decrements are signed: if you want to increment a bucket, pass a negative value.

### Setting the smallest share possible in a rate-limited group

Frequently, you don't want to divide the bytes available in a rate-limiting group up evenly among all bufferevents in every tick.  For example, if you had 10,000 active bufferevents in a rate-limiting group with 10,000 bytes available for writing every tick, it wouldn't be efficient to let each bufferevent write only 1 byte per tick, due to the overheads of system calls and TCP headers.

To solve this, each rate-limiting group has a notion of its "minimum share". In the situation above, instead of every bufferevent being allowed to write 1 byte per tick, 10,000/SHARE bufferevents will be allowed to write SHARE bytes each every tick, and the rest will be allowed to write nothing.  Which bufferevents are allowed to write first is chosen randomly each tick.

The default minimum share is chosen to give decent performance, and is currently (as of 2.0.6-rc) set to 64. You can adjust this value with the following function:

Interface

```c
int bufferevent_rate_limit_group_set_min_share(
    struct bufferevent_rate_limit_group *group, size_t min_share);
```

Setting the min_share to 0 disables the minimum-share code entirely.

Libevent's rate-limiting has had minimum shares since it was first introduced.  The function to change them was first exposed in Libevent 2.0.6-rc.

### Limitations of the rate-limiting implementation

As of Libevent 2.0, there are some limitations to the rate-limiting implementation that you should know.

- Not every bufferevent type supports rate limiting well, or at all.
- Bufferevent rate limiting groups cannot nest, and a bufferevent can only be in a single rate limiting group at a time.
- The rate limiting implementation only counts bytes transferred in TCP packets as data, doesn't include TCP headers.
- The read-limiting implementation relies on the TCP stack noticing that the application is only consuming data at a certain rate, and pushing back on the other side of the TCP connection when its buffers get full.
- Some implementations of bufferevents (particularly the windows IOCP implementation) can over-commit.
- Buckets start out with one full tick's worth of traffic.  This means that a bufferevent can start reading or writing immediately, and not wait until a full tick has passed.  It also means, though, that a bufferevent that has been rate limited for N.1 ticks can potentially transfer N+1 ticks worth of traffic.
- Ticks cannot be smaller than 1 millisecond, and all fractions of a millisecond are ignored.

/// TODO: Write an example for rate-limiting

## Bufferevents and SSL

Bufferevents can use the OpenSSL library to implement the SSL/TLS secure transport layer.  Because many applications don't need or want to link OpenSSL, this functionality is implemented in a separate library installed as "libevent_openssl".  Future versions of Libevent could add support for other SSL/TLS libraries such as NSS or GnuTLS, but right now OpenSSL is all that's there.

OpenSSL functionality was introduced in Libevent 2.0.3-alpha, though it didn't work so well before Libevent 2.0.5-beta or Libevent 2.0.6-rc.

This section is not a tutorial on OpenSSL, SSL/TLS, or cryptography in general.

These functions are all declared in the header "event2/bufferevent_ssl.h".

## Setting up and using an OpenSSL-based bufferevent

Interface

```c
enum bufferevent_ssl_state {
    BUFFEREVENT_SSL_OPEN = 0,
    BUFFEREVENT_SSL_CONNECTING = 1,
    BUFFEREVENT_SSL_ACCEPTING = 2
};

struct bufferevent *
bufferevent_openssl_filter_new(struct event_base *base,
    struct bufferevent *underlying,
    SSL *ssl,
    enum bufferevent_ssl_state state,
    int options);

struct bufferevent *
bufferevent_openssl_socket_new(struct event_base *base,
    evutil_socket_t fd,
    SSL *ssl,
    enum bufferevent_ssl_state state,
    int options);
```

You can create two kinds of SSL bufferevents: a filter-based bufferevent that communicates over another underlying bufferevent, or a socket-based bufferevent that tells OpenSSL to communicate with the network directly over.  In either case, you must provide an SSL object and a description of the SSL object's state.  The state should be BUFFEREVENT_SSL_CONNECTING if the SSL is currently performing negotiation as a client, BUFFEREVENT_SSL_ACCEPTING if the SSL is currently performing negotiation as a server, or BUFFEREVENT_SSL_OPEN if the SSL handshake is done.

The usual options are accepted; BEV_OPT_CLOSE_ON_FREE makes the SSL object and the underlying fd or bufferevent get closed when the openssl bufferevent itself is closed.

Once the handshake is complete, the new bufferevent's event callback gets invoked with BEV_EVENT_CONNECTED in flags.

If you're creating a socket-based bufferevent and the SSL object already has a socket set, you do not need to provide the socket yourself: just pass -1.  You can also set the fd later with bufferevent_setfd().

/// TODO: Remove this once bufferevent_shutdown() API has been finished.

Note that when BEV_OPT_CLOSE_ON_FREE is set on a SSL bufferevent, a clean shutdown will not be performed on the SSL connection.  This has two problems: first, the connection will seem to have been "broken" by the other side, rather than having been closed cleanly: the other party will not be able to tell whether you closed the connection, or whether it was broken by an attacker or third party.  Second, OpenSSL will treat the session as "bad", and removed from the session cache. This can cause significant performance degradation on SSL applications under load.

Currently the only workaround is to do lazy SSL shutdowns manually. While this breaks the TLS RFC, it will make sure that sessions will stay in cache once closed. The following code implements this workaround.

//BUILD: SKIP
Example

```c
SSL *ctx = bufferevent_openssl_get_ssl(bev);

/*
 * SSL_RECEIVED_SHUTDOWN tells SSL_shutdown to act as if we had already
 * received a close notify from the other end.  SSL_shutdown will then
 * send the final close notify in reply.  The other end will receive the
 * close notify and send theirs.  By this time, we will have already
 * closed the socket and the other end's real close notify will never be
 * received.  In effect, both sides will think that they have completed a
 * clean shutdown and keep their sessions valid.  This strategy will fail
 * if the socket is not ready for writing, in which case this hack will
 * lead to an unclean shutdown and lost session on the other end.
 */
SSL_set_shutdown(ctx, SSL_RECEIVED_SHUTDOWN);
SSL_shutdown(ctx);
bufferevent_free(bev);
```

Interface

```c
SSL *bufferevent_openssl_get_ssl(struct bufferevent *bev);
```

This function returns the SSL object used by an OpenSSL bufferevent, or NULL if 'bev' is not an OpenSSL-based bufferevent.

Interface

```c
unsigned long bufferevent_get_openssl_error(struct bufferevent *bev);
```

This function returns the first pending OpenSSL error for a given bufferevent's operations, or 0 if there was no pending error.  The error format is as returned by ERR_get_error() in the openssl library.

Interface

```c
int bufferevent_ssl_renegotiate(struct bufferevent *bev);
```

Calling this function tells the SSL to renegotiate, and the bufferevent to invoke appropriate callbacks.  This is an advanced topic; you should generally avoid it unless you really know what you're doing, especially since many SSL versions have had known security issues related to renegotiation.

Interface

```c
int bufferevent_openssl_get_allow_dirty_shutdown(struct bufferevent *bev);
void bufferevent_openssl_set_allow_dirty_shutdown(struct bufferevent *bev,
    int allow_dirty_shutdown);
```

All good versions of the SSL protocol (that is, SSLv3 and all TLS versions) support an authenticated shutdown operation that enables the parties to distinguish an intentional close from an accidental or maliciously induced termination in the underling buffer.  By default, we treat anything besides a proper shutdown as an error on the connection. If the allow_dirty_shutdown flag is set to 1, however, we treat a close in the connection as a BEV_EVENT_EOF.

The allow_dirty_shutdown functions were added in Libevent 2.1.1-alpha.

//BUILD: SKIP

Example: A simple SSL-based echo server

```c
include::examples_R6a/R6a_ssl_server.c[]
```

### Some notes on threading and OpenSSL

The built in threading mechanisms of Libevent do not cover OpenSSL locking. Since OpenSSL uses a myriad of global variables, you must still configure OpenSSL to be thread safe. While this process is outside the scope of Libevent, this topic comes up enough to warrant discussion.

//BUILD: SKIP

Example: A very simple example of how to enable thread safe OpenSSL

```c
include::examples_R6a/R6a_ssl_lock_init.c[]
```

_Go back to [Index](README.md)_
