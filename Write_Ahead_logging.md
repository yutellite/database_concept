PostgreSQL 7.2.8 Documentation
===
Chapter 12. Write-Ahead Logging (WAL)
---
Table of Contents
---
[12.1. General Description]()  
    [12.1.1. Immediate Benefits of WAL]()  
    [12.1.2. Future Benefits]()  
[12.2. Implementation]()  
[12.3. WAL Configuration]()  

> Author: Vadim Mikheev and Oliver Elphick  

#### 12.1. General Description ####  

Write Ahead Logging (WAL) is a standard approach to transaction logging. Its detailed description may be found in most (if not all) books about transaction processing. Briefly, WAL's central concept is that changes to data files (where tables and indexes reside) must be written only after those changes have been logged - that is, when log records have been flushed to permanent storage. When we follow this procedure, we do not need to flush data pages to disk on every transaction commit, because we know that in the event of a crash we will be able to recover the database using the log: any changes that have not been applied to the data pages will first be redone from the log records (this is roll-forward recovery, also known as REDO) and then changes made by uncommitted transactions will be removed from the data pages (roll-backward recovery - UNDO).

#### 12.1.1. Immediate Benefits of WAL
The first obvious benefit of using WAL is a significantly reduced number of disk writes, since only the log file needs to be flushed to disk at the time of transaction commit; in multiuser environments, commits of many transactions may be accomplished with a single fsync() of the log file. Furthermore, the log file is written sequentially, and so the cost of syncing the log is much less than the cost of flushing the data pages.

The next benefit is consistency of the data pages. The truth is that, before WAL, PostgreSQL was never able to guarantee consistency in the case of a crash. Before WAL, any crash during writing could result in:

* index tuples pointing to nonexistent table rows

* index tuples lost in split operations

* totally corrupted table or index page content, because of partially written data pages

Problems with indexes (problems 1 and 2) could possibly have been fixed by additional fsync() calls, but it is not obvious how to handle the last case without WAL; WAL saves the entire data page content in the log if that is required to ensure page consistency for after-crash recovery.
#### 12.1.2. Future Benefits
UNDO operation is not implemented. This means that changes made by aborted transactions will still occupy disk space and that we still need a permanent pg_clog file to hold the status of transactions, since we are not able to re-use transaction identifiers. Once UNDO is implemented, pg_clog will no longer be required to be permanent; it will be possible to remove pg_clog at shutdown. (However, the urgency of this concern has decreased greatly with the adoption of a segmented storage method for pg_clog --- it is no longer necessary to keep old pg_clog entries around forever.)

With UNDO, it will also be possible to implement savepoints to allow partial rollback of invalid transaction operations (parser errors caused by mistyping commands, insertion of duplicate primary/unique keys and so on) with the ability to continue or commit valid operations made by the transaction before the error. At present, any error will invalidate the whole transaction and require a transaction abort.

WAL offers the opportunity for a new method for database on-line backup and restore (BAR). To use this method, one would have to make periodic saves of data files to another disk, a tape or another host and also archive the WAL log files. The database file copy and the archived log files could be used to restore just as if one were restoring after a crash. Each time a new database file copy was made the old log files could be removed. Implementing this facility will require the logging of data file and index creation and deletion; it will also require development of a method for copying the data files (operating system copy commands are not suitable).

A difficulty standing in the way of realizing these benefits is that they require saving WAL entries for considerable periods of time (eg, as long as the longest possible transaction if transaction UNDO is wanted). The present WAL format is extremely bulky since it includes many disk page snapshots. This is not a serious concern at present, since the entries only need to be kept for one or two checkpoint intervals; but to achieve these future benefits some sort of compressed WAL format will be needed.

#### 12.2. Implementation
WAL is automatically enabled from release 7.1 onwards. No action is required from the administrator with the exception of ensuring that the additional disk-space requirements of the WAL logs are met, and that any necessary tuning is done (see Section 12.3).

WAL logs are stored in the directory $PGDATA/pg_xlog, as a set of segment files, each 16 MB in size. Each segment is divided into 8 kB pages. The log record headers are described in access/xlog.h; record content is dependent on the type of event that is being logged. Segment files are given ever-increasing numbers as names, starting at 0000000000000000. The numbers do not wrap, at present, but it should take a very long time to exhaust the available stock of numbers.

The WAL buffers and control structure are in shared memory, and are handled by the backends; they are protected by lightweight locks. The demand on shared memory is dependent on the number of buffers. The default size of the WAL buffers is 8 buffers of 8 kB each, or 64 kB total.

It is of advantage if the log is located on another disk than the main database files. This may be achieved by moving the directory, pg_xlog, to another location (while the postmaster is shut down, of course) and creating a symbolic link from the original location in $PGDATA to the new location.

