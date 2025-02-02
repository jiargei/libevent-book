# Helper functions and types for Libevent

The <event2/util.h> header defines many functions that you might find helpful for implementing portable applications using Libevent. Libevent uses these types and functions internally.

## Basic types

### evutil_socket_t

Most everywhere except Windows, a socket is an int, and the operating system hands them out in numeric order.  Using the Windows socket API, however, a socket is of type SOCKET, which is really a pointer-like OS handle, and the order you receive them is undefined.  We define the evutil_socket_t type to be an integer that can hold the output of socket() or accept() without risking pointer truncation on Windows.

Definition

```c
#ifdef WIN32
#define evutil_socket_t intptr_t
#else
#define evutil_socket_t int
#endif
```

This type was introduced in Libevent 2.0.1-alpha.

### Standard integer types

Often you will find yourself on a C system that missed out on the 21st century and therefore does not implement the standard C99 stdint.h header.  For this situation, Libevent defines its own versions of the bit-width-specific integers from stdint.h:

|  Type      |Width  |Signed | Maximum       | Minimum |
| --- | --- | --- | --- | --- |
|ev_uint64_t |64     |No     | EV_UINT64_MAX | 0 |
|ev_int64_t  |64     |Yes    | EV_INT64_MAX  | EV_INT64_MIN |
|ev_uint32_t |32     |No     | EV_UINT32_MAX | 0 |
|ev_int32_t  |32     |Yes    | EV_INT32_MAX  | EV_INT32_MIN |
|ev_uint16_t |16     |No     | EV_UINT16_MAX | 0 |
|ev_int16_t  |16     |Yes    | EV_INT16_MAX  | EV_INT16_MIN |
|ev_uint8_t  |8      |No     | EV_UINT8_MAX  | 0 |
|ev_int8_t   |8      |Yes    | EV_INT8_MAX   | EV_INT8_MIN |

As in the C99 standard, each type has exactly the specified width, in bits.

These types were introduced in Libevent 1.4.0-beta.  The MAX/MIN constants first appeared in Libevent 2.0.4-alpha.

### Miscellaneous compatibility types

The ev_ssize_t type is defined to ssize_t (signed size_t) on platforms that have one, and to a reasonable default on platforms that don't.  The largest possible value of ev_ssize_t is EV_SSIZE_MAX; the smallest is EV_SSIZE_MIN. (The largest possible value for size_t is EV_SIZE_MAX, in case your platform doesn't define a SIZE_MAX for you.)

The ev_off_t type is used to represent offset into a file or a chunk of memory.  It is defined to off_t on platforms with a reasonable off_t definition, and to ev_int64_t on Windows.

Some implementations of the sockets API provide a length type, socklen_t, and some do not.  The ev_socklen_t is defined to this type where it exists, and a reasonable default otherwise.

The ev_intptr_t type is a signed integer that is large enough to hold a pointer without loss of bits.  The ev_uintptr_t type is an unsigned integer large enough to hold a pointer without loss of bits.

The ev_ssize_t type was added in Libevent 2.0.2-alpha.  The ev_socklen_t type was new in Libevent 2.0.3-alpha.  The ev_intptr_t and ev_uintptr_t types, and the EV_SSIZE_MAX/MIN macros, were added in Libevent 2.0.4-alpha.  The ev_off_t type first appeared in Libevent 2.0.9-rc.

## Timer portability functions

Not every platform defines the standard timeval manipulation functions, so we provide our own implementations.

Interface

```c
#define evutil_timeradd(tvp, uvp, vvp) /* ... */
#define evutil_timersub(tvp, uvp, vvp) /* ... */
```

These macros add or subtract (respectively) their first two arguments, and stores the result in the third.

Interface

```c
#define evutil_timerclear(tvp) /* ... */
#define evutil_timerisset(tvp) /* ... */
```

Clearing a timeval sets its value to zero.  Checking whether it is set returns true if it is nonzero and false otherwise.

Interface

```c
#define evutil_timercmp(tvp, uvp, cmp)
```

The evutil_timercmp macro compares two timevals, and yields true if they are in the relationship specified by the relational operator 'cmp'.  For example, 'evutil_timercmp(t1, t2, <=)' means, "Is t1 <= t2?"  Note that unlike some operating systems' versions, Libevent's timercmp supports all the C relational operations (that is, <, >, ==, !=, <=, and >=).

Interface

```c
int evutil_gettimeofday(struct timeval *tv, struct timezone *tz);
```

The evutil_gettimeofday function sets 'tv' to the current time.  The tz argument is unused.

//BUILD: FUNCTIONBODY INC:event2/util.h INC:stdio.h

Example

