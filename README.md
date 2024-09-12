# Предварительный анализ систем при переходе на PG

## Работы до начала анализа

```
Подключаем к эластику (за 1 месяц до анализа)
Доступ к забиску или графане
Доступ к продуктиву на открытие внешних обработок - для получения структуры хранения
Доступ к MS SQL
Уточняем размер технологического окна
```
## Структура хранения базы

### Размер БД
```sql
SELECT
    DB_Name(database_id) as [Database Name],
    CAST(SUM(size * 8.0 / 1024 / 1024) AS numeric(10)) as [Size, Gb]
FROM
    sys.master_files
GROUP BY
    database_id
order by
    [Size, Gb] desc
```
Исходя из размера базы определяем методику её миграции на PG

Расчет следующий:

**Если** [Размер базы] / [Скорость миграции (2 Гб/минуту)] > [Время технологического окна] **Тогда** Миграция через план обмена

**Иначе** Миграция через выгрузку базы (через DT или через утилиту ibcmd)

Если на сервере MS SQL несколько баз, то их также необходимо будет мигрировать на PG или на другой MSSQL сервер для полного освобождения рассматриваемого сервера.

По каждой базе на сервере СУБД необходимо получить информацию следующего вида: Название базы, Размер, Владелец, Ответственный за развитие, Номер ИС
### Размер и число записей в таблицах
```sql
SELECT
    t.Name AS TableName,
    s.Name AS SchemaName,
    p.Rows AS RowCounts,
    SUM(a.total_pages) * 8 / 1024 / 1024 AS TotalSpaceGB,
    SUM(a.used_pages) * 8 / 1024 / 1024 AS UsedSpaceGB,
    (SUM(a.total_pages) - SUM(a.used_pages)) * 8 AS UnusedSpaceKB
FROM
    sys.tables t
    INNER JOIN sys.indexes i ON t.object_id = i.object_id
    INNER JOIN sys.partitions p ON i.object_id = p.object_id
    AND i.index_id = p.index_id
    INNER JOIN sys.allocation_units a ON p.partition_id = a.container_id
    LEFT OUTER JOIN sys.schemas s ON t.schema_id = s.schema_id
WHERE
    t.Name NOT LIKE 'dt%'
    AND t.is_ms_shipped = 0
    AND i.object_id > 255
GROUP BY
    t.Name,
    s.Name,
    p.Rows
ORDER BY
    UsedSpaceGB DESC;

GO
```
Если размер базы более 1Тб, то необходимо выполнить анализ структуры БД на предмет хранения в ней файлов или больших
таблиц с итогами.

