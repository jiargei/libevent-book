# Using DNS with Libevent: high and low-level functionality

Libevent provides a few APIs to use for resolving DNS names, and a facility for implementing simple DNS servers.

We'll start by describing the higher-level facilities for name lookup, and then describe the low-level and server facilities.

.Note

There are known limitations in Libevent's current DNS client implementation. It doesn't support TCP lookups, DNSSec, or arbitrary record types.  We'd like to fix all of these in some future version of Libevent, but for now, they're not there.

## Preliminaries: Portable blocking name resolution

To aid in porting programs that already use blocking name resolution, Libevent provides a portable implementation of the standard getaddrinfo() interface.  This can be helpful when your program needs to run on platforms where either there is no getaddrinfo() function, or where getaddrinfo() doesn't conform to the standard as well as our replacement.  (There are shockingly many of each.)

The getaddrinfo() interface is specified in RFC 3493, section 6.1.  See the "Compatibility Notes" section below for a summary of how we fall short of a conformant implemenation.

.Interface

```c
struct evutil_addrinfo {
    int ai_flags;
    int ai_family;
    int ai_socktype;
    int ai_protocol;
    size_t ai_addrlen;
    char *ai_canonname;
    struct sockaddr *ai_addr;
    struct evutil_addrinfo *ai_next;
};

#define EVUTIL_AI_PASSIVE     /* ... */
#define EVUTIL_AI_CANONNAME   /* ... */
#define EVUTIL_AI_NUMERICHOST /* ... */
#define EVUTIL_AI_NUMERICSERV /* ... */
#define EVUTIL_AI_V4MAPPED    /* ... */
#define EVUTIL_AI_ALL         /* ... */
#define EVUTIL_AI_ADDRCONFIG  /* ... */

int evutil_getaddrinfo(const char *nodename, const char *servname,
    const struct evutil_addrinfo *hints, struct evutil_addrinfo **res);
void evutil_freeaddrinfo(struct evutil_addrinfo *ai);
const char *evutil_gai_strerror(int err);
```

The evutil_getaddrinfo() function tries to resolve the provided nodename and servname fields, according to the rules you give it in 'hints', and build you a linked list of evutil_addrinfo structures and store them in *res.  It returns 0 on success, and a nonzero error code on failure.

You must provide at least one of 'nodename' and 'servname'.  If 'nodename' is provided, it is either a literal IPv4 address (like "127.0.0.1"), a literal IPv6 address (like "::1"), or a DNS name (like "www.example.com").  If 'servname' is provided, it is either the symbolic name of a network service (like "https") or a string containing a port number given in decimal (like "443").

If you do not specify 'servname', then the port values in \*res will be set to zero.  If you do not specify 'nodename', then the addresses in *res will either be for localhost (by default), or for "any" (if EVUTIL_AI_PASSIVE is set.)

The ai_flags field of 'hints' tells evutil_getaddrinfo how to perform the lookup.  It can contain zero or more of the flags below, ORed together.

EVUTIL_AI_PASSIVE::
    This flag indicates that we're going to be using the address for listening, not for connection.  Ordinarily this makes no difference, except when 'nodename' is NULL: for connecting, a NULL nodename is localhost (127.0.0.1 or ::1), whereas when listening, a NULL node name is ANY (0.0.0.0 or ::0).

EVUTIL_AI_CANONNAME::
    If this flag is set, we try to report the canonical name for the host in the ai_canonname field.

EVUTIL_AI_NUMERICHOST::
    When this flag is set, we only resolve numeric IPv4 and IPv6 addresses; if the 'nodename' would require a name lookup, we instead give an EVUTIL_EAI_NONAME error.

EVUTIL_AI_NUMERICSERV::
    When this flag is set, we only resolve numeric service names. If the 'servname' is neither NULL nor a decimal integer, give an EVUTIL_EAI_NONAME error.

EVUTIL_AI_V4MAPPED::
    This flag indicates that if ai_family is AF_INET6, and no IPv6 addresses are found, any IPv4 addresses in the result should be returned as v4-mapped IPv6 addresses.  It is not currently supported by evutil_getaddrinfo() unless the OS supports it.

EVUTIL_AI_ALL::
    If this flag and EVUTIL_AI_V4MAPPED are both set, then IPv4 addresses in the result included in the result as 4-mapped IPv6 addresses, whether there are any IPv6 addresses or not.  It is not currently supported by evutil_getaddrinfo() unless the OS supports it.

