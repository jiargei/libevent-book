# Connection listeners: accepting TCP connections

The evconnlistener mechanism gives you a way to listen for and accept incoming TCP connections.

All the functions and types in this section are declared in event2/listener.h. They first appeared in Libevent 2.0.2-alpha, unless otherwise noted.

## Creating or freeing an evconnlistener

Interface

```c
struct evconnlistener *evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd);
struct evconnlistener *evconnlistener_new_bind(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    const struct sockaddr *sa, int socklen);
void evconnlistener_free(struct evconnlistener *lev);
```

The two evconnlistener_new*() functions both allocate and return a new connection listener object. A connection listener uses an event_base to note when there is a new TCP connection on a given listener socket. When a new connection arrives, it invokes the callback function you give it.

In both functions, the 'base' parameter is an event_base that the listener should use to listen for connections. The 'cb' function is a callback to invoke when a new connection is received; if 'cb' is NULL, the listener is treated as disabled until a callback is set. The 'ptr' pointer will be passed to the callback. The 'flags' argument controls the behavior of the listener -- more on this below. The 'backlog' parameter controls the maximum number of pending connections that the network stack should allow to wait in a not-yet-accepted state at any time; see documentation for your system's listen() function for more details. If 'backlog' is negative, Libevent tries to pick a good value for the backlog; if it is zero, Libevent assumes that you have already called listen() on the socket you are providing it.

The functions differ in how they set up their listener socket. The evconnlistener_new() function assumes that you have already bound a socket to the port you want to listen on, and that you're passing the socket in as 'fd'. If you want Libevent to allocate and bind to a socket on its own, call evconnlistener_new_bind(), and pass in the sockaddr you want to bind to, and its length.

TIP: [When using evconnlistener_new, make sure your listening socket is in non-blocking mode by using evutil_make_socket_nonblocking or by manually setting the correct socket option. When the listening socket is left in blocking mode, undefined behavior might occur.]

To free a connection listener, pass it to evconnlistener_free().

### Recognized flags

These are the flags you can pass to the 'flags' argument of the evconnlistener_new() function. You can give any number of these, OR'd together.

LEV_OPT_LEAVE_SOCKETS_BLOCKING::
    By default, when the connection listener accepts a new incoming socket, it sets it up to be nonblocking so that you can use it with the rest of Libevent. Set this flag if you do not want this behavior.

LEV_OPT_CLOSE_ON_FREE::
    If this option is set, the connection listener closes its underlying socket when you free it.

LEV_OPT_CLOSE_ON_EXEC::
    If this option is set, the connection listener sets the close-on-exec flag on the underlying listener socket. See your platform documentation for fcntl and FD_CLOEXEC for more information.

LEV_OPT_REUSEABLE::
    By default on some platforms, once a listener socket is closed, no other socket can bind to the same port until a while has passed. Setting this option makes Libevent mark the socket as reusable, so that once it is closed, another socket can be opened to listen on the same port.

LEV_OPT_THREADSAFE::
    Allocate locks for the listener, so that it's safe to use it from multiple threads. New in Libevent 2.0.8-rc.

LEV_OPT_DISABLED::
    Initialize the listener to be disabled, not enabled. You can turn it on manually with evconnlistener_enable(). New in Libevent 2.1.1-alpha.

LEV_OPT_DEFERRED_ACCEPT::
    If possible, tell the kernel to not announce sockets as having been accepted until some data has been received on them, and they are ready for reading. Do not use this option if your protocol _does not_ start out with the client transmitting data, since in that case this option will sometimes cause the kernel to never tell you about the connection. Not all operating systems support this option: on ones that don't, this option has no effect. New in Libevent 2.1.1-alpha.

### The connection listener callback

Interface

```c
typedef void (*evconnlistener_cb)(struct evconnlistener *listener,
    evutil_socket_t sock, struct sockaddr *addr, int len, void *ptr);
```

When a new connection is received, the provided callback function is invoked. The 'listener' argument is the connection listener that received the connection. The 'sock' argument is the new socket itself. The 'addr' and 'len' arguments are the address from which the connection was received and the length of that address respectively. The 'ptr' argument is the user-supplied pointer that was passed to evconnlistener_new().

## Enabling and disabling an evconnlistener

Interface

```c
int evconnlistener_disable(struct evconnlistener *lev);
int evconnlistener_enable(struct evconnlistener *lev);
```

These functions temporarily disable or reenable listening for new connections.

## Adjusting an evconnlistener's callback

Interface

```c
void evconnlistener_set_cb(struct evconnlistener *lev,
    evconnlistener_cb cb, void *arg);
```

This function adjusts the callback and callback argument of an existing evconnlistener. It was introduced in 2.0.9-rc.

## Inspecting an evconnlistener

Interface

```c
evutil_socket_t evconnlistener_get_fd(struct evconnlistener *lev);
struct event_base *evconnlistener_get_base(struct evconnlistener *lev);
```

These functions return a listener's associated socket and event_base respectively.

The evconnlistener_get_fd() function first appeared in Libevent 2.0.3-alpha.

## Detecting errors

You can set an error callback that gets informed whenever an accept() call fails on the listener. This can be important to do if you're facing an error condition that would lock the process unless you addressed it.

Interface

```c
typedef void (*evconnlistener_errorcb)(struct evconnlistener *lis, void *ptr);
void evconnlistener_set_error_cb(struct evconnlistener *lev,
    evconnlistener_errorcb errorcb);
```

If you use evconnlistener_set_error_cb() to set an error callback on a listener, the callback will be invoked every time that an error occurs on the listener. It will receive the listener as its first argument, and the argument passed as 'ptr' to evconnlistener_new() as its second argument.

This function was introduced in Libevent 2.0.8-rc.

## Example code: an echo server

//BUILD: SKIP
Example

```c
include::examples_R8/R8_echo_server.c[]
```

_Go back to [Index](README.md)_
