Lab 2: ClickHouse Lab 
---------------------

Евгений Павленко, группа J4140


Схемы таблиц
-------------

1. Входная таблица transactions:
```sql
CREATE TABLE epavlenko_412010.transactions ON CLUSTER kube_clickhouse_cluster
(
    user_id_out UInt32,
    user_id_in UInt32,
    amount Float32,
    datetime DateTime,
    important UInt8
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(datetime)
ORDER BY (user_id_out, user_id_in, datetime);
```

![image](https://github.com/MrShaller/MrShaller/assets/62774239/ca47f405-c8de-4099-b40c-21eb36ce1ec5)

Обоснование выбранного метода шардирования:
------------------------------------------
Был выбран метод шардирования по user_id_out, это поле является уникальным индентификатором пользователя, участвующего в транзакции. Выбор между user_id_in и user_id_out в целом только шардирует данные по кластеру по разному, в целом, разница небольшая.

Списоĸ выбранных MV и запросы
-----------------------------

Задача № 3: Создать MV для нахождения суммы всех входящих и исходящих транзакций от каждого пользователя.
----------------------------------------------------------------------------------------------------------------------

1. Materialized view transactions_sums_months:
```sql
CREATE MATERIALIZED VIEW epavlenko_412010.transactions_sums_months
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree
ORDER BY (user_id, date)
POPULATE AS
WITH
sum_in AS (
    SELECT
        user_id_in AS user_id,
        toStartOfMonth(datetime) AS date,
        sum(amount) AS sum_in
    FROM epavlenko_412010.distr_transactions
    GROUP BY
        user_id,
        date
),
sum_out AS (
    SELECT
        user_id_out AS user_id,
        toStartOfMonth(datetime) AS date,
        sum(amount) AS sum_out
    FROM epavlenko_412010.distr_transactions
    GROUP BY
        user_id,
        date
)
SELECT
    sum_in.user_id,
    sum_in.date,
    sum_out,
    sum_in
FROM sum_in
INNER JOIN sum_out ON (sum_in.user_id = sum_out.user_id) AND (sum_in.date = sum_out.date)
ORDER BY 
    user_id,
    date
```

2. Пример вывода:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/7eb6c5d9-e844-45a6-98ee-426797873126)


Задача № 4: Создать MV для нахождения saldo каждого пользователя (баланс на текущий момент).
---------------------------------------------------------------------------------------------------------------------

1. Materialized view saldo_transactions:
```sql
CREATE MATERIALIZED VIEW epavlenko_412010.saldo_transactions
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree
ORDER BY user_id
POPULATE AS
WITH
saldo_in AS (
    SELECT
        user_id_in AS user_id,
        sum(amount) AS saldo_in
    FROM epavlenko_412010.distr_transactions
    GROUP BY user_id
),
saldo_out AS (
    SELECT
        user_id_out AS user_id,
        sum(amount) AS saldo_out
    FROM epavlenko_412010.distr_transactions
    GROUP BY user_id
)
SELECT
    saldo_in.user_id,
    saldo_out as transactions_out,
    saldo_in as transactions_in,
    (saldo_in - saldo_out) as saldo
FROM saldo_in
INNER JOIN saldo_out ON (saldo_in.user_id = saldo_out.user_id)
ORDER BY user_id
```

Пример вывода:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/ddfb896f-21f4-4e9f-9bdc-4b326cf639b5)
