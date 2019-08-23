PostgreSQL 7.1.3 Documentation
===
Chapter 9. Write-Ahead Logging (WAL)
---
Table of Contents
---
[9.1. General Description]()  
    [9.1.1. Immediate Benefits of WAL]()  
    [9.1.2. Future Benefits]()  
[9.2. Implementation]()  
[9.3. WAL Configuration]()  

> Author: Vadim Mikheev and Oliver Elphick  

#### 9.1. General Description ####  

Write Ahead Logging (WAL) is a standard approach to transaction logging. Its detailed description may be found in most (if not all) books about transaction processing. Briefly, WAL's central concept is that changes to data files (where tables and indices reside) must be written only after those changes have been logged - that is, when log records have been flushed to permanent storage. When we follow this procedure, we do not need to flush data pages to disk on every transaction commit, because we know that in the event of a crash we will be able to recover the database using the log: any changes that have not been applied to the data pages will first be redone from the log records (this is roll-forward recovery, also known as REDO) and then changes made by uncommitted transactions will be removed from the data pages (roll-backward recovery - UNDO).

#### 9.1.1. Immediate Benefits of WAL
The first obvious benefit of using WAL is a significantly reduced number of disk writes, since only the log file needs to be flushed to disk at the time of transaction commit; in multi-user environments, commits of many transactions may be accomplished with a single fsync() of the log file. Furthermore, the log file is written sequentially, and so the cost of syncing the log is much less than the cost of flushing the data pages.

The next benefit is consistency of the data pages. The truth is that, before WAL, PostgreSQL was never able to guarantee consistency in the case of a crash. Before WAL, any crash during writing could result in:

index tuples pointing to non-existent table rows

index tuples lost in split operations

totally corrupted table or index page content, because of partially written data pages

Problems with indices (problems 1 and 2) could possibly have been fixed by additional fsync() calls, but it is not obvious how to handle the last case without WAL; WAL saves the entire data page content in the log if that is required to ensure page consistency for after-crash recovery.
#### 9.1.2. Future Benefits
In this first release of WAL, UNDO operation is not implemented, because of lack of time. This means that changes made by aborted transactions will still occupy disk space and that we still need a permanent pg_log file to hold the status of transactions, since we are not able to re-use transaction identifiers. Once UNDO is implemented, pg_log will no longer be required to be permanent; it will be possible to remove pg_log at shutdown, split it into segments and remove old segments.

With UNDO, it will also be possible to implement savepoints to allow partial rollback of invalid transaction operations (parser errors caused by mistyping commands, insertion of duplicate primary/unique keys and so on) with the ability to continue or commit valid operations made by the transaction before the error. At present, any error will invalidate the whole transaction and require a transaction abort.

WAL offers the opportunity for a new method for database on-line backup and restore (BAR). To use this method, one would have to make periodic saves of data files to another disk, a tape or another host and also archive the WAL log files. The database file copy and the archived log files could be used to restore just as if one were restoring after a crash. Each time a new database file copy was made the old log files could be removed. Implementing this facility will require the logging of data file and index creation and deletion; it will also require development of a method for copying the data files (operating system copy commands are not suitable).

#### 9.2. Implementation
WAL is automatically enabled from release 7.1 onwards. No action is required from the administrator with the exception of ensuring that the additional disk-space requirements of the WAL logs are met, and that any necessary tuning is done (see Section 9.3).

WAL logs are stored in the directory $PGDATA/pg_xlog, as a set of segment files, each 16 MB in size. Each segment is divided into 8 kB pages. The log record headers are described in access/xlog.h; record content is dependent on the type of event that is being logged. Segment files are given sequential numbers as names, starting at 0000000000000000. The numbers do not wrap, at present, but it should take a very long time to exhaust the available stock of numbers.

The WAL buffers and control structure are in shared memory, and are handled by the backends; they are protected by spinlocks. The demand on shared memory is dependent on the number of buffers; the default size of the WAL buffers is 64 kB.

It is of advantage if the log is located on another disk than the main database files. This may be achieved by moving the directory, pg_xlog, to another location (while the postmaster is shut down, of course) and creating a symbolic link from the original location in $PGDATA to the new location.

The aim of WAL, to ensure that the log is written before database records are altered, may be subverted by disk drives that falsely report a successful write to the kernel, when, in fact, they have only cached the data and not yet stored it on the disk. A power failure in such a situation may still lead to irrecoverable data corruption; administrators should try to ensure that disks holding PostgreSQL's data and log files do not make such false reports.

#### 9.2.1. Database Recovery with WAL
After a checkpoint has been made and the log flushed, the checkpoint's position is saved in the file pg_control. Therefore, when recovery is to be done, the backend first reads pg_control and then the checkpoint record; next it reads the redo record, whose position is saved in the checkpoint, and begins the REDO operation. Because the entire content of the pages is saved in the log on the first page modification after a checkpoint, the pages will be first restored to a consistent state.

Using pg_control to get the checkpoint position speeds up the recovery process, but to handle possible corruption of pg_control, we should actually implement the reading of existing log segments in reverse order -- newest to oldest -- in order to find the last checkpoint. This has not yet been done in release 7.1.
