# Setting up the Libevent library

Libevent has a few global settings that are shared across the entire process. These affect the entire library.

You *must* make any changes to these settings before you call any other part of the Libevent library. If you don't, Libevent could wind up in an inconsistent state.

## Log messages in Libevent

Libevent can log internal errors and warnings. It also logs debugging messages if it was compiled with logging support. By default, these messages are written to stderr. You can override this behavior by providing your own logging function.

Interface

```c

#define EVENT_LOG_DEBUG 0
#define EVENT_LOG_MSG   1
#define EVENT_LOG_WARN  2
#define EVENT_LOG_ERR   3

/* Deprecated; see note at the end of this section */
#define _EVENT_LOG_DEBUG EVENT_LOG_DEBUG
#define _EVENT_LOG_MSG   EVENT_LOG_MSG
#define _EVENT_LOG_WARN  EVENT_LOG_WARN
#define _EVENT_LOG_ERR   EVENT_LOG_ERR

typedef void (*event_log_cb)(int severity, const char *msg);

void event_set_log_callback(event_log_cb cb);

```

To override Libevent's logging behavior, write your own function matching the signature of event_log_cb, and pass it as an argument to event_set_log_callback(). Whenever Libevent wants to log a message, it will pass it to the function you provided. You can have Libevent return to its default behavior by calling event_set_log_callback() again with NULL as an argument.

### Examples

```c

#include <event2/event.h>
#include <stdio.h>

static void discard_cb(int severity, const char *msg)
{
    /* This callback does nothing. */
}

static FILE *logfile = NULL;
static void write_to_file_cb(int severity, const char *msg)
{
    const char *s;
    if (!logfile)
        return;
    switch (severity) {
        case _EVENT_LOG_DEBUG: s = "debug"; break;
        case _EVENT_LOG_MSG:   s = "msg";   break;
        case _EVENT_LOG_WARN:  s = "warn";  break;
        case _EVENT_LOG_ERR:   s = "error"; break;
        default:               s = "?";     break; /* never reached */
    }
    fprintf(logfile, "[%s] %s\n", s, msg);
}

/* Turn off all logging from Libevent. */
void suppress_logging(void)
{
    event_set_log_callback(discard_cb);
}

/* Redirect all Libevent log messages to the C stdio file 'f'. */
void set_logfile(FILE *f)
{
    logfile = f;
    event_set_log_callback(write_to_file_cb);
}
```

### NOTE

It is not safe to invoke Libevent functions from within a user-provided
event_log_cb callback!  For instance, if you try to write a log callback that
uses bufferevents to send warning messages to a network socket, you are
likely to run into strange and hard-to-diagnose bugs. This restriction may
be removed for some functions in a future version of Libevent.

Ordinarily, debug logs are not enabled, and are not sent to the logging
callback. You can turn them on manually, if Libevent was built to support
them.

Interface

```c

#define EVENT_DBG_NONE 0
#define EVENT_DBG_ALL 0xffffffffu

void event_enable_debug_logging(ev_uint32_t which);

```

Debugging logs are verbose, and not necessarily useful under most circumstances. Calling event_enable_debug_logging() with EVENT_DBG_NONE gets default behavior; calling it with EVENT_DBG_ALL turns on all the supported debugging logs. More fine-grained options may be supported in future versions.

These functions are declared in <event2/event.h>. They first appeared in Libevent 1.0c, except for event_enable_debug_logging(), which first appeared in Libevent 2.1.1-alpha.

### COMPATIBILITY NOTE

Before Libevent 2.0.19-stable, the EVENT_LOG_* macros had names that began with an underscore: _EVENT_LOG_DEBUG, _EVENT_LOG_MSG, _EVENT_LOG_WARN, and _EVENT_LOG_ERR. These older names are deprecated, and should only be used for backward compatibility with Libevent 2.0.18-stable and earlier. They may be removed in a future version of Libevent.

## Handling fatal errors

When Libevent detects a non-recoverable internal error (such as a corrupted data structure), its default behavior is to call exit() or abort() to leave the currently running process. These errors almost always mean that there is a bug somewhere: either in your code, or in Libevent itself.

You can override Libevent's behavior if you want your application to handle fatal errors more gracefully, by providing a function that Libevent should call in lieu of exiting.

Interface

```c

typedef void (*event_fatal_cb)(int err);
void event_set_fatal_callback(event_fatal_cb cb);

```