При этом рекомендуется хранение файлов выносить за пределы БД ( [Возможность БСП](https://its.1c.ru/db/sdadmin/content/392/hdoc "Возможность БСП"); [Возможность 1С 8.3.22](https://infostart.ru/journal/news/mir-1s/novyy-mekhanizm-khranilishche-dvoichnykh-dannykh-v-1s-predpriyatie-8-3-22_1562699/ "Возможность 1С 8.3.22") )

Большие таблицы с итогами можно очищать перед миграцией и пересчитывать в базе поле миграции.

### Максимальный размер записи

```sql
SELECT
    MAX(record.Size) / 1024 / 1024 AS [ (mb)],
    CASE
        WHEN MAX(record.Size) / 1024 / 1024 > 1000 THEN ''
        ELSE ''
    END AS [ PG]
FROM
    dbo._InfoRg2133 --
    CROSS APPLY (
        SELECT
            datalength([_Fld2135]) -- varbinary(max) image
    ) record(Size);
```

Если максимальный размер записи более 1Гб, то это вызовет ошибку при миграции на PostgreSQL через DT. Данная проблема

связана с ограничениями PG на размер записи

### Нетиповые объекты базы

Механики ниже необходимы для базового представления, итоговые данные о наличии таких объектов необходимо запрашивать у
ответственных за развитие ИС


**Индексы**

Получение списка ВСЕХ индексов

Для получения кастомных индексов в базе 1С была разработана обработка, которая показывает, какие индексы были изменены

/добавлены/удалены и генерирует код создания аналогичного индекса в PG

Для конвертации индексов можно использовать инструмент https://www.sqlines.com/online

```sql
select 
	i.[name] as index_name,
    substring(column_names, 1, len(column_names)-1) as [columns],
    case when i.[type] = 1 then 'Clustered index'
        when i.[type] = 2 then 'Nonclustered unique index'
        when i.[type] = 3 then 'XML index'
        when i.[type] = 4 then 'Spatial index'
        when i.[type] = 5 then 'Clustered columnstore index'
        when i.[type] = 6 then 'Nonclustered columnstore index'
        when i.[type] = 7 then 'Nonclustered hash index'
        end as index_type,
    case when i.is_unique = 1 then 'Unique'
        else 'Not unique' end as [unique],
    schema_name(t.schema_id) + '.' + t.[name] as table_view,
    case when t.[type] = 'U' then 'Table'
        when t.[type] = 'V' then 'View'
        end as [object_type]
from sys.objects t
    inner join sys.indexes i
        on t.object_id = i.object_id
    cross apply (select col.[name] + ', '
                    from sys.index_columns ic
                        inner join sys.columns col
                            on ic.object_id = col.object_id
                            and ic.column_id = col.column_id
                    where ic.object_id = t.object_id
                        and ic.index_id = i.index_id
                            order by key_ordinal
                            for xml path ('') ) D (column_names)
where t.is_ms_shipped <> 1
and index_id > 0
order by i.[name]
```

**ВАЖНО!!!** При тестовой миграции необходимо провести сравнение числа индексов по таблицам для повторного контроля наличия нетиповых индексов


**Таблицы**

```sql
SELECT
    LOWER(name) AS [],
    xtype AS [],
    crdate AS [ ]
FROM
    sysobjects
WHERE
    NOT xtype IN ('P', 'FN')
    AND xtype = 'U'
    AND NOT LEFT(LOWER(name), 4) IN (
        '_ref',
        '_inf',
        '_con',
        '_enu',
        '_cki',
        '_tas',
        '_sch',
        '_bpr',
        '_chr',
        '_dbc',
        '_dat',
        '_bot',
        '_nod',
        '_doc',
        '_acc'
    )
    AND NOT LOWER(name) IN (
        '_urlexternaldata',
        '_extdatasrcprms',
        '_yearoffset',
        '_accopt',
        'config',
        'configsave',
        'params',
        '_extensionsinfo',
        'files',
        '_extensionsinfongs',
        'configcas',
        '_extensionsrestruct',
        '_extensionsrestructngs',
        'configcassave',
        'v8users',
        'ibversion',
        '_systemsettings',
        '_errorprocessingsettings',
        '_commonsettings',
        '_repsettings',
        '_repvarsettings',
        '_frmdtsettings',
        '_usersworkhistory',
        '_dynlistsettings',
        '_odatasettings',
        '_ecs',
        'dbschema',
        'schemastorage'
    )
```

**Представления, процедуры, функции и триггеры**

```sql
SELECT
    LOWER(name) AS name,
    '' AS type,
    crdate AS [ ]
FROM
    sysobjects
WHERE
    xtype = 'P'
    AND name NOT LIKE 'dt_%'
UNION
ALL
SELECT
    LOWER(name) AS name,
    '' AS type,
    crdate AS [ ]
FROM
    sysobjects
WHERE
    xtype = 'FN'
UNION
ALL
SELECT
    LOWER(TABLE_NAME) AS name,
    '' AS type,
    '' AS [ ]
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    TABLE_TYPE = 'VIEW'
    AND TABLE_NAME NOT IN ('syssegments', 'sysconstraints')
UNION
ALL
SELECT
    LOWER(name) AS name,
    '' AS type,
    crdate AS [ ]
FROM
    sysobjects
WHERE
    xtype = 'TR'
```

Полученный список предварительный, его необходимо дополнить ответственному за развитие системы.

## Нагрузка на СУБД

### Характеристики сервера

**Реализация (физика или ВМ)**

Влияет на то, как будет создаваться сервер-замена с PG. Если это ВМ, то поднимается аналогичная ВМ, выполняется миграция, если сервер физический, то необходимо поднять аналогичный или лучше по характеристикам сервер, выполнить миграцию,

Необходимые данные:
```
CPU (Частота, ядра, поколение)
RAM (объем)
Диски (Размер, тип)
Текущая утилизация CPU (возможно нужна оптимизация, которая снимет нагрузку или апгрейд)
Свободное место на диске с базой (возможно потребуется апгрейд)
```
### Использование ресурсов в разрезе баз данных

Если рассматриваемая база соседствует на сервере с другими базами, то необходимо понимать сколько ресурсов потребляет
конкретная ИС


### CPU time

```sql
WITH DB_CPU_Stats AS (
    SELECT
        DatabaseID,
        DB_Name(DatabaseID) AS [DatabaseName],
        SUM(total_worker_time) AS [CPU_Time_Ms]
    FROM
        sys.dm_exec_query_stats AS qs
        CROSS APPLY (
            SELECT
                CONVERT(int, value) AS [DatabaseID]
            FROM
                sys.dm_exec_plan_attributes(qs.plan_handle)
            WHERE
                attribute = N'dbid'
        ) AS F_DB
    GROUP BY
        DatabaseID
)
SELECT
    ROW_NUMBER() OVER(
        ORDER BY
            [CPU_Time_Ms] DESC
    ) AS [row_num],
    DatabaseName,
    [CPU_Time_Ms],
    CAST(
        [CPU_Time_Ms] * 1.0 / SUM([CPU_Time_Ms]) OVER() * 100.0 AS DECIMAL(5, 2)
    ) AS [CPUPercent]
FROM
    DB_CPU_Stats
WHERE
    DatabaseID > 4 -- system databases
    AND DatabaseID <> 32767 -- ResourceDB
ORDER BY
    row_num OPTION (RECOMPILE);
```

### RAM

```sql
SELECT
    DB_NAME(database_id) AS DB,
    COUNT(row_count) * 8.00 / 1024.00 AS MB,
    COUNT(row_count) * 8.00 / 1024.00 / 1024.00 AS GB
FROM
    sys.dm_os_buffer_descriptors
GROUP BY
    database_id
ORDER BY
    MB DESC
```

### HDD read/write

```sql
WITH DB_Disk_Reads_Stats AS (
    SELECT
        DatabaseID,
        DB_Name(DatabaseID) AS [DatabaseName],
        SUM(total_physical_reads) AS [physical_reads]
    FROM
        sys.dm_exec_query_stats AS qs
        CROSS APPLY (
            SELECT
                CONVERT(int, value) AS [DatabaseID]
            FROM
                sys.dm_exec_plan_attributes(qs.plan_handle)
            WHERE
                attribute = N'dbid'
        ) AS F_DB
    GROUP BY
        DatabaseID
)
SELECT
    ROW_NUMBER() OVER(
        ORDER BY
            [physical_reads] DESC
    ) AS [row_num],
    DatabaseName,
    [physical_reads],
    CAST(
        [physical_reads] * 1.0 / SUM([physical_reads]) OVER() * 100.0 AS DECIMAL(5, 2)
    ) AS [Physical_Reads_Percent]
FROM
    DB_Disk_Reads_Stats
WHERE
    DatabaseID > 4 -- system databases
    AND DatabaseID <> 32767 -- ResourceDB
ORDER BY
    row_num OPTION (RECOMPILE);
```

## Анализ функционала нагружающего СУБД

### OpenSearch (Тех.журнал)

Частые запросы 
Ресурсоемкие запросы

### MS SQL

**Частые запросы**
```sql
SELECT
    TOP 100 [Execution count] = execution_count,
    [Execution time] = total_worker_time,
    [Avg execution time] = [total_worker_time] / execution_count,
    [Individual Query] = SUBSTRING (
        qt.text,
        qs.statement_start_offset / 2,
        (
            CASE
                WHEN qs.statement_end_offset = - THEN LEN(CONVERT(NVARCHAR(MAX), qt.text)) * 2
                ELSE qs.statement_end_offset
            END - qs.statement_start_offset
        ) / 2
    ),
    [Parent Query] = qt.text,
    DatabaseName = DB_NAME(qt.dbid)
FROM
    sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) as qt
ORDER BY
    [Execution count] DESC;
```

