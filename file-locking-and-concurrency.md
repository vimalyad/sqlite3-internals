# SQLite Concurrency & Journaling

## The Reader-Writer Problem

When readers continuously hold locks on a shared resource, writers are starved of access. This typically occurs when no proper solution to the problem has been implemented.

SQLite's newer locking and journaling mechanism was designed to improve concurrency, reduce writer starvation, and support atomic commits across transactions involving multiple database files.

---

## The Pager Module

The Pager module is responsible for making SQLite ACID — Atomic, Consistent, Isolated, and Durable.

It ensures that changes happen all at once: either all of them occur, or none do. It prevents two or more processes from accessing the database in incompatible ways simultaneously, and guarantees that once changes are written, they persist until explicitly deleted.

The Pager also maintains an in-memory cache of portions of the disk file.

---

## Locking

SQLite uses five distinct locking states to manage concurrent access.

**UNLOCKED** — No locks are held by anyone. Cached data is not treated as correct until verified. No process has yet claimed any form of access.

**SHARED** — The database may be read but not written. Multiple readers may hold a SHARED lock simultaneously. A write is only permitted once no SHARED locks remain.

**RESERVED** — A process intends to write in the future but has not yet begun. Only one RESERVED lock may exist at a time, but it can coexist with multiple SHARED locks. Unlike PENDING, new SHARED locks may still be acquired while a RESERVED lock is active.

**PENDING** — A process is waiting to write as soon as all existing SHARED locks are released, at which point it will escalate to EXCLUSIVE. While a PENDING lock is active, no new SHARED locks are granted, though existing ones are allowed to finish.

**EXCLUSIVE** — Required for any write operation. Only one EXCLUSIVE lock is permitted at a time, and no other lock of any kind may coexist with it. SQLite minimizes the duration of EXCLUSIVE locks to keep concurrency as high as possible.

The operating system interface layer tracks all five states. The Pager module tracks only four, as the PENDING state is transient.

---

## Rollback Journal

The rollback journal is an ordinary disk file located in the same directory as the database file. Its name matches the database file with a `-journal` suffix appended.

It records the initial size of the database so that, if the database grows during a transaction, it can be restored to its original size on rollback.

---

## Super Journal

When multiple databases are used together via the `ATTACH` command, each database maintains its own rollback journal along with a string that stores the name of the super journal.

A super journal is an aggregate journal file that holds the names of all rollback journals involved in a cross-database transaction. It is created whenever a transaction spans multiple databases, or when the databases involved are not attached to one another.

---

## Hot Journals

If an application or host machine crashes mid-write, the journal or write-ahead log may be left on disk. This file contains the information needed to restore the database to a consistent state.

A hot journal is only created when a write is in progress at the time of the crash. Since this is relatively rare, hot journals are seldom encountered in practice.

A journal is considered hot if all of the following are true:

- It exists on disk.
- Its size exceeds 512 bytes.
- Its header is non-zero and well-formed.
- Its super journal exists, or the super journal name is an empty string.
- No RESERVED lock is held on the corresponding database file.

Before reading from a database file, SQLite always checks for the presence of a hot journal. If one is found, it is used to roll the database back to a consistent state before any read proceeds