To use these functions, you first define a new function that Libevent should call upon encountering a fatal error, then you pass it to event_set_fatal_callback(). Later, if Libevent encounters a fatal error, it will call the function you provided.

Your function *should not* return control to Libevent; doing so may cause undefined behavior, and Libevent might exit anyway to avoid crashing. Once your function has been called, you should not call any other Libevent function.

These functions are declared in <event2/event.h>. They first appeared in Libevent 2.0.3-alpha.

## Memory management

By default, Libevent uses the C library's memory management functions to allocate memory from the heap. You can have Libevent use another memory manager by providing your own replacements for malloc, realloc, and free. You might want to do this if you have a more efficient allocator that you want Libevent to use, or if you have an instrumented allocator that you want Libevent to use in order to look for memory leaks.

Interface

```c

void event_set_mem_functions(void *(*malloc_fn)(size_t sz), 
                             void *(*realloc_fn)(void *ptr, size_t sz),  
                             void (*free_fn)(void *ptr));

```

Here's a simple example that replaces Libevent's allocation functions with variants that count the total number of bytes that are allocated. In reality, you'd probably want to add locking here to prevent errors when Libevent is running in multiple threads.

Example

```c

#include <event2/event.h>
#include <sys/types.h>
#include <stdlib.h>

/* This union's purpose is to be as big as the largest of all the
 * types it contains. */
union alignment {
    size_t sz;
    void *ptr;
    double dbl;
};
/* We need to make sure that everything we return is on the right
   alignment to hold anything, including a double. */
#define ALIGNMENT sizeof(union alignment)

/* We need to do this cast-to-char* trick on our pointers to adjust
   them; doing arithmetic on a void* is not standard. */
#define OUTPTR(ptr) (((char*)ptr)+ALIGNMENT)
#define INPTR(ptr) (((char*)ptr)-ALIGNMENT)

static size_t total_allocated = 0;
static void *replacement_malloc(size_t sz)
{
    void *chunk = malloc(sz + ALIGNMENT);
    if (!chunk) return chunk;
    total_allocated += sz;
    *(size_t*)chunk = sz;
    return OUTPTR(chunk);
}
static void *replacement_realloc(void *ptr, size_t sz)
{
    size_t old_size = 0;
    if (ptr) {
        ptr = INPTR(ptr);
        old_size = *(size_t*)ptr;
    }
    ptr = realloc(ptr, sz + ALIGNMENT);
    if (!ptr)
        return NULL;
    *(size_t*)ptr = sz;
    total_allocated = total_allocated - old_size + sz;
    return OUTPTR(ptr);
}
static void replacement_free(void *ptr)
{
    ptr = INPTR(ptr);
    total_allocated -= *(size_t*)ptr;
    free(ptr);
}
void start_counting_bytes(void)
{
    event_set_mem_functions(replacement_malloc,
                            replacement_realloc,
                            replacement_free);
}

```

### NOTES

- Replacing the memory management functions affects all future calls to allocate, resize, or free memory from Libevent. Therefore, you need to make sure that you replace the functions _before_ you call any other Libevent functions. Otherwise, Libevent will use your version of free to deallocate memory returned from the C library's version of malloc.
- Your malloc and realloc functions need to return memory chunks with the same alignment as the C library.
- Your realloc function needs to handle realloc(NULL, sz) correctly (that is, by treating it as malloc(sz)).
- Your realloc function needs to handle realloc(ptr, 0) correctly (that is, by treating it as free(ptr)).
- Your free function does not need to handle free(NULL).
- Your malloc function does not need to handle malloc(0).
- The replaced memory management functions need to be threadsafe if you are using Libevent from more than one thread.
- Libevent will use these functions to allocate memory that it returns to you. Thus, if you want to free memory that is allocated and returned by a Libevent function, and you have replaced the malloc and realloc functions, then you will probably have to use your replacement free function to free it.

The event_set_mem_functions() function is declared in <event2/event.h>. It first appeared in Libevent 2.0.1-alpha.

Libevent can be built with event_set_mem_functions() disabled. If it is, then programs using event_set_mem_functions will not compile or link. In Libevent 2.0.2-alpha and later, you can detect the presence of event_set_mem_functions() by checking whether the EVENT_SET_MEM_FUNCTIONS_IMPLEMENTED macro is defined.

## Locks and threading

As you probably know if you're writing multithreaded programs, it isn't always safe to access the same data from multiple threads at the same time.

Libevent structures can generally work three ways with multithreading.

