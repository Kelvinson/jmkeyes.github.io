---
layout: post
title: Ring Buffers
date: 2011-04-09 23:32:54.000000000 -07:00
status: published
type: post
---
I built a general purpose ring buffer for a project I have been working 
on. I've seen a bunch of implementations on Stack Overflow and 
throughout some hobbyist forums for embedded devices, but none that I 
saw were very well documented, portable, or usable. It's fairly simple 
and compiles under fairly strict compiler settings.

I wont go into details about why ring buffers are a Good Thing™ 
because anyone looking to use one should already know why it's useful. 
This code is released under the ISC license, so if you find it useful 
enough to include in a project, please give credit where it's due. I've 
compiled this using GCC 4.5.2 with the flags `-O2 -Wall -Wextra -Werror 
-std=c99 -pedantic`. 

There are four functions for reading and writing data to and from the 
ring buffer. They're both fairly self explanatory: the ones that 
operate on memory are similar to the standard memcpy functions and the 
ones that operate on file descriptors do the same for read/write. They 
almost completely duplicate the semantics of their respective standard 
functions, except that they also provide wrap-around reading and 
writing to the circular buffer when necessary. Also, the ring buffer 
always allocates a power-of-two greater than or equal to the size 
requested during creation.

{% highlight c %}
/* ringbuffer.h */
#ifndef RINGBUFFER_H
#define RINGBUFFER_H

struct ringbuffer {
    unsigned long    size;         /**<< Allocated buffer size. */
    unsigned long    read_offset;  /**<< Buffer read index. */
    unsigned long    write_offset; /**<< Buffer write index. */
    unsigned char * storage;       /**<< Data storage area. */
};

int ringbuffer_create(struct ringbuffer **, unsigned long);
int ringbuffer_destroy(struct ringbuffer *);

int ringbuffer_flush(struct ringbuffer *, char);

long ringbuffer_fill_count(struct ringbuffer *);
long ringbuffer_free_count(struct ringbuffer *);

long ringbuffer_write_memory(struct ringbuffer *, void *, unsigned long);
long ringbuffer_read_memory(struct ringbuffer *, void *, unsigned long);

long ringbuffer_write_fd(struct ringbuffer *, int, unsigned long);
long ringbuffer_read_fd(struct ringbuffer *, int, unsigned long);

#endif /* RINGBUFFER_H */
{% endhighlight %}

{% highlight c %}
/* ringbuffer.c */

#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/uio.h>
#include <errno.h>

#include "ringbuffer.h"

static unsigned long next_power_of_two(unsigned long size) {
    size--;
    size |= (size >> 1);
    size |= (size >> 2);
    size |= (size >> 4);
    size |= (size >> 8);
    size |= (size >> 16);
    size |= (size >> 32);
    size++;

    return size;
}

/** int ringbuffer_create(struct ringbuffer **rb, unsigned long size);
 *
 * Allocates memory for a ring buffer instance at @a rb. The size is
 * rounded up to the nearest power-of-two of @a size. Its contents
 * are uninitialized by default.
 *
 * @param rb The address of a pointer to a ring buffer structure.
 * @param size The size of the allocated storage for this buffer.
 * @return Returns  0 on success and -1 on error, setting errno.
 */
int ringbuffer_create(struct ringbuffer **rb, unsigned long size) {
    if((*rb = malloc(sizeof **rb)) == NULL) {
        errno = ENOMEM; /* Allocation failed. */
        return -1;
    }

    size = next_power_of_two(size);

    (*rb)->size = size;

    if(((*rb)->storage = malloc(size)) == NULL) {
        errno = ENOMEM; /* Allocation failed. */
        return -1;
    }

    return 0;
}

/** int ringbuffer_destroy(struct ringbuffer *rb);
 *
 * Destroys the ringbuffer located at @a rb.
 * All allocated memory is freed.
 *
 * @param rb The address of a ring buffer structure.
 * @return Returns  0 on success and -1 on error, setting errno.
 */
int ringbuffer_destroy(struct ringbuffer *rb) {
    if(rb == NULL) {
        errno = EINVAL; /* Invalid argument. */
        return -1;
    }

    free(rb->storage);
    free(rb);

    return 0;
}

/** int ringbuffer_flush(struct ringbuffer *rb, char fill);
 *
 * Resets a ring buffer at @a rb to its initial state, filling its
 * contents with the repeated byte @a fill.
 *
 * @param rb The address of a ring buffer structure.
 * @param fill The byte to fill the storage area with.
 * @return Returns  0 on success and -1 on error, setting errno.
 */