EVUTIL_AI_ADDRCONFIG::
    If this flag is set, then IPv4 addresses are only included in the result if the system has a nonlocal IPv4 address, and IPv6 addresses are only included in the result if the system has a nonlocal IPv6 address.

The ai_family field of 'hints' is used to tell evutil_getaddrinfo() which addresses it should return.  It can be AF_INET to request IPv4 addresses only, AF_INET6 to request IPv6 addresses only, or AF_UNSPEC to request all available addresses.

The ai_socktype and ai_protocol fields of 'hints' are used to tell evutil_getaddrinfo() how you're going to use the address.  They're the same as the socktype and protocol fields you would pass to socket().

If evutil_getaddrinfo() is successful, it allocates a new linked list of evutil_addrinfo structures, where each points to the next with its "ai_next" pointer, and stores them in *res.  Because this value is heap-allocated, you will need to use evutil_freeaddrinfo to free it.

If it fails, it returns one of these numeric error codes:

EVUTIL_EAI_ADDRFAMILY::
    You requested an address family that made no sense for the nodename.

EVUTIL_EAI_AGAIN::
    There was a recoverable error in name resolution; try again later.

EVUTIL_EAI_FAIL::
    There was a non-recoverable error in name resolution; your resolver or your DNS server may be busted.

EVUTIL_EAI_BADFLAGS::
    The ai_flags field in hints was somehow invalid.

EVUTIL_EAI_FAMILY::
    The ai_family field in hints was not one we support.

EVUTIL_EAI_MEMORY::
    We ran out of memory while trying to answer your request.

EVUTIL_EAI_NODATA::
    The host you asked for exists, but has no address information associated with it.  (Or, it has no address information of the type you requested.)

EVUTIL_EAI_NONAME::
      The host you asked for doesn't seem to exist.

EVUTIL_EAI_SERVICE::
      The service you asked for doesn't seem to exist.

EVUTIL_EAI_SOCKTYPE::
      We don't support the socket type you asked for, or it isn't compatible with ai_protocol.

EVUTIL_EAI_SYSTEM::
      There was some other system error during name resolution.  Check errno for more information.

EVUTIL_EAI_CANCEL::
      The application requested that this DNS lookup should be canceled before it was finished.  The evutil_getaddrinfo() function never produces this error, but it can come from evdns_getaddrinfo() as described in the section below.

You can use evutil_gai_strerror() to convert one of these results into a human-readable string.

Note: If your OS defines struct addrinfo, then evutil_addrinfo is just an alias for your OS's built-in structure.  Similarly, if your operating system defines any of the AI_\* flags, then the corresponding EVUTIL_AI_\* flag is just an alias for the native flag; and if your operating system defines any of the EAI_* errors, then the corresponding EVUTIL_EAI_* code is the same as your platform's native error code.

.Example: Resolving a hostname and making a blocking connection

```c
#include <event2/util.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <stdio.h>
#include <string.h>
#include <assert.h>
#include <unistd.h>

evutil_socket_t
get_tcp_socket_for_host(const char *hostname, ev_uint16_t port)
{
    char port_buf[6];
    struct evutil_addrinfo hints;
    struct evutil_addrinfo *answer = NULL;
    int err;
    evutil_socket_t sock;

    /* Convert the port to decimal. */
    evutil_snprintf(port_buf, sizeof(port_buf), "%d", (int)port);

    /* Build the hints to tell getaddrinfo how to act. */
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_UNSPEC; /* v4 or v6 is fine. */
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP; /* We want a TCP socket */
    /* Only return addresses we can use. */
    hints.ai_flags = EVUTIL_AI_ADDRCONFIG;

    /* Look up the hostname. */
    err = evutil_getaddrinfo(hostname, port_buf, &hints, &answer);
    if (err != 0) {
          fprintf(stderr, "Error while resolving '%s': %s",
                  hostname, evutil_gai_strerror(err));
          return -1;
    }

    /* If there was no error, we should have at least one answer. */
    assert(answer);
    /* Just use the first answer. */
    sock = socket(answer->ai_family,
                  answer->ai_socktype,
                  answer->ai_protocol);
    if (sock < 0)
        return -1;
    if (connect(sock, answer->ai_addr, answer->ai_addrlen)) {
        /* Note that we're doing a blocking connect in this function.
         * If this were nonblocking, we'd need to treat some errors
         * (like EINTR and EAGAIN) specially. */
        EVUTIL_CLOSESOCKET(sock);
        return -1;
    }

    return sock;
}
```