- Some structures are inherently single-threaded: it is never safe to use them from more than one thread at the same time.
- Some structures are optionally locked: you can tell Libevent for each object whether you need to use it from multiple threads at once.
- Some structures are always locked: if Libevent is running with lock support, then they are always safe to use from multiple threads at once.

To get locking in Libevent, you must tell Libevent which locking functions to use. You need to do this before you call any Libevent function that allocates a structure that needs to be shared between threads.

If you are using the pthreads library, or the native Windows threading code, you're in luck. There are pre-defined functions that will set Libevent up to use the right pthreads or Windows functions for you.

Interface

```c

#ifdef WIN32
int evthread_use_windows_threads(void);
#define EVTHREAD_USE_WINDOWS_THREADS_IMPLEMENTED
#endif
#ifdef _EVENT_HAVE_PTHREADS
int evthread_use_pthreads(void);
#define EVTHREAD_USE_PTHREADS_IMPLEMENTED
#endif
```

Both functions return 0 on success, and -1 on failure.

If you need to use a different threading library, then you have a little more work ahead of you. You need to define functions that use your library to implement:

- Locks
  - locking
  - unlocking
  - lock allocation
  - lock destruction
  - Conditions
  - condition variable creation
  - condition variable destruction
  - waiting on a condition variable
  - signaling/broadcasting to a condition variable
- Threads
  - thread ID detection

Then you tell Libevent about these functions, using the evthread_set_lock_callbacks and evthread_set_id_callback interfaces.

Interface

```c

#define EVTHREAD_WRITE 0x04
#define EVTHREAD_READ  0x08
#define EVTHREAD_TRY   0x10

#define EVTHREAD_LOCKTYPE_RECURSIVE 1
#define EVTHREAD_LOCKTYPE_READWRITE 2

#define EVTHREAD_LOCK_API_VERSION 1

struct evthread_lock_callbacks {
       int lock_api_version;
       unsigned supported_locktypes;
       void *(*alloc)(unsigned locktype);
       void (*free)(void *lock, unsigned locktype);
       int (*lock)(unsigned mode, void *lock);
       int (*unlock)(unsigned mode, void *lock);
};

int evthread_set_lock_callbacks(const struct evthread_lock_callbacks *);

void evthread_set_id_callback(unsigned long (*id_fn)(void));

struct evthread_condition_callbacks {
        int condition_api_version;
        void *(*alloc_condition)(unsigned condtype);
        void (*free_condition)(void *cond);
        int (*signal_condition)(void *cond, int broadcast);
        int (*wait_condition)(void *cond, void *lock,
            const struct timeval *timeout);
};

int evthread_set_condition_callbacks(
        const struct evthread_condition_callbacks *);

```

The evthread_lock_callbacks structure describes your locking callbacks and their abilities. For the version described above, the lock_api_version field must be set to EVTHREAD_LOCK_API_VERSION. The supported_locktypes field must be set to a bitmask of the EVTHREAD_LOCKTYPE_* constants to describe which lock types you can support. (As of 2.0.4-alpha, EVTHREAD_LOCK_RECURSIVE is mandatory and EVTHREAD_LOCK_READWRITE is unused.)

The 'alloc' function must return a new lock of the specified type. The 'free' function must release all resources held by a lock of the specified type. The 'lock' function must try to acquire the lock in the specified mode, returning 0 on success and nonzero on failure. The 'unlock' function must try to unlock the lock, returning 0 on success and nonzero on failure.

Recognized lock types are:

0::
        A regular, not-necessarily recursive lock.

EVTHREAD_LOCKTYPE_RECURSIVE::
        A lock that does not block a thread already holding it from
        requiring it again. Other threads can acquire the lock once
        the thread holding it has unlocked it as many times as it was
        initially locked.

EVTHREAD_LOCKTYPE_READWRITE::
        A lock that allows multiple threads to hold it at once for
        reading, but only one thread at a time to hold it for writing.
        A writer excludes all readers.

Recognized lock modes are:

EVTHREAD_READ::
        For READWRITE locks only: acquire or release the lock for
        reading.

EVTHREAD_WRITE::
        For READWRITE locks only: acquire or release the lock for
        writing.

EVTHREAD_TRY::
        For locking only: acquire the lock only if the lock can be
        acquired immediately.

The id_fn argument must be a function returning an unsigned long identifying what thread is calling the function. It must always return the same number for the same thread, and must not ever return the same number for two different threads if they are both executing at the same time.

