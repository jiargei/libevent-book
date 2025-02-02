# Bufferevents: concepts and basics

Most of the time, an application wants to perform some amount of data buffering in addition to just responding to events.  When we want to write data, for example, the usual pattern runs something like:

- Decide that we want to write some data to a connection; put that   data in a buffer.
- Wait for the connection to become writable
- Write as much of the data as we can
- Remember how much we wrote, and if we still have more data to write,   wait for the connection to become writable again.

This buffered IO pattern is common enough that Libevent provides a generic mechanism for it.  A "bufferevent" consists of an underlying transport (like a socket), a read buffer, and a write buffer.  Instead of regular events, which give callbacks when the underlying transport is ready to be read or written, a bufferevent invokes its user-supplied callbacks when it has read or written enough data.

There are multiple types of bufferevent that all share a common interface.  As of this writing, the following types exist:

socket-based bufferevents::
    A bufferevent that sends and receives data from an underlying stream socket, using the event_* interface as its backend.

asynchronous-IO bufferevents::
    A bufferevent that uses the Windows IOCP interface to send and  receive data to an underlying stream socket.  (Windows only; experimental.)

filtering bufferevents::
    A bufferevent that processes incoming and outgoing data before passing it to an underlying bufferevent object--for example, to compress or translate data.

paired bufferevents::
    Two bufferevents that transmit data to one another.

.NOTE
As of Libevent 2.0.2-alpha, the bufferevents interfaces here are still not fully orthogonal across all bufferevent types.  In other words, not every interface described below will work on all bufferevent types. The Libevent developers intend to correct this in future versions.

.NOTE ALSO
Bufferevents currently only work for stream-oriented protocols like TCP. There may in the future be support for datagram-oriented protocols like UDP.

All of the functions and types in this section are declared in event2/bufferevent.h.  Functions specifically related to evbuffers are declared in event2/buffer.h; see the next chapter for information on those.

## Bufferevents and evbuffers

Every bufferevent has an input buffer and an output buffer.  These are of type "struct evbuffer".  When you have data to write on a bufferevent, you add it to the output buffer; when a bufferevent has data for you to read, you drain it from the input buffer.

The evbuffer interface supports many operations; we discuss them in a later section.

## Callbacks and watermarks

Every bufferevent has two data-related callbacks: a read callback and a write callback.  By default, the read callback is called whenever any data is read from the underlying transport, and the write callback is called whenever enough data from the output buffer is emptied to the underlying transport.  You can override the behavior of these functions by adjusting the read and write "watermarks" of the bufferevent.

Every bufferevent has four watermarks:

Read low-water mark::
   Whenever a read occurs that leaves the bufferevent's input buffer at this level or higher, the bufferevent's read callback is invoked. Defaults to 0, so that every read results in the read callback being invoked.

Read high-water mark::
   If the bufferevent's input buffer ever gets to this level, the bufferevent stops reading until enough data is drained from the input buffer to take us below it again.  Defaults to unlimited, so that we never stop reading because of the size of the input buffer.

Write low-water mark::
   Whenever a write occurs that takes us to this level or below, we invoke the write callback.  Defaults to 0, so that a write callback is not invoked unless the output buffer is emptied.

Write high-water mark::
   Not used by a bufferevent directly, this watermark can have special meaning when a bufferevent is used as the underlying transport of another bufferevent.  See notes on filtering bufferevents below.

A bufferevent also has an "error" or "event" callback that gets invoked to tell the application about non-data-oriented events, like when a connection is closed or an error occurs.  The following event flags are defined:

BEV_EVENT_READING::
   An event occured during a read operation on the bufferevent.  See the other flags for which event it was.

BEV_EVENT_WRITING::
   An event occured during a write operation on the bufferevent.  See the other flags for which event it was.

BEV_EVENT_ERROR::
   An error occurred during a bufferevent operation.  For more information on what the error was, call EVUTIL_SOCKET_ERROR().

BEV_EVENT_TIMEOUT::
   A timeout expired on the bufferevent.

BEV_EVENT_EOF::
   We got an end-of-file indication on the bufferevent.

