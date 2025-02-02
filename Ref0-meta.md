# Preliminaries

## Libevent from 10,000 feet

Libevent is a library for writing fast portable nonblocking IO. Its design goals are:

### Design goals

#### Portability

A program written using Libevent should work across all the platforms Libevent supports.  Even when there is no really _good_ way to do nonblocking IO, Libevent should support the so-so ways, so that your program can run in restricted environments.

#### Speed

Libevent tries to use the fastest available nonblocking IO implementations on each platform, and not to introduce much overhead as it does so.

#### Scalability

Libevent is designed to work well even with programs that need to have tens of thousands of active sockets.

#### Convenience

Whenever possible, the most natural way to write a program with Libevent should be the stable, portable way.

### Components

Libevent is divided into the following components:

#### evutil

Generic functionality to abstract out the differences between different platforms' networking implementations.

#### event and event_base

This is the heart of Libevent.  It provides an abstract API to the various platform-specific, event-based nonblocking IO backends. It can let you know when sockets are ready to read or write, do basic timeout functionality, and detect OS signals.

#### bufferevent

These functions provide a more convenient wrapper around Libevent's event-based core.  They let your application request buffered reads and writes, and rather than informing you when sockets are ready to do, they let you know when IO has actually occurred.

The bufferevent interface also has multiple backends, so that it can take advantage of systems that provide faster ways to do nonblocking IO, such as the Windows IOCP API.

#### evbuffer

This module implements the buffers underlying bufferevents, and provides functions for efficient and/or convenient access.

#### evhttp

A simple HTTP client/server implementation.

#### evdns

A simple DNS client/server implementation.

#### evrpc

A simple RPC implementation.

## The Libraries

When Libevent is built, by default it installs the following libraries:

### libevent_core

All core event and buffer functionality. This library contains all the event_base, evbuffer, bufferevent, and utility functions.

### libevent_extra

This library defines protocol-specific functionality that you may or may not want for your application, including HTTP, DNS, and RPC.

### libevent

This library exists for historical reasons; it contains the contents of both libevent_core and libevent_extra. You shouldn't use it; it may go away in a future version of Libevent.

The following libraries are installed only on some platforms:

### libevent_pthreads

This library adds threading and locking implementations based on the pthreads portable threading library.  It is separated from libevent\_core so that you don't need to link against pthreads to use Libevent unless you are _actually_ using Libevent in a multithreaded way.

### libevent_openssl

This library provides support for encrypted communications using bufferevents and the OpenSSL library.  It is separated from libevent\_core so that you don't need to link against OpenSSL to use Libevent unless you are _actually_ using encrypted connections.

## The Headers

All current public Libevent headers are installed under the 'event2' directory. Headers fall into three broad classes:

### API headers

An API header is one that defines current public interfaces to Libevent. These headers have no special suffix.

### Compatibility headers

A compatibility header includes definitions for deprecated functions. You shouldn't include it unless you're porting a program from an older version of Libevent.

### Structure headers

These headers define structures with relatively volatile layouts. Some of these are exposed in case you need fast access to structure component; some are exposed for historical reasons. Relying on any of the structures in headers directly can break your program's binary compatibility with other versions of Libevent, sometimes in hard-to-debug ways. These headers have the suffix "_struct.h"

(There are also older versions of the Libevent headers without the 'event2' directory.  See "If you have to work with an old version of Libevent" below.)

## If you have to work with an old version of Libevent

Libevent 2.0 has revised its APIs to be generally more rational and less error-prone.  If it's possible, you should write new programs to use the Libevent 2.0 APIs.  But sometimes you might need to work with the older APIs, either to update an existing application, or to support an environment where for some reason you can't install Libevent 2.0 or later.

Older versions of Libevent had fewer headers, and did not install them under "event2":

| OLD HEADER... | ...REPLACED BY CURRENT HEADERS |
| --- | --- |
| event.h    | event2/event*.h, event2/buffer*.h event2/bufferevent*.h, event2/tag*.h |
| evdns.h    | event2/dns*.h |
| evhttp.h   | event2/http*.h |
| evrpc.h    | event2/rpc*.h |
| evutil.h   | event2/util*.h |

In Libevent 2.0 and later, the old headers still exist as wrappers for the new headers.

Some other notes on working with older versions:

- Before 1.4, there was only one library, "libevent", that contained the functionality currently split into libevent_core and libevent_extra.
- Before 2.0, there was no support for locking; Libevent could be thread-safe, but only if you made sure to never use the same structure from two threads at the same time.

Individual sections below will discuss the obsolete APIs that you might encounter for specific areas of the codebase.

## Notes on version status

Versions of Libevent before 1.4.7 or so should be considered totally obsolete.  Versions of Libevent before 1.3e or so should be considered hopelessly bug-ridden.

(Also, please don't send the Libevent maintainers any new features for 1.4.x or earlier---it's supposed to stay as a stable release. And if you encounter a bug in 1.3x or earlier, please make sure that it still exists in the latest stable version before you report it: subsequent releases have happened for a reason.)

_Go back to [Index](README.md)_
