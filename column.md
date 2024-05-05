
# design v0

### DISK FORMAT

Only on DISK FORMAT is different, after being queries, let's deform original
tuples and convert them into column tuple.

A chunk (16 * BLCKSZ) or 1 BLCKSZ (in initial version) can only contain one
column's data. Maintain a per-table chunk2column map, so that we can easier to
get all the chunks that used for one column.

A special block to mantain the reference/map data to each column block.

{
    Page header [use pd_flags to identify the special page]
    ItemPointerData - the tuple itself
    ItemPointerData[] - points to each column data
    TODO: number of rows, this is needed when we support bulk insert.
}

The mechnism for the special page, the first 0 one is the special meta page,
and then each time the special meta page becomes full, then choose the next
meta page and add a special tuple which links to the next metapage.

Each real data block also will include a special tuple links to the next block
of this column.

[reference block] [column a block] [column b block] .. [reference block 2]

TO THINK:

How to find the target block with enough freespace efficiently.
One idea is to maintain multiple fsm tree for each type of block.


### DDL DML

functions:
create_column_store_table
insert_into_column_table

1. scan the block from 0 to find a target special block to save the
   reference/map data.
2. insert the reference tuple into the block.
3. Choose the real data block, no of metapage block + no of column * N(try to
   get N from 0 to ...).
4. insert the data in the real block.

select_from_column_table.

1. start from 0 speical meta page, get each reference.
2. Based on the reference, scan the required column blocks.
3. Combine the required column data into one row style tuple and return.

### WAL
Skip the WAL log.

### VACUUM
Don't support.

### INDEX
Don't support.

-------------------------------------------------------------------------------

# design v1

### DISK FORMAT

FSM.

### DDL DML
Support command: CREATE TABLE, INSERT, SELECT.

### WAL
### VACUUM
### INDEX

-------------------------------------------------------------------------------

# design v2

### DISK FORMAT

FSM

### DDL DML
No

### WAL
Generate the WAL log.

### VACUUM
### INDEX

-------------------------------------------------------------------------------

# design v3

### DISK FORMAT

### DDL DML
### WAL
### VACUUM
### INDEX

-------------------------------------------------------------------------------

# design v4

### DISK FORMAT
### DDL DML

UPDATE, DELETE

### WAL
### VACUUM
### INDEX

-------------------------------------------------------------------------------

# design v5

### DISK FORMAT
### DDL DML
### WAL
### VACUUM
### INDEX

See how to support VACUUM

-------------------------------------------------------------------------------

# design v6

### DISK FORMAT
### DDL DML
### WAL
### VACUUM
### INDEX