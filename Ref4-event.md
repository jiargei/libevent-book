# Working with events

Libevent's basic unit of operation is the 'event'. Every event represents a set of conditions, including:

- A file descriptor being ready to read from or write to.
- A file descriptor _becoming_ ready to read from or write to (Edge-triggered IO only).
- A timeout expiring.
- A signal occurring.
- A user-triggered event.

Events have similar lifecycles. Once you call a Libevent function to set up an event and associate it with an event base, it becomes *initialized*. At this point, you can 'add', which makes it *pending* in the base. When the event is pending, if the conditions that would trigger an event occur (e.g., its file descriptor changes state or its timeout expires), the event becomes *active*, and its (user-provided) callback function is run. If the event is configured *persistent*, it remains pending. If it is not persistent, it stops being pending when its callback runs. You can make a pending event non-pending by 'deleting' it, and you can 'add' a non-pending event to make it pending again.

## Constructing event objects

To create a new event, use the event_new() interface.

Interface

```c
#define EV_TIMEOUT      0x01
#define EV_READ         0x02
#define EV_WRITE        0x04
#define EV_SIGNAL       0x08
#define EV_PERSIST      0x10
#define EV_ET           0x20

typedef void (*event_callback_fn)(evutil_socket_t, short, void *);

struct event *event_new(struct event_base *base, evutil_socket_t fd,
    short what, event_callback_fn cb,
    void *arg);

void event_free(struct event *event);
```

The event\_new() function tries to allocate and construct a new event for use with 'base'. The 'what' argument is a set of the flags listed above. (Their semantics are described below.)  If 'fd' is nonnegative, it is the file that we'll observe for read or write events. When the event is active, Libevent will invoke the provided 'cb' function, passing it as arguments: the file descriptor 'fd', a bitfield of _all_ the events that triggered, and the value that was passed in for 'arg' when the function was constructed.

On an internal error, or invalid arguments, event_new() will return NULL.

All new events are initialized and non-pending. To make an event pending, call event_add() (documented below).

To deallocate an event, call event_free(). It is safe to call event_free() on an event that is pending or active: doing so makes the event non-pending and inactive before deallocating it.

Example

```c
#include <event2/event.h>

void cb_func(evutil_socket_t fd, short what, void *arg)
{
        const char *data = arg;
        printf("Got an event on socket %d:%s%s%s%s [%s]",
            (int) fd,
            (what&EV_TIMEOUT) ? " timeout" : "",
            (what&EV_READ)    ? " read" : "",
            (what&EV_WRITE)   ? " write" : "",
            (what&EV_SIGNAL)  ? " signal" : "",
            data);
}

void main_loop(evutil_socket_t fd1, evutil_socket_t fd2)
{
        struct event *ev1, *ev2;
        struct timeval five_seconds = {5,0};
        struct event_base *base = event_base_new();

        /* The caller has already set up fd1, fd2 somehow, and make them
           nonblocking. */

        ev1 = event_new(base, fd1, EV_TIMEOUT|EV_READ|EV_PERSIST, cb_func,
           (char*)"Reading event");
        ev2 = event_new(base, fd2, EV_WRITE|EV_PERSIST, cb_func,
           (char*)"Writing event");

        event_add(ev1, &five_seconds);
        event_add(ev2, NULL);
        event_base_dispatch(base);
}
```

The above functions are defined in <event2/event.h>, and first appeared in Libevent 2.0.1-alpha. The event_callback_fn type first appeared as a typedef in Libevent 2.0.4-alpha.

### The event flags

EV_TIMEOUT::
    This flag indicates an event that becomes active after a timeout elapses.

The EV_TIMEOUT flag is ignored when constructing an event: you can either set a timeout when you add the event, or not. It is set in the 'what' argument to the callback function when a timeout has occurred.

EV_READ::
    This flag indicates an event that becomes active when the provided file descriptor is ready for reading.

