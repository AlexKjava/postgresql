```sql
-- Размер табличных пространств
SELECT spcname, pg_size_pretty(pg_tablespace_size(spcname))
FROM pg_tablespace
WHERE spcname<>'pg_global';

-- Размер баз данных
SELECT pg_database.datname,
       pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;

-- Размер схем в базе данных
SELECT A.schemaname,
       pg_size_pretty (SUM(pg_relation_size(C.oid))) as table,
       pg_size_pretty (SUM(pg_total_relation_size(C.oid)-pg_relation_size(C.oid))) as index,
       pg_size_pretty (SUM(pg_total_relation_size(C.oid))) as table_index,
       SUM(n_live_tup)
FROM pg_class C
         LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
         INNER JOIN pg_stat_user_tables A ON C.relname = A.relname
WHERE nspname NOT IN ('pg_catalog', 'information_schema')
  AND C .relkind <> 'i'
  AND nspname !~ '^pg_toast'
GROUP BY A.schemaname;

-- Размер таблиц
SELECT schemaname,
       C.relname AS "relation",
       pg_size_pretty (pg_relation_size(C.oid)) as table,
       pg_size_pretty (pg_total_relation_size (C.oid)-pg_relation_size(C.oid)) as index,
       pg_size_pretty (pg_total_relation_size (C.oid)) as table_index,
       n_live_tup
FROM pg_class C
         LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
         LEFT JOIN pg_stat_user_tables A ON C.relname = A.relname
WHERE nspname NOT IN ('pg_catalog', 'information_schema')
  AND C.relkind <> 'i'
  AND nspname !~ '^pg_toast'
ORDER BY pg_total_relation_size (C.oid) DESC

--     Мониторинг блокировок
SELECT COALESCE(blockingl.relation::regclass::text, blockingl.locktype) AS locked_item,
       now() - blockeda.query_start                                     AS waiting_duration,
       blockeda.pid                                                     AS blocked_pid,
       blockeda.query                                                   AS blocked_query,
       blockedl.mode                                                    AS blocked_mode,
       blockinga.pid                                                    AS blocking_pid,
       blockinga.query                                                  AS blocking_query,
       blockingl.mode                                                   AS blocking_mode
FROM pg_locks blockedl
         JOIN pg_stat_activity blockeda ON blockedl.pid = blockeda.pid
         JOIN pg_locks blockingl ON (blockingl.transactionid = blockedl.transactionid OR
                                     blockingl.relation = blockedl.relation AND
                                     blockingl.locktype = blockedl.locktype) AND blockedl.pid <> blockingl.pid
         JOIN pg_stat_activity blockinga ON blockingl.pid = blockinga.pid AND blockinga.datid = blockeda.datid
WHERE NOT blockedl.granted AND blockinga.datname = current_database();

-- Снятие блокировок
SELECT pg_cancel_backend(PID_ID);
OR
SELECT pg_terminate_backend(PID_ID);

select
    sa.usename as username,
    sa.client_addr,
    sa.backend_start,
    sa.query_start,
    sa.wait_event_type,
    sa.state,
    sa.query,
    lock.locktype,
    lock.relation::regclass as rel,
    lock.mode,
    lock.transactionid as tid,
    lock.virtualtransaction as vtid,
    lock.pid,
    lock.granted
from pg_catalog.pg_locks lock
         left join pg_catalog.pg_database db
                   on db.oid = lock.database
         left join pg_catalog.pg_stat_activity sa
                   on lock.pid = sa.pid
where not lock.pid = pg_backend_pid()
order by lock.pid;

-- Коэффициент кэширования (Cache Hit Ratio)
SELECT sum(heap_blks_read) as heap_read,
       sum(heap_blks_hit)  as heap_hit,
       sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
FROM
    pg_statio_user_tables;
-- Коэффициент кэширования - это показатель эффективности чтения, измеряемый долей операций чтения из кэша по
-- сравнению с общим количеством операций чтения как с диска, так и из кэша. За исключением случаев использования
-- хранилища данных, идеальный коэффициент кэширования составляет 99% или выше, что означает, что по крайней мере
-- 99% операций чтения выполняются из кэша и не более 1% - с диска

-- Использование индексов
SELECT relname,
       100 * idx_scan / (seq_scan + idx_scan) percent_of_times_index_used,
       n_live_tup rows_in_table
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 0
ORDER BY n_live_tup DESC;
-- Добавление индексов в вашу базу данных имеет большое значение для производительности запросов.
-- Индексы особенно важны для больших таблиц. Этот запрос показывает количество строк в таблицах и
-- процент времени использования индексов по сравнению с чтением без индексов. Идеальные кандидаты
-- для добавления индекса - это таблицы размером более 10000 строк с нулевым или низким использованием индекса.

-- Коэффициент кэширования индексов (Index Cache Hit Rate)
SELECT sum(idx_blks_read) as idx_read,
       sum(idx_blks_hit)  as idx_hit,
       (sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit) as ratio
FROM pg_statio_user_indexes;

-- Неиспользуемые индексы
SELECT schemaname, relname, indexrelname
FROM pg_stat_all_indexes
WHERE idx_scan = 0 and schemaname <> 'pg_toast' and  schemaname <> 'pg_catalog'


--     Раздувание базы данных (Database bloat)
SELECT
    current_database(), schemaname, tablename, /*reltuples::bigint, relpages::bigint, otta,*/
    ROUND((CASE WHEN otta=0 THEN 0.0 ELSE sml.relpages::float/otta END)::numeric,1) AS tbloat,
    CASE WHEN relpages < otta THEN 0 ELSE bs*(sml.relpages-otta)::BIGINT END AS wastedbytes,
    iname, /*ituples::bigint, ipages::bigint, iotta,*/
    ROUND((CASE WHEN iotta=0 OR ipages=0 THEN 0.0 ELSE ipages::float/iotta END)::numeric,1) AS ibloat,
    CASE WHEN ipages < iotta THEN 0 ELSE bs*(ipages-iotta) END AS wastedibytes
FROM (
         SELECT
             schemaname, tablename, cc.reltuples, cc.relpages, bs,
             CEIL((cc.reltuples*((datahdr+ma-
                                  (CASE WHEN datahdr%ma=0 THEN ma ELSE datahdr%ma END))+nullhdr2+4))/(bs-20::float)) AS otta,
             COALESCE(c2.relname,'?') AS iname, COALESCE(c2.reltuples,0) AS ituples, COALESCE(c2.relpages,0) AS ipages,
             COALESCE(CEIL((c2.reltuples*(datahdr-12))/(bs-20::float)),0) AS iotta /* very rough approximation, assumes all cols */
         FROM (
                  SELECT
                      ma,bs,schemaname,tablename,
                      (datawidth+(hdr+ma-(case when hdr%ma=0 THEN ma ELSE hdr%ma END)))::numeric AS datahdr,
                      (maxfracsum*(nullhdr+ma-(case when nullhdr%ma=0 THEN ma ELSE nullhdr%ma END))) AS nullhdr2
                  FROM (
                           SELECT
                               schemaname, tablename, hdr, ma, bs,
                               SUM((1-null_frac)*avg_width) AS datawidth,
                               MAX(null_frac) AS maxfracsum,
                               hdr+(
                                   SELECT 1+count(*)/8
                                   FROM pg_stats s2
                                   WHERE null_frac<>0 AND s2.schemaname = s.schemaname AND s2.tablename = s.tablename
                               ) AS nullhdr
                           FROM pg_stats s, (
                               SELECT
                                   (SELECT current_setting('block_size')::numeric) AS bs,
                                   CASE WHEN substring(v,12,3) IN ('8.0','8.1','8.2') THEN 27 ELSE 23 END AS hdr,
                                   CASE WHEN v ~ 'mingw32' THEN 8 ELSE 4 END AS ma
                               FROM (SELECT version() AS v) AS foo
                           ) AS constants
                           GROUP BY 1,2,3,4,5
                       ) AS foo
              ) AS rs
                  JOIN pg_class cc ON cc.relname = rs.tablename
                  JOIN pg_namespace nn ON cc.relnamespace = nn.oid AND nn.nspname = rs.schemaname AND nn.nspname <> 'information_schema'
                  LEFT JOIN pg_index i ON indrelid = cc.oid
                  LEFT JOIN pg_class c2 ON c2.oid = i.indexrelid
     ) AS sml
ORDER BY wastedbytes DESC;
-- Раздувание базы данных - это дисковое пространство, которое использовалось таблицей или индексом и доступно для
-- повторного использования базой данных, но не было освобождено. Раздувание происходит при обновлении таблиц или
-- индексов. Если у вас загруженная база данных с большим количеством операций удаления, раздувание может оставить
-- много неиспользуемого пространства в вашей базе данных и повлиять на производительность, если его не убрать.
-- Показатели wastedbytes для таблиц и wastedibytes для индексов покажет вам, есть ли у вас какие-либо серьезные
-- проблемы с раздуванием. Для борьбы с раздуванием существует команда VACUUM.

-- Проверка запусков VACUUM
    SELECT relname,
       last_vacuum,
       last_autovacuum
FROM pg_stat_user_tables;

-- Показывает количество открытых подключений
SELECT COUNT(*) as connections,
       backend_type
FROM pg_stat_activity
where state = 'active' OR state = 'idle'
GROUP BY backend_type
ORDER BY connections DESC;
-- Показывает открытые подключения ко всем базам данных в вашем экземпляре PostgreSQL.
-- Если у вас несколько баз данных в одном PostgreSQL, то в условие WHERE стоит добавить datname = 'Ваша_база_данных'.

-- Показывает выполняющиеся запросы
SELECT pid, age(clock_timestamp(), query_start), usename, query, state
FROM pg_stat_activity
WHERE state != 'idle' AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY query_start desc;

-- Размер таблиц и индексов, единым плоским списком по всей СУБД (чтобы понять что кушает больше всего)

SELECT concat(schemaname, '.', tablename, ':', indexname),
       pg_relation_size(concat(schemaname, '.', indexname)::regclass) as size,
       pg_size_pretty(pg_relation_size(concat(schemaname, '.', indexname)::regclass)) AS pretty_size
FROM pg_indexes
UNION
SELECT concat(schemaname, '.', relname),
       pg_table_size(relid) as size,
       pg_size_pretty(pg_table_size(relid)) AS pretty_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY 2 DESC

--     Размер одной таблицы суммарный и в пересчёте на одну строку (удобно для прикинуть сколько занимает одна
--     запись на диске)

SELECT l.metric, l.nr AS bytes
     , CASE WHEN is_size THEN pg_size_pretty(nr) END AS bytes_pretty
     , CASE WHEN is_size THEN nr / NULLIF(x.ct, 0) END AS bytes_per_row
FROM  (
          SELECT min(tableoid)        AS tbl      -- = 'public.tbl'::regclass::oid
               , count(*)             AS ct
               , sum(length(t::text)) AS txt_len  -- length in characters
          FROM   partners t                     -- заменить здесь на имя таблицы, которую нужно проанализировать
      ) x
          CROSS  JOIN LATERAL (
    VALUES
    (true , 'core_relation_size'               , pg_relation_size(tbl))
         , (true , 'visibility_map'                   , pg_relation_size(tbl, 'vm'))
         , (true , 'free_space_map'                   , pg_relation_size(tbl, 'fsm'))
         , (true , 'table_size_incl_toast'            , pg_table_size(tbl))
         , (true , 'indexes_size'                     , pg_indexes_size(tbl))
         , (true , 'total_size_incl_toast_and_indexes', pg_total_relation_size(tbl))
         , (true , 'live_rows_in_text_representation' , txt_len)
         , (false, '------------------------------'   , NULL)
         , (false, 'row_count'                        , ct)
         , (false, 'tuples'                      , (SELECT reltuples::int FROM pg_class WHERE pg_class.oid = x.tbl))
         , (false, 'pages'                      , (SELECT relpages::int FROM pg_class WHERE pg_class.oid = x.tbl))
    ) l(is_size, metric, nr);


-- Получение статистики выполнявшихся запросов - по суммарно потраченному СУБД времени на все запросы / по потраченному на выполнение одного запроса времени.

-- Для Postgres 13 и выше нужно заменить total_time на total_exec_time.

SELECT round(total_time::numeric, 2) AS total_time,
       calls,
       ROWS,
       round(total_time::numeric / calls, 2) AS avg_time,
       round((100 * total_time / sum(total_time::numeric) OVER ())::numeric, 2) AS percentage_cpu,
       query
FROM pg_stat_statements
ORDER BY total_time DESC -- для сортировки по суммарному времени выполнению всех копий запроса
-- ORDER BY avg_time DESC -- для сортировки по времени выполнения одного запроса
LIMIT 20;


-- получить физический размер файлов (хранилища) базы данных
SELECT pg_database_size(current_database());
SELECT pg_database_size('DATABASE NAME PASTE');
SELECT pg_size_pretty(pg_database_size(current_database()));

-- получить перечень таблиц базы данных.
SELECT table_name FROM information_schema.tables
WHERE table_schema NOT IN ('information_schema','pg_catalog');

-- все таблицы из указанной схемы текущей базы данных:
SELECT table_name FROM information_schema.tables
WHERE table_schema NOT IN ('information_schema', 'pg_catalog')
  AND table_schema IN('public', 'myschema');

-- размер данных таблицы
SELECT pg_relation_size('accounts');

-- вывести список таблиц текущей базы данных, отсортированный по размеру таблицы
SELECT relname, relpages FROM pg_class ORDER BY relpages DESC;
SELECT relname, relpages FROM pg_class ORDER BY relpages DESC LIMIT 1;

-- узнать имя, IP и используемый порт подключенных пользователей
SELECT datname,usename,client_addr,client_port FROM pg_stat_activity;

-- активность соединения конкретного пользователя
SELECT datname FROM pg_stat_activity WHERE usename = 'devuser';

-- Удалить все дубликаты :
DELETE FROM customers WHERE ctid NOT IN  (SELECT max(ctid) FROM customers GROUP BY customers.*);

-- найти записи с дубликатами:
SELECT * FROM customers WHERE ctid NOT IN (SELECT max(ctid) FROM customers GROUP BY customer_id);

-- Для того, чтобы остановить конкретный запрос,  с указанием id процесса (pid):
SELECT pg_cancel_backend(procpid);

-- Для того, чтобы прекратить работу запроса
SELECT pg_terminate_backend(procpid);

SHOW data_directory;

-- Получим перечень доступных типов данных
SELECT typname, typlen from pg_type where typtype='b';

-- Изменение настроек СУБД без перезагрузки
-- Настройки PostgreSQL находятся в специальных файлах вроде postgresql.conf и pg_hba.conf.
-- После изменения этих файлов нужно, чтобы СУБД снова получила настройки. Для этого производится
-- перезагрузка сервера баз данных. Понятно, что приходится это делать, но на продакшн-версии проекта,
-- которым пользуются тысячи пользователей, это очень нежелательно. Поэтому в PostgreSQL есть функция,
-- с помощью которой можно применить изменения без перезагрузки сервера:
SELECT pg_reload_conf();

--проверка статуса
service postgresql status

--справка postgresql
service postgresql


--войти в запись суперпользователя postgres
sudo -i -u postgres

--открыть консоль postgres
psql

--Базовые команды:
--список баз данных
\l 

--выход из консоли
\q

--Создаём свою базу данных:
createdb <name>

--Заходим снова в консоль БД и проверяем наличие БД
psql
\l
\q


--Удаление БД
dropdb <name>

--показать список пользователей
\du

--Сменим пароль для пользователя постгрес
ALTER USER postgres WITH PASSWORD '<password>';


--Создадим нового пользователя для работы с БД, работать из под рута не рекомендовано
\du
CREATE USER <username>  WITH PASSWORD '<pwd>';

--Теперь дадим права пользователю:
ALTER USER <username> WITH SUPERUSER;

--Удалить пользователя
DROP USER <username>;

--Выйти из консоли:
\q

--Выйти из учётной записи:
exit или ctrl + D

--Вызвать справку по postgres:
man psql


\c   \connect

\conninfo

\pset

\! pwd

\x  - gorizontal to vertical

select * from pg_tables where tablename='pg_class' \gx

\o file1.log

select * from pg_statistic;

\d pg_tables
```