The evthread_condition_callbacks structure describes callbacks related to condition variables. For the version described above, the lock_api_version field must be set to EVTHREAD_CONDITION_API_VERSION. The alloc_condition function must return a pointer to a new condition variable. It receives 0 as its argument. The free_condition function must release storage and resources held by a condition variable. The wait_condition function takes three arguments: a condition allocated by alloc_condition, a lock allocated by the evthread_lock_callbacks.alloc function you provided, and an optional timeout. The lock will be held whenever the function is called; the function must release the lock, and wait until the condition becomes signalled or until the (optional) timeout has elapsed. The wait_condition function should return -1 on an error, 0 if the condition is signalled, and 1 on a timeout. Before it returns, it should make sure it holds the lock again. Finally, the signal\_condition function should cause _one_ thread waiting on the condition to wake up (if its broadcast argument is false) and _all_ threads currently waiting on the condition to wake up (if its broadcast argument is true). It will only be held while holding the lock associated with the condition.

For more information on condition variables, look at the documentation for pthreads's pthread_cond_* functions, or Windows's CONDITION_VARIABLE functions.

Examples

```c
For an example of how to use these functions, see evthread_pthread.c and evthread_win32.c in the Libevent source distribution.
```

The functions in this section are declared in <event2/thread.h>. Most of them first appeared in Libevent 2.0.4-alpha. Libevent versions from 2.0.1-alpha through 2.0.3-alpha used an older interface to set locking functions. The event_use_pthreads() function requires you to link your program against the event_pthreads library.

The condition-variable functions were new in Libevent 2.0.7-rc; they were added to solve some otherwise intractable deadlock problems.

Libevent can be built with locking support disabled. If it is, then programs built to use the above thread-related functions will not run.

## Debugging lock usage

To help debug lock usage, Libevent has an optional "lock debugging" feature that wraps its locking calls in order to catch typical lock errors, including:

- unlocking a lock that we don't actually hold
- re-locking a non-recursive lock

If one of these lock errors occurs, Libevent exits with an assertion failure.

Interface

```c

void evthread_enable_lock_debugging(void);
#define evthread_enable_lock_debuging() evthread_enable_lock_debugging()

```

NOTE: This function MUST be called before any locks are created or used. To be safe, call it just after you set your threading functions.

This function was new in Libevent 2.0.4-alpha with the misspelled name "evthread_enable_lock_debuging()."  The spelling was fixed to evthread_enable_lock_debugging() in 2.1.2-alpha; both names are currently supported.

## Debugging event usage

There are some common errors in using events that Libevent can detect and report for you. They include:

- Treating an uninitialized struct event as though it were initialized.
- Try to reinitialize a pending struct event.

Tracking which events are initialized requires that Libevent use extra memory and CPU, so you should only enable debug mode when actually debugging your program.

Interface

```c

void event_enable_debug_mode(void);

```

This function must only be called before any event_base is created.

When using debug mode, you might run out of memory if your program uses a
large number of events created with event_assign() [not event_new()]. This
happens because Libevent has no way of telling when an event created with
event_assign() will no longer be used. (It can tell that an event_new()
event has become invalid when you call event_free() on it.)  If you want to
avoid running out of memory while debugging, you can explicitly tell Libevent
that such events are no longer to be treated as assigned:

Interface

```c
void event_debug_unassign(struct event *ev);
```

Calling event_debug_unassign() has no effect when debugging is not enabled.

Example

```c
#include <event2/event.h>
#include <event2/event_struct.h>

#include <stdlib.h>

void cb(evutil_socket_t fd, short what, void *ptr)
{
    /* We pass 'NULL' as the callback pointer for the heap allocated
     * event, and we pass the event itself as the callback pointer
     * for the stack-allocated event. */
    struct event *ev = ptr;

    if (ev)
        event_debug_unassign(ev);
}

/* Here's a simple mainloop that waits until fd1 and fd2 are both
 * ready to read. */
void mainloop(evutil_socket_t fd1, evutil_socket_t fd2, int debug_mode)
{
    struct event_base *base;
    struct event event_on_stack, *event_on_heap;

    if (debug_mode)
       event_enable_debug_mode();

    base = event_base_new();

    event_on_heap = event_new(base, fd1, EV_READ, cb, NULL);
    event_assign(&event_on_stack, base, fd2, EV_READ, cb, &event_on_stack);

    event_add(event_on_heap, NULL);
    event_add(&event_on_stack, NULL);

    event_base_dispatch(base);

    event_free(event_on_heap);
    event_base_free(base);
}
```

