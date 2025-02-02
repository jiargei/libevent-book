# A tiny introduction to asynchronous IO

Most beginning programmers start with blocking IO calls. An IO call is _synchronous_ if, when you call it, it does not return until the operation is completed, or until enough time has passed that your network stack gives up. When you call "connect()" on a TCP connection, for example, your operating system queues a SYN packet to the host on the other side of the TCP connection. It does not return control back to your application until either it has received a SYN ACK packet from the opposite host, or until enough time has passed that it decides to give up.

Here's an example of a really simple client using blocking network calls. It opens a connection to www.google.com, sends it a simple HTTP request, and prints the response to stdout.

//BUILD: SKIP

Example: A simple blocking HTTP client

```c
include::examples_01/01_sync_webclient.c[]
```

All of the network calls in the code above are _blocking_: the gethostbyname does not return until it has succeeded or failed in resolving www.google.com; the connect does not return until it has connected; the recv calls do not return until they have received data or a close; and the send call does not return until it has at least flushed its output to the kernel's write buffers.

Now, blocking IO is not necessarily evil. If there's nothing else you wanted your program to do in the meantime, blocking IO will work fine for you. But suppose that you need to write a program to handle multiple connections at once. To make our example concrete: suppose that you want to read input from two connections, and you don't know which connection will get input first. You can't say

//BUILD: FUNCTIONBODY INC:../example_stubs/sec01.h

Bad Example

```c
/* This won't work. */
char buf[1024];
int i, n;
while (i_still_want_to_read()) {
    for (i=0; i<n_sockets; ++i) {
        n = recv(fd[i], buf, sizeof(buf), 0);
        if (n==0)
            handle_close(fd[i]);
        else if (n<0)
            handle_error(fd[i], errno);
        else
            handle_input(fd[i], buf, n);
    }
}
```

because if data arrives on fd[2] first, your program won't even try reading from fd[2] until the reads from fd[0] and fd[1] have gotten some data and finished.

Sometimes people solve this problem with multithreading, or with multi-process servers. One of the simplest ways to do multithreading is with a separate process (or thread) to deal with each connection. Since each connection has its own process, a blocking IO call that waits for one connection won't make any of the other connections' processes block.

Here's another example program. It is a trivial server that listens for TCP connections on port 40713, reads data from its input one line at a time, and writes out the ROT13 obfuscation of line each as it arrives. It uses the Unix fork() call to create a new process for each incoming connection.

//BUILD: SKIP
.xample: Forking ROT13 server

```c
include::examples_01/01_rot13_server_forking.c[]
```

So, do we have the perfect solution for handling multiple connections at once?  Can I stop writing this book and go work on something else now?  Not quite. First off, process creation (and even thread creation) can be pretty expensive on some platforms. In real life, you'd want to use a thread pool instead of creating new processes. But more fundamentally, threads won't scale as much as you'd like. If your program needs to handle thousands or tens of thousands of connections at a time, dealing with tens of thousands of threads will not be as efficient as trying to have only a few threads per CPU.

But if threading isn't the answer to having multiple connections, what is? In the Unix paradigm, you make your sockets _nonblocking_. The Unix call to do this is:

```c
fcntl(fd, F_SETFL, O_NONBLOCK);
```

where fd is the file descriptor for the socket. footnote:[A file descriptor is the number the kernel assigns to the socket when you open it. You use this number to make Unix calls referring to the socket.]  Once you've made fd (the socket) nonblocking, from then on, whenever you make a network call to fd the call will either complete the operation immediately or return with a special error code to indicate "I couldn't make any progress now, try again."  So our two-socket example might be naively written as:

//BUILD: FUNCTIONBODY INC:../example_stubs/sec01.h

Bad Example: busy-polling all sockets

```c
/* This will work, but the performance will be unforgivably bad. */
int i, n;
char buf[1024];
for (i=0; i < n_sockets; ++i)
    fcntl(fd[i], F_SETFL, O_NONBLOCK);

while (i_still_want_to_read()) {
    for (i=0; i < n_sockets; ++i) {
        n = recv(fd[i], buf, sizeof(buf), 0);
        if (n == 0) {
            handle_close(fd[i]);
        } else if (n < 0) {
            if (errno == EAGAIN)
                 ; /* The kernel didn't have any data for us to read. */
            else
                 handle_error(fd[i], errno);
         } else {
            handle_input(fd[i], buf, n);
         }
    }
}
```

Now that we're using nonblocking sockets, the code above would _work_... but only barely. The performance will be awful, for two reasons. First, when there is no data to read on either connection the loop will spin indefinitely, using up all your CPU cycles. Second, if you try to handle more than one or two connections with this approach you'll do a kernel call for each one, whether it has any data for you or not. So what we need is a way to tell the kernel "wait until one of these sockets is ready to give me some data, and tell me which ones are ready."