EV_WRITE::
    This flag indicates an event that becomes active when the provided file descriptor is ready for writing.

EV_SIGNAL::
    Used to implement signal detection. See "Constructing signal events" below.

EV_PERSIST::
    Indicates that the event is 'persistent'. See "About Event Persistence" below.

EV_ET::
    Indicates that the event should be edge-triggered, if the underlying event_base backend supports edge-triggered events. This affects the semantics of EV_READ and EV_WRITE.

Since Libevent 2.0.1-alpha, any number of events may be pending for the same conditions at the same time. For example, you may have two events that will become active if a given fd becomes ready to read. The order in which their callbacks are run is undefined.

These flags are defined in <event2/event.h>. All have existed since before Libevent 1.0, except for EV_ET, which was introduced in Libevent 2.0.1-alpha.

### About Event Persistence

By default, whenever a pending event becomes active (because its fd is ready to read or write, or because its timeout expires), it becomes non-pending right before its callback is executed. Thus, if you want to make the event pending again, you can call event_add() on it again from inside the callback function.

If the EV_PERSIST flag is set on an event, however, the event is 'persistent.'  This means that event remains pending even when its callback is activated. If you want to make it non-pending from within its callback, you can call event_del() on it.

The timeout on a persistent event resets whenever the event's callback runs. Thus, if you have an event with flags EV_READ|EV_PERSIST and a timeout of five seconds, the event will become active:

- Whenever the socket is ready for reading.
- Whenever five seconds have passed since the event last became
    active.

### Creating an event as its own callback argument

Frequently, you might want to create an event that receives itself as a callback argument. You can't just pass a pointer to the event as an argument to event_new(), though, because it does not exist yet. To solve this problem, you can use event_self_cbarg().

Interface

```c
void *event_self_cbarg();
```

The event_self_cbarg() function returns a "magic" pointer which, when passed as an event callback argument, tells event_new() to create an event receiving itself as its callback argument.

Example

```c
#include <event2/event.h>

static int n_calls = 0;

void cb_func(evutil_socket_t fd, short what, void *arg)
{
    struct event *me = arg;

    printf("cb_func called %d times so far.\n", ++n_calls);

    if (n_calls > 100)
       event_del(me);
}

void run(struct event_base *base)
{
    struct timeval one_sec = { 1, 0 };
    struct event *ev;
    /* We're going to set up a repeating timer to get called called 100
       times. */
    ev = event_new(base, -1, EV_PERSIST, cb_func, event_self_cbarg());
    event_add(ev, &one_sec);
    event_base_dispatch(base);
}
```

This function can also be used with event_new(), evtimer_new(), evsignal_new(), event_assign(), evtimer_assign(), and evsignal_assign(). It won't work as a callback argument for non-events, however.

The event_self_cbarg() function was introduced in Libevent 2.1.1-alpha.

### Timeout-only events

As a convenience, there are a set of macros beginning with evtimer\_ that you can use in place of the event_* calls to allocate and manipulate pure-timeout events. Using these macros provides no benefit beyond improving the clarity of your code.

Interface

```c
#define evtimer_new(base, callback, arg) \
    event_new((base), -1, 0, (callback), (arg))
#define evtimer_add(ev, tv) \
    event_add((ev),(tv))
#define evtimer_del(ev) \
    event_del(ev)
#define evtimer_pending(ev, tv_out) \
    event_pending((ev), EV_TIMEOUT, (tv_out))
```

These macros have been present since Libevent 0.6, except for evtimer_new(), which first appeared in Libevent 2.0.1-alpha.

### Constructing signal events

Libevent can also watch for POSIX-style signals. To construct a handler for a signal, use:

Interface

```c
#define evsignal_new(base, signum, cb, arg) \
    event_new(base, signum, EV_SIGNAL|EV_PERSIST, cb, arg)
```

The arguments are as for event_new, except that we provide a signal number instead of a file descriptor.