These functions and constants were new in Libevent 2.0.3-alpha.  They are declared in event2/util.h.

## Non-blocking hostname resolution with evdns_getaddrinfo()

The main problem with the regular getaddrinfo() interface, and with evutil_getaddrinfo() above, is that they're blocking: when you call them, the thread you're in has to wait while they query your DNS server(s) and wait for a response.  Since you're using Libevent, that probably isn't the behavior you want.

So for nonblocking use, Libevent provides a set of functions to launch DNS requests, and use Libevent to wait for the server to answer.

.Interface

```c
typedef void (*evdns_getaddrinfo_cb)(
    int result, struct evutil_addrinfo *res, void *arg);
struct evdns_getaddrinfo_request;

struct evdns_getaddrinfo_request *evdns_getaddrinfo(
    struct evdns_base *dns_base,
    const char *nodename, const char *servname,
    const struct evutil_addrinfo *hints_in,
    evdns_getaddrinfo_cb cb, void *arg);

void evdns_getaddrinfo_cancel(struct evdns_getaddrinfo_request *req);
```

The evdns_getaddrinfo() function behaves just like evutil_getaddrinfo(), except that instead of blocking on DNS servers, it uses Libevent's low-level DNS facilities to look hostnames up for you.  Because it can't always return you the result immediately, you need to provide it a callback function of type evdns_getaddrinfo_cb, and an optional user-supplied argument for that callback function.

Additionally, you need to provide evdns_getaddrinfo() with a pointer to an evdns_base.  This structure holds the state  and configuration for Libevent's DNS resolver.  See the next section for more information on how to get one.

The evdns_getaddrinfo() function returns NULL if it fails or succeeds immediately.  Otherwise, it returns a pointer to an evdns_getaddrinfo_request.  You can use this to cancel the request with evdns_getaddrinfo_cancel() at any time before the request is finished.

Note that the callback function _will_ eventually be invoked whether evdns_getaddrinfo() returns NULL or not, and whether evdns_getaddrinfo_cancel() is called or not.

When you call evdns_getaddrinfo(), it makes its own internal copies of its nodename, servname, and hints arguments: you do not need to ensure that they continue to exist while the name lookup is in progress.

//BUILD: SKIP

.Example: Nonblocking lookups with evdns_getaddrinfo()

```c
include::examples_R9/R9_multilookup.c[]
```

These functions were new in Libevent 2.0.3-alpha.  They are declared in event2/dns.h.

## Creating and configuring an evdns_base

Before you can do nonblocking DNS lookups with evdns, you'll need to configure an evdns_base.  Each evdns_base stores a list of nameservers, and DNS configuration options, and tracks active and in-flight DNS requests.

.Interface

```c
struct evdns_base *evdns_base_new(struct event_base *event_base,
       int initialize);
void evdns_base_free(struct evdns_base *base, int fail_requests);
```

The evdns_base_new() function returns a new evdns_base on success, and NULL on failure.  If the 'initialize' argument is 1, it tries to configure the DNS base sensibly given your operating system's default. If it is 0, it leaves the evdns_base empty, with no nameservers or options configured.

When you no longer need an evdns_base, you can free it with evdns_base_free.  If its 'fail_requests' argument is true, it will make all in-flight requests get their callbacks invoked with a 'canceled' error code before it frees the base.

### Initializing evdns from the system configuration

If you want a little more control over how the evdns_base is initialized, you can pass 0 as the 'initialize' argument to evdns_base_new, and invoke one of these functions.

.Interface

```c
#define DNS_OPTION_SEARCH 1
#define DNS_OPTION_NAMESERVERS 2
#define DNS_OPTION_MISC 4
#define DNS_OPTION_HOSTSFILE 8
#define DNS_OPTIONS_ALL 15
int evdns_base_resolv_conf_parse(struct evdns_base *base, int flags,
                                 const char *filename);

#ifdef WIN32
int evdns_base_config_windows_nameservers(struct evdns_base *);
#define EVDNS_BASE_CONFIG_WINDOWS_NAMESERVERS_IMPLEMENTED
#endif
```

