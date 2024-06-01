> Занятие 12  
Резервное копирование и восстановление.
---
Упражнение 1 
--- 

Поготавливаем директорию под бэкапы:
```bash
root@MSI:~# sudo su postgres
postgres@MSI:/$ cd ../tmp
postgres@MSI:/tmp$ mkdir backup_pg
```
Создаем базу, таблицу и данные:

```sql
postgres=# create database otus;
CREATE DATABASE
postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=# create table otus_tbl (key_f bigint, val_f varchar(50));
CREATE TABLE
otus=# \dt
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | otus_tbl | table | postgres
(1 row)
otus=# insert into otus_tbl (key_f, val_f)
otus-# select i, left(md5(i::text), 10) from generate_series(1,100) s(i);
INSERT 0 100
```
Делаем логический бэкап таблицы и восстанавливаем в новую таблицу, проверяем результат:
```sql
otus=# \copy otus_tbl to '/tmp/backup_pg/otus_tbl.sql';
COPY 100
otus=# create table otus_tbl_dub (key_f bigint, val_f varchar(50));
CREATE TABLE
otus=# \copy otus_tbl_dub from '/tmp/backup_pg/otus_tbl.sql';
COPY 100
otus=# \dt
            List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | otus_tbl     | table | postgres
 public | otus_tbl_dub | table | postgres

otus=# select o1.*, o2.* from otus_tbl o1 natural join otus_tbl_dub o2 limit 10;
 key_f |   val_f    | key_f |   val_f
-------+------------+-------+------------
     1 | c4ca4238a0 |     1 | c4ca4238a0
     2 | c81e728d9d |     2 | c81e728d9d
     3 | eccbc87e4b |     3 | eccbc87e4b
     4 | a87ff679a2 |     4 | a87ff679a2
     5 | e4da3b7fbb |     5 | e4da3b7fbb
     6 | 1679091c5a |     6 | 1679091c5a
     7 | 8f14e45fce |     7 | 8f14e45fce
     8 | c9f0f895fb |     8 | c9f0f895fb
     9 | 45c48cce2e |     9 | 45c48cce2e
    10 | d3d9446802 |    10 | d3d9446802
(10 rows)
```
Делаем бэкап и восстановление при помощи **pg_dump** и **pg_restore**, восстанавливаем в новую базу и только таблицу *otus_tbl_dub*:

```bash
postgres@MSI:/$ pg_dump -d otus --create -Fc > /tmp/backup_pg/dump.gz
postgres@MSI:/$ ls -l tmp/backup_pg
total 20
-rw-rw-r-- 1 postgres postgres 4988 Jun  1 23:06 dump.gz
-rw-rw-r-- 1 postgres postgres 1392 Jun  1 22:50 otus_tbl.sql
postgres@MSI:/$ createdb otus_dub
postgres@MSI:/$ pg_restore -d otus_dub -t otus_tbl_dub /tmp/backup_pg/dump.gz
```
Проверяем результат:
```sql
postgres=# \l
                List of databases
   Name    |  Owner   | Encoding | Locale Provider 
-----------+----------+----------+-----------------
 otus      | postgres | UTF8     | libc            
 otus_dub  | postgres | UTF8     | libc            
 postgres  | postgres | UTF8     | libc            
 template0 | postgres | UTF8     | libc            
 template1 | postgres | UTF8     | libc            
(5 rows)

postgres=# \c otus_dub
You are now connected to database "otus_dub" as user "postgres".
otus_dub=# \dt
            List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | otus_tbl_dub | table | postgres
(1 row)

otus_dub=# select * from otus_tbl_dub limit 10;
 key_f |   val_f
-------+------------
     1 | c4ca4238a0
     2 | c81e728d9d
     3 | eccbc87e4b
     4 | a87ff679a2
     5 | e4da3b7fbb
     6 | 1679091c5a
     7 | 8f14e45fce
     8 | c9f0f895fb
     9 | 45c48cce2e
    10 | d3d9446802
(10 rows)

```