//BUILD: SKIP

Example

```c
struct event *hup_event;
struct event_base *base = event_base_new();

/* call sighup_function on a HUP signal */
hup_event = evsignal_new(base, SIGHUP, sighup_function, NULL);
```

Note that signal callbacks are run in the event loop after the signal occurs, so it is safe for them to call functions that you are not supposed to call from a regular POSIX signal handler.

WARNING: Don't set a timeout on a signal event. It might not be supported. [FIXME: is this true?]

There are also a set of convenience macros you can use when working with signal events.

Interface

```c
#define evsignal_add(ev, tv) \
    event_add((ev),(tv))
#define evsignal_del(ev) \
    event_del(ev)
#define evsignal_pending(ev, what, tv_out) \
    event_pending((ev), (what), (tv_out))
```

The evsignal_* macros have been present since Libevent 2.0.1-alpha. Prior versions called them signal_add(), signal_del(), and so on.

#### Caveats when working with signals

With current versions of Libevent, with most backends, only one event_base per process at a time can be listening for signals. If you add signal events to two event_bases at once ---even if the signals are different!--- only one event_base will receive signals.

The kqueue backend does not have this limitation.

### Setting up events without heap-allocation

For performance and other reasons, some people like to allocate events as a part of a larger structure. For each use of the event, this saves them:

- The memory allocator overhead for allocating a small object on the heap.
- The time overhead for dereferencing the pointer to the struct event.
- The time overhead from a possible additional cache miss if the event is not already in the cache.

Using this method risks breaking binary compatibility with other versions of of Libevent, which may have different sizes for the event structure.

These are _very_ small costs, and do not matter for most applications. You should just stick to using event_new() unless you *know* that you're incurring a significant performance penalty for heap-allocating your events. Using event_assign() can cause hard-to-diagnose errors with future versions of Libevent if they use a larger event structure than the one you're building with.

Interface

```c
int event_assign(struct event *event, struct event_base *base,
    evutil_socket_t fd, short what,
    void (*callback)(evutil_socket_t, short, void *), void *arg);
```

All the arguments of event_assign() are as for event_new(), except for the 'event' argument, which must point to an uninitialized event. It returns 0 on success, and -1 on an internal error or bad arguments.

Example

```c
#include <event2/event.h>
/* Watch out!  Including event_struct.h means that your code will not
 * be binary-compatible with future versions of Libevent. */
#include <event2/event_struct.h>
#include <stdlib.h>

struct event_pair {
         evutil_socket_t fd;
         struct event read_event;
         struct event write_event;
};
void readcb(evutil_socket_t, short, void *);
void writecb(evutil_socket_t, short, void *);
struct event_pair *event_pair_new(struct event_base *base, evutil_socket_t fd)
{
        struct event_pair *p = malloc(sizeof(struct event_pair));
        if (!p) return NULL;
        p->fd = fd;
        event_assign(&p->read_event, base, fd, EV_READ|EV_PERSIST, readcb, p);
        event_assign(&p->write_event, base, fd, EV_WRITE|EV_PERSIST, writecb, p);
        return p;
}
```

You can also use event_assign() to initialize stack-allocated or statically allocated events.

WARNING

Never call event_assign() on an event that is already pending in an event base. Doing so can lead to extremely hard-to-diagnose errors. If the event is already initialized and pending, call event_del() on it *before* you call event_assign() on it again.

There are convenience macros you can use to event_assign() a timeout-only or a signal event:

Interface

```c
#define evtimer_assign(event, base, callback, arg) \
    event_assign(event, base, -1, 0, callback, arg)
#define evsignal_assign(event, base, signum, callback, arg) \
    event_assign(event, base, signum, EV_SIGNAL|EV_PERSIST, callback, arg)
```

If you need to use event_assign() *and* retain binary compatibility with future versions of Libevent, you can ask the Libevent library to tell you at runtime how large a 'struct event' should be:

