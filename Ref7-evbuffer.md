# Evbuffers: utility functionality for buffered IO

Libevent's evbuffer functionality implements a queue of bytes, optimized for adding data to the end and removing it from the front.

Evbuffers are meant to be generally useful for doing the "buffer" part of buffered network IO. They do not provide functions to schedule the IO or trigger the IO when it's ready: that is what bufferevents do.

The functions in this chapter are declared in event2/buffer.h unless otherwise noted.

## Creating or freeing an evbuffer

Interface

```c
struct evbuffer *evbuffer_new(void);
void evbuffer_free(struct evbuffer *buf);
```

These functions should be relatively clear: evbuffer_new() allocates and returns a new empty evbuffer, and evbuffer_free() deletes one and all of its contents.

These functions have existed since Libevent 0.8.

## Evbuffers and Thread-safety

Interface

```c
int evbuffer_enable_locking(struct evbuffer *buf, void *lock);
void evbuffer_lock(struct evbuffer *buf);
void evbuffer_unlock(struct evbuffer *buf);
```

By default, it is not safe to access an evbuffer from multiple threads at once. If you need to do this, you can call evbuffer_enable_locking() on the evbuffer. If its 'lock' argument is NULL, Libevent allocates a new lock using the lock creation function that was provided to evthread_set_lock_creation_callback. Otherwise, it uses the argument as the lock.

The evbuffer_lock() and evbuffer_unlock() functions acquire and release the lock on an evbuffer respectively. You can use them to make a set of operations atomic. If locking has not been enabled on the evbuffer, these functions do nothing.

(Note that you do not need to call evbuffer_lock() and evbuffer_unlock() around _individual_ operations: if locking is enabled on the evbuffer, individual operations are already atomic. You only need to lock the evbuffer manually when you have more than one operation that need to execute without another thread butting in.)

These functions were all introduced in Libevent 2.0.1-alpha.

## Inspecting an evbuffer

Interface

```c
size_t evbuffer_get_length(const struct evbuffer *buf);
```

This function returns the number of bytes stored in an evbuffer.

It was introduced in Libevent 2.0.1-alpha.

Interface

```c
size_t evbuffer_get_contiguous_space(const struct evbuffer *buf);
```

This function returns the number of bytes stored contiguously at the front of the evbuffer. The bytes in an evbuffer may be stored in multiple separate chunks of memory; this function returns the number of bytes currently stored in the _first_ chunk.

It was introduced in Libevent 2.0.1-alpha.

## Adding data to an evbuffer: basics

Interface

```c
int evbuffer_add(struct evbuffer *buf, const void *data, size_t datlen);
```

This function appends the 'datlen' bytes in 'data' to the end of 'buf'. It returns 0 on success, and -1 on failure.

Interface

```c
int evbuffer_add_printf(struct evbuffer *buf, const char *fmt, ...)
int evbuffer_add_vprintf(struct evbuffer *buf, const char *fmt, va_list ap);
```

These functions append formatted data to the end of 'buf'. The format argument and other remaining arguments are handled as if by the C library functions "printf" and "vprintf" respectively. The functions return the number of bytes appended.

Interface

```c
int evbuffer_expand(struct evbuffer *buf, size_t datlen);
```

This function alters the last chunk of memory in the buffer, or adds a new chunk, such that the buffer is now large enough to contain datlen bytes without any further allocations.

//BUILD: FUNCTIONBODY INC:../example_stubs/Ref7.h

Examples

```c
/* Here are two ways to add "Hello world 2.0.1" to a buffer. */
/* Directly: */
evbuffer_add(buf, "Hello world 2.0.1", 17);

/* Via printf: */
evbuffer_add_printf(buf, "Hello %s %d.%d.%d", "world", 2, 0, 1);
```

The evbuffer_add() and evbuffer_add_printf() functions were introduced in Libevent 0.8; evbuffer_expand() was in Libevent 0.9, and evbuffer_add_vprintf() first appeared in Libevent 1.1.

## Moving data from one evbuffer to another

For efficiency, Libevent has optimized functions for moving data from one evbuffer to another.

Interface

```c
int evbuffer_add_buffer(struct evbuffer *dst, struct evbuffer *src);
int evbuffer_remove_buffer(struct evbuffer *src, struct evbuffer *dst,
    size_t datlen);
```

The evbuffer_add_buffer() function moves all data from 'src' to the end of 'dst'. It returns 0 on success, -1 on failure.

The evbuffer_remove_buffer() function moves exactly 'datlen' bytes from 'src' to the end of 'dst', copying as little as possible. If there are fewer than 'datlen' bytes to move, it moves all the bytes. It returns the number of bytes moved.

We introduced evbuffer_add_buffer() in Libevent 0.8; evbuffer_remove_buffer() was new in Libevent 2.0.1-alpha.

## Adding data to the front of an evbuffer

Interface

```c
int evbuffer_prepend(struct evbuffer *buf, const void *data, size_t size);
int evbuffer_prepend_buffer(struct evbuffer *dst, struct evbuffer* src);
```

These functions behave as evbuffer_add() and evbuffer\_add_buffer() respectively, except that they move data to the _front_ of the destination buffer.

These functions should be used with caution, and never on an evbuffer shared with a bufferevent. They were new in Libevent 2.0.1-alpha.

