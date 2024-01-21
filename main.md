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


# Clog buffer

TransactionIdCommitTree
    TransactionIdSetTreeStatus


a) Get the page from the xid

    TransactionIdToPage(xid);
        xid / (int64) CLOG_XACTS_PER_PAGE
        = xid / (BLCKSZ * CLOG_XACTS_PER_BYTE);



b) TransactionIdSetPageStatus

    1. lock the XactSLRULock

    2. TransactionIdSetPageStatusInternal

        2.1 SimpleLruReadPage
            SlruSelectLRUPage

            2.1.1 See if page already has a buffer assigned

                for (slotno = 0; slotno < shared->num_slots; slotno++)
                    if (shared->page_number[slotno] == pageno &&
                        shared->page_status[slotno] != SLRU_PAGE_EMPTY)
                        return return slotno;

            2.1.2 If we find any EMPTY slot, just select that one. Else choose a
                victim page to replace.
            
            2.1.3 Write the page if dirty, and try again.

                SlruInternalWritePage(ctl, bestvalidslot, NULL);
        
        2.2 ok = SlruPhysicalReadPage(ctl, pageno, slotno);

        2.3 Set the LSNs for this newly read-in page to zero.

            SimpleLruZeroLSNs(ctl, slotno);

        2.4 Set the subtransactions and main transaction

            TransactionIdSetStatusBit


The usage of clog:

Check if the transaction is committed or aborted, used mostly for tuple
visibility:

    TransactionIdDidCommit
    TransactionIdDidAbort
        TransactionIdGetStatus


```c

	/*
	 * Arrays holding info for each buffer slot.  Page number is undefined
	 * when status is EMPTY, as is page_lru_count.
	 */
	char	  **page_buffer;
...
	/*
	 * Optional array of WAL flush LSNs associated with entries in the SLRU
	 * pages.  If not zero/NULL, we must flush WAL before writing pages (true
	 * for pg_xact, false for multixact, pg_subtrans, pg_notify).  group_lsn[]
	 * has lsn_groups_per_page entries per buffer slot, each containing the
	 * highest LSN known for a contiguous group of SLRU entries on that slot's
	 * page.
	 */
	XLogRecPtr *group_lsn;
	int			lsn_groups_per_page;

	/*----------
	 * We mark a page "most recently used" by setting
	 *		page_lru_count[slotno] = ++cur_lru_count;
	 * The oldest page is therefore the one with the highest value of
	 *		cur_lru_count - page_lru_count[slotno]
	 * The counts will eventually wrap around, but this calculation still
	 * works as long as no page's age exceeds INT_MAX counts.
	 *----------
	 */
	int			cur_lru_count;
```

# Tuple format

Note that only t_data will be written into the disk.

    offnum = PageAddItem(pageHeader, (Item) tuple->t_data,
    				 tuple->t_len, InvalidOffsetNumber, false, true);

```c

typedef struct HeapTupleData
{
	uint32		t_len;			/* length of *t_data */
	ItemPointerData t_self;		/* SelfItemPointer */
	Oid			t_tableOid;		/* table the tuple came from */
	HeapTupleHeader t_data;		/* -> tuple header and data */
} HeapTupleData;

struct HeapTupleHeaderData
{
	union
	{
		HeapTupleFields t_heap;
		DatumTupleFields t_datum;
	}			t_choice;

	ItemPointerData t_ctid;		/* current TID of this or newer tuple (or a
								 * speculative insertion token) */

	/* Fields below here must match MinimalTupleData! */

	uint16		t_infomask2;	/* number of attributes + various flags */
	uint16		t_infomask;		/* various flag bits, see below */
	uint8		t_hoff;			/* sizeof header incl. bitmap, padding */

	/* ^ - 23 bytes - ^ */
	bits8		t_bits[FLEXIBLE_ARRAY_MEMBER];	/* bitmap of NULLs */

	/* MORE DATA FOLLOWS AT END OF STRUCT */
};
```

# VSM VM

16384 - table's filenode

16384_fsm - The free space map is stored in a file named with the filenode
number

16384_vm - to track which pages are known to have no dead tuples.

	/*
	 * Reset the next slot pointer. This encourages the use of low-numbered
	 * pages, increasing the chances that a later vacuum can truncate the
	 * relation.  We don't bother with a lock here, nor with marking the page
	 * dirty if it wasn't already, since this is just a hint.
	 */
	((FSMPage) PageGetContents(page))->fp_next_slot = 0;

For example, consider this tree:

		   7
	   7	   6
	 5	 7	 6	 5
	4 5 5 7 2 6 5 2
				T

Assume that the target node is the node indicated by the letter T, and we're
searching for a node with value of 6 or higher. The search begins at T. At the
first iteration, we move to the right, then to the parent, arriving at the
rightmost 5. At the second iteration, we move to the right, wrapping around,
then climb up, arriving at the 7 on the third level.  7 satisfies our search,
so we descend down to the bottom, following the path of sevens.  This is in
fact the first suitable page to the right of (allowing for wraparound) our
start point.

# column store Hydra

A meta table to record each data block which contains a chunk of data of the
same column.

Columnar storage functionality is dependent on several metadata tables that
reside in the columnar schema. For example, the columnar.stripe table contains
all stripes that are currently visible to our transaction, and this information
will be used to read and locate stripes within our columnar table.

The heap tables provide a consistent view of data in concurrent environments
via Postgres’ multi-version model (MVCC). This means that each SQL statement
sees a snapshot of data (a database version) as it was some time ago,
regardless of the current state of the underlying data. You can imagine a
situation when two concurrent transactions are active - A and B. If transaction
A adds rows to the table, then the other transaction will not be able to see
them because entries in columnar.stripe will not be visible to transaction B ,
even though they are visible to transaction A.

Each stripe contains up to 15 chunks (with each chunk contains up to 10,000
rows), and the metadata for each chunk is stored in columnar.chunk . This table
is used to filter chunks based on their min and max values. Each chunk column
has entries in this table, so when executing a filter (a WHERE clause) these
values are checked before reading the chunk based on min and max value.

Hydra's columnar DELETE command uses the mask column inside each row_mask row
to logically mark the rows that are deleted and hide them from future queries.

https://blog.hydra.so/blog/2023-03-02-columnar-updates-and-deletes


# WAL log insert

Constructing a WAL record begins with a call to XLogBeginInsert, followed by a
number of XLogRegister* calls. The registered data is collected in private
working memory, and finally assembled into a chain of XLogRecData structs by a
call to XLogRecordAssemble(). See access/transam/README for details.

XLogInsertRecord

1. Reserve the right amount of space from the WAL.

    	ReserveXLogInsertLocation(rechdr->xl_tot_len, &StartPos, &EndPos,
							  &rechdr->xl_prev);

2. Copy the record to the reserved WAL space.

		/*
		 * All the record data, including the header, is now ready to be
		 * inserted. Copy the record in the space reserved.
		 */
		CopyXLogRecordToWAL(rechdr->xl_tot_len, isLogSwitch, rdata,
							StartPos, EndPos, insertTLI);


                Get a pointer to the right place in the right WAL buffer to start inserting to.
                
                    currpos = GetXLogBuffer(CurrPos, tli);

                     * If the page is not initialized yet, it is initialized. That might require
                     * evicting an old dirty buffer from the buffer cache, which means I/O.

                        AdvanceXLInsertBuffer(ptr, tli, false);
                            XLogWrite(WriteRqst, tli, false);