Interface

```c
size_t event_get_struct_event_size(void);
```

This function returns the number of bytes you need to set aside for a struct event. As before, you should only be using this function if you know that heap-allocation is actually a significant problem in your program, since it can make your code much harder to read and write.

Note that event_get_struct_event_size() may in the future give you a value _smaller_ than 'sizeof(struct event)'. If this happens, it means that any extra bytes at the end of 'struct event' are only padding bytes reserved for use by a future version of Libevent.

Here's the same example as above, but instead of relying on the size of 'struct event' from event_struct.h, we use event_get_struct_size() to use the correct size at runtime.

Example

```c
#include <event2/event.h>
#include <stdlib.h>

/* When we allocate an event_pair in memory, we'll actually allocate
 * more space at the end of the structure. We define some macros
 * to make accessing those events less error-prone. */
struct event_pair {
         evutil_socket_t fd;
};

/* Macro: yield the struct event 'offset' bytes from the start of 'p' */
#define EVENT_AT_OFFSET(p, offset) \
            ((struct event*) ( ((char*)(p)) + (offset) ))
/* Macro: yield the read event of an event_pair */
#define READEV_PTR(pair) \
            EVENT_AT_OFFSET((pair), sizeof(struct event_pair))
/* Macro: yield the write event of an event_pair */
#define WRITEEV_PTR(pair) \
            EVENT_AT_OFFSET((pair), \
                sizeof(struct event_pair)+event_get_struct_event_size())

/* Macro: yield the actual size to allocate for an event_pair */
#define EVENT_PAIR_SIZE() \
            (sizeof(struct event_pair)+2*event_get_struct_event_size())

void readcb(evutil_socket_t, short, void *);
void writecb(evutil_socket_t, short, void *);
struct event_pair *event_pair_new(struct event_base *base, evutil_socket_t fd)
{
        struct event_pair *p = malloc(EVENT_PAIR_SIZE());
        if (!p) return NULL;
        p->fd = fd;
        event_assign(READEV_PTR(p), base, fd, EV_READ|EV_PERSIST, readcb, p);
        event_assign(WRITEEV_PTR(p), base, fd, EV_WRITE|EV_PERSIST, writecb, p);
        return p;
}
```

The event_assign() function defined in <event2/event.h>. It has existed since Libevent 2.0.1-alpha. It has returned an int since 2.0.3-alpha; previously, it returned void. The event_get_struct_event_size() function was introduced in Libevent 2.0.4-alpha. The event structure itself is defined in <event2/event_struct.h>.

## Making events pending and non-pending

Once you have constructed an event, it won't actually do anything until you have made it 'pending' by adding it. You do this with event_add:

Interface

```c
int event_add(struct event *ev, const struct timeval *tv);
```

Calling event_add on a non-pending event makes it pending in its configured base. The function returns 0 on success, and -1 on failure. If 'tv' is NULL, the event is added with no timeout. Otherwise, 'tv' is the size of the timeout in seconds and microseconds.

If you call event\_add() on an event that is _already_ pending, it will leave it pending, and reschedule it with the provided timeout. If the event is already pending, and you re-add it with the timeout NULL, event_add() will have no effect.

NOTE: Do not set 'tv' to the time at which you want the timeout to run. If you say "tv->tv_sec = time(NULL)+10;" on 1 January 2010, your timeout will wait 40 years, not 10 seconds.

Interface

```c
int event_del(struct event *ev);
```

Calling event_del on an initialized event makes it non-pending and non-active. If the event was not pending or active, there is no effect. The return value is 0 on success, -1 on failure.

NOTE: If you delete an event after it becomes active but before its callback has a chance to execute, the callback will not be executed.

Interface

```c
int event_remove_timer(struct event *ev);
```