## Rearranging the internal layout of an evbuffer

Sometimes you want to peek at the first N bytes of data in the front of an evbuffer, and see it as a contiguous array of bytes. To do this, you must first ensure that the front of the buffer really _is_ contiguous.

Interface

```c
unsigned char *evbuffer_pullup(struct evbuffer *buf, ev_ssize_t size);
```

The evbuffer_pullup() function "linearizes" the first 'size' bytes of 'buf', copying or moving them as needed to ensure that they are all contiguous and occupying the same chunk of memory. If 'size' is negative, the function linearizes the entire buffer. If 'size' is greater than the number of bytes in the buffer, the function returns NULL. Otherwise, evbuffer_pullup() returns a pointer to the first byte in buf.

Calling evbuffer_pullup() with a large size can be quite slow, since it potentially needs to copy the entire buffer's contents.

Example

```c
#include <event2/buffer.h>
#include <event2/util.h>

#include <string.h>

int parse_socks4(struct evbuffer *buf, ev_uint16_t *port, ev_uint32_t *addr)
{
    /* Let's parse the start of a SOCKS4 request!  The format is easy:
     * 1 byte of version, 1 byte of command, 2 bytes destport, 4 bytes of
     * destip. */
    unsigned char *mem;

    mem = evbuffer_pullup(buf, 8);

    if (mem == NULL) {
        /* Not enough data in the buffer */
        return 0;
    } else if (mem[0] != 4 || mem[1] != 1) {
        /* Unrecognized protocol or command */
        return -1;
    } else {
        memcpy(port, mem+2, 2);
        memcpy(addr, mem+4, 4);
        *port = ntohs(*port);
        *addr = ntohl(*addr);
        /* Actually remove the data from the buffer now that we know we
           like it. */
        evbuffer_drain(buf, 8);
        return 1;
    }
}
```

.Note
Calling evbuffer_pullup() with size equal to the value returned by evbuffer_get_contiguous_space() will not result in any data being copied or moved.

The evbuffer_pullup() function was new in Libevent 2.0.1-alpha: previous versions of Libevent always kept evbuffer data contiguous, regardless of the cost.

## Removing data from an evbuffer

Interface

```c
int evbuffer_drain(struct evbuffer *buf, size_t len);
int evbuffer_remove(struct evbuffer *buf, void *data, size_t datlen);
```

The evbuffer_remove() function copies and removes the first 'datlen' bytes from the front of 'buf' into the memory at 'data'. If there are fewer than 'datlen' bytes available, the function copies all the bytes there are. The return value is -1 on failure, and is otherwise the number of bytes copied.

The evbuffer_drain() function behaves as evbuffer_remove(), except that it does not copy the data: it just removes it from the front of the buffer. It returns 0 on success and -1 on failure.

Libevent 0.8 introduced evbuffer_drain(); evbuffer_remove() appeared in Libevent 0.9.

## Copying data out from an evbuffer

Sometimes you want to get a copy of the data at the start of a buffer without draining it. For example, you might want to see whether a complete record of some kind has arrived, without draining any of the data (as evbuffer_remove would do), or rearranging the buffer internally (as evbuffer_pullup() would do.)

Interface

```c
ev_ssize_t evbuffer_copyout(struct evbuffer *buf, void *data, size_t datlen);
ev_ssize_t evbuffer_copyout_from(struct evbuffer *buf,
     const struct evbuffer_ptr *pos,
     void *data_out, size_t datlen);
```

The evbuffer_copyout() behaves just like evbuffer_remove(), but does not drain any data from the buffer. That is, it copies the first 'datlen' bytes from the front of 'buf' into the memory at 'data'. If there are fewer than 'datlen' bytes available, the function copies all the bytes there are. The return value is -1 on failure, and is otherwise the number of bytes copied.

The evbuffer_copyout_from() function behaves like evbuffer_copyout(), but instead of copying bytes from the front of the buffer, it copies them beginning at the position provided in 'pos'. See "Searching within an evbuffer" below for information on the evbuffer_ptr structure.

If copying data from the buffer is too slow, use evbuffer_peek() instead.

Example

```c
#include <event2/buffer.h>
#include <event2/util.h>
#include <stdlib.h>
#include <stdlib.h>

int get_record(struct evbuffer *buf, size_t *size_out, char **record_out)
{
    /* Let's assume that we're speaking some protocol where records
       contain a 4-byte size field in network order, followed by that
       number of bytes. We will return 1 and set the 'out' fields if we
       have a whole record, return 0 if the record isn't here yet, and
       -1 on error. */
    size_t buffer_len = evbuffer_get_length(buf);
    ev_uint32_t record_len;
    char *record;

    if (buffer_len < 4)
       return 0; /* The size field hasn't arrived. */

   /* We use evbuffer_copyout here so that the size field will stay on
       the buffer for now. */
    evbuffer_copyout(buf, &record_len, 4);
    /* Convert len_buf into host order. */
    record_len = ntohl(record_len);
    if (buffer_len < record_len + 4)
        return 0; /* The record hasn't arrived */

    /* Okay, _now_ we can remove the record. */
    record = malloc(record_len);
    if (record == NULL)
        return -1;

    evbuffer_drain(buf, 4);
    evbuffer_remove(buf, record, record_len);

    *record_out = record;
    *size_out = record_len;
    return 1;
}
```

