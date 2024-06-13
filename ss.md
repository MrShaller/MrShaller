Lab 2: ClickHouse Lab 
---------------------

Евгений Павленко, группа J4151

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

1. Materialized view - in_transactions для входящих транзакций по месяцам для пользователей:
```sql
CREATE MATERIALIZED VIEW epavlenko_412010.in_transactions
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree
ORDER BY (user_id, month)
POPULATE AS
SELECT
    user_id_in AS user_id,
    toYYYYMM(datetime) AS month,
    sum(amount) AS sum_in
FROM epavlenko_412010.transactions
GROUP BY user_id_in, month
```

2. Распределенная таблица по MV, distr_in_transactions:
```sql
CREATE TABLE epavlenko_412010.distr_in_transactions
ON CLUSTER kube_clickhouse_cluster AS epavlenko_412010.in_transactions
ENGINE = Distributed(kube_clickhouse_cluster, epavlenko_412010,
in_transactions, xxHash64(user_id));
```

3. MV out_transactions и распределенная таблица distr_out_transactions для исходящих транзакций по месяцам для пользователей:
```sql
CREATE MATERIALIZED VIEW epavlenko_412010.out_transactions
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY (user_id, month)
POPULATE AS
SELECT
    user_id_out AS user_id,
    toYYYYMM(datetime) AS month,
    sum(amount) AS sum_out
FROM epavlenko_412010.transactions
GROUP BY user_id_out, month

CREATE TABLE epavlenko_412010.distr_out_transactions
ON CLUSTER kube_clickhouse_cluster AS epavlenko_412010.out_transactions
ENGINE = Distributed(kube_clickhouse_cluster, epavlenko_412010,
out_transactions, xxHash64(user_id));
```

4. VIEW transactions_by_monts, который собирает из этих двух MV результат:
```sql
CREATE VIEW epavlenko_412010.transactions_by_months ON CLUSTER kube_clickhouse_cluster
AS SELECT
    user_id,
    month,
    sum(sum_in) AS sum_in,
    sum(sum_out) AS sum_out
FROM
(
    SELECT
        user_id,
        month,
        sum_in,
        0 AS sum_out
    FROM
        epavlenko_412010.distr_in_transactions
    UNION ALL
    SELECT
        user_id,
        month,
        0 AS sum_in,
        sum_out
    FROM
        epavlenko_412010.distr_out_transactions
) AS combinatedview
GROUP BY user_id, month;
```


5. Пример команды вызова и результат:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/a73f14cf-83eb-40cb-9429-0a1fcdc3b447)

Задача № 4: Создать MV для нахождения saldo каждого пользователя (баланс на текущий момент).
---------------------------------------------------------------------------------------------------------------------

1. MV и распределенная таблица для входящих транзакций для каждого пользователя:
```sql
CREATE MATERIALIZED VIEW epavlenko_412010.in_saldo
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY user_id
POPULATE AS
SELECT
    user_id_in AS user_id,
    sum(amount) AS sum_in
FROM epavlenko_412010.transactions
GROUP BY user_id_in

CREATE TABLE epavlenko_412010.distr_in_saldo
ON CLUSTER kube_clickhouse_cluster AS epavlenko_412010.in_saldo
ENGINE = Distributed(kube_clickhouse_cluster, epavlenko_412010,
in_saldo, xxHash64(user_id));

```

2. MV и распределенная таблица для исходящих транзакций для каждого пользователя:
```sql
CREATE MATERIALIZED VIEW epavlenko_412010.out_saldo
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY user_id
POPULATE AS
SELECT
    user_id_out AS user_id,
    sum(amount) AS sum_out
FROM epavlenko_412010.transactions
GROUP BY user_id_out

CREATE TABLE epavlenko_412010.distr_out_saldo
ON CLUSTER kube_clickhouse_cluster AS epavlenko_412010.out_saldo
ENGINE = Distributed(kube_clickhouse_cluster, epavlenko_412010,
out_saldo, xxHash64(user_id));
```

3. VIEW:
```sql
CREATE VIEW epavlenko_412010.saldo ON CLUSTER kube_clickhouse_cluster
AS
SELECT
    user_id,
    sum(sum_in) AS sum_in,
    sum(sum_out) AS sum_out,
    (sum_in - sum_out) AS saldo
FROM
(
    SELECT
        user_id,
        sum_in,
        0 AS sum_out
    FROM
        epavlenko_412010.distr_in_saldo
    UNION ALL
    SELECT
        user_id,
        0 AS sum_in,
        sum_out
    FROM
        epavlenko_412010.distr_out_saldo
) AS combinedviewsaldo
GROUP BY user_id
```

Пример команды вызова и результат:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/bca1ed6a-bf17-4034-9681-56221014da7f)

