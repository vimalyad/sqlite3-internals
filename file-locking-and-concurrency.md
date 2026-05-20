Reader Writer Problem
─────────────────────────────────────────────────────────────────────────────
When the readers are frequently locking the resource and the writer is not
getting the chance to write. This mainly happens when a proper solution to
this problem is not implemented.

The new SQLite version introduced a new locking and journaling mechanism
designed to improve concurrency and to reduce the writer-starvation problem.
The new mechanism also allows atomic commits of transactions involving multiple
database files.

Pager Module
─────────────────────────────────────────────────────────────────────────────
It is responsible for making SQLite "ACID" (Atomic, Consistent, Isolated,
and Durable).

It ensures changes happen all at once, that either all changes occur or none
of them do, that two or more processes do not try to access the database in
incompatible ways at the same time, and that once changes have been written
they persist until explicitly deleted.

It also provides a memory cache of some of the contents of the disk file.

Locking
─────────────────────────────────────────────────────────────────────────────
UNLOCKED
No locks are held on the database by anyone. No data present in cache is
treated as correct until verified. Anyone can read/write (no one has
claimed access yet).

SHARED
The database is only allowed to be read, not written. Many different
readers are allowed to read by holding the SHARED lock. A write is only
allowed if there are no readers holding the lock.

RESERVED
It means that a process is planning on writing in the future. There is
only one RESERVED lock, but at the same time multiple SHARED locks may
coexist. RESERVED differs from PENDING in that new SHARED locks can be
acquired while there is a RESERVED lock.

PENDING
It means that a process is planning on writing as soon as all shared
locks are released so that it can get an EXCLUSIVE lock. No new shared
locks are permitted against the database if a PENDING lock is active,
though existing SHARED locks are allowed to continue.

EXCLUSIVE
It is needed to write to the database. Only one EXCLUSIVE lock is allowed
on the file and no other locks of any kind are allowed to coexist with
an EXCLUSIVE lock. In order to maximize concurrency, SQLite works to
minimize the amount of time that EXCLUSIVE locks are held.

The operating system interface layer understands and tracks all five locking
states. The pager module tracks only four of them since a PENDING lock is just
temporary.

Rollback Journal
─────────────────────────────────────────────────────────────────────────────
It is an ordinary disk file that is always located in the same directory as
the database file. It has the same name as the database file with the
addition of "-journal" suffix.

It stores the initial size of the database so, in case it grows in the
future, it can be tracked back to normal in a rollback.

Super Journal
─────────────────────────────────────────────────────────────────────────────
When we have multiple databases working together (using the ATTACH command),
each database has its own rollback journal as well as a string which stores
the name of the super journal.

A super journal is a file—specifically an aggregate journal—that contains
the names of all the database's rollback journals currently working
together. It is created when there is any transaction going on between
databases or when the databases are not ATTACHed.

Hot Journals
─────────────────────────────────────────────────────────────────────────────
If the application or the host computer crashes, the journal or write-ahead
log—which contains information needed to restore the database file to its
consistent state—may be left behind.

It is created only when there is an update going on the database and the
process or application itself crashes. Since these chances are quite low, we
almost never see them.

A journal is said to be hot if:
• It exists,
• Its size is greater than 512 bytes,
• The journal header is non-zero and well-formed,
• Its super-journal exists or the super-journal name is an empty string,
• There is no RESERVED lock on the corresponding database file.

Before reading from a database file, SQLite always checks whether there is any
hot journal. If it exists, it is used to roll back to a consistent state.
