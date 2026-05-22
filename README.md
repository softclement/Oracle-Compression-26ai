# Oracle Compression PoC (Oracle 23ai / 26ai)

Simple hands-on PoC to explore Oracle Database compression features.

This script compares:
* Normal Heap Table
* BASIC Compression
* OLTP Compression
* SecureFile LOB Compression
  
---

# Compression Types Covered
| Table      | Compression Type           |
| ---------- | -------------------------- |
| T_NORMAL   | No Compression             |
| T_COMPRESS | BASIC Compression          |
| T_OLTP     | OLTP Compression           |
| T_LOB      | SecureFile LOB Compression |

---

# What This PoC Demonstrates
* Table compression behavior
* Segment size comparison
* Compression metadata
* BASIC vs OLTP compression
* SecureFile LOB compression
* DML behavior with OLTP compression

---

# Notes

## BASIC Compression

```sql id="2o0tt5"
compress
```

* Better storage savings
* Good for bulk loads and reporting workloads

---

## OLTP Compression

```sql id="0m6l25"
compress for oltp
```

* Designed for OLTP workloads
* Better for INSERT/UPDATE/DELETE operations
* Requires Advanced Compression option/license

---

## SecureFile Compression

```sql id="s9y8va"
lob(doc) store as securefile
(
    compress high
)
```

* Compresses large CLOB/BLOB data
* Useful for XML, JSON, documents, logs

---

# Oracle Compression PoC Script

```sql id="0wwdml"
------------------------------------------------------------
-- ORACLE COMPRESSION PoC
-- NORMAL vs BASIC vs OLTP vs SECUREFILE LOB
------------------------------------------------------------

------------------------------------------------------------
-- CLEANUP
------------------------------------------------------------

drop table IF EXISTS t_normal purge;
drop table IF EXISTS t_compress purge;
drop table IF EXISTS t_oltp purge;
drop table IF EXISTS t_lob purge;

------------------------------------------------------------
-- CREATE NORMAL TABLE
------------------------------------------------------------

create table t_normal
(
    id        number,
    city      varchar2(100),
    remarks   varchar2(500)
);

------------------------------------------------------------
-- CREATE BASIC COMPRESSED TABLE
------------------------------------------------------------

create table t_compress
(
    id        number,
    city      varchar2(100),
    remarks   varchar2(500)
)
compress;

------------------------------------------------------------
-- CREATE OLTP COMPRESSED TABLE
------------------------------------------------------------

create table t_oltp
(
    id        number,
    city      varchar2(100),
    remarks   varchar2(500)
)
compress for oltp;

------------------------------------------------------------
-- CREATE SECUREFILE LOB COMPRESSED TABLE
------------------------------------------------------------

create table t_lob
(
    id    number,
    doc   clob
)
lob(doc) store as securefile
(
    compress high
);

------------------------------------------------------------
-- INSERT DATA INTO NORMAL TABLE
------------------------------------------------------------

insert into t_normal (id, city, remarks)
select
    level,
    'BANGALORE',
    'Oracle test data.......' || level
from dual
connect by level < 1000000;

commit;

------------------------------------------------------------
-- INSERT DATA INTO BASIC COMPRESSED TABLE
------------------------------------------------------------

insert into t_compress (id, city, remarks)
select
    level,
    'BANGALORE',
    'Oracle test data.......' || level
from dual
connect by level < 1000000;

commit;

------------------------------------------------------------
-- INSERT DATA INTO OLTP COMPRESSED TABLE
------------------------------------------------------------

insert into t_oltp (id, city, remarks)
select
    level,
    'BANGALORE',
    'Oracle test data.......' || level
from dual
connect by level < 1000000;

commit;

------------------------------------------------------------
-- INSERT LARGE LOB DATA
------------------------------------------------------------

insert into t_lob
select
    level,
    rpad(
        'Oracle SecureFile Compression Test ',
        5000,
        'X'
    )
from dual
connect by level <= 10000;

commit;

------------------------------------------------------------
-- GATHER STATS
------------------------------------------------------------

call dbms_stats.gather_table_stats(user,'T_NORMAL');
call dbms_stats.gather_table_stats(user,'T_COMPRESS');
call dbms_stats.gather_table_stats(user,'T_OLTP');
call dbms_stats.gather_table_stats(user,'T_LOB');

------------------------------------------------------------
-- TABLE COMPRESSION DETAILS
------------------------------------------------------------

select
    table_name,
    compression,
    compress_for
from user_tables
where table_name in
(
    'T_NORMAL',
    'T_COMPRESS',
    'T_OLTP',
    'T_LOB'
)
order by table_name;

------------------------------------------------------------
-- SEGMENT SIZE COMPARISON
------------------------------------------------------------

select
    segment_name,
    segment_type,
    round(bytes/1024/1024,2) mb
from user_segments
where segment_name in
(
    'T_NORMAL',
    'T_COMPRESS',
    'T_OLTP',
    'T_LOB'
)
order by mb desc;

------------------------------------------------------------
-- SECUREFILE LOB DETAILS
------------------------------------------------------------

select
    table_name,
    column_name,
    securefile,
    compression
from user_lobs
where table_name='T_LOB';

------------------------------------------------------------
-- UPDATE TEST FOR OLTP COMPRESSION
------------------------------------------------------------

update t_oltp
set remarks = remarks || ' UPDATED'
where mod(id,100)=0;

commit;

------------------------------------------------------------
-- FINAL SIZE CHECK
------------------------------------------------------------

select
    segment_name,
    segment_type,
    round(bytes/1024/1024,2) mb
from user_segments
where segment_name like 'T_%'
order by mb desc;

------------------------------------------------------------
-- VERIFY ROW COUNTS
------------------------------------------------------------

select 'T_NORMAL'    table_name, count(*) rows_count from t_normal
union all
select 'T_COMPRESS', count(*) from t_compress
union all
select 'T_OLTP',     count(*) from t_oltp
union all
select 'T_LOB',      count(*) from t_lob;
```

---

# Learning Outcome

* How Oracle compression reduces storage
* Difference between BASIC and OLTP compression
* Compression metadata behavior
* Segment size savings
* SecureFile LOB compression usage

---

# Tested On

* Oracle 26ai