The evbuffer_copyout() function first appeared in Libevent 2.0.5-alpha; evbuffer_copyout_from() was added in Libevent 2.1.1-alpha.

## Line-oriented input

Interface

```c
enum evbuffer_eol_style {
        EVBUFFER_EOL_ANY,
        EVBUFFER_EOL_CRLF,
        EVBUFFER_EOL_CRLF_STRICT,
        EVBUFFER_EOL_LF,
        EVBUFFER_EOL_NUL
};
char *evbuffer_readln(struct evbuffer *buffer, size_t *n_read_out,
    enum evbuffer_eol_style eol_style);
```

Many Internet protocols use line-based formats. The evbuffer_readln() function extracts a line from the front of an evbuffer and returns it in a newly allocated NUL-terminated string. If 'n_read_out' is not NULL, *'n_read_out' is set to the number of bytes in the string returned. If there is not a whole line to read, the function returns NULL. The line terminator is not included in the copied string.

The evbuffer_readln() function understands 4 line termination formats:

EVBUFFER_EOL_LF::
    The end of a line is a single linefeed character. (This is also known as "\n". It is ASCII value is 0x0A.)

EVBUFFER_EOL_CRLF_STRICT::
    The end of a line is a single carriage return, followed by a single linefeed. (This is also known as "\r\n". The ASCII values are 0x0D 0x0A).

EVBUFFER_EOL_CRLF::
    The end of the line is an optional carriage return, followed by a linefeed. (In other words, it is either a "\r\n" or a "\n".) This format is useful in parsing text-based Internet protocols, since the standards generally prescribe a "\r\n" line-terminator, but nonconformant clients sometimes say just "\n".

EVBUFFER_EOL_ANY::
    The end of line is any sequence of any number of carriage return and linefeed characters. This format is not very useful; it exists mainly for backward compatibility.

EVBUFFER_EOL_NUL::
    The end of line is a single byte with the value 0 -- that is, an ASCII NUL.

(Note that if you used event_set_mem_functions() to override the default malloc, the string returned by evbuffer_readln will be allocated by the malloc-replacement you specified.)

//BUILD: FUNCTIONBODY INC:../example_stubs/Ref7.h

Example

```c
char *request_line;
size_t len;

request_line = evbuffer_readln(buf, &len, EVBUFFER_EOL_CRLF);
if (!request_line) {
    /* The first line has not arrived yet. */
} else {
    if (!strncmp(request_line, "HTTP/1.0 ", 9)) {
        /* HTTP 1.0 detected ... */
    }
    free(request_line);
}
```

The evbuffer_readln() interface is available in Libevent 1.4.14-stable and later. EVBUFFER_EOL_NUL was added in Libevent 2.1.1-alpha.

## Searching within an evbuffer

The evbuffer_ptr structure points to a location within an evbuffer, and contains data that you can use to iterate through an evbuffer.

Interface

```c
struct evbuffer_ptr {
        ev_ssize_t pos;
        struct {
                /* internal fields */
        } _internal;
};
```

The 'pos' field is the only public field; the others should not be used by user code. It indicates a position in the evbuffer as an offset from the start.

Interface

```c
struct evbuffer_ptr evbuffer_search(struct evbuffer *buffer,
    const char *what, size_t len, const struct evbuffer_ptr *start);
struct evbuffer_ptr evbuffer_search_range(struct evbuffer *buffer,
    const char *what, size_t len, const struct evbuffer_ptr *start,
    const struct evbuffer_ptr *end);
struct evbuffer_ptr evbuffer_search_eol(struct evbuffer *buffer,
    struct evbuffer_ptr *start, size_t *eol_len_out,
    enum evbuffer_eol_style eol_style);
```

The evbuffer_search() function scans the buffer for an occurrence of the 'len'-character string 'what'. It returns an evbuffer_ptr containing the position of the string, or -1 if the string was not found. If the 'start' argument is provided, it's the position at which the search should begin; otherwise, the search is from the start of the string.

The evbuffer_search_range() function behaves as evbuffer_search, except that it only considers occurrences of 'what' that occur before the evbuffer_ptr 'end'.

The evbuffer_search_eol() function detects line-endings as evbuffer_readln(), but instead of copying out the line, returns an evbuffer_ptr to the start of the end-of-line characters(s). If eol_len_out is non-NULL, it is set to the length of the EOL string.

Interface

```c
enum evbuffer_ptr_how {
        EVBUFFER_PTR_SET,
        EVBUFFER_PTR_ADD
};
int evbuffer_ptr_set(struct evbuffer *buffer, struct evbuffer_ptr *pos,
    size_t position, enum evbuffer_ptr_how how);
```

The evbuffer_ptr_set function manipulates the position of an evbuffer_ptr 'pos' within 'buffer'. If 'how' is EVBUFFER_PTR_SET, the pointer is moved to an absolute position 'position' within the buffer. If it is EVBUFFER_PTR_ADD, the pointer moves 'position' bytes forward. This function returns 0 on success and -1 on failure.

Example

