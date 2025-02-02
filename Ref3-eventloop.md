# Working with an event loop

## Running the loop

Once you have an event_base with some events registered (see the next section about how to create and register events), you will want Libevent to wait for events and alert you about them.

Interface

```c
#define EVLOOP_ONCE		0x01
#define EVLOOP_NONBLOCK		0x02
#define EVLOOP_NO_EXIT_ON_EMPTY	0x04

int event_base_loop(struct event_base *base, int flags);
```

By default, the event_base_loop() function 'runs' an event_base until there are no more events registered in it. To run the loop, it repeatedly checks whether any of the registered events has triggered (for example, if a read event's file descriptor is ready to read, or if a timeout event's timeout is ready to expire). Once this happens, it marks all triggered events as "active", and starts to run them.

You can change the behavior of event_base_loop() by setting one or more flags in its 'flags' argument. If EVLOOP_ONCE is set, then the loop will wait until some events become active, then run active events until there are no more to run, then return. If EVLOOP_NONBLOCK is set, then the loop will not wait for events to trigger: it will only check whether any events are ready to trigger immediately, and run their callbacks if so.

Ordinarily, the loop will exit as soon as it has no pending or active events. You can override this behavior by passing the EVLOOP_NO_EXIT_ON_EMPTY flag for example, if you're going to be adding events from some other thread. If you do set EVLOOP_NO_EXIT_ON_EMPTY, the loop will keep running until somebody calls event_base_loopbreak(), or calls event_base_loopexit(), or an error occurs.

When it is done, event_base_loop() returns 0 if it exited normally, -1 if it exited because of some unhandled error in the backend, and 1 if it exited because there were no more pending or active events.

To aid in understanding, here's an approximate summary of the event_base_loop algorithm:

Pseudocode

```c
while (any events are registered with the loop,
        or EVLOOP_NO_EXIT_ON_EMPTY was set) {

    if (EVLOOP_NONBLOCK was set, or any events are already active)
        If any registered events have triggered, mark them active.
    else
        Wait until at least one event has triggered, and mark it active.

    for (p = 0; p < n_priorities; ++p) {
       if (any event with priority of p is active) {
          Run all active events with priority of p.
          break; /* Do not run any events of a less important priority */
       }
    }

    if (EVLOOP_ONCE was set or EVLOOP_NONBLOCK was set)
       break;
}
```

As a convenience, you can also call:

Interface

```c
int event_base_dispatch(struct event_base *base);
```

The event_base_dispatch() call is the same as event_base_loop(), with no flags set. Thus, it keeps running until there are no more registered events or until event_base_loopbreak() or event_base_loopexit() is called.

These functions are defined in <event2/event.h>. They have existed since Libevent 1.0.

## Stopping the loop

If you want an active event loop to stop running before all events are removed from it, you have two slightly different functions you can call.

Interface

```c
int event_base_loopexit(struct event_base *base,
                        const struct timeval *tv);
int event_base_loopbreak(struct event_base *base);
```

The event_base_loopexit() function tells an event_base to stop looping after a given time has elapsed. If the 'tv' argument is NULL, the event_base stops looping without a delay. If the event_base is currently running callbacks for any active events, it will continue running them, and not exit until they have all been run.

The event_base_loopbreak() function tells the event_base to exit its loop immediately. It differs from event_base_loopexit(base, NULL) in that if the event_base is currently running callbacks for any active events, it will exit immediately after finishing the one it's currently processing.

Note also that event_base_loopexit(base,NULL) and event_base_loopbreak(base) act differently when no event loop is running: loopexit schedules the next instance of the event loop to stop right after the next round of callbacks are run (as if it had been invoked with EVLOOP_ONCE) whereas loopbreak only stops a currently running loop, and has no effect if the event loop isn't running.

Both of these methods return 0 on success and -1 on failure.

Example: Shut down immediately

```c
#include <event2/event.h>

/* Here's a callback function that calls loopbreak */
void cb(int sock, short what, void *arg)
{
    struct event_base *base = arg;
    event_base_loopbreak(base);
}

void main_loop(struct event_base *base, evutil_socket_t watchdog_fd)
{
    struct event *watchdog_event;

    /* Construct a new event to trigger whenever there are any bytes to
       read from a watchdog socket. When that happens, we'll call the
       cb function, which will make the loop exit immediately without
       running any other active events at all.
     */
    watchdog_event = event_new(base, watchdog_fd, EV_READ, cb, base);

    event_add(watchdog_event, NULL);

    event_base_dispatch(base);
}
```

Example: Run an event loop for 10 seconds, then exit.