The aim of WAL, to ensure that the log is written before database records are altered, may be subverted by disk drives that falsely report a successful write to the kernel, when, in fact, they have only cached the data and not yet stored it on the disk. A power failure in such a situation may still lead to irrecoverable data corruption. Administrators should try to ensure that disks holding PostgreSQL's log files do not make such false reports.

#### 12.2.1. Database Recovery with WAL
After a checkpoint has been made and the log flushed, the checkpoint's position is saved in the file pg_control. Therefore, when recovery is to be done, the backend first reads pg_control and then the checkpoint record; then it performs the REDO operation by scanning forward from the log position indicated in the checkpoint record. Because the entire content of data pages is saved in the log on the first page modification after a checkpoint, all pages changed since the checkpoint will be restored to a consistent state.

Using pg_control to get the checkpoint position speeds up the recovery process, but to handle possible corruption of pg_control, we should actually implement the reading of existing log segments in reverse order -- newest to oldest -- in order to find the last checkpoint. This has not been implemented, yet.

#### 12.3. WAL Configuration
There are several WAL-related parameters that affect database performance. This section explains their use. Consult Section 3.4 for details about setting configuration parameters.

Checkpoints are points in the sequence of transactions at which it is guaranteed that the data files have been updated with all information logged before the checkpoint. At checkpoint time, all dirty data pages are flushed to disk and a special checkpoint record is written to the log file. As result, in the event of a crash, the recoverer knows from what record in the log (known as the redo record) it should start the REDO operation, since any changes made to data files before that record are already on disk. After a checkpoint has been made, any log segments written before the undo records are no longer needed and can be recycled or removed. (When WAL-based BAR is implemented, the log segments would be archived before being recycled or removed.)

The postmaster spawns a special backend process every so often to create the next checkpoint. A checkpoint is created every CHECKPOINT_SEGMENTS log segments, or every CHECKPOINT_TIMEOUT seconds, whichever comes first. The default settings are 3 segments and 300 seconds respectively. It is also possible to force a checkpoint by using the SQL command CHECKPOINT.

Reducing CHECKPOINT_SEGMENTS and/or CHECKPOINT_TIMEOUT causes checkpoints to be done more often. This allows faster after-crash recovery (since less work will need to be redone). However, one must balance this against the increased cost of flushing dirty data pages more often. In addition, to ensure data page consistency, the first modification of a data page after each checkpoint results in logging the entire page content. Thus a smaller checkpoint interval increases the volume of output to the log, partially negating the goal of using a smaller interval, and in any case causing more disk I/O.

There will be at least one 16MB segment file, and will normally not be more than 2 * CHECKPOINT_SEGMENTS + 1 files. You can use this to estimate space requirements for WAL. Ordinarily, when old log segment files are no longer needed, they are recycled (renamed to become the next sequential future segments). If, due to a short-term peak of log output rate, there are more than 2 * CHECKPOINT_SEGMENTS + 1 segment files, the unneeded segment files will be deleted instead of recycled until the system gets back under this limit.

There are two commonly used WAL functions: LogInsert and LogFlush. LogInsert is used to place a new record into the WAL buffers in shared memory. If there is no space for the new record, LogInsert will have to write (move to kernel cache) a few filled WAL buffers. This is undesirable because LogInsert is used on every database low level modification (for example, tuple insertion) at a time when an exclusive lock is held on affected data pages, so the operation needs to be as fast as possible. What is worse, writing WAL buffers may also force the creation of a new log segment, which takes even more time. Normally, WAL buffers should be written and flushed by a LogFlush request, which is made, for the most part, at transaction commit time to ensure that transaction records are flushed to permanent storage. On systems with high log output, LogFlush requests may not occur often enough to prevent WAL buffers being written by LogInsert. On such systems one should increase the number of WAL buffers by modifying the postgresql.conf WAL_BUFFERS parameter. The default number of WAL buffers is 8. Increasing this value will correspondingly increase shared memory usage.

The COMMIT_DELAY parameter defines for how many microseconds the backend will sleep after writing a commit record to the log with LogInsert but before performing a LogFlush. This delay allows other backends to add their commit records to the log so as to have all of them flushed with a single log sync. No sleep will occur if fsync is not enabled or if fewer than COMMIT_SIBLINGS other backends are not currently in active transactions; this avoids sleeping when it's unlikely that any other backend will commit soon. Note that on most platforms, the resolution of a sleep request is ten milliseconds, so that any nonzero COMMIT_DELAY setting between 1 and 10000 microseconds will have the same effect. Good values for these parameters are not yet clear; experimentation is encouraged.

The WAL_SYNC_METHOD parameter determines how PostgreSQL will ask the kernel to force WAL updates out to disk. All the options should be the same as far as reliability goes, but it's quite platform-specific which one will be the fastest. Note that this parameter is irrelevant if FSYNC has been turned off.

Setting the WAL_DEBUG parameter to any nonzero value will result in each LogInsert and LogFlush WAL call being logged to standard error. At present, it makes no difference what the nonzero value is. This option may be replaced by a more general mechanism in the future.
