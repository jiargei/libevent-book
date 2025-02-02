# Creating an event_base

Before you can use any interesting Libevent function, you need to allocate one or more event_base structures. Each event_base structure holds a set of events and can poll to determine which events are active.

If an event_base is set up to use locking, it is safe to access it between multiple threads. Its loop can only be run in a single thread, however. If you want to have multiple threads polling for IO, you need to have an event_base for each thread.

TIP: [A future version of Libevent may have support for event_bases that run events across multiple threads.]

Each event_base has a "method", or a backend that it uses to determine which events are ready. The recognized methods are:

- select
- poll
- epoll
- kqueue
- devpoll
- evport
- win32

The user can disable specific backends with environment variables. If you want to turn off the kqueue backend, set the EVENT_NOKQUEUE environment variable, and so on. If you want to turn off backends from within the program, see notes on event_config_avoid_method() below.

## Setting up a default event_base

The event_base_new() function allocates and returns a new event base with the default settings. It examines the environment variables and returns a pointer to a new event_base. If there is an error, it returns NULL.

When choosing among methods, it picks the fastest method that the OS supports.

Interface

```c
struct event_base *event_base_new(void);
```

For most programs, this is all you need.

The event_base_new() function is declared in <event2/event.h>. It first appeared in Libevent 1.4.3.

## Setting up a complicated event_base

If you want more control over what kind of event_base you get, you need to use an event_config. An event_config is an opaque structure that holds information about your preferences for an event_base. When you want an event_base, you pass the event_config to event_base_new_with_config().

Interface

```c
struct event_config *event_config_new(void);
struct event_base *event_base_new_with_config(const struct event_config *cfg);
void event_config_free(struct event_config *cfg);
```

To allocate an event_base with these functions, you call event_config_new() to allocate a new event_config. Then, you call other functions on the event_config to tell it about your needs. Finally, you call event_base_new_with_config() to get a new event_base. When you are done, you can free the event_config with event_config_free().

Interface

```c
int event_config_avoid_method(struct event_config *cfg, const char *method);

enum event_method_feature {
    EV_FEATURE_ET = 0x01,
    EV_FEATURE_O1 = 0x02,
    EV_FEATURE_FDS = 0x04,
};
int event_config_require_features(struct event_config *cfg,
                                  enum event_method_feature feature);

enum event_base_config_flag {
    EVENT_BASE_FLAG_NOLOCK = 0x01,
    EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
    EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
    EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
    EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,
    EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};
int event_config_set_flag(struct event_config *cfg,
    enum event_base_config_flag flag);
```

Calling event_config_avoid_method tells Libevent to avoid a specific available backend by name. Calling event_config_require_feature() tells Libevent not to use any backend that cannot supply all of a set of features. Calling event_config_set_flag() tells Libevent to set one or more of the run-time flags below when constructing the event base.

The recognized feature values for event_config_require_features are:

EV_FEATURE_ET::
    Requires a backend method that supports edge-triggered IO.

EV_FEATURE_O1::
    Requires a backend method where adding or deleting a single event, or having a single event become active, is an O(1) operation.

EV_FEATURE_FDS::
    Requires a backend method that can support arbitrary file descriptor types, and not just sockets.

The recognized option values for event_config_set_flag() are:

EVENT_BASE_FLAG_NOLOCK::
    Do not allocate locks for the event_base. Setting
    this option may save a little time for locking and releasing the
    event_base, but will make it unsafe and nonfunctional to access it
    from multiple threads.

EVENT_BASE_FLAG_IGNORE_ENV::
    Do not check the EVENT_* environment
    variables when picking which backend method to use. Think hard before
    using this flag: it can make it harder for users to debug the interactions
    between your program and Libevent.

EVENT_BASE_FLAG_STARTUP_IOCP::
    On Windows only, this flag makes Libevent
    enable any necessary IOCP dispatch logic on startup, rather than
    on-demand.

EVENT_BASE_FLAG_NO_CACHE_TIME::
    Instead of checking the current time every
    time the event loop is ready to run timeout callbacks, check it after
    every timeout callback. This can use more CPU than you necessarily
    intended, so watch out!

EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST::
    Tells Libevent that, if it decides to
    use the epoll backend, it is safe to use the faster "changelist"-based
    backend. The epoll-changelist backend can avoid needless system calls in
    cases where the same fd has its status modified more than once between
    calls to the backend's dispatch function, but it also trigger a kernel bug
    that causes erroneous results if you give Libevent any fds cloned by
    dup() or its variants. This flag has no effect if you use a backend
    other than epoll. You can also turn on the epoll-changelist option by
    setting the EVENT_EPOLL_USE_CHANGELIST environment variable.