The oldest solution that people still use for this problem is select(). The select() call takes three sets of fds (implemented as bit arrays): one for reading, one for writing, and one for "exceptions". It waits until a socket from one of the sets is ready and alters the sets to contain only the sockets ready for use.

Here is our example again, using select:

//BUILD: FUNCTIONBODY INC:../example_stubs/sec01.h

Example: Using select

```c
/* If you only have a couple dozen fds, this version won't be awful */
fd_set readset;
int i, n;
char buf[1024];

while (i_still_want_to_read()) {
    int maxfd = -1;
    FD_ZERO(&readset);

    /* Add all of the interesting fds to readset */
    for (i=0; i < n_sockets; ++i) {
         if (fd[i]>maxfd) maxfd = fd[i];
         FD_SET(fd[i], &readset);
    }

    /* Wait until one or more fds are ready to read */
    select(maxfd+1, &readset, NULL, NULL, NULL);

    /* Process all of the fds that are still set in readset */
    for (i=0; i < n_sockets; ++i) {
        if (FD_ISSET(fd[i], &readset)) {
            n = recv(fd[i], buf, sizeof(buf), 0);
            if (n == 0) {
                handle_close(fd[i]);
            } else if (n < 0) {
                if (errno == EAGAIN)
                     ; /* The kernel didn't have any data for us to read. */
                else
                     handle_error(fd[i], errno);
             } else {
                handle_input(fd[i], buf, n);
             }
        }
    }
}
```

And here's a reimplementation of our ROT13 server, using select() this time.

//BUILD: SKIP
.Example: select()-based ROT13 server

```c
include::examples_01/01_rot13_server_select.c[]
```

But we're still not done. Because generating and reading the select() bit arrays takes time proportional to the largest fd that you provided for select(), the select() call scales terribly when the number of sockets is high. footnote:[On the userspace side, generating and reading the bit arrays can be made to take time proportional to the number of fds that you provided for select(). But on the kernel side, reading the bit arrays takes time proportional to the largest fd in the bit array, which tends to be around _the total number of fds in use in the whole program_, regardless of how many fds are added to the sets in select().]

Different operating systems have provided different replacement functions for select. These include poll(), epoll(), kqueue(), evports, and /dev/poll. All of these give better performance than select(), and all but poll() give O(1) performance for adding a socket, removing a socket, and for noticing that a socket is ready for IO.

Unfortunately, none of the efficient interfaces is a ubiquitous standard. Linux has epoll(), the BSDs (including Darwin) have kqueue(), Solaris has evports and /dev/poll... _and none of these operating systems has any of the others_. So if you want to write a portable high-performance asynchronous application, you'll need an abstraction that wraps all of these interfaces, and provides whichever one of them is the most efficient.

And that's what the lowest level of the Libevent API does for you. It provides a consistent interface to various select() replacements, using the most efficient version available on the computer where it's running.

Here's yet another version of our asynchronous ROT13 server. This time, it uses Libevent 2 instead of select(). Note that the fd_sets are gone now: instead, we associate and disassociate events with a struct event_base, which might be implemented in terms of select(), poll(), epoll(), kqueue(), etc.

//BUILD: SKIP

Example: A low-level ROT13 server with Libevent

```c
include::examples_01/01_rot13_server_libevent.c[]
```

(Other things to note in the code: instead of typing the sockets as "int", we're using the type evutil_socket_t. Instead of calling fcntl(O_NONBLOCK) to make the sockets nonblocking, we're calling evutil_make_socket_nonblocking. These changes make our code compatible with the divergent parts of the Win32 networking API.)

## What about convenience?  (and what about Windows?)

You've probably noticed that as our code has gotten more efficient, it has also gotten more complex. Back when we were forking, we didn't have to manage a buffer for each connection: we just had a separate stack-allocated buffer for each process. We didn't need to explicitly track whether each socket was reading or writing: that was implicit in our location in the code. And we didn't need a structure to track how much of each operation had completed: we just used loops and stack variables.

Moreover, if you're deeply experienced with networking on Windows, you'll realize that Libevent probably isn't getting optimal performance when it's used as in the example above. On Windows, the way you do fast asynchronous IO is not with a select()-like interface: it's by using the IOCP (IO Completion Ports) API. Unlike all the fast networking APIs, IOCP does not alert your program when a socket is _ready_ for an operation that your program then has to perform. Instead, the program tells the Windows networking stack to _start_ a network operation, and IOCP tells the program when the operation has finished.

Fortunately, the Libevent 2 "bufferevents" interface solves both of these issues: it makes programs much simpler to write, and provides an interface that Libevent can implement efficiently on Windows _and_ on Unix.

Here's our ROT13 server one last time, using the bufferevents API.

//BUILD: SKIP

Example: A simpler ROT13 server with Libevent

```c
include::examples_01/01_rot13_server_bufferevent.c[]
```

## How efficient is all of this, really?

XXXX write an efficiency section here. The benchmarks on the libevent page are really out of date.

_Go back to [Index](README.md)_