Detailed event debugging is a feature which can only be enabled at compile-time using the CFLAGS environment variable "-DUSE_DEBUG". With this flag enabled, any program compiled against Libevent will output a very verbose log detailing low-level activity on the back-end. These logs include, but not limited to, the following:

- event additions
- event deletions
- platform specific event notification information

This feature cannot be enabled or disabled via an API call so it must only be used in developer builds.

These debugging functions were added in Libevent 2.0.4-alpha.

## Detecting the version of Libevent

New versions of Libevent can add features and remove bugs. Sometimes you'll want to detect the Libevent version, so that you can:

- Detect whether the installed version of Libevent is good enough to
  build your program.
- Display the Libevent version for debugging.
- Detect the version of Libevent so that you can warn the user about
  bugs, or work around them.

Interface

```c
#define LIBEVENT_VERSION_NUMBER 0x02000300
#define LIBEVENT_VERSION "2.0.3-alpha"
const char *event_get_version(void);
ev_uint32_t event_get_version_number(void);
```

The macros make available the compile-time version of the Libevent library; the functions return the run-time version. Note that if you have dynamically linked your program against Libevent, these versions may be different.

You can get a Libevent version in two formats: as a string suitable for displaying to users, or as a 4-byte integer suitable for numerical comparison. The integer format uses the high byte for the major version, the second byte for the minor version, the third byte for the patch version, and the low byte to indicate release status (0 for release, nonzero for a development series after a given release).

Thus, the released Libevent 2.0.1-alpha has the version number of [02 00 01 00], or 0x02000100. A development versions between 2.0.1-alpha and 2.0.2-alpha might have a version number of [02 00 01 08], or 0x02000108.

Example: Compile-time checks

```c
#include <event2/event.h>

#if !defined(LIBEVENT_VERSION_NUMBER) || LIBEVENT_VERSION_NUMBER < 0x02000100
#error "This version of Libevent is not supported; Get 2.0.1-alpha or later."
#endif

int
make_sandwich(void)
{
        /* Let's suppose that Libevent 6.0.5 introduces a make-me-a
           sandwich function. */
#if LIBEVENT_VERSION_NUMBER >= 0x06000500
        evutil_make_me_a_sandwich();
        return 0;
#else
        return -1;
#endif
}
```

Example: Run-time checks

```c
#include <event2/event.h>
#include <string.h>

int
check_for_old_version(void)
{
    const char *v = event_get_version();
    /* This is a dumb way to do it, but it is the only thing that works
       before Libevent 2.0. */
    if (!strncmp(v, "0.", 2) ||
        !strncmp(v, "1.1", 3) ||
        !strncmp(v, "1.2", 3) ||
        !strncmp(v, "1.3", 3)) {

        printf("Your version of Libevent is very old. If you run into bugs,"
               " consider upgrading.\n");
        return -1;
    } else {
        printf("Running with Libevent version %s\n", v);
        return 0;
    }
}

int
check_version_match(void)
{
    ev_uint32_t v_compile, v_run;
    v_compile = LIBEVENT_VERSION_NUMBER;
    v_run = event_get_version_number();
    if ((v_compile & 0xffff0000) != (v_run & 0xffff0000)) {
        printf("Running with a Libevent version (%s) very different from the "
               "one we were built with (%s).\n", event_get_version(),
               LIBEVENT_VERSION);
        return -1;
    }
    return 0;
}
```

The macros and functions in this section are defined in <event2/event.h>. The event_get_version() function first appeared in Libevent 1.0c; the others first appeared in Libevent 2.0.1-alpha.

## Freeing global Libevent structures

Even when you've freed all the objects that you allocated with Libevent, there will be a few globally allocated structures left over. This isn't usually a problem: once the process exits, they will all get cleaned up anyway. But having these structures can confuse some debugging tools into thinking that Libevent is leaking resources. If you need to make sure that Libevent has released all internal library-global data structures, you can call:

Interface

```c
void libevent_global_shutdown(void);
```

This function doesn't free any structures that were returned to you by a Libevent function. If you want to free everything before exiting, you'll need to free all events, event_bases, bufferevents, and so on yourself.

Calling libevent_global_shutdown() will make other Libevent functions behave unpredictably; don't call it except as the last Libevent function your program invokes. One exception is that libevent_global_shutdown() is idempotent: it is okay to call it even if it has already been called.

This function is declared in <event2/event.h>. It was introduced in Libevent 2.1.1-alpha.

_Go back to [Index](README.md)_