Finally, you can remove a pending event's timeout completely without deleting its IO or signal components. If the event had no timeout pending, event_remove_timer() has no effect. If the event had only a timeout but no IO or signal component, event_remove_timer() has the same effect as event_del(). The return value is 0 on success, -1 on failure.

These are defined in <event2/event.h>; event_add() and event_del() have existed since Libevent 0.1; event_remove_timer() was added in 2.1.2-alpha.

## Events with priorities

When multiple events trigger at the same time, Libevent does not define any order with respect to when their callbacks will be executed. You can define some events as more important than others by using priorities.

As discussed in an earlier section, each event_base has one or more priority values associated with it. Before adding an event to the event_base, but after initializing it, you can set its priority.

Interface

```c
int event_priority_set(struct event *event, int priority);
```

The priority of the event is a number between 0 and the number of priorities in an event_base, minus 1. The function returns 0 on success, and -1 on failure.

When multiple events of multiple priorities become active, the low-priority events are not run. Instead, Libevent runs the high priority events, then checks for events again. Only when no high-priority events are active are the low-priority events run.

Example

```c
#include <event2/event.h>

void read_cb(evutil_socket_t, short, void *);
void write_cb(evutil_socket_t, short, void *);

void main_loop(evutil_socket_t fd)
{
  struct event *important, *unimportant;
  struct event_base *base;

  base = event_base_new();
  event_base_priority_init(base, 2);
  /* Now base has priority 0, and priority 1 */
  important = event_new(base, fd, EV_WRITE|EV_PERSIST, write_cb, NULL);
  unimportant = event_new(base, fd, EV_READ|EV_PERSIST, read_cb, NULL);
  event_priority_set(important, 0);
  event_priority_set(unimportant, 1);

  /* Now, whenever the fd is ready for writing, the write callback will
     happen before the read callback. The read callback won't happen at
     all until the write callback is no longer active. */
}
```

When you do not set the priority for an event, the default is the number of queues in the event base, divided by 2.

This function is declared in <event2/event.h>. It has existed since Libevent 1.0.

## Inspecting event status

Sometimes you want to tell whether an event has been added, and check what it refers to.

Interface

```c
int event_pending(const struct event *ev, short what, struct timeval *tv_out);

#define event_get_signal(ev) /* ... */
evutil_socket_t event_get_fd(const struct event *ev);
struct event_base *event_get_base(const struct event *ev);
short event_get_events(const struct event *ev);
event_callback_fn event_get_callback(const struct event *ev);
void *event_get_callback_arg(const struct event *ev);
int event_get_priority(const struct event *ev);

void event_get_assignment(const struct event *event,
        struct event_base **base_out,
        evutil_socket_t *fd_out,
        short *events_out,
        event_callback_fn *callback_out,
        void **arg_out);
```

The event_pending function determines whether the given event is pending or active. If it is, and any of the flags EV_READ, EV_WRITE, EV_SIGNAL, and EV_TIMEOUT are set in the 'what' argument, the function returns all of the flags that the event is currently pending or active on. If 'tv_out' is provided, and EV_TIMEOUT is set in 'what', and the event is currently pending or active on a timeout, then 'tv_out' is set to hold the time when the event's timeout will expire.

The event_get_fd() and event_get_signal() functions return the configured file descriptor or signal number for an event. The event_get_base() function returns its configured event_base. The event_get_events() function returns the event flags (EV_READ, EV_WRITE, etc) of the event. The event_get_callback() and event_get_callback_arg() functions return the callback function and argument pointer. The event_get_priority() function returns the event's currently assigned priority.

The event_get_assignment() function copies all of the assigned fields of the event into the provided pointers. If any of the pointers is NULL, it is ignored.

Example