BEV_EVENT_CONNECTED::
   We finished a requested connection on the bufferevent.

(The above event names are new in Libevent 2.0.2-alpha.)

## Deferred callbacks

By default, a bufferevent callbacks are executed _immediately_ when the corresponding condition happens.  (This is true of evbuffer callbacks too; we'll get to those later.)  This immediate invocation can make trouble when dependencies get complex.  For example, suppose that there is a callback that moves data into evbuffer A when it grows empty, and another callback that processes data out of evbuffer A when it grows full.  Since these calls are all happening on the stack, you might risk a stack overflow if the dependency grows nasty enough.

To solve this, you can tell a bufferevent (or an evbuffer) that its callbacks should be _deferred_.  When the conditions are met for a deferred callback, rather than invoking it immediately, it is queued as part of the event_loop() call, and invoked after  the regular events' callbacks.

(Deferred callbacks were introduced in Libevent 2.0.1-alpha.)

## Option flags for bufferevents

You can use one or more flags when creating a bufferevent to alter its behavior.  Recognized flags are:

BEV_OPT_CLOSE_ON_FREE::
    When the bufferevent is freed, close the underlying transport. This will close an underlying socket, free an underlying bufferevent, etc.

BEV_OPT_THREADSAFE::
    Automatically allocate locks for the bufferevent, so that it's safe to use from multiple threads.

BEV_OPT_DEFER_CALLBACKS::
    When this flag is set, the bufferevent defers all of its callbacks, as described above.

BEV_OPT_UNLOCK_CALLBACKS::
    By default, when the bufferevent is set up to be threadsafe, the bufferevent's locks are held whenever the any user-provided callback is invoked.  Setting this option makes Libevent release the bufferevent's lock when it's invoking your callbacks.

(Libevent 2.0.5-beta introduced BEV_OPT_UNLOCK_CALLBACKS.  The other options above were new in Libevent 2.0.1-alpha.)

## Working with socket-based bufferevents

The simplest bufferevents to work with is the socket-based type.  A socket-based bufferevent uses Libevent's underlying event mechanism to detect when an underlying network socket is ready for read and/or write operations, and uses underlying network calls (like readv, writev, WSASend, or WSARecv) to transmit and receive data.

### Creating a socket-based bufferevent

You can create a socket-based bufferevent using bufferevent_socket_new():

Interface

```c
struct bufferevent *bufferevent_socket_new(
    struct event_base *base,
    evutil_socket_t fd,
    enum bufferevent_options options);
```

The 'base' is an event_base, and 'options' is a bitmask of bufferevent options (BEV_OPT_CLOSE_ON_FREE, etc).  The 'fd' argument is an optional file descriptor for a socket.  You can set fd to -1 if you want to set the file descriptor later.

TIP: [Make sure that the socket you provide to bufferevent_socket_new is in non-blocking mode. Libevent provides the convenience method evutil_make_socket_nonblocking for this.]

This function returns a bufferevent on success, and NULL on failure.

The bufferevent_socket_new() function was introduced in Libevent 2.0.1-alpha.

### Launching connections on socket-based bufferevents

If the bufferevent's socket is not yet connected, you can launch a new connection.

Interface

```c
int bufferevent_socket_connect(struct bufferevent *bev,
    struct sockaddr *address, int addrlen);
```

The address and addrlen arguments are as for the standard call connect().  If the bufferevent does not already have a socket set, calling this function allocates a new stream socket for it, and makes it nonblocking.

If the bufferevent *does* have a socket already, calling bufferevent_socket_connect() tells Libevent that the socket is not connected, and no reads or writes should be done on the socket until the connect operation has succeeded.

It is okay to add data to the output buffer before the connect is done.

This function returns 0 if the connect was successfully launched, and -1 if an error occurred.

Example

```c
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <sys/socket.h>
#include <string.h>

void eventcb(struct bufferevent *bev, short events, void *ptr)
{
    if (events & BEV_EVENT_CONNECTED) {
         /* We're connected to 127.0.0.1:8080.   Ordinarily we'd do
            something here, like start reading or writing. */
    } else if (events & BEV_EVENT_ERROR) {
         /* An error occured while connecting. */
    }
}

int main_loop(void)
{
    struct event_base *base;
    struct bufferevent *bev;
    struct sockaddr_in sin;

    base = event_base_new();

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;   
    sin.sin_addr.s_addr = htonl(0x7f000001); /* 127.0.0.1 */
    sin.sin_port = htons(8080); /* Port 8080 */

    bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);

    bufferevent_setcb(bev, NULL, NULL, eventcb, NULL);

    if (bufferevent_socket_connect(bev,
        (struct sockaddr *)&sin, sizeof(sin)) < 0) {
        /* Error starting connection */
        bufferevent_free(bev);
        return -1;
    }

    event_base_dispatch(base);
    return 0;
}
```

The bufferevent_socket_connect() function was introduced in Libevent-2.0.2-alpha.  Before then, you had to manually call connect() on your socket yourself, and when the connection was complete, the bufferevent would report it as a write.

Note that you only get a BEV_EVENT_CONNECTED event if you launch the connect() attempt using bufferevent_socket_connect().  If you call connect() on your own, the connection gets reported as a write.

If you want to call connect() yourself, but still get receive a BEV_EVENT_CONNECTED event when the connection succeeds, call bufferevent_socket_connect(bev, NULL, 0) after connect() returns -1 with errno equal to EAGAIN or EINPROGRESS.

This function was introduced in Libevent 2.0.2-alpha.

### Launching connections by hostname

Quite often, you'd like to combine resolving a hostname and connecting to it into a single operation.  There's an interface for that:

Interface

```c
int bufferevent_socket_connect_hostname(struct bufferevent *bev,
    struct evdns_base *dns_base, int family, const char *hostname,
    int port);
int bufferevent_socket_get_dns_error(struct bufferevent *bev);
```

This function resolves the DNS name 'hostname', looking for addresses of type 'family'.  (Allowable family types are AF_INET, AF_INET6, and AF_UNSPEC.)  If the name resolution fails, it invokes the event callback with an error event. If it succeeds, it launches a connection attempt just as bufferevent_connect would.

The dns_base argument is optional.  If it is NULL, then Libevent blocks while waiting for the name lookup to finish, which usually isn't what you want.  If it is provided, then Libevent uses it to look up the hostname asynchronously. See [chapter R9](Ref9-dns.md) for more info on DNS.

As with bufferevent_socket_connect(), this function tells Libevent that any existing socket on the bufferevent is not connected, and no reads or writes should be done on the socket until the resolve is finished and the connect operation has succeeded.

If an error occurs, it might be a DNS hostname lookup error.  You can find out what the most recent error was by calling
bufferevent_socket_get_dns_error(). If the returned error code is 0, no DNS error was detected.

//BUILD: SKIP

Example: Trivial HTTP v0 client.

```c
include::examples_R6/R6_http_client.c[]
```

The bufferevent_socket_connect_hostname() function was new in Libevent 2.0.3-alpha; bufferevent_socket_get_dns_error() was new in 2.0.5-beta.

## Generic bufferevent operations

The functions in this section work with multiple bufferevent implementations.

### Freeing a bufferevent

Interface

```c
void bufferevent_free(struct bufferevent *bev);
```

This function frees a bufferevent.  Bufferevents are internally reference-counted, so if the bufferevent has pending deferred callbacks when you free it, it won't be deleted until the callbacks are done.

The bufferevent_free() function does, however, try to free the bufferevent as soon as possible.  If there is pending data to write on the bufferevent, it probably won't be flushed before the bufferevent is freed.

If the BEV_OPT_CLOSE_ON_FREE flag was set, and this bufferevent has a socket or underlying bufferevent associated with it as its transport, that transport is closed when you free the bufferevent.

This function was introduced in Libevent 0.8.

### Manipulating callbacks, watermarks, and enabled operations

Interface

```c
typedef void (*bufferevent_data_cb)(struct bufferevent *bev, void *ctx);
typedef void (*bufferevent_event_cb)(struct bufferevent *bev,
    short events, void *ctx);

void bufferevent_setcb(struct bufferevent *bufev,
    bufferevent_data_cb readcb, bufferevent_data_cb writecb,
    bufferevent_event_cb eventcb, void *cbarg);

void bufferevent_getcb(struct bufferevent *bufev,
    bufferevent_data_cb *readcb_ptr,
    bufferevent_data_cb *writecb_ptr,
    bufferevent_event_cb *eventcb_ptr,
    void **cbarg_ptr);
```

The bufferevent_setcb() function changes one or more of the callbacks of a bufferevent.  The readcb, writecb, and eventcb functions are called (respectively) when enough data is read, when enough data is written, or when an event occurs.  The first argument of each is the bufferevent that has had the event happen.  The last argument is the value provided by the user in the 'cbarg' parameter of bufferevent_callcb(): You can use this to pass data to your callbacks.  The 'events' argument of the event callback is a bitmask of event flags: see "callbacks and watermarks" above.

You can disable a callback by passing NULL instead of the callback function.  Note all the callback functions on a bufferevent share a single 'cbarg' value, so changing it will affect all of them.

You can retrieve the currently set callbacks for a bufferevent by passing pointers to bufferevent_getcb(), which sets *readcb_ptr to the current read callback, \*writecb\_ptr to the current write callback, \*eventcb_ptr to the current event callback, and *cbarg_ptr to the current callback argument field.  Any of these pointers set to NULL will be ignored.

The bufferevent_setcb() function was introduced in Libevent 1.4.4.  The type names "bufferevent_data_cb" and "bufferevent_event_cb" were new in Libevent 2.0.2-alpha.  The bufferevent_getcb() function was added in 2.1.1-alpha.

Interface

```c
void bufferevent_enable(struct bufferevent *bufev, short events);
void bufferevent_disable(struct bufferevent *bufev, short events);

short bufferevent_get_enabled(struct bufferevent *bufev);
```

You can enable or disable the events EV_READ, EV_WRITE, or EV_READ|EV_WRITE on a bufferevent.  When reading or writing is not enabled, the bufferevent will not try to read or write data.

There is no need to disable writing when the output buffer is empty: the bufferevent automatically stops writing, and restarts again when there is data to write.

Similarly, there is no need to disable reading when the input buffer is up to its high-water mark: the bufferevent automatically stops reading, and restarts again when there is space to read.

By default, a newly created bufferevent has writing enabled, but not reading.

You can call bufferevent_get_enabled() to see which events are currently enabled on the bufferevent.

These functions were introduced in Libevent 0.8, except for bufferevent_get_enabled(), which was introduced in version 2.0.3-alpha.

Interface

```c
void bufferevent_setwatermark(struct bufferevent *bufev, short events,
    size_t lowmark, size_t highmark);
```

The bufferevent_setwatermark() function adjusts the read watermarks, the write watermarks, or both, of a single bufferevent.  (If EV_READ is set in the events field, the read watermarks are adjusted.  If EV_WRITE is set in the events field, the write watermarks are adjusted.)

A high-water mark of 0 is equivalent to "unlimited".

This function was first exposed in Libevent 1.4.4.

Example

```c
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <event2/util.h>

#include <stdlib.h>
#include <errno.h>
#include <string.h>

struct info {
    const char *name;
    size_t total_drained;
};

void read_callback(struct bufferevent *bev, void *ctx)
{
    struct info *inf = ctx;
    struct evbuffer *input = bufferevent_get_input(bev);
    size_t len = evbuffer_get_length(input);
    if (len) {
        inf->total_drained += len;
        evbuffer_drain(input, len);
        printf("Drained %lu bytes from %s\n",
             (unsigned long) len, inf->name);
    }
}

void event_callback(struct bufferevent *bev, short events, void *ctx)
{
    struct info *inf = ctx;
    struct evbuffer *input = bufferevent_get_input(bev);
    int finished = 0;

    if (events & BEV_EVENT_EOF) {
        size_t len = evbuffer_get_length(input);
        printf("Got a close from %s.  We drained %lu bytes from it, "
            "and have %lu left.\n", inf->name,
            (unsigned long)inf->total_drained, (unsigned long)len);
        finished = 1;
    }
    if (events & BEV_EVENT_ERROR) {
        printf("Got an error from %s: %s\n",
            inf->name, evutil_socket_error_to_string(EVUTIL_SOCKET_ERROR()));
        finished = 1;
    }
    if (finished) {
        free(ctx);
        bufferevent_free(bev);
    }
}

struct bufferevent *setup_bufferevent(void)
{
    struct bufferevent *b1 = NULL;
    struct info *info1;

    info1 = malloc(sizeof(struct info));
    info1->name = "buffer 1";
    info1->total_drained = 0;

    /* ... Here we should set up the bufferevent and make sure it gets
       connected... */

    /* Trigger the read callback only whenever there is at least 128 bytes
       of data in the buffer. */
    bufferevent_setwatermark(b1, EV_READ, 128, 0);

    bufferevent_setcb(b1, read_callback, NULL, event_callback, info1);

    bufferevent_enable(b1, EV_READ); /* Start reading. */
    return b1;
}
```

### Manipulating data in a bufferevent

Reading and writing data from the network does you no good if you can't look at it.  Bufferevents give you these methods to give them data to write, and to get the data to read:

Interface

```c
struct evbuffer *bufferevent_get_input(struct bufferevent *bufev);
struct evbuffer *bufferevent_get_output(struct bufferevent *bufev);
```

These two functions are very powerful fundamental: they return the input and output buffers respectively.  For full information on all the operations you can perform on an evbuffer type, see the next chapter.

Note that the application may only remove (not add) data on the input buffer, and may only add (not remove) data from the output buffer.

If writing on the bufferevent was stalled because of too little data (or if reading was stalled because of too much), then adding data to the output buffer (or removing data from the input buffer) will automatically restart it.

These functions were introduced in Libevent 2.0.1-alpha.

Interface

```c
int bufferevent_write(struct bufferevent *bufev,
    const void *data, size_t size);
int bufferevent_write_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);
```

These functions add data to a bufferevent's output buffer.  Calling bufferevent_write() adds 'size' bytes from the memory at 'data' to the end of the output buffer.  Calling bufferevent_write_buffer() removes the entire contents of 'buf' and puts them at the end of the output buffer.   Both return 0 if successful, or -1 if an error occurred.

These functions have existed since Libevent 0.8.

Interface

```c
size_t bufferevent_read(struct bufferevent *bufev, void *data, size_t size);
int bufferevent_read_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);
```

These functions remove data from a bufferevent's input buffer.  The bufferevent_read() function removes up to 'size' bytes from the input buffer, storing them into the memory at 'data'.  It returns the number of bytes actually removed.  The bufferevent_read_buffer() function drains the entire contents of the input buffer and places them into 'buf'; it returns 0 on success and -1 on failure.

Note that with bufferevent_read(), the memory chunk at 'data' must actually have enough space to hold 'size' bytes of data.

The bufferevent_read() function has existed since Libevent 0.8; bufferevent_read_buffer() was introduced in Libevent 2.0.1-alpha.

Example

```c
#include <event2/bufferevent.h>
#include <event2/buffer.h>

#include <ctype.h>

void
read_callback_uppercase(struct bufferevent *bev, void *ctx)
{
        /* This callback removes the data from bev's input buffer 128
           bytes at a time, uppercases it, and starts sending it
           back.

           (Watch out!  In practice, you shouldn't use toupper to implement
           a network protocol, unless you know for a fact that the current
           locale is the one you want to be using.)
         */

        char tmp[128];
        size_t n;
        int i;
        while (1) {
                n = bufferevent_read(bev, tmp, sizeof(tmp));
                if (n <= 0)
                        break; /* No more data. */
                for (i=0; i<n; ++i)
                        tmp[i] = toupper(tmp[i]);
                bufferevent_write(bev, tmp, n);
        }
}

struct proxy_info {
        struct bufferevent *other_bev;
};
void
read_callback_proxy(struct bufferevent *bev, void *ctx)
{
        /* You might use a function like this if you're implementing
           a simple proxy: it will take data from one connection (on
           bev), and write it to another, copying as little as
           possible. */
        struct proxy_info *inf = ctx;

        bufferevent_read_buffer(bev,
            bufferevent_get_output(inf->other_bev));
}

struct count {
        unsigned long last_fib[2];
};

void
write_callback_fibonacci(struct bufferevent *bev, void *ctx)
{
        /* Here's a callback that adds some Fibonacci numbers to the
           output buffer of bev.  It stops once we have added 1k of
           data; once this data is drained, we'll add more. */
        struct count *c = ctx;

        struct evbuffer *tmp = evbuffer_new();
        while (evbuffer_get_length(tmp) < 1024) {
                 unsigned long next = c->last_fib[0] + c->last_fib[1];
                 c->last_fib[0] = c->last_fib[1];
                 c->last_fib[1] = next;

                 evbuffer_add_printf(tmp, "%lu", next);
        }

        /* Now we add the whole contents of tmp to bev. */
        bufferevent_write_buffer(bev, tmp);

        /* We don't need tmp any longer. */
        evbuffer_free(tmp);
}
```

### Read- and write timeouts

As with other events, you can have a timeout get invoked if a certain amount of time passes without any data having been successfully written or read by a bufferevent.

Interface

```c
void bufferevent_set_timeouts(struct bufferevent *bufev,
    const struct timeval *timeout_read, const struct timeval *timeout_write);
```

Setting a timeout to NULL is supposed to remove it; however before Libevent 2.1.2-alpha this wouldn't work with all event types.  (As a workaround for older versions, you can try setting the timeout to a multi-day interval and/or having your eventcb function ignore BEV_TIMEOUT events when you don't want them.)