```c
#include <event2/buffer.h>
#include <string.h>

/* Count the total occurrences of 'str' in 'buf'. */
int count_instances(struct evbuffer *buf, const char *str)
{
    size_t len = strlen(str);
    int total = 0;
    struct evbuffer_ptr p;

    if (!len)
        /* Don't try to count the occurrences of a 0-length string. */
        return -1;

    evbuffer_ptr_set(buf, &p, 0, EVBUFFER_PTR_SET);

    while (1) {
         p = evbuffer_search(buf, str, len, &p);
         if (p.pos < 0)
             break;
         total++;
         evbuffer_ptr_set(buf, &p, 1, EVBUFFER_PTR_ADD);
    }

    return total;
}
```

.WARNING

Any call that modifies an evbuffer or its layout invalidates all outstanding evbuffer_ptr values, and makes them unsafe to use.

These interfaces were new in Libevent 2.0.1-alpha.

## Inspecting data without copying it

Sometimes, you want to read data in an evbuffer without copying it out (as evbuffer_copyout() does), and without rearranging the evbuffer's internal memory (as evbuffer_pullup() does). Sometimes you might want to see data in the middle of an evbuffer.

You can do this with:

Interface

```c
struct evbuffer_iovec {
    void *iov_base;
    size_t iov_len;
};

int evbuffer_peek(struct evbuffer *buffer, ev_ssize_t len,
    struct evbuffer_ptr *start_at,
    struct evbuffer_iovec *vec_out, int n_vec);
```

When you call evbuffer_peek(), you give it an array of evbuffer_iovec structures in 'vec_out'. The array's length is 'n_vec'. It sets these structures so that each one contains a pointer to a chunk of the evbuffer's internal RAM ('iov_base'), and the length of memory that is set in that chunk.

If 'len' is less than 0, evbuffer_peek() tries to fill all of the evbuffer_iovec structs you have given it. Otherwise, it fills them until either they are all used, or at least 'len' bytes are visible. If the function could give you all the data you asked for, it returns the number of evbuffer_iovec structures that it actually used. Otherwise, it returns the number that it would need in order to give what you asked for.

When 'ptr' is NULL, evbuffer_peek() starts at the beginning of the buffer. Otherwise, it starts at the pointer given in 'ptr'.

//BUILD: FUNCTIONBODY INC:event2/buffer.h INC:../example_stubs/Ref7.h

Examples

```c
{
    /* Let's look at the first two chunks of buf, and write them to stderr. */
    int n, i;
    struct evbuffer_iovec v[2];
    n = evbuffer_peek(buf, -1, NULL, v, 2);
    for (i=0; i<n; ++i) { /* There might be less than two chunks available. */
        fwrite(v[i].iov_base, 1, v[i].iov_len, stderr);
    }
}

{
    /* Let's send the first 4906 bytes to stdout via write. */
    int n, i, r;
    struct evbuffer_iovec *v;
    size_t written = 0;

    /* determine how many chunks we need. */
    n = evbuffer_peek(buf, 4096, NULL, NULL, 0);
    /* Allocate space for the chunks. This would be a good time to use
       alloca() if you have it. */
    v = malloc(sizeof(struct evbuffer_iovec)*n);
    /* Actually fill up v. */
    n = evbuffer_peek(buf, 4096, NULL, v, n);
    for (i=0; i<n; ++i) {
        size_t len = v[i].iov_len;
        if (written + len > 4096)
            len = 4096 - written;
        r = write(1 /* stdout */, v[i].iov_base, len);
        if (r<=0)
            break;
        /* We keep track of the bytes written separately; if we don't,
           we may write more than 4096 bytes if the last chunk puts
           us over the limit. */
        written += len;
    }
    free(v);
}

{
    /* Let's get the first 16K of data after the first occurrence of the
       string "start\n", and pass it to a consume() function. */
    struct evbuffer_ptr ptr;
    struct evbuffer_iovec v[1];
    const char s[] = "start\n";
    int n_written;

    ptr = evbuffer_search(buf, s, strlen(s), NULL);
    if (ptr.pos == -1)
        return; /* no start string found. */

    /* Advance the pointer past the start string. */
    if (evbuffer_ptr_set(buf, &ptr, strlen(s), EVBUFFER_PTR_ADD) < 0)
        return; /* off the end of the string. */

    while (n_written < 16*1024) {
        /* Peek at a single chunk. */
        if (evbuffer_peek(buf, -1, &ptr, v, 1) < 1)
            break;
        /* Pass the data to some user-defined consume function */
        consume(v[0].iov_base, v[0].iov_len);
        n_written += v[0].iov_len;

        /* Advance the pointer so we see the next chunk next time. */
        if (evbuffer_ptr_set(buf, &ptr, v[0].iov_len, EVBUFFER_PTR_ADD)<0)
            break;
    }
}
```

.Notes

- Modifying the data pointed to by the evbuffer_iovec can result in undefined behavior.
- If any function is called that modifies the evbuffer, the pointers that evbuffer_peek() yields may become invalid.
- If your evbuffer could be used in multiple threads, make sure to lock it with evbuffer_lock() before you call evbuffer_peek(), and unlock it once you are done using the extents that evbuffer_peek() gave you.

This function is new in Libevent 2.0.2-alpha.

## Adding data to an evbuffer directly

Sometimes you want to insert data info an evbuffer directly, without first writing it into a character array and then copying it in with evbuffer_add(). There are an advanced pair of functions you can use to do this: evbuffer_reserve_space() and evbuffer_commit_space(). As with evbuffer_peek(), these functions use the evbuffer_iovec structure to provide direct access to memory inside the evbuffer.