```c
struct timeval tv1, tv2, tv3;

/* Set tv1 = 5.5 seconds */
tv1.tv_sec = 5; tv1.tv_usec = 500*1000;

/* Set tv2 = now */
evutil_gettimeofday(&tv2, NULL);

/* Set tv3 = 5.5 seconds in the future */
evutil_timeradd(&tv1, &tv2, &tv3);

/* all 3 should print true */
if (evutil_timercmp(&tv1, &tv1, ==))  /* == "If tv1 == tv1" */
   puts("5.5 sec == 5.5 sec");
if (evutil_timercmp(&tv3, &tv2, >=))  /* == "If tv3 >= tv2" */
   puts("The future is after the present.");
if (evutil_timercmp(&tv1, &tv2, <))   /* == "If tv1 < tv2" */
   puts("It is no longer the past.");
```

These functions were introduced in Libevent 1.4.0-beta, except for evutil_gettimeofday(), which was introduced in Libevent 2.0.

NOTE: It wasn't safe to use <= or >= with timercmp before Libevent 1.4.4.

## Socket API compatibility

This section exists because, for historical reasons, Windows has never really implemented the Berkeley sockets API in a nice compatible (and nicely compatible) way.  Here are some functions you can use in order to pretend that it has.

Interface

```c
int evutil_closesocket(evutil_socket_t s);

#define EVUTIL_CLOSESOCKET(s) evutil_closesocket(s)
```

This function  closes a socket.  On Unix, it's an alias for close(); on Windows, it calls closesocket().   (You can't use close() on sockets on Windows, and nobody else defines a closesocket().)

The evutil_closesocket function was introduced in Libevent 2.0.5-alpha. Before then, you needed to call the EVUTIL_CLOSESOCKET macro.

Interface

```c
#define EVUTIL_SOCKET_ERROR()
#define EVUTIL_SET_SOCKET_ERROR(errcode)
#define evutil_socket_geterror(sock)
#define evutil_socket_error_to_string(errcode)
```

These macros access and manipulate socket error codes. EVUTIL_SOCKET_ERROR() returns the global error code for the last socket operation from this thread, and evutil_socket_geterror() does so for a particular socket.  (Both are errno on Unix-like systems.) EVUTIL_SET_SOCKET_ERROR() changes the current socket error code (like setting errno on Unix), and evutil_socket_error_to_string() returns a string representation of a given socket error code (like strerror() on Unix).

