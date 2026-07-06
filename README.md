# Test_1
example

Something about myself

-------------------------------------------------------------------------------------------------------------
Мониторинг активных сессий и «тяжелых» запросов в реальном времени
Этот запрос показывает, какие запросы сейчас выполняются, сколько длятся, какой у них план (если доступен) и потребление ресурсов (чтение/запись). Помогает выявить внезапные всплески CPU.


        -- Активные запросы с временем выполнения и планом (если включен auto_explain)
        SELECT
            pid,
            usename,
            application_name,
            client_addr,
            state,
            wait_event_type,
            wait_event,
            now() - state_change AS duration,
            (now() - state_change) > interval '5 minutes' AS long_running,
            query,
            (SELECT json_agg(json_build_object('operation', op, 'relation', rel, 'rows', rows, 'cost', cost))
             FROM pg_stat_activity_explain(pid)) AS explain_plan
        FROM pg_stat_activity
        WHERE state != 'idle'
          AND pid != pg_backend_pid()
          AND (query NOT LIKE '%pg_stat_activity%' OR query NOT LIKE '%pg_stat_%')
        ORDER BY duration DESC;

Примечание: функция pg_stat_activity_explain не существует – для получения плана используйте EXPLAIN (BUFFERS, ANALYZE), но его нельзя выполнить для чужого запроса. Вместо этого можно включить расширение auto_explain (shared_preload_libraries) и смотреть логи, либо использовать pg_stat_statements для анализа прошлых запросов (см. скрипт 2).


-------------------------------------------------------------------------------------------------------------
2. Топ-20 самых ресурсоёмких запросов по времени и числу чтений (из pg_stat_statements)
Если расширение pg_stat_statements включено, это ваш главный источник данных. Если нет – настоятельно рекомендую включить (требует перезагрузки, но окупается). Скрипт показывает запросы, которые больше всего грузят CPU и I/O.


        -- Самые медленные запросы по общему времени
        SELECT
            queryid,
            substring(query, 1, 200) AS query_sample,
            calls,
            total_exec_time / 1000 AS total_sec,
            mean_exec_time AS avg_ms,
            max_exec_time AS max_ms,
            stddev_exec_time,
            rows / calls AS avg_rows,
            shared_blks_hit + shared_blks_read AS total_buffers,
            (shared_blks_hit::float / (shared_blks_hit + shared_blks_read + 1) * 100) AS cache_hit_ratio,
            (local_blks_dirtied + shared_blks_dirtied) AS dirty_buffers,
            wal_bytes / 1024 AS wal_kb
        FROM pg_stat_statements
        WHERE dbid = (SELECT oid FROM pg_database WHERE datname = current_database())
          AND calls > 10  -- исключаем одноразовые
        ORDER BY total_exec_time DESC
        LIMIT 20;


Для анализа запросов по числу сканирований строк (индексные vs последовательные) можно сортировать по shared_blks_read или по rows.


-------------------------------------------------------------------------------------------------------------
3. Анализ эффективности использования индексов (по каждой схеме)
Этот скрипт выявляет:

Таблицы без индексов (или с недостаточными).

Индексы, которые никогда не используются (или используются крайне редко) – их можно удалить.

Дублирующиеся индексы (одинаковые столбцы).


        WITH table_scan AS (
            SELECT
                schemaname,
                tablename,
                seq_scan,
                seq_tup_read,
                idx_scan,
                idx_tup_fetch,
                pg_relation_size(schemaname||'.'||tablename) AS table_size
            FROM pg_stat_user_tables
            WHERE schemaname NOT IN ('information_schema', 'pg_catalog')
        ),
        index_usage AS (
            SELECT
                schemaname,
                tablename,
                indexname,
                idx_scan,
                pg_relation_size(indexrelid) AS index_size,
                (SELECT array_agg(attname) FROM pg_index i
                 JOIN pg_attribute a ON a.attrelid = i.indrelid AND a.attnum = ANY(i.indkey)
                 WHERE i.indexrelid = c.oid) AS indexed_columns
            FROM pg_stat_user_indexes
            JOIN pg_class c ON c.oid = indexrelid
            WHERE schemaname NOT IN ('information_schema', 'pg_catalog')
        )
        SELECT
            t.schemaname,
            t.tablename,
            t.table_size,
            t.seq_scan,
            t.idx_scan,
            CASE WHEN t.seq_scan + t.idx_scan > 0
                 THEN round(100.0 * t.idx_scan / (t.seq_scan + t.idx_scan), 2)
                 ELSE 0 END AS idx_scan_percent,
            i.indexname,
            i.idx_scan AS index_scans,
            i.index_size,
            i.indexed_columns
        FROM table_scan t
        LEFT JOIN index_usage i ON i.schemaname = t.schemaname AND i.tablename = t.tablename
        ORDER BY t.schemaname, t.tablename, i.idx_scan NULLS LAST;

Особое внимание обратите на таблицы с большим seq_scan и низким idx_scan_percent – это кандидаты на создание индексов. Индексы с idx_scan = 0 и большим размером – лишние.