Interface

```c
int evbuffer_reserve_space(struct evbuffer *buf, ev_ssize_t size,
    struct evbuffer_iovec *vec, int n_vecs);
int evbuffer_commit_space(struct evbuffer *buf,
    struct evbuffer_iovec *vec, int n_vecs);
```

The evbuffer_reserve_space() function gives you pointers to space inside the evbuffer. It expands the buffer as necessary to give you at least 'size' bytes. The pointers to these extents, and their lengths, will be stored in the array of vectors you pass in with 'vec'; 'n_vec' is the length of this array.

The value of 'n_vec' must be at least 1. If you provide only one vector, then Libevent will ensure that you have all the contiguous space you requested in a single extent, but it may have to rearrange the buffer or waste memory in order to do so. For better performance, provide at least 2 vectors. The function returns the number of provided vectors that it needed for the space you requested.

The data that you write into these vectors is not part of the buffer until you call evbuffer_commit_space(), which actually makes the data you wrote count as being in the buffer. If you want to commit less space than you asked for, you can decrease the iov_len field in any of the evbuffer_iovec structures you were given. You can also pass back fewer vectors than you were given. The evbuffer_commit_space() function returns 0 on success and -1 on failure.

.Notes and Caveats

- Calling any function that rearranges the evbuffer or adds data to it evbuffer will invalidate the pointers you got from evbuffer_reserve_space().
- In the current implementation, evbuffer_reserve_space() never uses more than two vectors, no matter how many the user supplies. This may change in a future release.
- It is safe to call evbuffer_reserve_space() any number of times.
- If your evbuffer could be used in multiple threads, make sure to lock it with evbuffer_lock() before you call evbuffer_reserve_space(), and unlock it once you commit.

//BUILD: FUNCTIONBODY INC:../example_stubs/Ref7.h

Example

```c
/* Suppose we want to fill a buffer with 2048 bytes of output from a
   generate_data() function, without copying. */
struct evbuffer_iovec v[2];
int n, i;
size_t n_to_add = 2048;

/* Reserve 2048 bytes.*/
n = evbuffer_reserve_space(buf, n_to_add, v, 2);
if (n<=0)
   return; /* Unable to reserve the space for some reason. */

for (i=0; i<n && n_to_add > 0; ++i) {
   size_t len = v[i].iov_len;
   if (len > n_to_add) /* Don't write more than n_to_add bytes. */
      len = n_to_add;
   if (generate_data(v[i].iov_base, len) < 0) {
      /* If there was a problem during data generation, we can just stop
         here; no data will be committed to the buffer. */
      return;
   }
   /* Set iov_len to the number of bytes we actually wrote, so we
      don't commit too much. */
   v[i].iov_len = len;
}

/* We commit the space here. Note that we give it 'i' (the number of
   vectors we actually used) rather than 'n' (the number of vectors we
   had available. */
if (evbuffer_commit_space(buf, v, i) < 0)
   return; /* Error committing */
```

//BUILD: FUNCTIONBODY INC:../example_stubs/Ref7.h

.Bad Examples

```c
/* Here are some mistakes you can make with evbuffer_reserve().
   DO NOT IMITATE THIS CODE. */
struct evbuffer_iovec v[2];

{
  /* Do not use the pointers from evbuffer_reserve_space() after
     calling any functions that modify the buffer. */
  evbuffer_reserve_space(buf, 1024, v, 2);
  evbuffer_add(buf, "X", 1);
  /* WRONG: This next line won't work if evbuffer_add needed to rearrange
     the buffer's contents. It might even crash your program. Instead,
     you add the data before calling evbuffer_reserve_space. */
  memset(v[0].iov_base, 'Y', v[0].iov_len-1);
  evbuffer_commit_space(buf, v, 1);
}

{
  /* Do not modify the iov_base pointers. */
  const char *data = "Here is some data";
  evbuffer_reserve_space(buf, strlen(data), v, 1);
  /* WRONG: The next line will not do what you want. Instead, you
     should _copy_ the contents of data into v[0].iov_base. */
  v[0].iov_base = (char*) data;
  v[0].iov_len = strlen(data);
  /* In this case, evbuffer_commit_space might give an error if you're
     lucky */
  evbuffer_commit_space(buf, v, 1);
}
```

These functions have existed with their present interfaces since Libevent 2.0.2-alpha.

## Network IO with evbuffers

The most common use case for evbuffers in Libevent is network IO. The interface for performing network IO on an evbuffer is:

Interface

```c
int evbuffer_write(struct evbuffer *buffer, evutil_socket_t fd);
int evbuffer_write_atmost(struct evbuffer *buffer, evutil_socket_t fd,
        ev_ssize_t howmuch);
int evbuffer_read(struct evbuffer *buffer, evutil_socket_t fd, int howmuch);
```

The evbuffer_read() function reads up to 'howmuch' bytes from the socket 'fd' onto the end of 'buffer'. It returns a number of bytes read on success, 0 on EOF, and -1 on an error. Note that the error may indicate that a nonblocking operation would not succeed; you need to check the error code for EAGAIN (or WSAEWOULDBLOCK on Windows). If 'howmuch' is negative, evbuffer_read() tries to guess how much to read itself.