The evdns_base_resolv_conf_parse() function will scan the resolv.conf formatted file stored in 'filename', and read in all the options from it that are listed in 'flags'.  (For more information on the resolv.conf file, see your local Unix manual pages.)

DNS_OPTION_SEARCH::
    Tells evdns to read the 'domain' and 'search' fields from the resolv.conf file and the 'ndots' option, and use them to decide which domains (if any) to search for hostnames that aren't fully-qualified.

DNS_OPTION_NAMESERVERS::
    This flag tells evdns to learn the nameservers from the resolv.conf file.

DNS_OPTION_MISC::
    Tells evdns to set other configuration options from the resolv.conf file.

DNS_OPTION_HOSTSFILE::
    Tells evdns to read a list of hosts from /etc/hosts as part of loading the resolv.conf file.

DNS_OPTIONS_ALL::
    Tells evdns to learn as much as it can from the resolv.conf file.

On Windows, you don't have a resolv.conf file to tell you where your nameservers are, so you can use the
evdns_base_config_windows_nameservers() function to read all your nameservers from your registry (or your NetworkParams, or wherever they're hidden).

#### The resolv.conf file format

The resolv.conf format we recognize is a text file, each line of which should either be empty, contain a comment starting with the # character, or consist of a token followed zero or more arguments.  The tokens we recognize are:

nameserver::
    Must be followed by the IP address of exactly one nameserver.  As an extension, Libevent allows you to specify a nonstandard port for the nameserver, using the IP:Port or the [IPv6]:port syntax.

domain::
    The local domain name.

search::
    A list of names to search when resolving local hostnames. Any name that has fewer than "ndots" dots in it is considered local, and if we can't resolve it as-is, we look in these domain names.  For example, if "search" is example.com and "ndots" is 1, then when the user asks us to resolve "www", we will consider "www.example.com".

options::
    A space-separated list of options.  Each option is given either as a bare string, or (if it takes an argument) in the option:value format.  Recognized options are:

    ndots:INTEGER;;
        Used to configure searching.  See "search" above.  Defaults to 1.
    timeout:FLOAT;;
        How long, in seconds, do we wait for a response from a DNS server before we assume we aren't getting one?  Defaults to 5 seconds.
    max-timeouts:INT;;
        How many times do we allow a nameserver to time-out in a row before we assume that it's down?  Defaults to 3.
    max-inflight:INT;;
        How many DNS requests do we allow to be pending at once?  (If we try to do more requests than this, the extras will stall until the earlier ones are answered or time out.)  Defaults to 64.
    attempts:INT;;
        How many times to we re-transmit a DNS request before giving up on it?  Defaults to 3.
    randomize-case:INT;;
        If nonzero, we randomize the case on outgoing DNS requests and make sure that replies have the same case as our requests.  This so-called "0x20 hack" can help prevent some otherwise simple active events against DNS.  Defaults to 1.
    bind-to:ADDRESS;;
        If provided, we bind to the given address whenever we send packets to a nameserver.  As of Libevent 2.0.4-alpha, it only applied to subsequent nameserver entries.
    initial-probe-timeout:FLOAT;;
        When we decide that a nameserver is down, we probe it with exponentially decreasing frequency to see if it has come back up.  This option configures the first timeout in the series, in seconds.  Defaults to 10.
    getaddrinfo-allow-skew:FLOAT;;
        When evdns_getaddrinfo() requests both an IPv4 address and an IPv6 address, it does so in separate DNS request packets, since some servers can't handle both requests in one packet.  Once it has an answer for one address type, it waits a little while to see if an answer for the other one comes in.  This option configures how long to wait, in seconds.  Defaults to 3 seconds.

Unrecognized tokens and options are ignored.

#### Configuring evdns manually

If you want even more fine-grained control over evdns's behavior, you can use these functions:

.Interface

```c
int evdns_base_nameserver_sockaddr_add(struct evdns_base *base,
                                 const struct sockaddr *sa, ev_socklen_t len,
                                 unsigned flags);
int evdns_base_nameserver_ip_add(struct evdns_base *base,
                                 const char *ip_as_string);
int evdns_base_load_hosts(struct evdns_base *base, const char *hosts_fname);

void evdns_base_search_clear(struct evdns_base *base);
void evdns_base_search_add(struct evdns_base *base, const char *domain);
void evdns_base_search_ndots_set(struct evdns_base *base, int ndots);

int evdns_base_set_option(struct evdns_base *base, const char *option,
    const char *val);

int evdns_base_count_nameservers(struct evdns_base *base);
```

The evdns_base_nameserver_sockaddr_add() function adds a nameserver to an existing evdns_base by its address.  The 'flags' argument is currently ignored, and should be 0 for forward-compatibility.  The function returns 0 on success and negative on failure.  (It was added in Libevent 2.0.7-rc.)

The evdns_base_nameserver_ip_add function adds a nameserver to an existing evdns_base.  It takes the nameserver in a text string, either as an IPv4 address, an IPv6 address, an IPv4 address with a port (IPv4:Port), or an IPv6 address with a port ([IPv6]:Port).  It returns 0 on success and negative on failure.

The evdns_base_load_hosts() function loads a hosts file (in the same format as /etc/hosts) from hosts_fname.  It also returns 0 on success and negative on failure.

The evdns_base_search_clear() function removes all current search suffixes (as configured by the 'search' option) from the evdns_base; the evdns_base_search_add() function adds a suffix.

The evdns_base_set_option() function sets a given option to a given value in the evdns_base.  Each one is given as a string.  (Before Libevent 2.0.3, the option name needed to have a colon after it.)

If you've just parsed a set of configuration files and want to see if any nameservers were added, you can use evdns_base_count_nameservers() to see how many there are.

#### Library-side configuration

There are a couple of functions you can use to specify library-wide settings for the evdns module:

.Interface

```c
typedef void (*evdns_debug_log_fn_type)(int is_warning, const char *msg);
void evdns_set_log_fn(evdns_debug_log_fn_type fn);
void evdns_set_transaction_id_fn(ev_uint16_t (*fn)(void));
```

For historical reasons, the evdns subsystem does its own logging; you can use evdns_set_log_fn() to give it a callback that does something with its messages besides discard them.

For security, evdns needs a good source of random numbers: it uses this to pick hard-to-guess transaction IDs and to randomize queries when using the 0x20 hack.  (See the "randomize-case" option for more info here.)  Older versions of Libevent, did not provide a secure RNG of its own, however.  You can give evdns a better random number generator by calling evdns_set_transaction_id_fn and giving it a function that returns a hard-to-predict two-byte unsigned integer.

In Libevent 2.0.4-alpha and later, Libevent uses its own built-in secure RNG; evdns_set_transaction_id_fn() has no effect.

// XXXX Add a reference for the 0x20 hack

##### Low-level DNS interfaces

Occasionally, you'll want the ability to launch specific DNS requests with more fine-grained control than you get from evdns_getaddrinfo(). Libevent gives you some interfaces to do that.

.Missing features

Right now, Libevent's DNS support lacks a few features that you'd expect from a low-level DNS system, like support for arbitrary request types and TCP requests.  If you need features that evdns doesn't have, please consider contributing a patch.  You might also look into a more full-featured DNS library like c-ares.

.Interface

```c
#define DNS_QUERY_NO_SEARCH /* ... */

#define DNS_IPv4_A         /* ... */
#define DNS_PTR            /* ... */
#define DNS_IPv6_AAAA      /* ... */

typedef void (*evdns_callback_type)(int result, char type, int count,
    int ttl, void *addresses, void *arg);

struct evdns_request *evdns_base_resolve_ipv4(struct evdns_base *base,
    const char *name, int flags, evdns_callback_type callback, void *ptr);
struct evdns_request *evdns_base_resolve_ipv6(struct evdns_base *base,
    const char *name, int flags, evdns_callback_type callback, void *ptr);
struct evdns_request *evdns_base_resolve_reverse(struct evdns_base *base,
    const struct in_addr *in, int flags, evdns_callback_type callback,
    void *ptr);
struct evdns_request *evdns_base_resolve_reverse_ipv6(
    struct evdns_base *base, const struct in6_addr *in, int flags,
    evdns_callback_type callback, void *ptr);
```

These resolve functions initiate a DNS request for a particular record.  Each takes an evdns_base to use for the request, a resource to look up (either a hostname for forward lookups, or an address for reverse lookups), a set of flags to determine how to do the lookup, a callback to invoke when the lookup is done, and a pointer to pass to the user-supplied callback.

The 'flags' argument is either 0 or DNS_QUERY_NO_SEARCH to explicitlys uppress searching in the list of search if the original search fails. DNS_QUERY_NOw_SEARCH has no effect for reverse lookups, since those never do searching.

When the request is done---either successfully or not---the callback function will be invoked.  The callback takes a 'result' that indicates success or an error code (see DNS Errors table below), a record type (one of DNS_IPv4_A, DNS_IPv6_AAAA, or DNS_PTR), the number of records in 'addresses', a time-to-live in seconds, the addresses themselves, and the user-supplied argument pointer.

The 'addresses' argument to the callback is NULL in the event of an error. For a PTR record, it's a NUL-terminated string.  For IPv4 records, it is an array of four-byte values in network order.  For IPv6 records, it is an array of 16-byte records in network order.  (Note that the number of addresses can be 0 even if there was no error.  This can happen when the name exists, but it has no records of the requested type.)

The errors codes that can be passed to the callback are as follows:

.DNS Errors

| Code                | Meaning |
| --- | --- |
|DNS_ERR_NONE         | No error occurred |
|DNS_ERR_FORMAT       | The server didn't understand the query |
|DNS_ERR_SERVERFAILED | The server reported an internal error |
|DNS_ERR_NOTEXIST     | There was no record with the given name |
|DNS_ERR_NOTIMPL      | The server doesn't understand this kind of query |
|DNS_ERR_REFUSED      | The server rejected the query for policy reasons |
|DNS_ERR_TRUNCATED    | The DNS record wouldn't fit in a UDP packet |
|DNS_ERR_UNKNOWN      | Unknown internal error |
|DNS_ERR_TIMEOUT      | We waited too long for an answer |
|DNS_ERR_SHUTDOWN     | The user asked us to shut down the evdns system |
|DNS_ERR_CANCEL       | The user asked us to cancel this request |
|DNS_ERR_NODATA       | The response arrived, but contained no answers |

(DNS_ERR_NODATA was new in 2.0.15-stable.)

You can decode these error codes to a human-readable string with:

.Interface

```c
const char *evdns_err_to_string(int err);
```

Each resolve function returns a pointer to an opaque 'evdns_request' structure.  You can use this to cancel the request at any point before the callback is invoked:

.Interface

```c
void evdns_cancel_request(struct evdns_base *base,
    struct evdns_request *req);
```

Canceling a request with this function makes its callback get invoked with the DNS_ERR_CANCEL result code.

## Suspending DNS client operations and changing nameservers

Sometimes you want to reconfigure or shut down the DNS subsystem without affecting in-flight DNS request too much.

Interface

```c
int evdns_base_clear_nameservers_and_suspend(struct evdns_base *base);
int evdns_base_resume(struct evdns_base *base);
```

If you call evdns_base_clear_nameservers_and_suspend() on an evdns_base, all nameservers are removed, and pending requests are left in limbo until later you re-add nameservers and call evdns_base_resume().

These functions return 0 on success and -1 on failure.  They were introduced in Libevent 2.0.1-alpha.

### DNS server interfaces

Libevent provides simple functionality for acting as a trivial DNS server and responding to UDP DNS requests.

This section assumes some familiarity with the DNS protocol.

## Creating and closing a DNS server

.Interface

```c
struct evdns_server_port *evdns_add_server_port_with_base(
    struct event_base *base,
    evutil_socket_t socket,
    int flags,
    evdns_request_callback_fn_type callback,
    void *user_data);

typedef void (*evdns_request_callback_fn_type)(
    struct evdns_server_request *request,
    void *user_data);

void evdns_close_server_port(struct evdns_server_port *port);
```

To begin listening for DNS requests, call evdns_add_server_port_with_base(). It takes an event_base to use for event handling; a UDP socket to listen on; a flags variable (always 0 for now); a callback function to call when a new DNS query is received; and a pointer to user data that will be passed to the callback.  It returns a new evdns_server_port object.

When you are done with the DNS server, you can pass it to evdns_close_server_port().

The evdns_add_server_port_with_base() function was new in 2.0.1-alpha; evdns_close_server_port() was introduced in 1.3.

## Examining a DNS request

Unfortunately, Libevent doesn't currently provide a great way to look at DNS requests via a programmatic interface.  Instead, you're stuck including event2/dns_struct.h and looking at the evdns_server_request structure manually.

It would be great if a future version of Libevent provided a better way to do this.

.Interface

```c
struct evdns_server_request {
    int flags;
    int nquestions;
    struct evdns_server_question **questions;
};
#define EVDNS_QTYPE_AXFR 252
#define EVDNS_QTYPE_ALL  255
struct evdns_server_question {
    int type;
    int dns_question_class;
    char name[1];
};
```

The 'flags' field of the request contains the DNS flags set in the request; the 'nquestions' field is the number of questions in the request; and 'questions' is an array of pointers to struct evdns_server_question.  Each evdns_server_question includes the resource type of the request (see below for a list of EVDNS_*_TYPE macros), the class of the request (typically EVDNS_CLASS_INET), and the name of the requested hostname.

These structures were introduced in Libevent 1.3.  Before Libevent 1.4, dns_question_class was called "class", which made trouble for the C++ people. C programs that still use the old "class" name will stop working in a future release.

.Interface

```c
int evdns_server_request_get_requesting_addr(struct evdns_server_request *req,
        struct sockaddr *sa, int addr_len);
```

Sometimes you'll want to know which address made a particular DNS request. You can check this by calling evdns_server_request_get_requesting_addr() on it.  You should pass in a sockaddr with enough storage to hold the address: struct sockaddr_storage is recommended.

This function was introduced in Libevent 1.3c.

## Responding to DNS requests

Every time your DNS server receives a request, the request is passed to the callback function you provided, along with your user_data pointer. The callback function must either respond to the request, ignore the request, or make sure that the request is eventually answered or ignored.

Before you respond to a request, you can add one or more answers to your response:

.Interface

```c
int evdns_server_request_add_a_reply(struct evdns_server_request *req,
    const char *name, int n, const void *addrs, int ttl);
int evdns_server_request_add_aaaa_reply(struct evdns_server_request *req,
    const char *name, int n, const void *addrs, int ttl);
int evdns_server_request_add_cname_reply(struct evdns_server_request *req,
    const char *name, const char *cname, int ttl);
```

The functions above all add a single RR (of type A, AAAA, or CNAME respectively) to the answers section of a DNS reply for the request 'req'. In each case the argument 'name' is the hostname to add an answer for, and 'ttl' is the time-to-live value of the answer in seconds.  For A and AAAA records, 'n' is the number of addresses to add, and 'addrs' is a pointer to the raw addresses, either given as a sequence of n\*4 bytes for IPv4 addresses in an A record, or as a sequence of n*16 bytes for IPv6 addresses in an AAAA record.

These functions return 0 on success and -1 on failure.

.Interface

```c
int evdns_server_request_add_ptr_reply(struct evdns_server_request *req,
    struct in_addr *in, const char *inaddr_name, const char *hostname,
    int ttl);
```

This function adds a PTR record to the answer section of a request.  The arguments 'req' and 'ttl' are as above.  You must provide exactly one of 'in' (an IPv4 address) or 'inaddr_name' (an address in the .arpa domain) to indicate which address you're providing a response for.  The 'hostname' argument is the answer for the PTR lookup.

.Interface

```c
#define EVDNS_ANSWER_SECTION 0
#define EVDNS_AUTHORITY_SECTION 1
#define EVDNS_ADDITIONAL_SECTION 2

#define EVDNS_TYPE_A       1
#define EVDNS_TYPE_NS       2
#define EVDNS_TYPE_CNAME   5
#define EVDNS_TYPE_SOA       6
#define EVDNS_TYPE_PTR      12
#define EVDNS_TYPE_MX      15
#define EVDNS_TYPE_TXT      16
#define EVDNS_TYPE_AAAA      28

#define EVDNS_CLASS_INET   1

int evdns_server_request_add_reply(struct evdns_server_request *req,
    int section, const char *name, int type, int dns_class, int ttl,
    int datalen, int is_name, const char *data);
```

This function adds an arbitrary RR to the DNS reply of a request 'req'. The 'section' argument describes which section to add it to, and should be one of the EVDNS_*\_SECTION values.  The 'name' argument is the name field of the RR.  The 'type' argument is the 'type' field of the RR, and should be one of the EVDNS_TYPE_* values if possible.  The 'dns_class' argument is the class field of the RR, and should generally be EVDNS_CLASS_INET.  The 'ttl' argument is the time-to-live in seconds of the RR.  The rdata and rdlength fields of the RR will be generated from the 'datalen' bytes provided in 'data'.  If is_name is true, the data will be encoded as a DNS name (i.e., with DNS name compression).  Otherwise, it's included verbatim.

.Interface

```c
int evdns_server_request_respond(struct evdns_server_request *req, int err);
int evdns_server_request_drop(struct evdns_server_request *req);
```

The evdns_server_request_respond() function sends a DNS response to a request, including all of the RRs that you attached to it, with the error code 'err'.  If you get a request that you don't want to respond to, you can ignore it by calling evdns_server_request_drop() on it to release all the associated memory and bookkeeping structures.

.Interface

```c
#define EVDNS_FLAGS_AA    0x400
#define EVDNS_FLAGS_RD    0x080

void evdns_server_request_set_flags(struct evdns_server_request *req,
                                    int flags);
```

If you want to set any flags on your response message, you can call this function at any time before you send the response.

All the functions in this section were introduced in Libevent 1.3, except for evdns_server_request_set_flags() which first appeared in Libevent 2.0.1-alpha.

## DNS Server example

//BUILD: SKIP

.Example: A trivial DNS responder

```c
include::examples_R9/R9_dns_server.c[]
```

## Obsolete DNS interfaces

.Obsolete Interfaces

```c
void evdns_base_search_ndots_set(struct evdns_base *base,
                                 const int ndots);
int evdns_base_nameserver_add(struct evdns_base *base,
    unsigned long int address);
void evdns_set_random_bytes_fn(void (*fn)(char *, size_t));

struct evdns_server_port *evdns_add_server_port(evutil_socket_t socket,
    int flags, evdns_request_callback_fn_type callback, void *user_data);
```

Calling evdns_base_search_ndots_set() is equivalent to using evdns_base_set_option() with the "ndots" option.

The evdns_base_nameserver_add() function behaves as evdns_base_nameserver_ip_add(), except it can only add nameservers with IPv4 addresses.  It takes them, idiosyncratically, as four bytes in network order.

Before Libevent 2.0.1-alpha, there was no way to specify a event base for a DNS server port.  You had to use evdns_add_server_port() instead, which took the default event_base.

From Libevent 2.0.1-alpha through 2.0.3-alpha, you could use evdns_set_random_bytes_fn to specify a function to use for generating random numbers instead of evdns_set_transaction_id_fn.  It no longer has any effect, now that Libevent provides its own secure RNG.

The DNS_QUERY_NO_SEARCH flag has also been called DNS_NO_SEARCH.

Before Libevent 2.0.1-alpha, there was no separate notion of an evdns_base: all information in the evdns subsystem was stored globally, and the functions that manipulated it took no evdns_base as an argument.  They are all now deprecated, and declared only in event2/dns_compat.h.  They are implemented via a single global evdns_base; you can access this base by calling the evdns_get_global_base() function introduced in Libevent 2.0.3-alpha.

| Current function                           | Obsolete global-evdns_base version |
| --- | --- |
| event_base_new()                           | evdns_init() |
| evdns_base_free()                          | evdns_shutdown() |
| evdns_base_nameserver_add()                | evdns_nameserver_add() |
| evdns_base_count_nameservers()             | evdns_count_nameservers() |
| evdns_base_clear_nameservers_and_suspend() | evdns_clear_nameservers_and_suspend() |
| evdns_base_resume()                        | evdns_resume() |
| evdns_base_nameserver_ip_add()             | evdns_nameserver_ip_add() |
| evdns_base_resolve_ipv4()                  | evdns_resolve_ipv4() |
| evdns_base_resolve_ipv6()                  | evdns_resolve_ipv6() |
| evdns_base_resolve_reverse()               | evdns_resolve_reverse() |
| evdns_base_resolve_reverse_ipv6()          | evdns_resolve_reverse_ipv6() |
| evdns_base_set_option()                    | evdns_set_option() |
| evdns_base_resolv_conf_parse()             | evdns_resolv_conf_parse() |
| evdns_base_search_clear()                  | evdns_search_clear() |
| evdns_base_search_add()                    | evdns_search_add() |
| evdns_base_search_ndots_set()              | evdns_search_ndots_set() |
| evdns_base_config_windows_nameservers()    | evdns_config_windows_nameservers() |

The EVDNS_CONFIG_WINDOWS_NAMESERVERS_IMPLEMENTED macro is defined if and only if evdns_config_windows_nameservers() is available.

// Move nameserver_add to dns_compat.h ?

// Move the set_random_bytes function to dns_compat.h ?

_Go back to [Index](README.md)_