```c
#include <event2/event.h>
#include <stdio.h>

/* Change the callback and callback_arg of 'ev', which must not be
 * pending. */
int replace_callback(struct event *ev, event_callback_fn new_callback,
    void *new_callback_arg)
{
    struct event_base *base;
    evutil_socket_t fd;
    short events;

    int pending;

    pending = event_pending(ev, EV_READ|EV_WRITE|EV_SIGNAL|EV_TIMEOUT,
                            NULL);
    if (pending) {
        /* We want to catch this here so that we do not re-assign a
         * pending event. That would be very very bad. */
        fprintf(stderr,
                "Error! replace_callback called on a pending event!\n");
        return -1;
    }

    event_get_assignment(ev, &base, &fd, &events,
                         NULL /* ignore old callback */ ,
                         NULL /* ignore old callback argument */);

    event_assign(ev, base, fd, events, new_callback, new_callback_arg);
    return 0;
}
```

These functions are declared in <event2/event.h>. The event_pending() function has existed since Libevent 0.1. Libevent 2.0.1-alpha introduced event_get_fd() and event_get_signal(). Libevent 2.0.2-alpha introduced event_get_base(). Libevent 2.1.2-alpha added event_get_priority(). The others were new in Libevent 2.0.4-alpha.

## Finding the currently running event

For debugging or other purposes, you can get a pointer to the currently running event.

Interface

```c
struct event *event_base_get_running_event(struct event_base *base);
```

Note that this function's behavior is only defined when it's called from within the provided event_base's loop. Calling it from another thread is not supported, and can cause undefined behavior.

This function is declared in <event2/event.h>. It was introduced in Libevent 2.1.1-alpha.

## Configuring one-off events

If you don't need to add an event more than once, or delete it once it has been added, and it doesn't have to be persistent, you can use event_base_once().

Interface

```c
int event_base_once(struct event_base *, evutil_socket_t, short,
  void (*)(evutil_socket_t, short, void *), void *, const struct timeval *);
```

This function's interface is the same as event_new(), except that it does not support EV_SIGNAL or EV_PERSIST. The scheduled event is inserted and run with the default priority. When the callback is finally done, Libevent frees the internal event structure itself. The return value is 0 on success, -1 on failure.

Events inserted with event_base_once cannot be deleted or manually activated: if you want to be able to cancel an event, create it with the regular event_new() or event_assign() interfaces.

Note also that at up to Libevent 2.0, if the event is never triggered, the internal memory used to hold it will never be freed. Starting in Libevent 2.1.2-alpha, these events _are_ freed when the event_base is freed, even if they haven't activated, but still be aware: if there's some storage associated with their callback arguments, that storage won't be released unless your program has done something to track and release it.

## Manually activating an event

Rarely, you may want to make an event active even though its conditions have not triggered.

Interface

```c
void event_active(struct event *ev, int what, short ncalls);
```

This function makes an event 'ev' become active with the flags 'what' (a combination of EV_READ, EV_WRITE, and EV_TIMEOUT). The event does not need to have previously been pending, and activating it does not make it pending.

Warning: calling event_active() recursively on the same event may result in resource exhaustion. The following snippet of code is an example of how event_active can be used incorrectly.

//BUILD: INC:event2/event.h

Bad Example: making an infinite loop with event_active()

```c
struct event *ev;

static void cb(int sock, short which, void *arg) {
        /* Whoops: Calling event_active on the same event unconditionally
           from within its callback means that no other events might not get
           run! */

    event_active(ev, EV_WRITE, 0);
}

int main(int argc, char **argv) {
    struct event_base *base = event_base_new();

    ev = event_new(base, -1, EV_PERSIST | EV_READ, cb, NULL);

    event_add(ev, NULL);

    event_active(ev, EV_WRITE, 0);

    event_base_loop(base, 0);

    return 0;
}
```

This creates a situation where the event loop is only executed once and calls the function "cb" forever.

//BUILD: INC:event2/event.h

Example: Alternative solution to the above problem using timers