The evbuffer_write_atmost() function tries to write up to 'howmuch' bytes from the front of 'buffer' onto the socket 'fd'. It returns a number of bytes written on success, and -1 on failure. As with evbuffer_read(), you need to check the error code to see whether the error is real, or just indicates that nonblocking IO could not be completed immediately. If you give a negative value for 'howmuch', we try to write the entire contents of the buffer.

Calling evbuffer_write() is the same as calling evbuffer_write_atmost() with a negative 'howmuch' argument: it attempts to flush as much of the buffer as it can.

On Unix, these functions should work on any file descriptor that supports read and write. On Windows, only sockets are supported.

Note that when you are using bufferevents, you do not need to call these IO functions; the bufferevents code does it for you.

The evbuffer_write_atmost() function was introduced in Libevent 2.0.1-alpha.

## Evbuffers and callbacks

Users of evbuffers frequently want to know when data is added to or removed from an evbuffer. To support this, Libevent provides a generic evbuffer callback mechanism.

Interface

```c
struct evbuffer_cb_info {
        size_t orig_size;
        size_t n_added;
        size_t n_deleted;
};

typedef void (*evbuffer_cb_func)(struct evbuffer *buffer,
    const struct evbuffer_cb_info *info, void *arg);
```

An evbuffer callback is invoked whenever data is added to or removed from the evbuffer. It receives the buffer, a pointer to an evbuffer_cb_info structure, and a user-supplied argument. The evbuffer_cb_info structure's orig_size field records how many bytes there were on the buffer before its size changed; its n_added field records how many bytes were added to the buffer, and its n_deleted field records how many bytes were removed.

Interface

```c
struct evbuffer_cb_entry;
struct evbuffer_cb_entry *evbuffer_add_cb(struct evbuffer *buffer,
    evbuffer_cb_func cb, void *cbarg);
```

The evbuffer_add_cb() function adds a callback to an evbuffer, and returns an opaque pointer that can later be used to refer to this particular callback instance. The 'cb' argument is the function that will be invoked, and the 'cbarg' is the user-supplied pointer to pass to the function.

You can have multiple callbacks set on a single evbuffer. Adding a new callback does not remove old callbacks.

Example

```c
#include <event2/buffer.h>
#include <stdio.h>
#include <stdlib.h>

/* Here's a callback that remembers how many bytes we have drained in
   total from the buffer, and prints a dot every time we hit a
   megabyte. */
struct total_processed {
    size_t n;
};
void count_megabytes_cb(struct evbuffer *buffer,
    const struct evbuffer_cb_info *info, void *arg)
{
    struct total_processed *tp = arg;
    size_t old_n = tp->n;
    int megabytes, i;
    tp->n += info->n_deleted;
    megabytes = ((tp->n) >> 20) - (old_n >> 20);
    for (i=0; i<megabytes; ++i)
        putc('.', stdout);
}

void operation_with_counted_bytes(void)
{
    struct total_processed *tp = malloc(sizeof(*tp));
    struct evbuffer *buf = evbuffer_new();
    tp->n = 0;
    evbuffer_add_cb(buf, count_megabytes_cb, tp);

    /* Use the evbuffer for a while. When we're done: */
    evbuffer_free(buf);
    free(tp);
}
```

Note in passing that freeing a nonempty evbuffer does not count as draining data from it, and that freeing an evbuffer does not free the user-supplied data pointer for its callbacks.

If you don't want a callback to be permanently active on a buffer, you can _remove_ it (to make it gone for good), or disable it (to turn it off for a while):

Interface

```c
int evbuffer_remove_cb_entry(struct evbuffer *buffer,
    struct evbuffer_cb_entry *ent);
int evbuffer_remove_cb(struct evbuffer *buffer, evbuffer_cb_func cb,
    void *cbarg);

#define EVBUFFER_CB_ENABLED 1
int evbuffer_cb_set_flags(struct evbuffer *buffer,
                          struct evbuffer_cb_entry *cb,
                          ev_uint32_t flags);
int evbuffer_cb_clear_flags(struct evbuffer *buffer,
                          struct evbuffer_cb_entry *cb,
                          ev_uint32_t flags);
```

You can remove a callback either by the evbuffer_cb_entry you got when you added it, or by the callback and pointer you used. The evbuffer_remove_cb() functions return 0 on success and -1 on failure.

The evbuffer_cb_set_flags() function and the evbuffer_cb_clear_flags() function make a given flag be set or cleared on a given callback respectively. Right now, only one user-visible flag is supported: 'EVBUFFER_CB_ENABLED'. The flag is set by default. When it is cleared, modifications to the evbuffer do not cause this callback to get invoked.

Interface

```c
int evbuffer_defer_callbacks(struct evbuffer *buffer, struct event_base *base);
```

As with bufferevent callbacks, you can cause evbuffer callbacks to not run immediately when the evbuffer is changed, but rather to be 'deferred' and run as part of the event loop of a given event base. This can be helpful if you have multiple evbuffers whose callbacks potentially cause data to be added and removed from one another, and you want to avoid smashing the stack.

If an evbuffer's callbacks are deferred, then when they are finally invoked, they may summarize the results for multiple operations.

Like bufferevents, evbuffers are internally reference-counted, so that it is safe to free an evbuffer even if it has deferred callbacks that have not yet executed.