EVENT_BASE_FLAG_PRECISE_TIMER::
    By default, Libevent tries to use the fastest available timing mechanism
    that the operating system provides. If there is a slower timing
    mechanism that provides more fine-grained timing precision, this
    flag tells Libevent to use that timing mechanism instead. If the
    operating system provides no such slower-but-more-precise mechanism,
    this flag has no effect.

The above functions that manipulate an event_config all return 0 on success, -1 on failure.

NOTE: It is easy to set up an event_config that requires a backend that your OS does not provide. For example, as of Libevent 2.0.1-alpha, there is no O(1) backend for Windows, and no backend on Linux that provides both EV_FEATURE_FDS and EV_FEATURE_O1. If you have made a configuration that Libevent can't satisfy, event_base_new_with_config() will return NULL.

Interface

```c
int event_config_set_num_cpus_hint(struct event_config *cfg, int cpus)
```

This function is currently only useful with Windows when using IOCP, though it may become useful for other platforms in the future. Calling it tells the event_config that the event_base it generates should try to make good use of a given number of CPUs when multithreading. Note that this is only a hint: the event base may wind up using more or fewer CPUs than you select.

Interface

```c
int event_config_set_max_dispatch_interval(struct event_config *cfg,
    const struct timeval *max_interval, int max_callbacks,
    int min_priority);
```

This function prevents priority inversion by limiting how many low-priority event callbacks can be invoked before checking for more high-priority events. If max_interval is non-null, the event loop checks the time after each callback, and re-scans for high-priority events if max_interval has passed. If max_callbacks is nonnegative, the event loop also checks for more events after max_callbacks callbacks have been invoked. These rules apply to any event of min_priority or higher.

//BUILD: FUNCTIONBODY INC:event2/event.

Example: Preferring edge-triggered backends

```c
struct event_config *cfg;
struct event_base *base;
int i;

/* My program wants to use edge-triggered events if at all possible. So
   I'll try to get a base twice: Once insisting on edge-triggered IO, and
   once not. */
for (i=0; i<2; ++i) {
    cfg = event_config_new();

    /* I don't like select. */
    event_config_avoid_method(cfg, "select");

    if (i == 0)
        event_config_require_features(cfg, EV_FEATURE_ET);

    base = event_base_new_with_config(cfg);
    event_config_free(cfg);
    if (base)
        break;

    /* If we get here, event_base_new_with_config() returned NULL. If
       this is the first time around the loop, we'll try again without
       setting EV_FEATURE_ET. If this is the second time around the
       loop, we'll give up. */
}
```

//BUILD: FUNCTIONBODY INC:event2/event.h

Example: Avoiding priority-inversion

```c
struct event_config *cfg;
struct event_base *base;

cfg = event_config_new();
if (!cfg)
   /* Handle error */;

/* I'm going to have events running at two priorities. I expect that
   some of my priority-1 events are going to have pretty slow callbacks,
   so I don't want more than 100 msec to elapse (or 5 callbacks) before
   checking for priority-0 events. */
struct timeval msec_100 = { 0, 100*1000 };
event_config_set_max_dispatch_interval(cfg, &msec_100, 5, 1);

base = event_base_new_with_config(cfg);
if (!base)
   /* Handle error */;

event_base_priority_init(base, 2);
```

These functions and types are declared in <event2/event.h>.

The EVENT_BASE_FLAG_IGNORE_ENV flag first appeared in Libevent 2.0.2-alpha. The EVENT_BASE_FLAG_PRECISE_TIMER flag first appeared in Libevent 2.1.2-alpha. The  event_config_set_num_cpus_hint() function was new in Libevent 2.0.7-rc, and event_config_set_max_dispatch_interval() was new in 2.1.1-alpha. Everything else in this section first appeared in Libevent 2.0.1-alpha.

## Examining an event_base's backend method

Sometimes you want to see which features are actually available in an event_base, or which method it's using.

Interface

```c
const char **event_get_supported_methods(void);
```

The event_get_supported_methods() function returns a pointer to an array of the names of the methods supported in this version of Libevent. The last element in the array is NULL.

//BUILD: FUNCTIONBODY INC:event2/event.h

Example

```c
int i;
const char **methods = event_get_supported_methods();
printf("Starting Libevent %s. Available methods are:\n",
    event_get_version());
for (i=0; methods[i] != NULL; ++i) {
    printf("    %s\n", methods[i]);
}
```

