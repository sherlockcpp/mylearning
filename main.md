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


---

XLogReadBufferForRedoExtended
XLogReadBufferForRedo

Reads a block referenced by a WAL record into shared buffer cache, and
determines what needs to be done to redo the changes to it.

---

PrefetchBuffer().

This is named by analogy to ReadBuffer but doesn't actually allocate a buffer.
Instead it tries to ensure that a future ReadBuffer for the given block will
not be delayed by the I/O.  Prefetching is optional.

Note: Prefetch will look for the buffer while not pin it.


# Buffer

BufferDescPadded BufferDescriptors[... ]

GetBufferDescriptor will get the buffdesc from the BufferDescriptors.
Note that LockBufHdr the bufferdesc.

Normal process of reading a buffer (Usage:
heapam_scan_sample_next_block/heapgettup_pagemode/heapgettup):

a) Allocate the buffer

  1. If the buffer is in the buffer pool already, then return.
  
            LWLockAcquire(newPartitionLock, LW_SHARED);
            existing_buf_id = BufTableLookup(&newTag, newHash);
                valid = PinBuffer(buf, strategy);

  2. If not exists, get a new buffer.
  
      GetVictimBuffer
          StrategyGetBuffer
  
          2.1. return if buffer is available in ring buffer
  
          2.2. it will notify the bgwriter.
          and then try to get a buffer from freelist StrategyControl->firstFreeBuffer and add it to the ring buffer.

  3. Nothing on the freelist, so run the "clock sweep" algorithm.

a.2) after getting the buffer, pin (buf_state += BUF_REFCOUNT_ONE; ...) and
unlock the buffer.

a.3) If the buffer was dirty(buf_state & BM_DIRTY), try to write it out.

    FlushBuffer
    ScheduleBufferTagForWriteback

b) Set necessary bits for the buffer.

    victim_buf_state |= BM_TAG_VALID | BUF_USAGECOUNT_ONE;
    StartBufferIO
        buf_state |= BM_IO_IN_PROGRESS;

c) read data from disk and write it into the buffer

    #define BufHdrGetBlock(bufHdr)	((Block) (BufferBlocks + ((Size) (bufHdr)->buf_id) * BLCKSZ))
    bufBlock = isLocalBuf ? LocalBufHdrGetBlock(bufHdr) : BufHdrGetBlock(bufHdr);

    smgrread(smgr, forkNum, blockNum, bufBlock);

        1. get the target segment.
        v = _mdfd_getseg(reln, forknum, blocknum, false,
				 EXTENSION_FAIL | EXTENSION_CREATE_RECOVERY);

            1.1. if an existing and opened segment, we're done
                targetseg = blkno / ((BlockNumber) RELSEG_SIZE);
                v = &reln->md_seg_fds[forknum][targetseg];
                return v

            1.2 open the fork if not yet.
                v = mdopenfork(reln, forknum, behavior);

            1.2. The target segment is not yet open. Iterate over all the segments
            between the last opened and the target segment.

                for (nextsegno = reln->md_num_open_segs[forknum];
                nextsegno <= targetseg; nextsegno++)
                    v = _mdfd_openseg(reln, forknum, nextsegno, flags);

        2. read data FileRead


The process of extending buffers:

a) Acquire victim buffers for extension without holding extension lock.

    for (uint32 i = 0; i < extend_by; i++)
        buffers[i] = GetVictimBuffer(strategy, io_context);
        buf_block = BufHdrGetBlock(GetBufferDescriptor(buffers[i] - 1));

b) Insert all the buffer into Buf hash

    for (uint32 i = 0; i < extend_by; i++)
        existing_id = BufTableInsert(&tag, hash, victim_buf_hdr->buf_id);

c) smgrzeroextend

d) Set BM_VALID, terminate IO, and wake up any waiters

    for (uint32 i = 0; i < extend_by; i++)
        TerminateBufferIO(buf_hdr, false, BM_VALID, true);


Write tuple into buffer(RelationPutHeapTuple):

a) Add the tuple to the page PageAddItem

    PageAddItemExtended

        1. Select offsetNumber to place the new item at
            for (offsetNumber = FirstOffsetNumber;
            	 offsetNumber < limit;	/* limit is maxoff+1 */
            	 offsetNumber++)
            {
            	itemId = PageGetItemId(page, offsetNumber);

        2. insert the item.

            itemId = PageGetItemId(page, offsetNumber);
            memcpy((char *) page + upper, item, size);

            adjust page header
	        phdr->pd_lower = (LocationIndex) lower;
	        phdr->pd_upper = (LocationIndex) upper;


b) Update tuple->t_self to the actual position where it was stored

    ItemPointerSet(&(tuple->t_self), BufferGetBlockNumber(buffer), offnum);


# Fork type

```c
/*
 * Stuff for fork names.
 *
 * The physical storage of a relation consists of one or more forks.
 * The main fork is always created, but in addition to that there can be
 * additional forks for storing various metadata. ForkNumber is used when
 * we need to refer to a specific fork in a relation.
 */
typedef enum ForkNumber
{
	InvalidForkNumber = -1,
	MAIN_FORKNUM = 0,
	FSM_FORKNUM,
	VISIBILITYMAP_FORKNUM,
	INIT_FORKNUM,
	/*
	 * NOTE: if you add a new fork, change MAX_FORKNUM and possibly
	 * FORKNAMECHARS below, and update the forkNames array in
	 * src/common/relpath.c
	 */
} ForkNumber;

...
	/*
	 * for md.c; per-fork arrays of the number of open segments
	 * (md_num_open_segs) and the segments themselves (md_seg_fds).
	 */
	int			md_num_open_segs[MAX_FORKNUM + 1];
	struct _MdfdVec *md_seg_fds[MAX_FORKNUM + 1];

	/* if unowned, list link in list of all unowned SMgrRelations */
	dlist_node	node;
} SMgrRelationData;
```

