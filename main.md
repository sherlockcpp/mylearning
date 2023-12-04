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