The read timeout will trigger if the bufferevent waits at least 'timeout_read' seconds while trying to read read.  The write timeout will trigger if the bufferevent waits at least 'timeout_write' seconds while trying to write data.

Note that the timeouts only count when the bufferevent would like to read or write.  In other words, the read timeout is not enabled if reading is disabled on the bufferevent, or if the input buffer is full (at its high-water mark).  Similarly, the write timeout is not enabled if writing is disabled, or if there is no data to write.

When a read or write timeout occurs, the corresponding read or write operation becomes disabled on the bufferevent.  The event callback is then invoked with either BEV_EVENT_TIMEOUT|BEV_EVENT_READING or BEV_EVENT_TIMEOUT|BEV_EVENT_WRITING.

This functions has existed since Libevent 2.0.1-alpha.  It didn't behave consistently across bufferevent types until Libevent 2.0.4-alpha.

### Initiating a flush on a bufferevent

Interface

```c
int bufferevent_flush(struct bufferevent *bufev,
    short iotype, enum bufferevent_flush_mode state);
```

Flushing a bufferevent tells the bufferevent to force as many bytes as possible to be read to or written from the underlying transport, ignoring other restrictions that might otherwise keep them from being written.  Its detailed function depends on the type of the bufferevent.

The iotype argument should be EV_READ, EV_WRITE, or EV_READ|EV_WRITE to indicate whether bytes being read, written, or both should be processed.  The state argument may be one of BEV_NORMAL, BEV_FLUSH, or BEV_FINISHED.  BEV_FINISHED indicates that the other side should be told that no more data will be sent; the distinction between BEV_NORMAL and BEV_FLUSH depends on the type of the bufferevent.