NOTE: This function returns a list of the methods that Libevent was compiled to support. It is possible that your operating system will not in fact support them all when Libevent tries to run. For example, you could be on a version of OSX where kqueue is too buggy to use.

Interface

```c
const char *event_base_get_method(const struct event_base *base);
enum event_method_feature event_base_get_features(const struct event_base *base);
```

The event_base_get_method() call returns the name of the actual method in use by an event_base. The event_base_get_features() call returns a bitmask of the features that it supports.

//BUILD: FUNCTIONBODY INC:event2/event.h

Example

```c
struct event_base *base;
enum event_method_feature f;

base = event_base_new();
if (!base) {
    puts("Couldn't get an event_base!");
} else {
    printf("Using Libevent with backend method %s.",
        event_base_get_method(base));
    f = event_base_get_features(base);
    if ((f & EV_FEATURE_ET))
        printf("  Edge-triggered events are supported.");
    if ((f & EV_FEATURE_O1))
        printf("  O(1) event notification is supported.");
    if ((f & EV_FEATURE_FDS))
        printf("  All FD types are supported.");
    puts("");
}
```

These functions are defined in <event2/event.h>. The event_base_get_method() call was first available in Libevent 1.4.3. The others first appeared in Libevent 2.0.1-alpha.

## Deallocating an event_base

When you are finished with an event_base, you can deallocate it with event_base_free().

Interface

```c
void event_base_free(struct event_base *base);
```

Note that this function does not deallocate any of the events that are currently associated with the event_base, or close any of their sockets, or free any of their pointers.

The event_base_free() function is defined in <event2/event.h>. It was first implemented in Libevent 1.2.

## Setting priorities on an event_base

Libevent supports setting multiple priorities on an event. By default, though, an event_base supports only a single priority level. You can set the number of priorities on an event_base by calling event_base_priority_init().

Interface

```c
int event_base_priority_init(struct event_base *base, int n_priorities);
```

This function returns 0 on success and -1 on failure. The 'base' argument is the event_base to modify, and n_priorities is the number of priorities to support. It must be at least 1. The available priorities for new events will be numbered from 0 (most important) to n_priorities-1 (least important).

There is a constant, EVENT_MAX_PRIORITIES, that sets the upper bound on the value of n_priorities. It is an error to call this function with a higher value for n_priorities.

NOTE: You *must* call this function before any events become active. It is best to call it immediately after creating the event_base.

To find the number of priorities currently supported by a base, you can call event_base_getnpriorities().

Interface

```c
int event_base_get_npriorities(struct event_base *base);
```

The return value is equal to the number of priorities configured in the base. So if event_base_get_npriorities() returns 3, then allowable priority values are 0, 1, and 2.

Example

```c
For an example, see the documentation for event_priority_set below.
```

By default, all new events associated with this base will be initialized with priority equal to n_priorities / 2.

The event_base_priority_init function is defined in <event2/event.h>. It has been available since Libevent 1.0. The event_base_get_npriorities() function was new in Libevent 2.1.1-alpha.

## Reinitializing an event_base after fork()

Not all event backends persist cleanly after a call to fork(). Thus, if your program uses fork() or a related system call in order to start a new process, and you want to continue using an event_base after you have forked, you may need to reinitialize it.

Interface

```c
int event_reinit(struct event_base *base);
```

The function returns 0 on success, -1 on failure.

//BUILD: FUNCTIONBODY INC:event2/event.h INC:../example_stubs/Ref2.h INC:unistd.h

Example

```c
struct event_base *base = event_base_new();

/* ... add some events to the event_base ... */

if (fork()) {
    /* In parent */
    continue_running_parent(base); /*...*/
} else {
    /* In child */
    event_reinit(base);
    continue_running_child(base); /*...*/
}
```

The event_reinit() function is defined in <event2/event.h>. It was first available in Libevent 1.4.3-alpha.

## Obsolete event_base functions

Older versions of Libevent relied pretty heavily on the idea of a "current" event_base. The "current" event_base was a global setting shared across all threads. If you forgot to specify which event_base you wanted, you got the current one. Since event_bases weren't threadsafe, this could get pretty error-prone.

Instead of event_base_new(), there was:

Interface

```c
struct event_base *event_init(void);
```

This function worked like event_base_new(), and set the current base to the allocated base. There was no other way to change the current base.

Some of the event_base functions in this section had variants that operated on the current base. These functions behaved as the current functions, except that they took no base argument.

| Current function              | Obsolete current-base version |
| --- | --- |
| event_base_priority_init()    | event_priority_init() |
| event_base_get_method()       | event_get_method() |

_Go back to [Index](README.md)_
