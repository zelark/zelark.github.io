---
title: Исследуем блокировки в PostgreSQL
layout: post
published: true
date: 2015-05-23
category: dev
tags: [postgres]
---
Сегодня предлагаю вам вольный перевод весьма увлекательной и забавной статьи [«Exploring Query Locks in Postgres»](http://big-elephants.com/2013-09/exploring-query-locks-in-postgres/).

Понимание того как работают блокировки является ключом к написанию правильных запросов способных выполняться параллельно. Чтобы узнать как работают блокировки и увидеть, что происходит внутри базы данных, давайте рассмотрим наглядный пример.

### Песочница
Для начала создадим «песочницу»:

~~~sql
create database sandbox;

create table toys (
  id serial not null,
  name character varying(36),
  usage integer not null default 0,
  constraint toys_pkey primary key (id)
);

insert into toys(name) values('car'),('digger'),('shovel');
~~~

Откроем два терминала, в каждом из них подключимся к только что созданной базе данных sandbox. Чтобы не путаться, дадим им имена. Пусть это будут Алиса и Боб. Изменить подсказку командной строки можно с помощью команды `\set`:

    \set PROMPT1 '[Alice] %/> '

Первой появляется Алиса и осматривает игрушки:

    [Alice] sandbox> begin;
    BEGIN
    [Alice] sandbox> select * from toys;

     id |  name  | usage
    ----+--------+-------
      1 | car    |     0
      2 | digger |     0
      3 | shovel |     0
    (3 rows)

Обратите внимание, что оператор `begin` начинает транзакцию явно. В этом случае она будет продолжаться до тех пор, пока мы не зафиксируем её, сделаем `commit`, или не откатим, сделаем `rollback`.

Если бы Боб сейчас посмотрел на игрушки, он увидел бы то же самое:

    [Bob] sandbox> begin;
    BEGIN
    [Bob] sandbox> select * from toys;

     id |  name  | usage
    ----+--------+-------
      1 | car    |     0
      2 | digger |     0
      3 | shovel |     0
    (3 rows)

Таким образом параллельное выполнение двух операторов `select` не мешает работе каждого из них. Именно такого поведения мы ожидаем от надёжной и высокопроизводительной базы данных.

### pg_lock
Однако, транзакции Алисы и Боба до сих пор открыты. Чтобы посмотреть какие блокировки были установлены, откроем третий терминал и назовём его Ева:

    \set PROMPT1 '[Eve] %/> '

~~~sql
select
  lock.locktype,
  lock.relation::regclass,
  lock.mode,
  lock.transactionid as tid,
  lock.virtualtransaction as vtid,
  lock.pid,
  lock.granted
from pg_catalog.pg_locks lock
  left join pg_catalog.pg_database db
    on db.oid = lock.database
where (db.datname = 'sandbox' or db.datname is null)
  and not lock.pid = pg_backend_pid()
order by lock.pid;
~~~

      locktype  | relation  |      mode       | tid | vtid  |  pid  | granted
    ------------+-----------+-----------------+-----+-------+-------+---------
     relation   | toys_pkey | AccessShareLock |     | 6/268 | 45265 | t
     relation   | toys      | AccessShareLock |     | 6/268 | 45265 | t
     virtualxid |           | ExclusiveLock   |     | 6/268 | 45265 | t
     relation   | toys_pkey | AccessShareLock |     | 1/282 | 45263 | t
     relation   | toys      | AccessShareLock |     | 1/282 | 45263 | t
     virtualxid |           | ExclusiveLock   |     | 1/282 | 45263 | t
    (6 rows)

Представление pg_lock показывает активные блокировки. Условие `(db.datname = 'sandbox' or db.datname is null)` оставляет только те записи которые относятся к «песочнице», а условие `not pid = pg_backend_pid()` исключает записи текущей сессии. Наконец, чтобы колонка `relation` стала более информативной было использовано приведение типа к `regclass`.

Посмотрим на пятую строку:

      locktype  | relation  |      mode       | tid | vtid  |  pid  | granted
    ------------+-----------+-----------------+-----+-------+-------+---------
     relation   | toys      | AccessShareLock |     | 1/282 | 45263 | t

Виртуальной транзакцией 1/282, на таблицу `toys` наложена блокировка `AccessShareLock`, при этом блокировка считается выданной (is granted). Пока всё идёт хорошо, Боб и Алиса счастливы, ведь они оба видят — игрушки можно взять.

Обратите внимание, каждая транзакция удерживает блокировку `ExclusiveLock` на своей виртуальной транзакции `virtualxid`.

Алиса решает взять машинку:

    [Alice] sandbox> update toys set usage = usage + 1 where id = 1;
    UPDATE 1

Никаких проблем. Посмотрим как выглядит таблица блокировок теперь:

       locktype    | relation  |       mode       |   tid    | vtid  |  pid  | granted
    ---------------+-----------+------------------+----------+-------+-------+---------
     relation      | toys_pkey | AccessShareLock  |          | 6/268 | 45265 | t
     relation      | toys      | AccessShareLock  |          | 6/268 | 45265 | t
     virtualxid    |           | ExclusiveLock    |          | 6/268 | 45265 | t
     relation      | toys_pkey | AccessShareLock  |          | 1/282 | 45263 | t
     relation      | toys_pkey | RowExclusiveLock |          | 1/282 | 45263 | t
     relation      | toys      | AccessShareLock  |          | 1/282 | 45263 | t
     relation      | toys      | RowExclusiveLock |          | 1/282 | 45263 | t
     virtualxid    |           | ExclusiveLock    |          | 1/282 | 45263 | t
     transactionid |           | ExclusiveLock    | 24273800 | 1/282 | 45263 | t
    (9 rows)

### transactionid
В таблице `toys` на записи с машинкой теперь стоит блокировка `RowExclusiveLock`. Также появился реальный идентификатор транзакции `transactionid` на котором удерживается блокировка `ExclusiveLock`. Такой идентификатор появляется у каждой транзакции потенциально меняющей состояние базы данных.

### MVCC
Поскольку транзакция Алисы не зафиксирована, Боб видит прежние данные:

    [Bob] sandbox> select * from toys;

     id |  name  | usage
    ----+--------+-------
      1 | car    |     0
      2 | digger |     0
      3 | shovel |     0
    (3 rows)

Мы не знаем, будет ли Алиса фиксировать или откатывать свою транзакцию. Следовательно, Боб видит содержимое таблицы неизменным.

Для того, чтобы каждый пользователь видел согласованное стостояние базы данных, постгрес использует механизм управления конкурентным доступом с помощью многоверсионности MVCC (Multi Version Concurrency Control).

### Блокирующие запросы
Допустим Боб тоже хочет поиграть машинкой (типичная ситуация для детей). Боб выполняет следующий запрос:

    [Bob] sandbox> update toys set usage = usage + 1 where id = 1;

но ничего не происходит. Ему нужно подождать пока Алиса завершит свою транзакцию. Снова посмотрим в таблицу блокировок:

       locktype    | relation  |       mode       |   tid    | vtid  |  pid  | granted
    ---------------+-----------+------------------+----------+-------+-------+---------
     relation      | toys_pkey | AccessShareLock  |          | 6/268 | 45265 | t
     relation      | toys_pkey | RowExclusiveLock |          | 6/268 | 45265 | t
     relation      | toys      | AccessShareLock  |          | 6/268 | 45265 | t
     relation      | toys      | RowExclusiveLock |          | 6/268 | 45265 | t
     virtualxid    |           | ExclusiveLock    |          | 6/268 | 45265 | t
     relation      | toys_pkey | AccessShareLock  |          | 1/282 | 45263 | t
     relation      | toys_pkey | RowExclusiveLock |          | 1/282 | 45263 | t
     relation      | toys      | AccessShareLock  |          | 1/282 | 45263 | t
     relation      | toys      | RowExclusiveLock |          | 1/282 | 45263 | t
     virtualxid    |           | ExclusiveLock    |          | 1/282 | 45263 | t
     transactionid |           | ExclusiveLock    | 24273800 | 1/282 | 45263 | t
     tuple         | toys      | ExclusiveLock    |          | 6/268 | 45265 | t
     transactionid |           | ExclusiveLock    | 24273801 | 6/268 | 45265 | t
     transactionid |           | ShareLock        | 24273800 | 6/268 | 45265 | f
    (14 rows)

Теперь у Боба тоже есть `transactionid` и он просит выдать ему `ShareLock` на `transactionid` Алисы — «Мам, я тоже хочу поиграть машинкой». Поскольку две блокировки конфликтуют друг c другом, запрос Боба не удовлетворён (is not granted). Он будет висеть в таком состоянии до тех пор, пока Алиса не снимет `ExclusiveLock`, завершив свою транзакцию.

### pg_stats_activity
[pg_stat_activity](http://www.postgresql.org/docs/9.4/static/monitoring-stats.html#PG-STAT-ACTIVITY-VIEW) ещё одно интересное представление (view) из pg_catalog'а. Оно показывает запросы выполняющиеся в данный момент:

~~~sql
select query, state, waiting, pid
from pg_stat_activity
where datname = 'sandbox'
  and not (state = 'idle' or pid = pg_backend_pid());
~~~

                          query                      |        state        | waiting |  pid
    -------------------------------------------------+---------------------+---------+-------
     update toys set usage = usage + 1 where id = 1; | active              | t       | 45265
     update toys set usage = usage + 1 where id = 1; | idle in transaction | f       | 45263
    (2 rows)

Мы видим, что запрос Алисы простаивает в ожидании поддтверждения транзакции (idle in transaction), в то время как запрос Боба активен и подвис (is waiting).

Чтобы увидеть кто кого заблокировал, объединим два запроса в один:

~~~sql
select
  bda.pid as blocked_pid,
  bda.query as blocked_query,
  bga.pid as blocking_pid,
  bga.query as blocking_query
from pg_catalog.pg_locks bdl
  join pg_stat_activity bda
    on bda.pid = bdl.pid
  join pg_catalog.pg_locks bgl
    on bgl.pid != bdl.pid
    and bgl.transactionid = bdl.transactionid
  join pg_stat_activity bga
    on bga.pid = bgl.pid
where not bdl.granted
  and bga.datname = 'sandbox';
~~~

     blocked_pid |                  blocked_query                  | blocking_pid |                 blocking_query
    -------------+-------------------------------------------------+--------------+-------------------------------------------------
           45265 | update toys set usage = usage + 1 where id = 1; |        45263 | update toys set usage = usage + 1 where id = 1;
    (1 row)

Если бы Алиса решила откатить или зафиксировать свою транзакцию, блокировка `ExclusiveLock` была бы снята и Боб получил бы `ShareLock`. После этого он мог бы зафиксировать свою транзакцию, и запись в таблице была бы обновлена независимо от решения Алисы.

    [Alice] sandbox> rollback;
    ROLLBACK

    [Bob] sandbox> commit;
    COMMIT
    [Bob] sandbox> select * from toys;

     id |  name  | usage
    ----+--------+-------
      2 | digger |     0
      3 | shovel |     0
      1 | car    |     1
    (3 rows)

Конечно, если бы Боб и Алиса решили играть разными игрушками, конфликтной ситуации между ними не возникло бы вообще.

### Явные блокировки
Другая типичная ситуация для детей, когда один из них хочет забрать все игрушки без реальной необходимости:

    [Alice] sandbox> begin;
    BEGIN
    [Alice] sandbox> lock table toys in access exclusive mode;
    LOCK TABLE

Хотя Алиса и не взяла ни одной игрушки, Боб всё равно должен ждать.

    [Bob] sandbox> begin; update toys set usage = usage + 1 where id = 2;
    BEGIN

Таблица блокировок теперь выглядит так:

      locktype  | relation |        mode         | tid | vtid  |  pid  | granted
    ------------+----------+---------------------+-----+-------+-------+---------
     virtualxid |          | ExclusiveLock       |     | 6/284 | 45265 | t
     virtualxid |          | ExclusiveLock       |     | 1/294 | 45263 | t
     relation   | toys     | RowExclusiveLock    |     | 6/284 | 45265 | f
     relation   | toys     | AccessExclusiveLock |     | 1/294 | 45263 | t
    (4 rows)

Поскольку Алиса удерживает `AccessExclusiveLock` без изменения состояния базы данных, то она не получила свой `transactionid`. У Боба его тоже нет, потому что он не получил `RowExclusiveLock` на таблицу `toys`. В этой ситуации, запрос для отображения блокировок который мы использовали ранее, нам не поможет, т. к. он использует объединение по `transactionid`.

     blocked_pid | blocked_query | blocking_pid | blocking_query
    -------------+---------------+--------------+----------------
    (0 rows)

Таким образом, Ева думает, что всё хорошо, в то время как следующий запрос:

~~~sql
select pid, query, now() - query_start as waiting_duration
from pg_catalog.pg_stat_activity
where datname = 'sandbox'
  and waiting;
~~~

      pid  |                      query                      | waiting_duration
    -------+-------------------------------------------------+------------------
     35929 | update toys set usage = usage + 1 where id = 2; | 00:01:34.519518
    (1 row)

показывает обратное. Обратите внимание на столбец `waiting_duration`, он рассчитывается как разница между `now()` и `query_start`. Благодаря этому, видно сколько времени запрос уже висит.

Сделав объединение по столбцам `relation` и `locktype`, мы снова можем видеть кто кого блокирует:

~~~sql
select
  bgl.relation::regclass,
  bda.pid as blocked_pid,
  bda.query as blocked_query,
  bdl.mode as blocked_mode,
  bga.pid AS blocking_pid,
  bga.query as blocking_query,
  bgl.mode as blocking_mode
from pg_catalog.pg_locks bdl
  join pg_stat_activity bda
    on bda.pid = bdl.pid
  join pg_catalog.pg_locks bgl
    on bdl.pid != bgl.pid
    and bgl.relation = bdl.relation
    and bgl.locktype = bdl.locktype
  join pg_stat_activity bga
    on bga.pid = bgl.pid
where not bdl.granted
  and bga.datname = 'sandbox';
~~~

     relation | blocked_pid |                  blocked_query                  |   blocked_mode   | blocking_pid |              blocking_query               |    blocking_mode
    ----------+-------------+-------------------------------------------------+------------------+--------------+-------------------------------------------+---------------------
     toys     |       35929 | update toys set usage = usage + 1 where id = 2; | RowExclusiveLock |        35937 | lock table toys in access exclusive mode; | AccessExclusiveLock
    (1 row)

Алисе было сказано, что некрасиво делать явную блокировку без видимой на то причины. Она фиксирует свою транзакцию без каких-либо изменений, а Боб может взять игрушку.

    [Alice] sandbox> commit;
    COMMIT

    UPDATE 1
    [Bob] sandbox>

Но его транзакция всё ещё открыта. Если мы посмотрим в таблицу блокировок, то увидим следующее:

       locktype    | relation  |       mode       |  tid  | vtid |  pid  | granted
    ---------------+-----------+------------------+-------+------+-------+---------
     relation      | toys_pkey | RowExclusiveLock |       | 4/51 | 35929 | t
     virtualxid    |           | ExclusiveLock    |       | 4/51 | 35929 | t
     relation      | toys      | RowExclusiveLock |       | 4/51 | 35929 | t
     transactionid |           | ExclusiveLock    | 19307 | 4/51 | 35929 | t
    (4 rows)

Лишь после того как Боб получил `RowExclusiveLock`, к его транзакции был добавлен `transactionid`. Боб рад и делает коммит:

    [Bob] sandbox> commit;
    COMMIT

### RowExclusiveLock
Поскольку Алиса не знает какую игрушку она хочет взять, а ставить явную блокировку ей не разрешили, она пробует другой подход:

    [Alice] sandbox> begin; select * from toys for update;
    BEGIN

     id |  name  | usage
    ----+--------+-------
      2 | digger |     1
      3 | shovel |     0
      1 | car    |     1
    (3 rows)

На детском языке это бы звучало примерно так: «Хочу видеть все игрушки и может быть я возьму одну, но пока не знаю какую. А до тех пор я не хочу чтобы кто-то другой прикасался к ним».

Тем временем Боб хочет взять лопатку, но конечно не может этого сделать, его транзакция подвисает:

    [Bob] sandbox> begin; update toys set usage = usage + 1 where id = 3;
    BEGIN

Ева видит следующую ситуацию:

       locktype    | relation  |       mode       |  tid  | vtid |  pid  | granted
    ---------------+-----------+------------------+-------+------+-------+---------
     transactionid |           | ShareLock        | 19309 | 4/55 | 35929 | f
     relation      | toys      | RowExclusiveLock |       | 4/55 | 35929 | t
     virtualxid    |           | ExclusiveLock    |       | 4/55 | 35929 | t
     transactionid |           | ExclusiveLock    | 19310 | 4/55 | 35929 | t
     tuple         | toys      | ExclusiveLock    |       | 4/55 | 35929 | t
     relation      | toys_pkey | RowExclusiveLock |       | 4/55 | 35929 | t
     relation      | toys      | RowShareLock     |       | 5/17 | 35937 | t
     virtualxid    |           | ExclusiveLock    |       | 5/17 | 35937 | t
     relation      | toys_pkey | AccessShareLock  |       | 5/17 | 35937 | t
     transactionid |           | ExclusiveLock    | 19309 | 5/17 | 35937 | t
    (10 rows)

Боб совершенно ясно хочет изменить состояние базы данных поэтому он получил `transactionid` равный 19310, но снова вынужден ждать получения `ShareLock` на транзакцию Алисы с номером 19309.

### Объединяем блокировки и активности
Пришло время объединить таблицу блокировок и таблицу активности вместе, так, чтобы всегда видеть кто кого заблокировал:

~~~sql
select
  coalesce(bgl.relation::regclass::text, bgl.locktype) as locked_item,
  now() - bda.query_start as waiting_duration,
  bda.pid as blocked_pid,
  bda.query as blocked_query,
  bdl.mode as blocked_mode,
  bga.pid as blocking_pid,
  bga.query as blocking_query,
  bgl.mode as blocking_mode
from pg_catalog.pg_locks bdl
  join pg_stat_activity bda
    on bda.pid = bdl.pid
  join pg_catalog.pg_locks bgl
    on bgl.pid != bdl.pid
    and (bgl.transactionid = bdl.transactionid
      or bgl.relation = bdl.relation and bgl.locktype = bdl.locktype)
  join pg_stat_activity bga
    on bga.pid = bgl.pid
    and bga.datid = bda.datid
where not bdl.granted
  and bga.datname = current_database();
~~~

      locked_item  | waiting_duration | blocked_pid |                  blocked_query                  | blocked_mode | blocking_pid |         blocking_query         | blocking_mode
    ---------------+------------------+-------------+-------------------------------------------------+--------------+--------------+--------------------------------+---------------
     transactionid | 00:03:32.330397  |       35929 | update toys set usage = usage + 1 where id = 3; | ShareLock    |        35937 | select * from toys for update; | ExclusiveLock
    (1 row)

Для оценки времени блокирования запроса был добавлен столбец `waiting_duration`, а также функция `current_database()` используемая в условии.

Ева не может запомнить этот длиннющий запрос и создаёт представление:

~~~sql
create view lock_monitor as (
  select
    coalesce(bgl.relation::regclass::text, bgl.locktype) as locked_item,
    now() - bda.query_start as waiting_duration,
    bda.pid as blocked_pid,
    bda.query as blocked_query,
    bdl.mode as blocked_mode,
    bga.pid as blocking_pid,
    bga.query as blocking_query,
    bgl.mode as blocking_mode
  from pg_catalog.pg_locks bdl
    join pg_stat_activity bda
      on bda.pid = bdl.pid
    join pg_catalog.pg_locks bgl
      on bgl.pid != bdl.pid
      and (bgl.transactionid = bdl.transactionid
        or bgl.relation = bdl.relation and bgl.locktype = bdl.locktype)
    join pg_stat_activity bga
      on bga.pid = bgl.pid
      and bga.datid = bda.datid
  where not bdl.granted
    and bga.datname = current_database()
);
~~~

С помощью него, она легко узнает что задумали её дети:

    [eve] sandbox> select * from lock_monitor;

      locked_item  | waiting_duration | blocked_pid |                  blocked_query                  | blocked_mode | blocking_pid |         blocking_query         | blocking_mode
    ---------------+------------------+-------------+-------------------------------------------------+--------------+--------------+--------------------------------+---------------
     transactionid | 00:06:19.986426  |       35929 | update toys set usage = usage + 1 where id = 3; | ShareLock    |        35937 | select * from toys for update; | ExclusiveLock
    (1 row)

Выпив чашку чая и успокоившись, Ева решает почитать [руководство по явным блокировкам в постгресе](http://www.postgresql.org/docs/9.1/static/explicit-locking.html), узнать какие бывают виды блокировок и то, как они конфликтуют друг с другом.