The bufferevent_flush() function returns -1 on failure, 0 if no data was flushed, or 1 if some data was flushed.

Currently (as of Libevent 2.0.5-beta), bufferevent_flush() is only implemented for some bufferevent types.  In particular, socket-based bufferevents don't have it.

### Type-specific bufferevent functions

These bufferevent functions are not supported on all bufferevent types.

Interface

```c
int bufferevent_priority_set(struct bufferevent *bufev, int pri);
int bufferevent_get_priority(struct bufferevent *bufev);
```

This function adjusts the priority of the events used to implement 'bufev' to 'pri'.  See event_priority_set() for more information on priorities.

This function returns 0 on success, and -1 on failure.  It works on socket-based bufferevents only.

The bufferevent_priority_set() function was introduced in Libevent 1.0; bufferevent_get_priority() didn't appear until Libevent 2.1.2-alpha.

Interface

```c
int bufferevent_setfd(struct bufferevent *bufev, evutil_socket_t fd);
evutil_socket_t bufferevent_getfd(struct bufferevent *bufev);
```

These functions set or return the file descriptor for a fd-based event.  Only socket-based bufferevents support setfd().  Both return -1 on failure; setfd() returns 0 on success.

The bufferevent_setfd() function was introduced in Libevent 1.4.4; the bufferevent_getfd() function was introduced in Libevent 2.0.2-alpha.