```c

struct event *ev;
struct timeval tv;

static void cb(int sock, short which, void *arg) {
   if (!evtimer_pending(ev, NULL)) {
       event_del(ev);
       evtimer_add(ev, &tv);
   }
}

int main(int argc, char **argv) {
   struct event_base *base = event_base_new();

   tv.tv_sec = 0;
   tv.tv_usec = 0;

   ev = evtimer_new(base, cb, NULL);

   evtimer_add(ev, &tv);

   event_base_loop(base, 0);

   return 0;
}
```

//BUILD: INC:event2/event.h

Example: Alternative solution to the above problem using event_config_set_max_dispatch_interval()

```c
struct event *ev;

static void cb(int sock, short which, void *arg) {
    event_active(ev, EV_WRITE, 0);
}

int main(int argc, char **argv) {
        struct event_config *cfg = event_config_new();
        /* Run at most 16 callbacks before checking for other events. */
        event_config_set_max_dispatch_interval(cfg, NULL, 16, 0);
    struct event_base *base = event_base_new_with_config(cfg);
    ev = event_new(base, -1, EV_PERSIST | EV_READ, cb, NULL);

    event_add(ev, NULL);

    event_active(ev, EV_WRITE, 0);

    event_base_loop(base, 0);

    return 0;
}
```

This function is defined in <event2/event.h>. It has existed since Libevent 0.3.

## Optimizing common timeouts

Current versions of Libevent use a binary heap algorithm to keep track of pending events' timeouts. A binary heap gives performance of order O(lg n) for adding and deleting each event timeout. This is optimal if you're adding events with a randomly distributed set of timeout values, but not if you have a large number of events with the same timeout.

For example, suppose you have ten thousand events, each of which should trigger its timeout five seconds after it was added. In a situation like this, you could get O(1) performance for each timeout by using a doubly-linked queue implementation.

Naturally, you wouldn't want to use a queue for all of your timeout values, since a queue is only faster for constant timeout values. If some of the timeouts are more-or-less randomly distributed, then adding one of those timeouts to a queue would take O(n) time, which would be significantly worse than a binary heap.

Libevent lets you solve this by placing some of your timeouts in queues, and others in the binary heap. To do this, you ask Libevent for a special "common timeout" timeval, which you then use to add events having that timeval. If you have a very large number of events with a single common timeout, using this optimization should improve timeout performance.

Interface

```c
const struct timeval *event_base_init_common_timeout(
    struct event_base *base, const struct timeval *duration);
```

This function takes as its arguments an event_base, and the duration of the common timeout to initialize. It returns a pointer to a special struct timeval that you can use to indicate that an event should be added to an O(1) queue rather than the O(lg n) heap. This special timeval can be copied or assigned freely in your code. It will only work with the specific base you used to construct it. Do not rely on its actual contents: Libevent uses them to tell itself which queue to use.

Example

```c
#include <event2/event.h>
#include <string.h>

/* We're going to create a very large number of events on a given base,
 * nearly all of which have a ten-second timeout. If initialize_timeout
 * is called, we'll tell Libevent to add the ten-second ones to an O(1)
 * queue. */
struct timeval ten_seconds = { 10, 0 };

void initialize_timeout(struct event_base *base)
{
    struct timeval tv_in = { 10, 0 };
    const struct timeval *tv_out;
    tv_out = event_base_init_common_timeout(base, &tv_in);
    memcpy(&ten_seconds, tv_out, sizeof(struct timeval));
}

int my_event_add(struct event *ev, const struct timeval *tv)
{
    /* Note that ev must have the same event_base that we passed to
       initialize_timeout */
    if (tv && tv->tv_sec == 10 && tv->tv_usec == 0)
        return event_add(ev, &ten_seconds);
    else
        return event_add(ev, tv);
}
```

As with all optimization functions, you should avoid using the common_timeout functionality unless you're pretty sure that it matters for you.

This functionality was introduced in Libevent 2.0.4-alpha.

## Telling a good event apart from cleared memory