```c
#include <event2/event.h>

void run_base_with_ticks(struct event_base *base)
{
  struct timeval ten_sec;

  ten_sec.tv_sec = 10;
  ten_sec.tv_usec = 0;

  /* Now we run the event_base for a series of 10-second intervals, printing
     "Tick" after each. For a much better way to implement a 10-second
     timer, see the section below about persistent timer events. */
  while (1) {
     /* This schedules an exit ten seconds from now. */
     event_base_loopexit(base, &ten_sec);

     event_base_dispatch(base);
     puts("Tick");
  }
}
```

Sometimes you may want to tell whether your call to event_base_dispatch() or event_base_loop() exited normally, or because of a call to event_base_loopexit() or event_base_break(). You can use these functions to tell whether loopexit or break was called:

Interface

```c
int event_base_got_exit(struct event_base *base);
int event_base_got_break(struct event_base *base);
```

These two functions will return true if the loop was stopped with event_base_loopexit() or event_base_break() respectively, and false otherwise. Their values will be reset the next time you start the event loop.

These functions are declared in <event2/event.h>. The event_break_loopexit() function was first implemented in Libevent 1.0c; event_break_loopbreak() was first implemented in Libevent 1.4.3.

## Re-checking for events

Ordinarily, Libevent checks for events, then runs all the active events with the highest priority, then checks for events again, and so on. But sometimes you might want to stop Libevent right after the current callback has been run, and tell it to scan again. By analogy to event_base_loopbreak(), you can do this with the function event_base_loopcontinue().

Interface

```c
int event_base_loopcontinue(struct event_base *);
```

Calling event_base_loopcontinue() has no effect if we aren't currently running event callbacks.

This function was introduced in Libevent 2.1.2-alpha.

## Checking the internal time cache

Sometimes you want to get an approximate view of the current time inside an event callback, and you want to get it without calling gettimeofday() yourself (presumably because your OS implements gettimeofday() as a syscall, and you're trying to avoid syscall overhead).

From within a callback, you can ask Libevent for its view of the current time when it began executing this round of callbacks:

Interface

```c
int event_base_gettimeofday_cached(struct event_base *base,
    struct timeval *tv_out);
```

The event_base_gettimeofday_cached() function sets the value of its 'tv_out' argument to the cached time if the event_base is currently executing callbacks. Otherwise, it calls evutil_gettimeofday() for the actual current time. It returns 0 on success, and negative on failure.

Note that since the timeval is cached when Libevent starts running callbacks, it will be at least a little inaccurate. If your callbacks take a long time to run, it may be *very* inaccurate. To force an immediate cache update, you can call this function:

Interface

```c
int event_base_update_cache_time(struct event_base *base);
```

It returns 0 on success and -1 on failure, and has no effect if the base was not running its event loop.

The event_base_gettimeofday_cached() function was new in Libevent 2.0.4-alpha. Libevent 2.1.1-alpha added event_base_update_cache_time().

## Dumping the event_base status

Interface

```c
void event_base_dump_events(struct event_base *base, FILE *f);
```

For help debugging your program (or debugging Libevent!) you might sometimes want a complete list of all events added in the event_base and their status. Calling event_base_dump_events() writes this list to the stdio file provided.

The list is meant to be human-readable; its format *will* change in future versions of Libevent.

This function was introduced in Libevent 2.0.1-alpha.

## Running a function over every event in an event_base

Interface

```c
typedef int (*event_base_foreach_event_cb)(const struct event_base *,
    const struct event *, void *);

int event_base_foreach_event(struct event_base *base,
                             event_base_foreach_event_cb fn,
                             void *arg);
```

You can use event_base_foreach_event() to iterate over every currently active or pending event associated with an event_base(). The provided callback will be invoked exactly once per event, in an unspecified order. The third argument of event_base_foreach_event() will be passed as the third argument to each invocation of the callback.

The callback function must return 0 to continue iteration, or some other integer to stop iterating. Whatever value the callback function finally returns will then be returned by event_base_foreach_function().

Your callback function *must not* modify any of the events that it receives, or add or remove any events to the event base, or otherwise modify any event associated with the event base, or undefined behavior can occur, up to or including crashes and heap-smashing.

The event_base lock will be held for the duration of the call to event_base_foreach_event() -- this will block other threads from doing anything useful with the event_base, so make sure that your callback doesn't take a long time.

This function was added in Libevent 2.1.2-alpha.

## Obsolete event loop functions

As discussed above, older versions of Libevent APIs had a global notion of a "current" event_base.

Some of the event loop functions in this section had variants that operated on the current base. These functions behaved as the current functions, except that they took no base argument.

| Current function              | Obsolete current-base version |
| --- | --- |
| event_base_dispatch()         | event_dispatch() |
| event_base_loop()             | event_loop() |
| event_base_loopexit()         | event_loopexit() |
| event_base_loopbreak()        | event_loopbreak() |

NOTE: Because event\_base did not support locking before Libevent 2.0, these functions weren't completely threadsafe: it was not permissible to call the \_loopbreak() or \_loopexit() functions from a thread other than the one executing the event loop.

_Go back to [Index](README.md)_