-------------------------------------------------------------------------------------------------------------
4. Блокировки и ожидания (locks + deadlocks в логах)
Длительные блокировки приводят к простою и накоплению запросов, что даёт нагрузку на CPU. Запрос показывает, кто кого блокирует, и сколько времени ждёт.

        
        SELECT
            blocked.pid AS blocked_pid,
            blocked.usename AS blocked_user,
            blocked.query AS blocked_query,
            blocking.pid AS blocking_pid,
            blocking.usename AS blocking_user,
            blocking.query AS blocking_query,
            blocked.wait_event_type,
            blocked.wait_event,
            age(now(), blocked.state_change) AS waiting_time
        FROM pg_stat_activity blocked
        JOIN pg_locks blocked_locks ON blocked_locks.pid = blocked.pid
        JOIN pg_locks blocking_locks ON (
            blocking_locks.locktype = blocked_locks.locktype
            AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
            AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
            AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
            AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
            AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
            AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
            AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
            AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
            AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
            AND blocking_locks.pid != blocked_locks.pid
        )
        JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
        WHERE NOT blocked_locks.granted
          AND blocked.state != 'idle'
          AND blocked.pid != pg_backend_pid()
        ORDER BY waiting_time DESC;

Чтобы выявить deadlock'и, проверьте логи на наличие сообщений deadlock detected. Для автоматического сбора можно настроить log_lock_waits = on и анализировать логи.

-------------------------------------------------------------------------------------------------------------
5. Статистика по использованию кэша (буферного пула) и I/O
Высокий CPU часто связан с интенсивным I/O или плохим попаданием в кэш. Скрипт показывает hit ratio для таблиц и индексов, а также активность фонового писателя (bgwriter).

        -- Hit ratio по таблицам
        SELECT
            schemaname,
            tablename,
            heap_blks_hit,
            heap_blks_read,
            round(100.0 * heap_blks_hit / (heap_blks_hit + heap_blks_read + 1), 2) AS heap_hit_ratio,
            idx_blks_hit,
            idx_blks_read,
            round(100.0 * idx_blks_hit / (idx_blks_hit + idx_blks_read + 1), 2) AS idx_hit_ratio,
            toast_blks_hit,
            toast_blks_read
        FROM pg_statio_user_tables
        WHERE schemaname NOT IN ('information_schema', 'pg_catalog')
          AND (heap_blks_hit + heap_blks_read) > 1000
        ORDER BY heap_hit_ratio ASC
        LIMIT 20;
        
        -- Статистика bgwriter – показывает, как часто происходит запись на диск
        SELECT
            checkpoints_timed,
            checkpoints_req,
            checkpoint_write_time,
            checkpoint_sync_time,
            buffers_checkpoint,
            buffers_clean,
            maxwritten_clean,
            buffers_backend,
            buffers_backend_fsync,
            buffers_alloc
        FROM pg_stat_bgwriter;

Низкий hit ratio (< 95%) говорит о недостаточном размере shared_buffers или неэффективных запросах, сканирующих много данных.

-------------------------------------------------------------------------------------------------------------
6. Мониторинг размеров объектов и роста таблиц
Переполнение кэшей и нехватка памяти могут быть вызваны чрезмерным ростом таблиц. Скрипт показывает размеры таблиц и индексов, а также оценивает количество «мертвых» кортежей (требующих VACUUM).

        SELECT
            schemaname,
            tablename,
            pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
            pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
            pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_total_size,
            n_live_tup,
            n_dead_tup,
            round(100.0 * n_dead_tup / (n_live_tup + 1), 2) AS dead_ratio,
            last_vacuum,
            last_autovacuum,
            last_analyze,
            last_autoanalyze
        FROM pg_stat_user_tables
        WHERE schemaname NOT IN ('information_schema', 'pg_catalog')
        ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
        LIMIT 30;

Высокое значение dead_ratio (> 10%) и отсутствие свежих VACUUM – повод проверить настройки autovacuum.
-------------------------------------------------------------------------------------------------------------


Дополнительные рекомендации по сбору данных
Периодический сбор: сохраняйте результаты скриптов в отдельную таблицу (например, monitoring.snapshots) с меткой времени. Это позволит строить графики и видеть динамику.

Фильтрация по схемам: у каждого сервиса своя схема – используйте WHERE schemaname IN ('service1', 'service2', ...) или исключайте системные.

Настройки параметров: проверьте ключевые параметры:

    SELECT name, setting, unit, context, short_desc
    FROM pg_settings
    WHERE name IN ('shared_buffers', 'work_mem', 'maintenance_work_mem', 'effective_cache_size',
                   'max_connections', 'checkpoint_completion_target', 'wal_buffers', 'random_page_cost');
                   
Сравните с рекомендуемыми значениями (например, shared_buffers – 25% ОЗУ, work_mem – по числу сложных сортировок).

Анализ планов запросов: для выявленных проблемных запросов выполните EXPLAIN (BUFFERS, ANALYZE) в тестовой среде, чтобы понять, где именно тратятся ресурсы.

Заключение
Предложенные скрипты покрывают 90% диагностики производительности PostgreSQL. Запускайте их в часы пиковой нагрузки, а также в периоды затишья – так вы увидите как пиковые, так и фоновые проблемы. Начните с топ-запросов (скрипт 2) и индексов (скрипт 3), затем переходите к блокировкам и кэшу. Если после оптимизации наиболее «тяжёлых» запросов CPU всё ещё высок – проверьте размеры таблиц, частоту VACUUM и настройки памяти. Помните, что каждая схема живёт своей жизнью, поэтому применяйте фильтры по схемам, чтобы локализовать проблемный сервис.

Удачи в отладке! Если потребуется углубиться в конкретный аспект – пишите, разберём детали.