Libevent provides functions that you can use to distinguish an initialized event from memory that has been cleared by setting it to 0 (for example, by allocating it with calloc() or clearing it with memset() or bzero()).

Interface

```c
int event_initialized(const struct event *ev);

#define evsignal_initialized(ev) event_initialized(ev)
#define evtimer_initialized(ev) event_initialized(ev)
```

Warning

These functions can't reliably distinguish between an initialized event and a hunk of uninitialized memory. You should not use them unless you know that the memory in question is either cleared or initialized as an event.

Generally, you shouldn't need to use these functions unless you've got a pretty specific application in mind. Events returned by event_new() are always initialized.

Example

```c
#include <event2/event.h>
#include <stdlib.h>

struct reader {
    evutil_socket_t fd;
};

#define READER_ACTUAL_SIZE() \
    (sizeof(struct reader) + \
     event_get_struct_event_size())

#define READER_EVENT_PTR(r) \
    ((struct event *) (((char*)(r))+sizeof(struct reader)))

struct reader *allocate_reader(evutil_socket_t fd)
{
    struct reader *r = calloc(1, READER_ACTUAL_SIZE());
    if (r)
        r->fd = fd;
    return r;
}

void readcb(evutil_socket_t, short, void *);
int add_reader(struct reader *r, struct event_base *b)
{
    struct event *ev = READER_EVENT_PTR(r);
    if (!event_initialized(ev))
        event_assign(ev, b, r->fd, EV_READ, readcb, r);
    return event_add(ev, NULL);
}
```

The event_initialized() function has been present since Libevent 0.3.

## Obsolete event manipulation functions

Pre-2.0 versions of Libevent did not have event_assign() or event_new(). Instead, you had event_set(), which associated the event with the "current" base. If you had more than one base, you needed to remember to call event_base_set() afterwards to make sure that the event was associated with the base you actually wanted to use.

Interface

```c
void event_set(struct event *event, evutil_socket_t fd, short what,
        void(*callback)(evutil_socket_t, short, void *), void *arg);
int event_base_set(struct event_base *base, struct event *event);
```

The event_set() function was like event_assign(), except for its use of the current base. The event_base_set() function changes the base associated with an event.

There were variants of event_set() for dealing more conveniently with timers and signals: evtimer_set() corresponded roughly to evtimer_assign(), and evsignal_set() corresponded roughly to evsignal_assign().

Versions of Libevent before 2.0 used "signal_" as the prefix for the signal-based variants of event_set() and so on, rather than "evsignal_". (That is, they had signal_set(), signal_add(), signal_del(), signal_pending(), and signal_initialized().)  Truly ancient versions of Libevent (before 0.6) used "timeout_" instead of "evtimer_". Thus, if you're doing code archeology, you might see timeout_add(), timeout_del(), timeout_initialized(), timeout_set(), timeout_pending(), and so on.

In place of the event_get_fd() and event_get_signal() functions, older versions of Libevent (before 2.0) used two macros called EVENT_FD() and EVENT_SIGNAL(). These macros inspected the event structure's contents directly and so prevented binary compatibility between versions; in 2.0 and later they are just aliases for event_get_fd() and event_get_signal().

Since versions of Libevent before 2.0 did not have locking support, it wasn't safe to call any of the functions that change an event's state with respect to a base from outside the thread running the base. These include event_add(), event_del(), event_active(), and event_base_once().

There was also an event_once() function that played the role of event_base_once(), but used the current base.

The EV_PERSIST flag did not interoperate sensibly with timeouts before Libevent 2.0. Instead resetting the timeout whenever the event was activated, the EV_PERSIST flag did nothing with the timeout.

Libevent versions before 2.0 did not support having multiple events inserted at the same time with the same fd and the same READ/WRITE. In other words, only one event at a time could be waiting for read on each fd, and only one event at a time could be waiting for write on each fd.

_Go back to [Index](README.md)_