Interface

```c
struct event_base *bufferevent_get_base(struct bufferevent *bev);
```

This function returns the event_base of a bufferevent.  It was introduced in 2.0.9-rc.

Interface

```c
struct bufferevent *bufferevent_get_underlying(struct bufferevent *bufev);
```

This function returns the bufferevent that another bufferevent is using as a transport, if any.  For information on when this situation would occur, see notes on filtering bufferevents.

This function was introduced in Libevent 2.0.2-alpha.

### Manually locking and unlocking a bufferevent

As with evbuffers, sometimes you want to ensure that a number of operations on a bufferevent are all performed atomically.  Libevent exposes functions that you can use to manually lock and unlock a bufferevent.

Interface

```c
void bufferevent_lock(struct bufferevent *bufev);
void bufferevent_unlock(struct bufferevent *bufev);
```

Note that locking a bufferevent has no effect if the bufferevent was not given the BEV_OPT_THREADSAFE thread on creation, or if Libevent's threading support wasn't activated.

Locking the bufferevent with this function will lock its associated evbuffers as well.  These functions are recursive: it is safe to lock a bufferevent for which you already hold the lock.  You must, of course, call unlock once for every time that you locked the bufferevent.

These functions were introduced in Libevent 2.0.6-rc.

### Obsolete bufferevent functionality

