PostgreSQL 7.1.3 Documentation

Chapter 9. Write-Ahead Logging (WAL)
Table of Contents
9.1. General Description
9.1.1. Immediate Benefits of WAL
9.1.2. Future Benefits
9.2. Implementation
9.3. WAL Configuration
Author: Vadim Mikheev and Oliver Elphick

9.1. General Description
Write Ahead Logging (WAL) is a standard approach to transaction logging. Its detailed description may be found in most (if not all) books about transaction processing. Briefly, WAL's central concept is that changes to data files (where tables and indices reside) must be written only after those changes have been logged - that is, when log records have been flushed to permanent storage. When we follow this procedure, we do not need to flush data pages to disk on every transaction commit, because we know that in the event of a crash we will be able to recover the database using the log: any changes that have not been applied to the data pages will first be redone from the log records (this is roll-forward recovery, also known as REDO) and then changes made by uncommitted transactions will be removed from the data pages (roll-backward recovery - UNDO).

9.1.1. Immediate Benefits of WAL
The first obvious benefit of using WAL is a significantly reduced number of disk writes, since only the log file needs to be flushed to disk at the time of transaction commit; in multi-user environments, commits of many transactions may be accomplished with a single fsync() of the log file. Furthermore, the log file is written sequentially, and so the cost of syncing the log is much less than the cost of flushing the data pages.

The next benefit is consistency of the data pages. The truth is that, before WAL, PostgreSQL was never able to guarantee consistency in the case of a crash. Before WAL, any crash during writing could result in:

index tuples pointing to non-existent table rows

index tuples lost in split operations

totally corrupted table or index page content, because of partially written data pages

Problems with indices (problems 1 and 2) could possibly have been fixed by additional fsync() calls, but it is not obvious how to handle the last case without WAL; WAL saves the entire data page content in the log if that is required to ensure page consistency for after-crash recovery.
9.1.2. Future Benefits
In this first release of WAL, UNDO operation is not implemented, because of lack of time. This means that changes made by aborted transactions will still occupy disk space and that we still need a permanent pg_log file to hold the status of transactions, since we are not able to re-use transaction identifiers. Once UNDO is implemented, pg_log will no longer be required to be permanent; it will be possible to remove pg_log at shutdown, split it into segments and remove old segments.

With UNDO, it will also be possible to implement savepoints to allow partial rollback of invalid transaction operations (parser errors caused by mistyping commands, insertion of duplicate primary/unique keys and so on) with the ability to continue or commit valid operations made by the transaction before the error. At present, any error will invalidate the whole transaction and require a transaction abort.

WAL offers the opportunity for a new method for database on-line backup and restore (BAR). To use this method, one would have to make periodic saves of data files to another disk, a tape or another host and also archive the WAL log files. The database file copy and the archived log files could be used to restore just as if one were restoring after a crash. Each time a new database file copy was made the old log files could be removed. Implementing this facility will require the logging of data file and index creation and deletion; it will also require development of a method for copying the data files (operating system copy commands are not suitable).