This entire callback system was new in Libevent 2.0.1-alpha. The evbuffer_cb_(set|clear)_flags() functions have existed with their present interfaces since 2.0.2-alpha.

## Avoiding data copies with evbuffer-based IO

Really fast network programming often calls for doing as few data copies as possible. Libevent provides some mechanisms to help out with this.

Interface

```c
typedef void (*evbuffer_ref_cleanup_cb)(const void *data,
    size_t datalen, void *extra);

int evbuffer_add_reference(struct evbuffer *outbuf,
    const void *data, size_t datlen,
    evbuffer_ref_cleanup_cb cleanupfn, void *extra);
```

This function adds a piece of data to the end of an evbuffer by reference. No copy is performed: instead, the evbuffer just stores a pointer to the 'datlen' bytes stored at 'data'. Therefore, the pointer must remain valid for as long as the evbuffer is using it. When the evbuffer no longer needs data, it will call the provided "cleanupfn" function with the provided "data" pointer, "datlen" value, and "extra" pointer as arguments. This function returns 0 on success, -1 on failure.

Example

```c
#include <event2/buffer.h>
#include <stdlib.h>
#include <string.h>

/* In this example, we have a bunch of evbuffers that we want to use to
   spool a one-megabyte resource out to the network. We do this
   without keeping any more copies of the resource in memory than
   necessary. */

#define HUGE_RESOURCE_SIZE (1024*1024)
struct huge_resource {
    /* We keep a count of the references that exist to this structure,
       so that we know when we can free it. */
    int reference_count;
    char data[HUGE_RESOURCE_SIZE];
};

struct huge_resource *new_resource(void) {
    struct huge_resource *hr = malloc(sizeof(struct huge_resource));
    hr->reference_count = 1;
    /* Here we should fill hr->data with something. In real life,
       we'd probably load something or do a complex calculation.
       Here, we'll just fill it with EEs. */
    memset(hr->data, 0xEE, sizeof(hr->data));
    return hr;
}

void free_resource(struct huge_resource *hr) {
    --hr->reference_count;
    if (hr->reference_count == 0)
        free(hr);
}

static void cleanup(const void *data, size_t len, void *arg) {
    free_resource(arg);
}

/* This is the function that actually adds the resource to the
   buffer. */
void spool_resource_to_evbuffer(struct evbuffer *buf,
    struct huge_resource *hr)
{
    ++hr->reference_count;
    evbuffer_add_reference(buf, hr->data, HUGE_RESOURCE_SIZE,
        cleanup, hr);
}

```

The evbuffer_add_reference() function has had is present interface since 2.0.2-alpha.

## Adding a file to an evbuffer

Some operating systems provide ways to write files to the network without ever copying the data to userspace. You can access these mechanisms, where available, with the simple interface:

Interface

```c
int evbuffer_add_file(struct evbuffer *output, int fd, ev_off_t offset,
    size_t length);
```

The evbuffer_add_file() function assumes that it has an open file descriptor (not a socket, for once!) 'fd' that is available for reading. It adds 'length' bytes from the file, starting at position 'offset', to the end of 'output'. It returns 0 on success, or -1 on failure.

.WARNING

In Libevent 2.0.x, the only reliable thing to do with data added this way was to send it to the network with evbuffer_write*(), drain it with evbuffer_drain(), or move it to another evbuffer with evbuffer_*_buffer(). You couldn't reliably extract it from the buffer with evbuffer_remove(), linearize it with evbuffer_pullup(), and so on. Libevent 2.1.x tries to fix this limitation.

If your operating system supports splice() or sendfile(), Libevent will use it to send data from 'fd' to the network directly when call evbuffer_write(), without copying the data to user RAM at all. If splice/sendfile don't exist, but you have mmap(), Libevent will mmap the file, and your kernel can hopefully figure out that it never needs to copy the data to userspace. Otherwise, Libevent will just read the data from disk into RAM.

The file descriptor will be closed after the data is flushed from the evbuffer, or when the evbuffer is freed. If that's not what you want, or if you want finer-grained control over the file, see the file_segment functionality below.

This function was introduced in Libevent 2.0.1-alpha.

## Fine-grained control with file segments

The evbuffer_add_file() interface is inefficient for adding the same file more than once, since it takes ownership of the file.

.Interface

```c
struct evbuffer_file_segment;

struct evbuffer_file_segment *evbuffer_file_segment_new(
    int fd, ev_off_t offset, ev_off_t length, unsigned flags);
void evbuffer_file_segment_free(struct evbuffer_file_segment *seg);
int evbuffer_add_file_segment(struct evbuffer *buf,
    struct evbuffer_file_segment *seg, ev_off_t offset, ev_off_t length);
```

The evbuffer_file_segment_new() function creates and returns a new evbuffer_file_segment object to represent a piece of the underlying file stored in 'fd' that begins at 'offset' and contains 'length' bytes. On error, it return NULL.

File segments are implemented with sendfile, splice, mmap, CreateFileMapping, or malloc()-and-read(), as appropriate. They're created using the most lightweight supported mechanism, and transition to a heavier-weight mechanism as needed. (For example, if your OS supports sendfile and mmap, then a file segment can be implemented using only sendfile, until you try to actually inspect its contents. At that point, it needs to be mmap()ed.)  You can control the fine-grained behavior of a file segment with these flags:

