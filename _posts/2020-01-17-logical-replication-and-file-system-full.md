---
layout: post
title:  "Resolving No space left on device errors when using Postgres replication"
comments: true
date:   2020-01-17 23:21:58 +0200
---

On a new project, one task had been to transfer continuously production data to another database, applying anonymization on-the-fly and using push-only mode.
These were internal security requirements.  
A lot of people think of Kafka for such a streaming task but, I chose Postgres replication, since the target database was Postgres and it's now avaiable in core Postgres.  
Other have written on this replication capabilities, and documentation is succinct but useful.  
Monitoring is quite limited but overall, it works.  
 
However, I recently faced a situation where suddenly, the suscriber was stuck (named here 'my_project_sub') :

    2020-01-17 11:27:22.624 CET [10560] DETAIL:  There are no running transactions.
    2020-01-17 11:27:51.609 CET [10560] ERROR:  could not write to data file for XID 1875256: No space left on device
    2020-01-17 11:27:52.521 CET [10576] LOG:  starting logical decoding for slot "my_project_sub"
    2020-01-17 11:27:52.521 CET [10576] DETAIL:  Streaming transactions committing after 20/B95CB4C8, reading WAL from 1F/65E7D700.
    2020-01-17 11:27:52.521 CET [10576] LOG:  logical decoding found consistent point at 1F/65E7D700
    2020-01-17 11:27:52.521 CET [10576] DETAIL:  There are no running transactions.
    2020-01-17 11:28:30.133 CET [10576] ERROR:  could not write to data file for XID 1669761: No space left on device
    2020-01-17 11:28:30.296 CET [10584] LOG:  starting logical decoding for slot "my_project_sub"
    2020-01-17 11:28:30.296 CET [10584] DETAIL:  Streaming transactions committing after 20/B95CB4C8, reading WAL from 1F/65E7D700.
    2020-01-17 11:28:30.299 CET [10584] LOG:  logical decoding found consistent point at 1F/65E7D700
    2020-01-17 11:28:30.299 CET [10584] DETAIL:  There are no running transactions.
    2020-01-17 11:28:58.158 CET [10584] ERROR:  could not write to data file for XID 1869710: No space left on device
    2020-01-17 11:28:59.231 CET [10602] LOG:  starting logical decoding for slot "my_project_sub"
    2020-01-17 11:28:59.231 CET [10602] DETAIL:  Streaming transactions committing after 20/B95CB4C8, reading WAL from 1F/65E7D700.
    2020-01-17 11:28:59.231 CET [10602] LOG:  logical decoding found consistent point at 1F/65E7D700
    2020-01-17 11:28:59.231 CET [10602] DETAIL:  There are no running transactions.
    2020-01-17 11:29:27.192 CET [10602] ERROR:  could not write to data file for XID 1869710: No space left on device

I had no filesystem that was full so it was at first a bit intriguing.
Data are on their own large filesystem and pg_wal is also redirected to another non full filesystem.
But it probably had to do with replication since this server had no other source of data volume spike. 

So I did the same than what is needed for `pg_wal` : using a symbolic link to redirect location of replication related folders from postgres root to another large filesystem
Here, the target folders will located on `/data/postgresql/`, wich is mapped to a large file system.

There are 2 folders that need to be linked : `pg_logical` and `replslot`

    # sudo mv pg_logical pg_logical_ori
    # sudo ln -s /data/postgresql/pg_logical /var/lib/pgsql/12/data/

chown on a symbolic link :

    # sudo chown -h postgres:postgres pg_logical