The bufferevent backend code underwent substantial revision between Libevent 1.4 and Libevent 2.0.  In the old interface, it was sometimes normal to build with access to the internals of the struct bufferevent, and to use macros that relied on this access.

To make matters confusing, the old code sometimes used names for bufferevent functionality that were prefixed with "evbuffer".

Here's a brief guideline of what things used to be called before Libevent 2.0:

| Current name               | Old name |
| --- | --- |
| bufferevent_data_cb        | evbuffercb |
| bufferevent_event_cb       | everrorcb |
| BEV_EVENT_READING          | EVBUFFER_READ |
| BEV_EVENT_WRITE            | EVBUFFER_WRITE |
| BEV_EVENT_EOF              | EVBUFFER_EOF |
| BEV_EVENT_ERROR            | EVBUFFER_ERROR |
| BEV_EVENT_TIMEOUT          | EVBUFFER_TIMEOUT |
| bufferevent_get_input(b)   | EVBUFFER_INPUT(b) |
| bufferevent_get_output(b)  | EVBUFFER_OUTPUT(b) |

The old functions were defined in event.h, not in event2/bufferevent.h.

If you still need access to the internals of the common parts of the bufferevent struct, you can include event2/bufferevent_struct.h.  We recommend against it: the contents of struct bufferevent WILL change between versions of Libevent.  The macros and names in this section are available if you include event2/bufferevent_compat.h.

The interface to set up a bufferevent differed in older versions:

Interface

```c
struct bufferevent *bufferevent_new(evutil_socket_t fd,
    evbuffercb readcb, evbuffercb writecb, everrorcb errorcb, void *cbarg);
int bufferevent_base_set(struct event_base *base, struct bufferevent *bufev);
```

The bufferevent_new() function creates a socket bufferevent only, and does so on the deprecated "default" event_base.  Calling bufferevent_base_set adjusts the event_base of a socket bufferevent only.

Instead of setting timeouts as struct timeval, they were set as numbers of seconds:

Interface

```c
void bufferevent_settimeout(struct bufferevent *bufev,
    int timeout_read, int timeout_write);
```

Finally, note that the underlying evbuffer implementation for Libevent versions before 2.0 was pretty inefficient, to the point where using bufferevents for high-performance apps was kind of questionable.

_Go back to [Index](README.md)_