EVBUF_FS_CLOSE_ON_FREE::
    If this flag is set, freeing the file segment with evbuffer_file_segment_free() will close the underlying file.

EVBUF_FS_DISABLE_MMAP::
    If this flag is set, the file_segment will never use a mapped-memory style backend (CreateFileMapping, mmap) for this file, even if that would be appropriate.

EVBUF_FS_DISABLE_SENDFILE::
    If this flag is set, the file_segment will never use a sendfile-style backend (sendfile, splice) for this file, even if that would be appropriate.

EVBUF_FS_DISABLE_LOCKING::
    If this flag is set, no locks are allocated for the file segment: it won't be safe to use it in any way where it can be seen by multiple threads.

Once you have an evbuffer_file_segment, you can add some or all of it to an evbuffer using evbuffer_add_file_segment(). The 'offset' argument here refers to an offset within the file segment, not to an offset within the file itself.

When you no longer want to use a file segment, you can free it with evbuffer_file_segment_free(). The actual storage won't be released until no evbuffer any longer holds a reference to a piece of the file segment.

Interface

```c
typedef void (*evbuffer_file_segment_cleanup_cb)(
    struct evbuffer_file_segment const *seg, int flags, void *arg);

void evbuffer_file_segment_add_cleanup_cb(struct evbuffer_file_segment *seg,
    evbuffer_file_segment_cleanup_cb cb, void *arg);
```

You can add a callback function to a file segment that will be invoked when the final reference to the file segment has been released and the file segment is about to get freed. This callback *must not* attempt to revivify the file segment, add it to any buffers, or so on.

These file-segment functions first appeared in Libevent 2.1.1-alpha; evbuffer_file_segment_add_cleanup_cb() was added in 2.1.2-alpha.

## Adding an evbuffer to another by reference

You can also add one evbuffer's to another by reference: rather than removing the contents of one buffer and adding them to another, you give one evbuffer a reference to another, and it behaves as though you had copied all the bytes in.

.Interface

```c
int evbuffer_add_buffer_reference(struct evbuffer *outbuf,
    struct evbuffer *inbuf);
```

The evbuffer_add_buffer_reference() function behaves as though you had copied all the data from 'outbuf' to 'inbuf', but does not perform any unnecessary copies. It returns 0 if successful and -1 on failure.

Note that subsequent changes to the contents of 'inbuf' are not reflected in 'outbuf': this function adds the current contents of the evbuffer by reference, not the evbuffer itself.

Note also that you cannot nest buffer references: a buffer that has already been the 'outbuf' of one evbuffer_add_buffer_reference call cannot be the 'inbuf' of another.

This function was introduced in Libevent 2.1.1-alpha.

## Making an evbuffer add- or remove-only

Interface

```c
int evbuffer_freeze(struct evbuffer *buf, int at_front);
int evbuffer_unfreeze(struct evbuffer *buf, int at_front);
```

You can use these functions to temporarily disable changes to the front or end of an evbuffer. The bufferevent code uses them internally to prevent accidental modifications to the front of an output buffer, or the end of an input buffer.

The evbuffer_freeze() functions were introduced in Libevent 2.0.1-alpha.

## Obsolete evbuffer functions

The evbuffer interface changed a lot in Libevent 2.0. Before then, every evbuffers was implemented as a contiguous chunk of RAM, which made access very inefficient.

The event.h header used to expose the internals of struct evbuffer. These are no longer available; they changed too much between 1.4 and 2.0 for any code that relied on them to work.

To access the number of bytes in an evbuffer, there was an EVBUFFER_LENGTH() macro. The actual data was available with EVBUFFER_DATA(). These are both available in event2/buffer_compat.h. Watch out, though: EVBUFFER_DATA(b) is an alias for evbuffer_pullup(b, -1), which can be very expensive.

Some other deprecated interfaces are:

.Deprecated Interface

```c
char *evbuffer_readline(struct evbuffer *buffer);
unsigned char *evbuffer_find(struct evbuffer *buffer,
    const unsigned char *what, size_t len);
```

The evbuffer_readline() function worked like the current evbuffer_readln(buffer, NULL, EVBUFFER_EOL_ANY).

The evbuffer_find() function would search for the first occurrence of a string in a buffer, and return a pointer to it. Unlike evbuffer_search(), it could only find the first string. To stay compatible with old code that uses this function, it now linearizes the entire buffer up to the end of the located string.

The callback interface was different too:

.Deprecated Interface

```c
typedef void (*evbuffer_cb)(struct evbuffer *buffer,
    size_t old_len, size_t new_len, void *arg);
void evbuffer_setcb(struct evbuffer *buffer, evbuffer_cb cb, void *cbarg);
```

An evbuffer could only have one callback set at a time, so setting a new callback would disable the previous callback, and setting a callback of NULL was the preferred way to disable a callbacks.

Instead of getting an evbuffer_cb_info_structure, the function was called with the old and new lengths of the evbuffer. Thus, if old_len was greater than new_len, data was drained. If new_len was greater than old_len, data was added. It was not possible to defer callbacks, and so adds and deletes were never batched into a single callback invocation.

The obsolete functions here are still available in event2/buffer_compat.h.

_Go back to [Index](README.md)_