(We need these functions because Windows doesn't use errno for errors from socket functions, but instead uses WSAGetLastError().)

Note that the Windows socket errors are not the same as the standard-C errors you would see in errno; watch out.

Interface

```c
int evutil_make_socket_nonblocking(evutil_socket_t sock);
```

Even the call you need to do nonblocking IO on a socket is not portable to Windows.  The evutil_make_socket_nonblocking() function takes a new socket (from socket() or accept()) and turns it into a nonblocking socket.  (It sets O_NONBLOCK on Unix and FIONBIO on Windows.)

Interface

```c
int evutil_make_listen_socket_reuseable(evutil_socket_t sock);
```

This function makes sure that the address used by a listener socket will be available to another socket immediately after the socket is closed.  (It sets SO_REUSEADDR on Unix and does nothing on Windows. You don't want to use SO_REUSEADDR on Windows; it means something different there.)

Interface

```c
int evutil_make_socket_closeonexec(evutil_socket_t sock);
```

This call tells the operating system that this socket should be closed if we ever call exec().  It sets the FD_CLOEXEC flag on Unix, and does nothing on Windows.

Interface

```c
int evutil_socketpair(int family, int type, int protocol,
        evutil_socket_t sv[2]);
```

This function behaves as the Unix socketpair() call: it makes two sockets that are connected with each other and can be used with ordinary socket IO calls.  It stores the two sockets in sv[0] and sv[1], and returns 0 for success and -1 for failure.

On Windows, this only supports family AF_INET, type SOCK_STREAM, and protocol 0.  Note that this can fail on some Windows hosts where firewall software has cleverly firewalled 127.0.0.1 to keep the host from talking to itself.

These functions were introduced in Libevent 1.4.0-beta, except for evutil_make_socket_closeonexec(), which was new in Libevent 2.0.4-alpha.

## Portable string manipulation functions

Interface

```c
ev_int64_t evutil_strtoll(const char *s, char **endptr, int base);
```

This function behaves as strtol, but handles 64-bit integers.  On some platforms, it only supports Base 10.

Interface

```c
int evutil_snprintf(char *buf, size_t buflen, const char *format, ...);
int evutil_vsnprintf(char *buf, size_t buflen, const char *format, va_list ap);
```

These snprintf-replacement functions behave as the standard snprintf and vsnprintf interfaces.  They return the number of bytes that would have been written into the buffer had it been long enough, not counting the terminating NUL byte.  (This behavior conforms to the C99 snprintf() standard, and is in contrast to the Windows _snprintf(), which returns a negative number if the string would not fit in the buffer.)

The evutil_strtoll() function has been in Libevent since 1.4.2-rc. These other functions first appeared in version 1.4.5.

## Locale-independent string manipulation functions

Sometimes, when implementing ASCII-based protocols, you want to manipulate strings according to ASCII's notion of character type, regardless of your current locale.  Libevent provides a few functions to help with this:

Interface

```c
int evutil_ascii_strcasecmp(const char *str1, const char *str2);
int evutil_ascii_strncasecmp(const char *str1, const char *str2, size_t n);
```

These functions behave as strcasecmp() and strncasecmp(), except that they always compare using the ASCII character set, regardless of the current locale.  The evutil_ascii_str[n]casecmp() functions were first exposed in Libevent 2.0.3-alpha.

## IPv6 helper and portability functions

Interface

```c
const char *evutil_inet_ntop(int af, const void *src, char *dst, size_t len);
int evutil_inet_pton(int af, const char *src, void *dst);
```

These functions behave as the standard inet_ntop() and inet_pton() functions for parsing and formatting IPv4 and IPv6 addresses, as specified in RFC3493.  That is, to format an IPv4 address, you call evutil_inet_ntop() with 'af' set to AF_INET, 'src' pointing to a struct in_addr, and 'dst' pointing to a character buffer of size 'len'.  For an IPv6 address, 'af' is AF_INET6 and 'src' is a struct in6_addr.  To parse an IPv4 address, call evutil_inet_pton() with 'af' set to AF_INET or AF_INET6, the string to parse in 'src', and 'dst' pointing to an in_addr or an in_addr6 as appropriate.

The return value from evutil_inet_ntop() is NULL on failure and otherwise points to dst.  The return value from evutil_inet_pton() is 0 on success and -1 on failure.

Interface

```c
int evutil_parse_sockaddr_port(const char *str, struct sockaddr *out,
    int *outlen);
```

This function parses an address from 'str' and writes the result to 'out'.  The 'outlen' argument must point to an integer holding the number of bytes available in 'out'; it is altered to hold the number of bytes actually used.  This function returns 0 on success and -1 on failure.  It recognizes the following address formats:

- [ipv6]:port (as in "[ffff::]:80")
- ipv6 (as in "ffff::")
- [ipv6] (as in "[ffff::]")
- ipv4:port (as in "1.2.3.4:80")
- ipv4 (as in "1.2.3.4")

If no port is given, the port in the resulting sockaddr is set to 0.

Interface

```c
int evutil_sockaddr_cmp(const struct sockaddr *sa1,
    const struct sockaddr *sa2, int include_port);
```

The evutil_sockaddr_cmp() function compares two addresses, and returns negative if sa1 precedes sa2, 0 if they are equal, and positive if sa2 precedes sa1.  It works for AF_INET and AF_INET6 addresses, and returns undefined output for other addresses.  It's guaranteed to give a total order for these addresses, but the ordering may change between Libevent versions.

If the 'include_port' argument is false, then two sockaddrs are treated as equal if they differ only in their port.  Otherwise, sockaddrs with different ports are treated as unequal.

These functions were introduced in Libevent 2.0.1-alpha, except for evutil_sockaddr_cmp(), which introduced in 2.0.3-alpha.

## Structure macro portability functions

Interface

```c
#define evutil_offsetof(type, field) /* ... */
```

As the standard offsetof macro, this macro yields the number of bytes from the start of 'type' at which 'field' occurs.

This macro was introduced in Libevent 2.0.1-alpha.  It was buggy in every version before Libevent 2.0.3-alpha.

## Secure random number generator

Many applications (including evdns) need a source of hard-to-predict random numbers for their security.

Interface

```c
void evutil_secure_rng_get_bytes(void *buf, size_t n);
```

This function fills n-byte buffer at 'buf' with 'n' bytes of random data.

If your platform provides the arc4random() function, Libevent uses that. Otherwise, it uses its own implementation of arc4random(), seeded by your operating system's entropy pool (CryptGenRandom on Windows, /dev/urandom everywhere else).

Interface

```c
int evutil_secure_rng_init(void);
void evutil_secure_rng_add_bytes(const char *dat, size_t datlen);
```

You do not need to manually initialize the secure random number generator, but if you want to make sure it is successfully initialized, you can do so by calling evutil_secure_rng_init().  It seeds the RNG (if it was not already seeded) and returns 0 on success. If it returns -1, Libevent wasn't able to find a good source of entropy on your OS, and you can't use the RNG safely without initializing it yourself.

If you are running in an environment where your program is likely to drop privileges (for example, by running chroot()), you should call evutil_secure_rng_init() before you do so.

You can add more random bytes to the entropy pool yourself by calling evutil_secure_rng_add_bytes(); this shouldn't be necessary in typical use.

These functions are new in Libevent 2.0.4-alpha.

_Go back to [Index](README.md)_
