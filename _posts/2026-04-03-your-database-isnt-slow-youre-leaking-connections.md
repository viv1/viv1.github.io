---
layout: post
title: "Your Database Isn't Slow, You're Leaking Connections"
date: 2026-04-04 16:00
category: Databases
tags: ["python", "postgres", "pgbouncer", "connection-pooling", "connection-leak", "debugging", "tcp", "linux", "performance", "psycopg2", "file-descriptors"]
description: "A deep dive into what a database connection really is at every level, how connection leaks happen, how to diagnose them from application to TCP layer, and a hands-on playground to see it all in action."
---

- [Introduction](#introduction)
- [What Is a Database Connection, Really?](#what-is-a-database-connection-really)
   * [At the OS Level: File Descriptors](#at-the-os-level-file-descriptors)
   * [At the Network Level: TCP](#at-the-network-level-tcp)
   * [At the Postgres Level: Backend Processes](#at-the-postgres-level-backend-processes)
   * [The Full Startup Handshake](#the-full-startup-handshake)
   * [What "Closing" a Connection Means](#what-closing-a-connection-means)
- [Why Connection Pools Exist](#why-connection-pools-exist)
   * [PgBouncer](#pgbouncer)
   * [PgBouncer Pooling Modes](#pgbouncer-pooling-modes)
- [What a Connection Leak Is](#what-a-connection-leak-is)
- [How Connection Leaks Happen in Python](#how-connection-leaks-happen-in-python)
- [Diagnosing: Application Level](#diagnosing-application-level)
- [Diagnosing: PgBouncer Level](#diagnosing-pgbouncer-level)
- [Diagnosing: Database Level](#diagnosing-database-level)
- [Diagnosing: OS and Network Level](#diagnosing-os-and-network-level)
- [Fixing Connection Leaks](#fixing-connection-leaks)
- [Experiment: See It Happen](#experiment-see-it-happen)
- [Conclusion](#conclusion)
- [References](#references)

## Introduction

Here's a pattern I have seen play out too many times. An app works fine for hours, sometimes days. Then database operations start timing out. Someone restarts the app, everything goes back to normal. A few hours later, same thing. The team starts talking about increasing the connection pool size, maybe upgrading the database instance.

But the app only has a few hundred users. It should not need that many connections. Something is being held and not released.

More often than not, the problem is a connection leak. And before you can understand how connections leak, you need to understand what a connection actually is. Not the abstract object your ORM gives you, but what is happening at every level of the stack.

> The examples in this article use Python with `psycopg2` (the most widely used PostgreSQL adapter for Python) and Postgres, but the underlying concepts apply to any language and database. Connection leaks look the same whether you are writing Java with HikariCP, Go with `database/sql`, or Node.js with `pg`. The TCP sockets, file descriptors, and database backend processes behave identically.
{:.prompt-info}

## What Is a Database Connection, Really?

When your Python code calls `psycopg2.connect()`, a lot happens under the hood. Let's trace through every layer.

### At the OS Level: File Descriptors

Everything in Unix is a file, and a network connection is no exception. When your Python process wants to connect to Postgres, the kernel creates a **socket**, which is just a special type of file descriptor.

A file descriptor is an integer that the kernel assigns to represent an open I/O resource. Your process has a table of these:

```
fd 0  -->  stdin
fd 1  -->  stdout
fd 2  -->  stderr
fd 3  -->  /var/log/app.log
fd 4  -->  TCP socket to 10.0.0.5:5432   <-- this is your DB connection
fd 5  -->  TCP socket to 10.0.0.5:5432   <-- another DB connection
```

You can see these in real time for any process:

```bash
# list all file descriptors for a process
ls -la /proc/<pid>/fd

# count them
ls /proc/<pid>/fd | wc -l

# see just the network sockets
ls -la /proc/<pid>/fd | grep socket
```

There are limits on how many file descriptors a process can have:

```bash
# per-process limit
ulimit -n
# typically 1024 or 65535

# system-wide limit
cat /proc/sys/fs/file-max
```

This is the first reason why connection leaks matter. Every leaked connection is a file descriptor that never gets released. Leak enough of them and your process hits its fd limit. After that, it can't open new files, new sockets, nothing. The process is effectively dead.

### At the Network Level: TCP

The file descriptor is the local handle, but the connection itself is a TCP session between your app and Postgres. This starts with the three-way handshake:

```
  Python app (client)                          PostgreSQL (server)
        |                                            |
        |  ---- SYN (seq=100) ---------------------> |
        |                                            |
        |  <--- SYN-ACK (seq=300, ack=101) --------- |
        |                                            |
        |  ---- ACK (seq=101, ack=301) ------------> |
        |                                            |
        |        [ TCP connection established ]       |
        |                                            |
```

At this point, the kernel on both sides creates a socket data structure. It allocates send and receive buffers (typically 128KB each). The connection is tracked by a 4-tuple: `(src_ip, src_port, dst_ip, dst_port)`.

You can see every TCP connection on your system:

```bash
# show all connections to Postgres
ss -tnp | grep 5432
```

The output tells you the state, buffer sizes, and which process owns each connection. A healthy system has connections that come and go. A leaking system has ESTABLISHED connections that pile up and never close.

### At the Postgres Level: Backend Processes

This is where it gets expensive. Postgres uses a **process-per-connection** model. When your TCP connection completes and the startup handshake finishes, the Postgres `postmaster` process forks a new child process to handle this connection.

```
  postmaster (PID 1)
    |
    |-- postgres: user1 mydb idle          (PID 100)   <-- your connection
    |-- postgres: user1 mydb idle          (PID 101)   <-- another connection
    |-- postgres: user2 mydb active        (PID 102)
    |-- background writer                  (PID 10)
    |-- walwriter                          (PID 11)
    |-- autovacuum launcher                (PID 12)
```

Each backend process is a full OS process with its own memory allocation. On a typical setup, each connection consumes somewhere around 5-10MB of memory. This is why Postgres has a `max_connections` setting (default 100), and why you don't want to set it to 1000. 1000 connections means 1000 forked processes, each with its own memory, all context-switching on the same CPU cores.

You can see these directly:

```bash
ps aux | grep "postgres:"
```

Or from within Postgres:

```sql
SELECT pid, usename, application_name, state, query, state_change
FROM pg_stat_activity
WHERE backend_type = 'client backend';
```

### The Full Startup Handshake

After TCP is established, Postgres has its own application-level protocol:

```
  Python app                                        PostgreSQL
       |                                                  |
       |  ---- StartupMessage --------------------------> |
       |       (protocol version, user, database)         |
       |                                                  |
       |  <--- AuthenticationMD5Password ----------------- |
       |                                                  |
       |  ---- PasswordMessage -------------------------> |
       |       (MD5 hash of password + salt)              |
       |                                                  |
       |  <--- AuthenticationOk -------------------------- |
       |                                                  |
       |  <--- ParameterStatus (server_version) ---------- |
       |  <--- ParameterStatus (client_encoding) --------- |
       |  <--- ParameterStatus (DateStyle) --------------- |
       |  <--- BackendKeyData (PID, secret key) ---------- |
       |  <--- ReadyForQuery ('I' = idle) ---------------- |
       |                                                  |
       |       [ Connection ready for queries ]           |
```

This whole exchange happens every time you open a connection. TCP handshake, authentication, parameter negotiation. It is not instantaneous. On a local machine it takes a few milliseconds. Over a network with TLS, it can be 10-50ms. That adds up when you are doing it on every request.

### What "Closing" a Connection Means

When you properly close a connection, this is what happens:

```
  Python app                                        PostgreSQL
       |                                                  |
       |  ---- Terminate message -----------------------> |
       |                                                  |  Backend process exits
       |  ---- FIN -------------------------------------> |
       |  <--- FIN-ACK ---------------------------------- |
       |  <--- FIN -------------------------------------- |
       |  ---- ACK -------------------------------------> |
       |                                                  |
       |       [ TCP connection closed ]                  |
       |       [ File descriptor released ]               |
       |       [ Postgres backend process gone ]          |
```

The Terminate message tells Postgres to clean up. The FIN/FIN-ACK exchange tears down the TCP session. The kernel frees the socket buffers. The file descriptor is released back to the process's fd table. On the Postgres side, the backend process exits, its memory is freed.

When a connection leaks, none of this happens. The TCP socket stays ESTABLISHED. The Postgres backend process sits there consuming memory. The file descriptor stays allocated. The kernel buffers stay allocated. Everything just sits there, doing nothing, waiting for a close that never comes.

## Why Connection Pools Exist

Opening a connection is expensive. You saw the full sequence above: TCP handshake, authentication, parameter exchange, Postgres forking a process. Closing is not free either. Doing this on every database operation is wasteful.

A connection pool keeps a set of connections open and reuses them. Your application borrows a connection, runs a query, returns it to the pool. The expensive setup happens once, not on every request.

```
  Without pool:

  Request 1:  [ TCP + Auth + Fork ]  ->  Query  ->  [ Terminate + FIN ]
  Request 2:  [ TCP + Auth + Fork ]  ->  Query  ->  [ Terminate + FIN ]
  Request 3:  [ TCP + Auth + Fork ]  ->  Query  ->  [ Terminate + FIN ]

  Every request pays the full cost.


  With pool:

  Startup:    [ TCP + Auth + Fork ] x pool_size     (one-time cost)

  Request 1:  borrow conn-2  ->  Query  ->  return conn-2
  Request 2:  borrow conn-1  ->  Query  ->  return conn-1
  Request 3:  borrow conn-2  ->  Query  ->  return conn-2

  Connections reused. No setup/teardown per request.
```

### PgBouncer

In Python web applications, you typically run multiple worker processes (gunicorn, uvicorn). Each worker is its own OS process with its own memory space. They can't share connections across processes.

If you have 4 gunicorn workers, each with a pool of 10 connections, that's 40 connections to Postgres. Scale to 10 workers and you are at 100 connections, which is Postgres's default `max_connections`. And you haven't even added connections for migrations, monitoring, cron jobs, or your colleague running queries in psql.

This is where PgBouncer comes in. It sits between your application and Postgres as a lightweight connection proxy. Your application connects to PgBouncer (which is cheap, just a few KB per connection), and PgBouncer maintains a much smaller pool of actual Postgres connections.

```
  gunicorn worker-1 ---\
  gunicorn worker-2 ----+---> PgBouncer ----> PostgreSQL
  gunicorn worker-3 ---/     (20 server      (20 backend
  gunicorn worker-4 ---/      connections)     processes)

  40 client connections multiplexed into 20 server connections.
```

### PgBouncer Pooling Modes

PgBouncer has three modes that control when a server connection is assigned to a client and when it is returned to the pool:

**Session mode**: Client gets a server connection when it connects. Keeps it until the client disconnects. This is the safest mode but offers the least connection sharing. Basically the same as connecting directly to Postgres but with the ability to queue clients when `max_connections` is reached.

**Transaction mode**: Client gets a server connection when it runs `BEGIN`. Connection is returned when the transaction ends (`COMMIT` or `ROLLBACK`). Between transactions, the client has no server connection. This is the most commonly used mode.

**Statement mode**: Client gets a server connection for a single statement. Connection is returned immediately after the statement completes. Multi-statement transactions are not supported. Rarely used in practice.

> Transaction mode is what most production setups use. It gives the best balance between connection reuse and compatibility. But it means you cannot use session-level features like `SET` statements, prepared statements, or `LISTEN/NOTIFY` across transactions.
{:.prompt-info}

## What a Connection Leak Is

A connection leak is when a connection is borrowed from the pool and never returned. The pool thinks it is in use. Postgres sees it as idle. The TCP socket stays open. The file descriptor stays allocated. The backend process sits there consuming memory.

The insidious part is how gradual it is:

```
  Time 0:   Pool [ conn-1: idle ] [ conn-2: idle ] [ conn-3: idle ]
            Available: 3/3

  Time 1:   App borrows conn-1, queries, returns it.
            Pool [ conn-1: idle ] [ conn-2: idle ] [ conn-3: idle ]
            Available: 3/3  (healthy)

  Time 2:   App borrows conn-2, exception thrown, no close.
            Pool [ conn-1: idle ] [ conn-2: LEAKED ] [ conn-3: idle ]
            Available: 2/3

  Time 3:   App borrows conn-3, same bug.
            Pool [ conn-1: idle ] [ conn-2: LEAKED ] [ conn-3: LEAKED ]
            Available: 1/3

  Time 4:   App borrows conn-1, same bug.
            Pool [ conn-1: LEAKED ] [ conn-2: LEAKED ] [ conn-3: LEAKED ]
            Available: 0/3

  Time 5:   App tries to borrow. Pool empty. Waits for timeout.
            "QueuePool limit of size 3 overflow 0 reached,
             connection timed out"
```

With a pool of 20 connections and a leak rate of 1 per hour, your app runs fine for 20 hours. Then everything falls over. Restart the app, pool resets, cycle repeats. This is why connection leaks are hard to catch in testing. They don't show up in a 5-minute test run.

## How Connection Leaks Happen in Python

Before we look at the leak patterns, a quick primer on `psycopg2` if you haven't used it.

`psycopg2` is a Python adapter for PostgreSQL. When you call `psycopg2.connect(...)`, it establishes a TCP connection to Postgres, completes the authentication handshake, and returns a **connection object**. This is the object that represents everything we discussed above: a file descriptor, a TCP socket, and a Postgres backend process on the other end.

From the connection, you create a **cursor** using `conn.cursor()`. A cursor is what you use to execute SQL queries and fetch results. Think of the connection as the pipe to the database, and the cursor as the thing you send queries through. A single connection can have multiple cursors, but they all share the same underlying TCP connection and transaction state.

By default, `psycopg2` sets `autocommit = False`. This means every query you execute is wrapped in a transaction. Postgres sees a `BEGIN` before your first query, and the transaction stays open until you explicitly call `conn.commit()` or `conn.rollback()`. If you set `conn.autocommit = True`, each query is committed immediately and no transaction is held open. This distinction matters a lot for connection leaks because a connection with an open transaction is in `idle in transaction` state, which holds locks and consumes more resources than a plain `idle` connection.

### The Obvious Case: No Close

```python
import psycopg2

def get_user(user_id):
    conn = psycopg2.connect("dbname=mydb user=myuser")
    cur = conn.cursor()
    cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    user = cur.fetchone()
    # forgot conn.close() and cur.close()
    return user
```

Every call to `get_user()` opens a connection and never closes it. With 100 users hitting this endpoint, you burn through 100 connections in no time.

### The Exception Path

This is more common and harder to spot:

```python
def transfer_money(from_id, to_id, amount):
    conn = psycopg2.connect("dbname=mydb user=myuser")
    cur = conn.cursor()
    cur.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s",
                (amount, from_id))
    cur.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s",
                (amount, to_id))
    conn.commit()
    cur.close()
    conn.close()  # this line never runs if an exception is thrown above
```

If the second UPDATE fails (say, `to_id` doesn't exist), the exception propagates up. `conn.close()` never executes. The connection is now leaked, and worse, it is stuck in an open transaction holding locks.

The fix is to use `try/finally` so `conn.close()` always runs, even if an exception is thrown:

```python
def transfer_money(from_id, to_id, amount):
    conn = psycopg2.connect("dbname=mydb user=myuser")
    try:
        with conn.cursor() as cur:
            cur.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s",
                        (amount, from_id))
            cur.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s",
                        (amount, to_id))
            conn.commit()
    finally:
        conn.close()  # always runs, even on exception
```

> With `psycopg2`, the `with` block on a connection does **not** close the connection. It only manages the transaction (commits on success, rolls back on exception). You still need to explicitly call `conn.close()` or wrap it in `try/finally`. `psycopg3` fixed this, the `with` block does close the connection.
{:.prompt-info}

### Idle in Transaction

This is one of the worst kinds of leaks because it holds locks:

```python
# using psycopg2's built-in connection pool
from psycopg2 import pool

db_pool = pool.SimpleConnectionPool(1, 10, "dbname=mydb user=myuser")

conn = db_pool.getconn()
cur = conn.cursor()
cur.execute("SELECT * FROM orders WHERE status = 'pending' FOR UPDATE")
# ... some processing that takes a long time or crashes
# no COMMIT, no ROLLBACK, no close
```

The connection is now in `idle in transaction` state. It holds row-level locks on every matching row in the `orders` table. Other transactions trying to update those rows will block and wait. You haven't just leaked a connection, you have blocked other parts of your application.

### SQLAlchemy Session Not Closed

[SQLAlchemy](https://www.sqlalchemy.org/){:target="_blank"} is a popular Python library that provides connection pooling and an ORM (Object Relational Mapper) on top of database adapters like `psycopg2`. Instead of writing raw SQL, you work with Python objects. It manages its own connection pool internally, which means leaked sessions translate directly to leaked connections from the pool.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

engine = create_engine("postgresql://user:pass@localhost/mydb", pool_size=5)

def get_orders():
    session = Session(engine)
    orders = session.query(Order).filter_by(status='pending').all()
    return orders
    # session never closed, connection never returned to pool
```

The session holds a connection from the pool. Since it is never closed, the connection is never returned. Every call to `get_orders()` checks out another connection.

### Background Thread Leak

```python
import threading
import psycopg2

def process_batch(batch_id):
    conn = psycopg2.connect("dbname=mydb user=myuser")
    cur = conn.cursor()
    cur.execute("SELECT * FROM batches WHERE id = %s", (batch_id,))
    # process...
    # thread finishes, but conn is never closed
    # Python's garbage collector MIGHT clean it up, eventually, maybe

for i in range(100):
    t = threading.Thread(target=process_batch, args=(i,))
    t.start()
```

Each thread opens a connection. When the thread finishes, the connection object goes out of scope but it is not guaranteed to be closed immediately. Python's garbage collector will get to it eventually, but "eventually" could be a long time, and by then you may have already exhausted the pool or hit the Postgres connection limit.

## Diagnosing: Application Level

### SQLAlchemy Pool Status

If you are using SQLAlchemy, the pool exposes useful metrics:

```python
from sqlalchemy import create_engine

engine = create_engine("postgresql://user:pass@localhost/mydb",
                       pool_size=5, max_overflow=0)

# check pool status
print(f"Pool size: {engine.pool.size()}")
print(f"Checked out: {engine.pool.checkedout()}")
print(f"Checked in: {engine.pool.checkedin()}")
print(f"Overflow: {engine.pool.overflow()}")
```

In a healthy app, `checkedout()` should fluctuate near zero. If it keeps climbing and never goes down, you have a leak.

### SQLAlchemy Connection Checkout Events

SQLAlchemy has an event system that lets you log every checkout and checkin:

```python
from sqlalchemy import event
import traceback
import time

@event.listens_for(engine, "checkout")
def on_checkout(dbapi_conn, connection_record, connection_proxy):
    connection_record.info['checkout_time'] = time.time()
    connection_record.info['checkout_stack'] = traceback.format_stack()

@event.listens_for(engine, "checkin")
def on_checkin(dbapi_conn, connection_record):
    checkout_time = connection_record.info.get('checkout_time')
    if checkout_time:
        duration = time.time() - checkout_time
        if duration > 30:  # connection held for more than 30 seconds
            print(f"Connection held for {duration:.1f}s")
            print("Checked out from:")
            print("".join(connection_record.info['checkout_stack']))
```

This logs a warning with the full stack trace whenever a connection is held for too long. The stack trace tells you exactly which function checked out the connection and didn't return it.

## Diagnosing: PgBouncer Level

PgBouncer has an admin console you can connect to like a regular database:

```bash
psql -h 127.0.0.1 -p 6432 -U pgbouncer pgbouncer
```

### SHOW POOLS

```sql
SHOW POOLS;
```

```
 database |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait
----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------
 mydb     | myuser   |        15 |          5 |        10 |      0  |       0 |         0 |        0 |      12
```

The columns that matter:
- `cl_active`: Client connections currently linked to a server connection. These are doing work.
- `cl_waiting`: Clients waiting for a server connection. If this is non-zero, your pool is saturated.
- `sv_active`: Server connections currently in use.
- `sv_idle`: Server connections sitting idle in the pool, available for reuse.
- `maxwait`: How long the oldest waiting client has been waiting (seconds).

In a healthy system, `cl_waiting` is 0 and `sv_idle` is greater than 0. If `cl_waiting` keeps growing and `sv_idle` is 0, connections are being held and not returned.

### SHOW CLIENTS

```sql
SHOW CLIENTS;
```

Shows every client connection with its state, which database it is using, how long it has been connected, and which server connection it is linked to. Look for clients that have been connected for an unusually long time.

### SHOW SERVERS

```sql
SHOW SERVERS;
```

Shows the PgBouncer-to-Postgres connections. Look for server connections in `active` state for extended periods. These correspond to connections that a client has borrowed but hasn't returned.

## Diagnosing: Database Level

### pg_stat_activity

This is the single most useful view for debugging connection issues in Postgres.

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    state_change,
    now() - state_change AS state_duration,
    wait_event_type,
    wait_event,
    left(query, 80) AS query
FROM pg_stat_activity
WHERE backend_type = 'client backend'
ORDER BY state_duration DESC;
```

The `state` column tells you what each connection is doing:

- `active`: Running a query right now.
- `idle`: Connected, not doing anything, waiting for the next query.
- `idle in transaction`: Inside an open transaction (`BEGIN` was sent) but not currently running a query. This is often the smoking gun for a leak.
- `idle in transaction (aborted)`: Same as above but the transaction hit an error. It is stuck until a `ROLLBACK` is issued.

**What to look for**: Connections in `idle in transaction` with a `state_duration` of minutes or hours. These are almost certainly leaked. Something opened a transaction and never committed, rolled back, or closed the connection.

### Quick Health Check

```sql
SELECT state, count(*)
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY state;
```

```
        state         | count
----------------------+-------
 active               |     3
 idle                 |     7
 idle in transaction  |    15
```

If `idle in transaction` is the largest group, you have a problem.

### Who Is Blocking Whom

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    now() - blocked.state_change AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks kl ON kl.locktype = bl.locktype
    AND kl.database IS NOT DISTINCT FROM bl.database
    AND kl.relation IS NOT DISTINCT FROM bl.relation
    AND kl.page IS NOT DISTINCT FROM bl.page
    AND kl.tuple IS NOT DISTINCT FROM bl.tuple
    AND kl.pid != bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid
WHERE NOT bl.granted;
```

This shows you which connection is blocked and which connection is blocking it. If the blocking connection is `idle in transaction`, it is holding a lock without doing anything, which is a leaked connection.

## Diagnosing: OS and Network Level

When you don't have access to the application logs, or you want to verify what the app is reporting, go to the OS.

### TCP Connections

```bash
# all connections to Postgres port
ss -tnp | grep 5432
```

```
ESTAB  0  0  10.0.0.2:45678  10.0.0.5:5432  users:(("python",pid=1234,fd=4))
ESTAB  0  0  10.0.0.2:45679  10.0.0.5:5432  users:(("python",pid=1234,fd=5))
ESTAB  0  0  10.0.0.2:45680  10.0.0.5:5432  users:(("python",pid=1234,fd=6))
...
```

Count them:

```bash
ss -tnp | grep 5432 | wc -l
```

If your pool max is 20 and you see 50 ESTABLISHED connections, something outside the pool is creating connections (or the pool itself is misconfigured).

### File Descriptors

```bash
# count open fds for your Python process
ls /proc/<pid>/fd | wc -l

# see which ones are sockets
ls -la /proc/<pid>/fd | grep socket

# get the socket details
cat /proc/<pid>/net/tcp
```

### lsof

```bash
# show all connections to Postgres
lsof -i :5432 -n -P
```

This shows the process name, PID, fd number, and connection state for every connection to port 5432.

### tcpdump

This is the lowest level. You can watch connections being opened and (not) closed in real time:

```bash
# watch for new connections (SYN packets)
tcpdump -i any port 5432 and 'tcp[tcpflags] & tcp-syn != 0'

# watch for connection closes (FIN packets)
tcpdump -i any port 5432 and 'tcp[tcpflags] & tcp-fin != 0'
```

In a healthy system, you see SYN packets when the pool starts up, and then very few after that (connections are reused). If you see a steady stream of SYN packets but very few FIN packets, connections are being opened and never closed. That is a leak visible at the packet level.

## Fixing Connection Leaks

### Always Use Context Managers

This is the most important fix. Make it impossible to forget to close:

```python
# psycopg2 - note: with block manages transaction, not connection lifetime
import psycopg2

conn = psycopg2.connect("dbname=mydb user=myuser")
try:
    with conn:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM users")
            rows = cur.fetchall()
finally:
    conn.close()

# psycopg3 - with block closes the connection
import psycopg

with psycopg.connect("dbname=mydb user=myuser") as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM users")
        rows = cur.fetchall()

# SQLAlchemy
from sqlalchemy import create_engine, text

engine = create_engine("postgresql://user:pass@localhost/mydb")

with engine.connect() as conn:
    result = conn.execute(text("SELECT * FROM users"))
    rows = result.fetchall()
# connection returned to pool when with block exits, even on exception
```

### Set Postgres Timeouts

Postgres can automatically kill leaked connections:

```sql
-- kill connections that are idle in transaction for more than 5 minutes
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';

-- kill connections that have been idle for more than 10 minutes
ALTER SYSTEM SET idle_session_timeout = '10min';  -- Postgres 14+

SELECT pg_reload_conf();
```

These are safety nets. They don't fix the leak, they limit the damage.

### Configure PgBouncer Timeouts

```ini
[pgbouncer]
; close server connection if it has been idle for more than 10 minutes
server_idle_timeout = 600

; close server connection if it has been connected for more than 1 hour
server_lifetime = 3600

; cancel query if it runs longer than 3 minutes
query_timeout = 180

; close client connection if it has been idle in transaction for more than 2 minutes
client_idle_timeout = 120
```

### SQLAlchemy Pool Configuration

```python
engine = create_engine(
    "postgresql://user:pass@localhost/mydb",
    pool_size=5,             # max number of persistent connections
    max_overflow=0,          # don't create connections beyond pool_size
    pool_timeout=30,         # seconds to wait for a connection before giving up
    pool_recycle=1800,       # recycle connections after 30 minutes
    pool_pre_ping=True,      # verify connection is alive before using it
)
```

`pool_pre_ping=True` is important. It sends a lightweight query (`SELECT 1`) before handing a connection to your code. If the connection is dead (server restarted, network blip, timeout killed it), it gets discarded and a fresh one is created. Without this, your code gets a dead connection and fails with a confusing error.

## Experiment: See It Happen

Let's set up a playground where you can cause a connection leak, watch it happen across every layer, and then fix it. You will need Docker and Python installed.

### Setup

Create a working directory and a virtual environment:

```bash
mkdir conn-leak-playground && cd conn-leak-playground
python3 -m venv venv
source venv/bin/activate
pip install psycopg2-binary
```

**docker-compose.yml:**

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
      POSTGRES_DB: testdb
    ports:
      - "5433:5432"
    command:
      - "postgres"
      - "-c"
      - "max_connections=25"
      - "-c"
      - "log_connections=on"
      - "-c"
      - "log_disconnections=on"

  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DATABASE_URL: "postgres://testuser:testpass@postgres:5432/testdb"
      POOL_MODE: transaction
      DEFAULT_POOL_SIZE: 10
      MAX_CLIENT_CONN: 100
      LOG_CONNECTIONS: 1
      LOG_DISCONNECTIONS: 1
    ports:
      - "6433:5432"
    depends_on:
      - postgres
```

> We use port `5433` for Postgres and `6433` for PgBouncer to avoid conflicts with any local Postgres instance you might have running on the default port `5432`.
{:.prompt-info}

Start the services:

```bash
docker compose up -d
```

**setup_db.py** - Create a test table with some data:

```python
import psycopg2

conn = psycopg2.connect(
    host="localhost", port=5433,
    dbname="testdb", user="testuser", password="testpass"
)
conn.autocommit = True
cur = conn.cursor()

cur.execute("""
    CREATE TABLE IF NOT EXISTS orders (
        id SERIAL PRIMARY KEY,
        status VARCHAR(20),
        amount DECIMAL(10,2),
        created_at TIMESTAMP DEFAULT now()
    )
""")

cur.execute("DELETE FROM orders")
for i in range(1000):
    cur.execute(
        "INSERT INTO orders (status, amount) VALUES (%s, %s)",
        ('pending' if i % 3 == 0 else 'completed', round(10 + i * 0.5, 2))
    )

print(f"Inserted 1000 orders")
cur.close()
conn.close()
```

```bash
python setup_db.py
```

Before we start the scenarios, let's check the baseline:

```sql
SELECT state, count(*)
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY state;
```

```
 state  | count
--------+-------
 active |     1
```

Just one connection: the query we ran to check. Clean slate.

### Scenario 1: Healthy Connections

**healthy.py:**

```python
import psycopg2
import threading
import time

DB_CONFIG = dict(host="localhost", port=5433,
                 dbname="testdb", user="testuser", password="testpass")

def do_query(thread_id):
    conn = psycopg2.connect(**DB_CONFIG)
    try:
        with conn:
            with conn.cursor() as cur:
                cur.execute("SELECT count(*) FROM orders WHERE status = 'pending'")
                result = cur.fetchone()[0]
                time.sleep(0.1)  # simulate some processing
    finally:
        conn.close()
    print(f"  Thread {thread_id:2d}: got {result} pending orders, connection closed")

print("Starting 20 threads, each opens and properly closes a connection...\n")
threads = []
for i in range(20):
    t = threading.Thread(target=do_query, args=(i,))
    threads.append(t)
    t.start()
    time.sleep(0.05)  # stagger slightly

for t in threads:
    t.join()

print("\nAll threads done.")
```

Run it:

```bash
python healthy.py
```

```
Starting 20 threads, each opens and properly closes a connection...

  Thread  0: got 334 pending orders, connection closed
  Thread  1: got 334 pending orders, connection closed
  Thread  2: got 334 pending orders, connection closed
  ...
  Thread 19: got 334 pending orders, connection closed

All threads done.
```

Now check `pg_stat_activity`:

```
pg_stat_activity after all threads completed:
  active                         : 1
  Total client connections: 1
```

Back to 1 connection (our monitoring query). Every connection was opened, used, and properly closed. This is what healthy looks like.

### Scenario 2: Leaking Connections

**leak.py:**

```python
import psycopg2
import threading
import time

DB_CONFIG = dict(host="localhost", port=5433,
                 dbname="testdb", user="testuser", password="testpass")

leaked_connections = []  # prevent garbage collection

def do_query_leaky(thread_id):
    conn = psycopg2.connect(**DB_CONFIG)
    cur = conn.cursor()
    cur.execute("SELECT count(*) FROM orders WHERE status = 'pending'")
    result = cur.fetchone()[0]
    time.sleep(0.1)
    # no cur.close()
    # no conn.close()
    leaked_connections.append(conn)  # prevent GC from cleaning up
    print(f"  Thread {thread_id:2d}: got {result} pending orders, connection NOT closed")

print("Starting 20 threads, each opens a connection and NEVER closes it...\n")
threads = []
for i in range(20):
    t = threading.Thread(target=do_query_leaky, args=(i,))
    threads.append(t)
    t.start()
    time.sleep(0.05)

for t in threads:
    t.join()

print(f"\nAll threads done. {len(leaked_connections)} connections leaked.")
```

Run it:

```bash
python leak.py
```

```
Starting 20 threads, each opens a connection and NEVER closes it...

  Thread  0: got 334 pending orders, connection NOT closed
  Thread  1: got 334 pending orders, connection NOT closed
  ...
  Thread 19: got 334 pending orders, connection NOT closed

All threads done. 20 connections leaked.
```

Now check `pg_stat_activity`:

```
pg_stat_activity after all threads completed:
  active                         : 1
  idle in transaction            : 20
  Total client connections: 21
```

There they are. 20 connections sitting in `idle in transaction`, doing absolutely nothing. Notice it says `idle in transaction` and not just `idle`. This is because `psycopg2` has `autocommit = False` by default, so every query implicitly starts a transaction. Since we never called `conn.commit()` or `conn.rollback()`, those transactions are still open. Each leaked connection is holding a Postgres backend process, a TCP socket, kernel buffers, and a file descriptor.

You can see the individual leaked connections by querying `pg_stat_activity`:

```sql
SELECT pid, state, now() - state_change AS duration, left(query, 60) AS query
FROM pg_stat_activity
WHERE backend_type = 'client backend' AND pid != pg_backend_pid()
ORDER BY state_change
LIMIT 10;
```

```
   PID  State                 Duration         Query
   ---  -----                 --------         -----
   113  idle in transaction   0:00:02.161657   SELECT count(*) FROM orders WHERE status = 'pending'
   114  idle in transaction   0:00:02.119119   SELECT count(*) FROM orders WHERE status = 'pending'
   115  idle in transaction   0:00:02.065612   SELECT count(*) FROM orders WHERE status = 'pending'
   ...
```

The `query` column shows the last query each connection ran. The `duration` column shows how long it has been stuck in that state. In a real production system, this duration would be hours, not seconds.

Now what happens when we try to open more connections? Remember, we set `max_connections=25` in our Postgres config, and Postgres reserves a few connections for superuser and background processes. With 20 connections already leaked, we are at the limit. In production, this is the moment your app starts throwing connection errors and users start seeing timeouts.

### Scenario 3: Direct vs PgBouncer

Let's see what happens when we leak connections through PgBouncer instead of directly to Postgres. We just saw that 20 leaked direct connections created 20 Postgres backend processes. Does PgBouncer help?

It depends on whether the leaked connections have open transactions.

#### 3a: PgBouncer with autocommit=True (no open transactions)

With `autocommit = True`, each query is its own transaction that completes immediately. In PgBouncer's transaction mode, the server connection is released back to the pool as soon as the transaction ends. The client connection to PgBouncer is still leaked, but PgBouncer is no longer tying up a Postgres backend for it.

**leak_pgbouncer_autocommit.py:**

```python
import psycopg2
import threading
import time

# Connect through PgBouncer (port 6433) instead of directly to Postgres
DB_CONFIG = dict(host="localhost", port=6433,
                 dbname="testdb", user="testuser", password="testpass")

leaked_connections = []

def do_query_leaky(thread_id):
    conn = psycopg2.connect(**DB_CONFIG)
    conn.autocommit = True  # no open transaction, PgBouncer can reclaim server conn
    cur = conn.cursor()
    cur.execute("SELECT count(*) FROM orders WHERE status = 'pending'")
    result = cur.fetchone()[0]
    time.sleep(0.1)
    leaked_connections.append(conn)
    print(f"  Thread {thread_id:2d}: got {result} pending orders, connection NOT closed")

print("Starting 20 threads via PgBouncer (autocommit=True)...\n")
threads = []
for i in range(20):
    t = threading.Thread(target=do_query_leaky, args=(i,))
    threads.append(t)
    t.start()
    time.sleep(0.05)

for t in threads:
    t.join()

print(f"\nAll threads done. {len(leaked_connections)} connections leaked.")
```

```bash
python leak_pgbouncer_autocommit.py
```

Check `pg_stat_activity` on Postgres:

```
pg_stat_activity (via PgBouncer, autocommit=True):
  idle                           : 1
  Total Postgres backend processes: 2
  Client connections leaked: 20
```

20 client connections leaked, but only 2 Postgres backend processes (1 idle server connection + our monitoring query). Compare that to Scenario 2 where the same 20 leaked connections created 21 Postgres backend processes.

PgBouncer absorbed the damage. The leaked client connections are sitting in PgBouncer's memory (a few KB each), not consuming Postgres backend processes at 5-10MB each.

#### 3b: PgBouncer with autocommit=False (open transactions)

Now the same thing but with `autocommit = False` (psycopg2's default). Each leaked connection holds an open transaction. In transaction mode, PgBouncer keeps the server connection assigned to that client until the transaction ends. Since the transaction never ends (no commit, no rollback, no close), the server connection is stuck.

PgBouncer has `DEFAULT_POOL_SIZE=10` server connections. What happens when we try to leak 15?

**leak_pgbouncer_no_autocommit.py:**

```python
import psycopg2
import threading
import time

DB_CONFIG = dict(host="localhost", port=6433,
                 dbname="testdb", user="testuser", password="testpass")

leaked_connections = []

def do_query_leaky(thread_id):
    conn = psycopg2.connect(**DB_CONFIG, connect_timeout=8)
    # autocommit=False is the default - each query opens a transaction
    cur = conn.cursor()
    cur.execute("SELECT count(*) FROM orders WHERE status = 'pending'")
    result = cur.fetchone()[0]
    time.sleep(0.1)
    leaked_connections.append(conn)
    print(f"  Thread {thread_id:2d}: got {result}, leaked with OPEN TRANSACTION")

print("Starting 15 threads via PgBouncer (autocommit=False)...")
print("PgBouncer pool size: 10 server connections\n")
threads = []
for i in range(15):
    t = threading.Thread(target=do_query_leaky, args=(i,))
    threads.append(t)
    t.start()
    time.sleep(0.3)

# Wait, but don't wait forever
for t in threads:
    t.join(timeout=15)

stuck = sum(1 for t in threads if t.is_alive())
print(f"\nResults:")
print(f"  Connections leaked: {len(leaked_connections)}")
print(f"  Threads still stuck waiting: {stuck}")
```

```bash
python leak_pgbouncer_no_autocommit.py
```

```
Starting 15 threads via PgBouncer (autocommit=False)...
PgBouncer pool size: 10 server connections

  Thread  0: got 334, leaked with OPEN TRANSACTION
  Thread  1: got 334, leaked with OPEN TRANSACTION
  ...
  Thread  9: got 334, leaked with OPEN TRANSACTION

Results:
  Connections leaked: 10
  Threads still stuck waiting: 5
```

The first 10 threads succeed and leak their connections with open transactions. That exhausts all 10 of PgBouncer's server connections. Threads 10-14 are stuck. They connected to PgBouncer (which is cheap), but when they try to run a query, PgBouncer needs to assign a server connection and there are none available. Those threads hang indefinitely, waiting for a server connection that will never be released.

Check `pg_stat_activity`:

```
pg_stat_activity (via PgBouncer, autocommit=False):
  idle in transaction            : 10
  Total Postgres backend processes: 11
```

All 10 server connections are tied up with `idle in transaction`. PgBouncer is fully saturated. It can't help anyone.

This is the important takeaway: **PgBouncer only helps if transactions are short-lived.** If your leaked connections hold open transactions, PgBouncer's server connections get stuck just like direct Postgres connections. You end up with the same problem, just with a different bottleneck (PgBouncer's pool instead of Postgres's `max_connections`).

### Scenario 4: Leaking Connections in Transactions (With Lock Contention)

This is the nastier variant. Instead of just leaking idle connections, we leak connections that are inside an open transaction holding row locks.

**leak_transaction.py:**

```python
import psycopg2
import threading
import time

DB_CONFIG = dict(host="localhost", port=5433,
                 dbname="testdb", user="testuser", password="testpass")

leaked_connections = []

def do_query_leaky_txn(thread_id):
    conn = psycopg2.connect(**DB_CONFIG)
    cur = conn.cursor()
    # BEGIN is implicit with psycopg2 (autocommit is off by default)
    # Each thread locks a different set of rows to avoid blocking each other
    start_id = thread_id * 5 + 1
    end_id = start_id + 4
    cur.execute("SELECT * FROM orders WHERE id BETWEEN %s AND %s FOR UPDATE",
                (start_id, end_id))
    rows = cur.fetchall()
    time.sleep(0.1)
    # no COMMIT, no ROLLBACK, no close
    # connection is now "idle in transaction" AND holding row locks
    leaked_connections.append(conn)
    print(f"  Thread {thread_id:2d}: locked rows {start_id}-{end_id}, txn NOT committed")

print("Starting 10 threads, each locks rows and never commits...\n")
threads = []
for i in range(10):
    t = threading.Thread(target=do_query_leaky_txn, args=(i,))
    threads.append(t)
    t.start()
    time.sleep(0.05)

for t in threads:
    t.join()

print(f"\n{len(leaked_connections)} connections leaked in open transactions.")
```

Run it:

```bash
python leak_transaction.py
```

```
Starting 10 threads, each locks rows and never commits...

  Thread  0: locked rows 1-5, txn NOT committed
  Thread  1: locked rows 6-10, txn NOT committed
  Thread  2: locked rows 11-15, txn NOT committed
  ...
  Thread  9: locked rows 46-50, txn NOT committed

10 connections leaked in open transactions.
```

Check `pg_stat_activity`:

```
pg_stat_activity:
  active                         : 1
  idle in transaction            : 10
```

Now here's where it gets painful. Try to UPDATE those locked rows:

```python
# In another terminal
cur.execute("SET statement_timeout = '3s'")
cur.execute("UPDATE orders SET amount = amount + 1 WHERE id BETWEEN 1 AND 50")
```

```
ERROR:  canceling statement due to statement timeout
CONTEXT:  while updating tuple (0,1) in relation "orders"
```

The UPDATE timed out after 3 seconds. The leaked transactions are holding `FOR UPDATE` locks on rows 1-50 and they will never release them because nobody is going to commit or rollback those transactions. Any other part of your application that tries to modify those rows will hang until it times out.

This is the real danger of leaking connections inside transactions. It is not just about running out of connections. It is about holding locks on data that the rest of your application needs.

### Scenario 5: Leak With Safeguards

Same leak as Scenario 4, but now we configure Postgres to automatically kill connections that have been idle in a transaction for too long.

**set_safeguard.py:**

```python
import psycopg2

conn = psycopg2.connect(
    host="localhost", port=5433,
    dbname="testdb", user="testuser", password="testpass"
)
conn.autocommit = True
cur = conn.cursor()

cur.execute("ALTER SYSTEM SET idle_in_transaction_session_timeout = '10s';")
cur.execute("SELECT pg_reload_conf();")

print("Set idle_in_transaction_session_timeout to 10 seconds.")
cur.close()
conn.close()
```

```bash
python set_safeguard.py
```

Now run the same `leak_transaction.py` from Scenario 4 again.

**Immediately after the leak:**

```
pg_stat_activity IMMEDIATELY after leak:
  idle in transaction            : 10
```

Same as before. 10 connections stuck in open transactions.

**12 seconds later:**

```
pg_stat_activity AFTER timeout:
  (no client connections besides monitor)
  Total client connections: 1
```

Postgres killed all 10 leaked connections automatically. The backend processes are gone, the file descriptors are released, and most importantly the row locks are released.

Now the UPDATE that was blocked in Scenario 4 works fine:

```
Attempting UPDATE on previously locked rows...
  UPDATE succeeded! Rows updated: 50
  Postgres killed the leaked transactions, locks released.
```

This is the safety net. It does not fix your code. The leak still happens. But `idle_in_transaction_session_timeout` limits the blast radius. Instead of leaked connections accumulating until your application dies, Postgres cleans them up after a configurable timeout.

Don't forget to reset after experimenting:

```python
cur.execute("ALTER SYSTEM RESET idle_in_transaction_session_timeout;")
cur.execute("SELECT pg_reload_conf();")
```

### Cleanup

```bash
docker compose down -v
```

## Conclusion

A database that seems slow with only a few hundred users is almost never a capacity problem. Before you start tweaking pool sizes or scaling your database, check if connections are actually being returned.

Start with `pg_stat_activity`. If you see connections piling up in `idle` or `idle in transaction`, that is your answer. Trace it to your code, find the missing `close()` or the uncaught exception, and fix it.

Set up the safety nets regardless. `idle_in_transaction_session_timeout` in Postgres and timeout settings in PgBouncer won't fix your code, but they will prevent a single leaked connection from taking down your entire application.

And remember, while the code examples here are in Python, the diagnostic tools and concepts are language-agnostic. `pg_stat_activity`, `ss`, and `tcpdump` don't care what language opened the connection.

## Related Articles

If you found this useful, you might also like these:

- [Simplifying the complicated world of Libraries - Part 1](/posts/simplifying-libraries-part-1/) - A deep dive into how library binaries, static and dynamic linking, and compatibility across platforms work. Helps understand the layers below your application code.
- [Python on the web - High cost of synchronous uWSGI](/posts/high-cost-of-sync-uwsgi/) - Another case where a limited pool of resources (uWSGI workers) gets exhausted by slow operations, similar to how leaked connections exhaust your database pool.

## References

- [PostgreSQL pg_stat_activity](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ACTIVITY-VIEW){:target="_blank"}
- [PgBouncer Documentation](https://www.pgbouncer.org/config.html){:target="_blank"}
- [psycopg2 Connection Context Manager](https://www.psycopg.org/docs/connection.html#connection.autocommit){:target="_blank"}
- [psycopg3 Connection](https://www.psycopg.org/psycopg3/docs/basic/usage.html){:target="_blank"}
- [SQLAlchemy Connection Pool](https://docs.sqlalchemy.org/en/20/core/pooling.html){:target="_blank"}
- [Linux File Descriptors](https://man7.org/linux/man-pages/man2/open.2.html){:target="_blank"}
- [ss(8) - Linux Socket Statistics](https://man7.org/linux/man-pages/man8/ss.8.html){:target="_blank"}
