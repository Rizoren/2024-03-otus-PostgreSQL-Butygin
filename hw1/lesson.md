> Занятие 2  
SQL & RDBMS. Введение в PostgreSQL.
---
Окружение
---
+ Основная ОС: Windows 10 x64  
  - WSL: Debian  
  - Windows Terminal
  - Docker Desktop
+ Docker: Установлен на Debian  
  - postgres:latest
  - dpage/pgadmin4:8.5

---
Упражнение 1 
--- 
Создание таблицы и первичное заполнение  
```sql
create table persons(id serial, first_name text, second_name text);  
insert into persons(first_name, second_name) values('ivan', 'ivanov');  
insert into persons(first_name, second_name) values('petr', 'petrov');  
commit;  
show transaction isolation level;

/* output */
 transaction_isolation
-----------------------
 read committed
(1 row)
```
---
Упражнение 2 
--- 
Работа с 2 сессиями с уровнем изоляции: **read committed**   
```sql
/* 1 session */
insert into persons(first_name, second_name) values('sergey', 'sergeev');

/* 2 session */  
select * from persons;

 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

/* 1 session */
commit;  

/* 2 session */
select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
**Вывод:**\
При уровне изоляции "read committed" другие сессии не "видят" изменений в рамках незавершенных транзакций.  

---
Упражнение 3 
--- 
Работа с 2 сессиями с уровнем изоляции: **repeatable read**  
```sql
/* 1 session */
\set AUTOCOMMIT FALSE
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
insert into persons(first_name, second_name) values('sveta', 'svetova');

select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  6 | sveta      | svetova
(4 rows)

/* 2 session */  
\set AUTOCOMMIT FALSE
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(4 rows)

/* 1 session */
commit;  

/* 2 session */
select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

/* 2 session */
commit;
select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  6 | sveta      | svetova
(4 rows)
```
**Вывод:**\
При уровне изоляции "repeatable read" сессия работает со снимком данных на момент начала. Успешное завершение транзакций в других сессиях никак не влияет на данные в исходной сессии. После завершения транзакции в исходной сессии и запуска новой снимок данных будет обновлен до состояния на момен начала новой транзакции.   