int ringbuffer_flush(struct ringbuffer *rb, char fill) {
    if(rb == NULL) {
        errno = EINVAL; /* Invalid Argument. */
        return -1;
    }

    memset(&rb->storage[0], fill, rb->size);

    rb->read_offset = 0;
    rb->write_offset = 0;

    return 0;
}

/** long ringbuffer_fill_count(struct ringbuffer *rb);
 *
 * Returns the total number of bytes fill in the ring buffer
 * at @a rb. When the ring buffer is full this returns zero.
 *
 * @param rb The address of a ring buffer structure.
 * @return Returns 0 on success and -1 on error, setting errno.
 */
long ringbuffer_fill_count(struct ringbuffer *rb) {
    long bytes = 0;

    if(rb == NULL) {
        errno = EINVAL; /* Invalid argument. */
        return -1;
    }

    if(rb->write_offset >= rb->read_offset) {
        bytes = (rb->write_offset - rb->read_offset);
    } else if(rb->write_offset < rb->read_offset) {
        bytes = (rb->write_offset + rb->size - rb->read_offset);
    }

    return bytes;
}

/** long ringbuffer_free_count(struct ringbuffer *rb);
 *
 * Returns the total number of bytes free in the ring buffer
 * at @a rb. When the ring buffer is empty, this returns zero.
 *
 * @param rb The address of a ring buffer structure.
 * @return Returns 0 on success and -1 on error, setting errno.
 */
long ringbuffer_free_count(struct ringbuffer *rb) {
    if(rb == NULL) {
        errno = EINVAL; /* Invalid argument. */
        return -1;
    }

    return (rb->size - ringbuffer_fill_count(rb));
}

/** long ringbuffer_write_memory(struct ringbuffer *rb, void *ptr, long amount);
 *
 * Copies the memory located in a ring buffer at @a rb to the buffer
 * @a ptr, attempting to copy @a amount bytes. The caller should
 * ensure there is enough memory available at @a ptr before calling
 * this function.
 *
 * @param rb The address of a ring buffer structure.
 * @param ptr The address to copy data into.
 * @param amount The total amount of data to copy.
 * @return Returns the number of bytes copied on success or -1 on error, setting errno.
 */
long ringbuffer_write_memory(struct ringbuffer *rb, void *ptr, long amount) {
    unsigned long idx = 0, end = 0, max = 0;
    unsigned char *buffer = ptr;

    if(rb == NULL) {
        errno = EINVAL; /* Invalid argument. */
        return -1;
    }

    if(amount > (long) rb->size) {
        errno = ENOMEM; /* Not enough memory */
        return -1;
    }

    if(amount > ringbuffer_fill_count(rb)) {
        amount = ringbuffer_fill_count(rb);
    }

    if (buffer != NULL) {
        idx = rb->read_offset;
        end = (rb->read_offset + amount) & (rb->size - 1);
        max = rb->size;

        if(idx < end) {
            memcpy(buffer, &rb->storage[idx], end - idx);
        } else if(idx > end) {
            memcpy(buffer, &rb->storage[idx], max - idx);
            memcpy(buffer + (max - idx), &rb->storage[0], end);
        }
    }

    rb->read_offset += amount;
    rb->read_offset &= rb->size - 1;

    return amount;
}

/** long ringbuffer_read_memory(struct ringbuffer *rb, void *ptr, long amount);
 *
 * Copies the data located at @a ptr into a ring buffer at @a rb,
 * attempting to copy @a amount bytes. If there is not enough
 * memory in the ring buffer to store the total of @a amount bytes,
 * then it copies as much as possible without overwriting other
 * stored data.
 *
 * @param rb The address of a ring buffer structure.
 * @param ptr The address to copy data from.
 * @param amount The total amount of data to copy.
 * @return Returns the number of bytes copied on success or -1 on error, setting errno.
 */
