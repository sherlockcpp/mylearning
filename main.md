# md.c

This code manages relations that reside on magnetic disk. Include Low level
functions to read/write data from/to disk.

mdwrite
    -> _mdfd_getseg

        Find the segment of the relation holding the specified block. (fd.c)

        -> PathNameOpenFile
            -> PathNameOpenFilePerm
                -> BasicOpenFilePerm
                    -> open (system function)

    -> FileWrite

Related info:

Segment [block]

RELSEG_SIZE:

    RELSEG_SIZE is the maximum number of blocks allowed in one disk file. Thus, the
    maximum size of a single file is RELSEG_SIZE * BLCKSZ; relations bigger than
    that are divided into multiple files. RELSEG_SIZE * BLCKSZ must be less than
    your OS' limit on file size. This is often 2 GB or 4GB in a 32-bit operating
    system, unless you have large file support enabled. By default, we make the
    limit 1 GB to avoid any possible integer-overflow problems within the OS. A
    limit smaller than necessary only means we divide a large relation into more
    chunks than necessary, so it seems best to err in the direction of a small
    limit. A power-of-2 value is recommended to save a few cycles in md.c, but is
    not absolutely required. Changing RELSEG_SIZE requires an initdb.

Vector write style:

    buffers_to_iov

        convert the *buffer into iovec.

    FileWriteV
        pwritev

            The writev() system call works just like write(2) except that multiple buffers are written out.
            The pointer iov points to an array of iovec structures, defined in <sys/uio.h> as:

                struct iovec {
                    void  *iov_base;    /* Starting address */
                    size_t iov_len;     /* Number of bytes to transfer */
                };

# buffer

Principal entry points:

ReadBuffer() -- find or create a buffer holding the requested page, and pin it
so that no one can destroy it while this process is using it.

ReleaseBuffer() -- unpin a buffer

MarkBufferDirty() -- mark a pinned buffer's contents as "dirty". The disk write
is delayed until buffer replacement or checkpoint.

See also these files:
		freelist.c -- chooses victim for buffer replacement
		buf_table.c -- manages the buffer lookup table


# smgrprefetch

FilePrefetch -> posix_fadvise

Programs can use posix_fadvise() to announce an intention to access file data
in a specific pattern in the future, thus allowing the kernel to perform
appropriate optimiza‐ tions.

posix_fadvise - POSIX_FADV_WILLNEED

The specified data will be accessed in the near future. POSIX_FADV_WILLNEED
initiates a nonblocking read of the specified region into the page cache.  The
amount of data read may be decreased by thekernel depending on  vir‐tual memory
load.  (A few megabytes will usually be fully satisfied, and more is rarely
useful.)



# WAL prefetch

lrq_prefetch -> XLogPrefetcherNextBlock

Returns LRQ_NEXT_AGAIN if no more WAL data is available yet.

Returns LRQ_NEXT_IO if the next block reference is for a main fork block that
isn't in the buffer pool, and the kernel has been asked to start reading it to
make a future read system call faster. An LSN is written to *lsn, and the I/O
will be considered to have completed once that LSN is replayed.

Returns LRQ_NEXT_NO_IO if we examined the next block reference and found that
it was already in the buffer pool, or we decided for various reasons not to
prefetch.

The basic logic is that: it will try to search the block in the buffer, if
exists, then don't need to prefetch, if not, call smgrprefetch to ask Kernel to
prefetch the data so that the next read will be faster.

i.e
 - Cache hit, nothing to do.
 - Cache miss, I/O (presumably) started.