replslot is also concerned :

    # ls -alh pg_replslot/my_project_sub/

    -rw------- 1 postgres postgres  8.9M Jan 17 11:40 xid-1669761-lsn-20-70000000.spill
    -rw------- 1 postgres postgres  1.4M Jan 17 11:39 xid-1669761-lsn-20-7000000.spill
    -rw------- 1 postgres postgres   12M Jan 17 11:40 xid-1669761-lsn-20-71000000.spill
    -rw------- 1 postgres postgres   14M Jan 17 11:40 xid-1669761-lsn-20-72000000.spill
    -rw------- 1 postgres postgres   10M Jan 17 11:40 xid-1669761-lsn-20-73000000.spill
    -rw------- 1 postgres postgres  5.7M Jan 17 11:40 xid-1669761-lsn-20-74000000.spill
    -rw------- 1 postgres postgres  6.1M Jan 17 11:40 xid-1669761-lsn-20-75000000.spill
    -rw------- 1 postgres postgres  6.4M Jan 17 11:40 xid-1669761-lsn-20-76000000.spill
    -rw------- 1 postgres postgres   14M Jan 17 11:40 xid-1669761-lsn-20-77000000.spill
    -rw------- 1 postgres postgres  9.0M Jan 17 11:40 xid-1669761-lsn-20-78000000.spill
    -rw------- 1 postgres postgres   12M Jan 17 11:40 xid-1669761-lsn-20-79000000.spill
    -rw------- 1 postgres postgres   16M Jan 17 11:40 xid-1669761-lsn-20-7A000000.spill
    -rw------- 1 postgres postgres   13M Jan 17 11:40 xid-1669761-lsn-20-7B000000.spill
    -rw------- 1 postgres postgres  4.3M Jan 17 11:40 xid-1669761-lsn-20-7C000000.spill
    -rw------- 1 postgres postgres  9.5M Jan 17 11:40 xid-1669761-lsn-20-7D000000.spill
    -rw------- 1 postgres postgres   14M Jan 17 11:40 xid-1669761-lsn-20-7E000000.spill
    -rw------- 1 postgres postgres  1.7M Jan 17 11:40 xid-1669761-lsn-20-7F000000.spill
    -rw------- 1 postgres postgres  8.3M Jan 17 11:40 xid-1669761-lsn-20-80000000.spill
    -rw------- 1 postgres postgres  1.3M Jan 17 11:39 xid-1669761-lsn-20-8000000.spill
    -rw------- 1 postgres postgres   14M Jan 17 11:40 xid-1669761-lsn-20-81000000.spill
    -rw------- 1 postgres postgres  6.4M Jan 17 11:40 xid-1669761-lsn-20-82000000.spill
    -rw------- 1 postgres postgres  9.3M Jan 17 11:40 xid-1669761-lsn-20-83000000.spill
    -rw------- 1 postgres postgres  6.1M Jan 17 11:40 xid-1669761-lsn-20-84000000.spill
    -rw------- 1 postgres postgres  5.4M Jan 17 11:40 xid-1669761-lsn-20-85000000.spill
    -rw------- 1 postgres postgres  1.4M Jan 17 11:40 xid-1669761-lsn-20-86000000.spill
    -rw------- 1 postgres postgres  1.4M Jan 17 11:40 xid-1669761-lsn-20-87000000.spill
    -rw------- 1 postgres postgres  316K Jan 17 11:40 xid-1669761-lsn-20-88000000.spill
    -rw------- 1 postgres postgres  5.3M Jan 17 11:40 xid-1669761-lsn-20-89000000.spill
    -rw------- 1 postgres postgres  317K Jan 17 11:40 xid-1669761-lsn-20-8A000000.spill
    -rw------- 1 postgres postgres  1.1M Jan 17 11:39 xid-1669761-lsn-20-9000000.spill
    -rw------- 1 postgres postgres  1.4M Jan 17 11:39 xid-1669761-lsn-20-A000000.spill
    -rw------- 1 postgres postgres  3.2M Jan 17 11:39 xid-1669761-lsn-20-B000000.spill
    -rw------- 1 postgres postgres  2.6M Jan 17 11:39 xid-1669761-lsn-20-C000000.spill
    -rw------- 1 postgres postgres  2.1M Jan 17 11:39 xid-1669761-lsn-20-D000000.spill
    -rw------- 1 postgres postgres  3.3M Jan 17 11:39 xid-1669761-lsn-20-E000000.spill
    -rw------- 1 postgres postgres  4.7M Jan 17 11:39 xid-1669761-lsn-20-F000000.spill

We can see that a lot of relatively big files are created on a short timespan.
That is probably the spike of data creation which is causing the `No space left on device` error blocking replication, even if afterwards, disk occupation is not 100%.

    # sudo tar cvfz /data/postgresql/pg_replslot.tar.gz pg_replslot/
    
    # sudo mv pg_replslot/ pg_replslot_ori
    
    # sudo ln -s /data/postgresql/pg_replslot /var/lib/pgsql/12/data/
    
    # sudo chown -h postgres:postgres pg_replslot
    
    
    # systemctl start postgresql-12.service
    # systemctl status postgresql-12.service
    ● postgresql-12.service - PostgreSQL 12 database server
       Loaded: loaded (/usr/lib/systemd/system/postgresql-12.service; disabled; vendor preset: disabled)
       Active: active (running) since Fri 2020-01-17 12:29:06 CET; 8s ago
         Docs: https://www.postgresql.org/docs/12/static/
      Process: 11526 ExecStartPre=/usr/pgsql-12/bin/postgresql-12-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
     Main PID: 11532 (postmaster)
       CGroup: /system.slice/postgresql-12.service
               ├─11532 /usr/pgsql-12/bin/postmaster -D /var/lib/pgsql/12/data/
               ├─11534 postgres: logger
               ├─11537 postgres: checkpointer
               ├─11538 postgres: background writer
               ├─11539 postgres: walwriter
               ├─11540 postgres: autovacuum launcher
               ├─11541 postgres: stats collector
               ├─11542 postgres: logical replication launcher
               └─11543 postgres: ginko_user ginko 163.113.181.28(56546) idle
    
    ...

=> The suscriber starts again immediately :

    2020-01-17 11:40:19.178 CET [31424] LOG:  background worker "logical replication worker" (PID 18157) exited with exit code 1
    2020-01-17 11:40:19.179 CET [31424] LOG:  background worker "logical replication launcher" (PID 31434) exited with exit code 1
    2020-01-17 11:40:19.183 CET [31429] LOG:  shutting down
    2020-01-17 11:40:19.243 CET [18191] FATAL:  the database system is shutting down
    2020-01-17 11:40:19.913 CET [31424] LOG:  database system is shut down
    2020-01-17 12:30:46.583 CET [18272] LOG:  database system was shut down at 2020-01-17 11:40:19 CET
    2020-01-17 12:30:46.584 CET [18272] LOG:  recovered replication state of node 1 to 20/B95A26B8
    2020-01-17 12:30:46.591 CET [18269] LOG:  database system is ready to accept connections
    2020-01-17 12:30:46.639 CET [18279] LOG:  logical replication apply worker for subscription "my_project_sub" has started