long ringbuffer_read_memory(struct ringbuffer *rb, void *ptr, long amount) {
    unsigned long idx = 0, end = 0, max = 0;
    unsigned char *buffer = ptr;

    if(rb == NULL) {
        errno = EINVAL; /* Invalid argument. */
        return -1;
    }

    if(amount > (long) rb->size) {
        errno = ENOMEM; /* Not enough memory. */
        return -1;
    }

    if(amount > ringbuffer_free_count(rb)) {
        amount = ringbuffer_free_count(rb);
    }

    if (buffer != NULL) {
        idx = rb->write_offset;
        end = (rb->write_offset + amount) & (rb->size - 1);
        max = rb->size;

        if(idx < end) {
            memcpy(&rb->storage[idx], buffer, end - idx);
        } else if(idx > end) {
            memcpy(&rb->storage[idx], buffer, max - idx);
            memcpy(&rb->storage[0], buffer + (max - idx), end);
        }
    }

    rb->write_offset += amount;
    rb->write_offset &= rb->size - 1;

    return amount;
}

/** long ringbuffer_write_fd(struct ringbuffer *rb, int fd, long amount);
 *
 * Writes the data located in the ring buffer at @a rb onto the file descriptor
 * (or socket) @a fd. Provides semantics similar to the standard write() call.
 *
 * @param rb The address of a ring buffer structure.
 * @param fd The file descriptor to copy data onto.
 * @param amount The total amount of data to copy.
 * @return Returns the number of bytes copied on success or -1 on error, setting errno.
 */
long ringbuffer_write_fd(struct ringbuffer *rb, int fd, long amount) {
    unsigned long idx = 0, end = 0, max = 0, bytes = 0;
    struct iovec iov[2];
    int count = 0;

    if(rb == NULL) {
        errno = EINVAL; /* Invalid argument. */
        return -1;
    }

    if(amount > (long) rb->size) {
        errno = ENOMEM; /* Not enough memory. */
        return -1;
    }

    if(amount > ringbuffer_free_count(rb)) {
        amount = ringbuffer_free_count(rb);
    }

    idx = rb->read_offset;
    end = (rb->read_offset + amount) & (rb->size - 1);
    max = rb->size;

    if(idx < end) {
        count = 1;
        iov[0].iov_base = &rb->storage[idx];
        iov[0].iov_len  = end - idx;
    } else if(idx > end) {
        count = 2;
        iov[0].iov_base = &rb->storage[idx];
        iov[0].iov_len  = max - idx;
        iov[1].iov_base = &rb->storage[0];
        iov[1].iov_len  = end;
    }

    bytes = writev(fd, iov, count);

    if(bytes <= 0) {
        if(errno != EAGAIN && errno != EWOULDBLOCK) {
            /* Actually do something to register an error. */
            return -1;
        }
    } else {
        rb->read_offset += bytes;
        rb->read_offset &= rb->size - 1;
    }

    return bytes;
}

/** long ringbuffer_read_fd(struct ringbuffer *rb, int fd, long amount);
 *
 * Reads data from the file descriptor (or socket) @a fd into the ring
 * buffer at @a rb. Provides semantics similar to the standard read() call.
 *
 * @param rb The address of a ring buffer structure.
 * @param fd The file descriptor to copy data onto.
 * @param amount The total amount of data to copy.
 * @return Returns the number of bytes copied on success or -1 on error, setting errno.
 */
long ringbuffer_read_fd(struct ringbuffer *rb, int fd, long amount) {
    unsigned long idx = 0, end = 0, max = 0, bytes = 0;
    struct iovec iov[2];
    int count = 0;

    if(rb == NULL) {
        errno = EINVAL; /* Invalid argument. */
        return -1;
    }

    if(amount > (long) rb->size) {
        errno = ENOMEM; /* Not enough memory. */
        return -1;
    }

    if(amount > ringbuffer_fill_count(rb)) {
        amount = ringbuffer_fill_count(rb);
    }

    idx = rb->write_offset;
    end = (rb->write_offset + amount) & (rb->size - 1);
    max = rb->size;

    if(idx < end) {
        count = 1;
        iov[0].iov_base = &rb->storage[idx];
        iov[0].iov_len  = end - idx;
    } else if(idx > end) {
        count = 2;
        iov[0].iov_base = &rb->storage[idx];
        iov[0].iov_len  = max - idx;
        iov[1].iov_base = &rb->storage[0];
        iov[1].iov_len  = end;
    }

    bytes = readv(fd, iov, count);

    if(bytes <= 0) {
        if(errno != EAGAIN && errno != EWOULDBLOCK) {
            /* Actually do something to register an error. */
            return -1;
        }
    } else {
        rb->write_offset += bytes;
        rb->write_offset &= rb->size - 1;
    }

    return bytes;
}
{% endhighlight %}

