ClickHouse Lab Report
---------------------

Витко Михаил, группа J4140


Схемы таблиц
-------------

1. Входная таблица mvitko_411071.transactions:
```sql
CREATE TABLE mvitko_411071.transactions ON CLUSTER kube_clickhouse_cluster
(
    user_id_in UInt32,
    user_id_out UInt32,
    amount Float32,
    important UInt8,
    datetime DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(datetime)
ORDER BY (user_id_out, user_id_in, datetime);
```
![image](https://github.com/MrShaller/MrShaller/assets/62774239/c31dab0a-e98d-4206-b5a3-99cfc355c3d8)


2. Таблицы для 3 пункта:


Таблица mvitko_411071.incoming_transactions_3:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/a7701958-ed39-43da-b86d-716c9e21c94a)

Содержит сумму входящих транзакций для каждого user_id (на каждый месяц)


Таблица mvitko_411071.outcoming_transactions_3:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/801048dc-3796-448a-b7d2-451dc83997e5)

Содержит сумму исходящих транзакций для каждого user_id (на каждый месяц)


Результирующая VIEW mvitko_411071.summ_transactions_view:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/e4fc130d-6efe-49aa-9973-8a05bd3e886c)

Содержит собранную таблицу из предыдущих двух


3. Таблицы для 4 пункта:

Таблица mvitko_411071.saldo_incoming_4:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/9c7f7067-cf11-4c20-baba-19e43460f722)

Содержит сумму всех входящих транзакций каждого пользователя


Таблица mvitko_411071.saldo_outcoming_4:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/0bdede0c-c1ff-4eb1-85da-dcc2a3686d85)

Содержит сумму всех исходящих транзакций каждого пользователя


Результирующая VIEW mvitko_411071.saldo_view:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/4537a53d-7f62-4882-95b6-bf4a5fd81f07)

Содержит результат двух предыдущих таблиц

Обоснование выбранного метода шардирования:
------------------------------------------
Был выбран метод шардирования по user_id_out в целях распределения шардов по этому полю, думаю, чаще всего запросы на практике(и на работе) идут именно на это поле.

Списоĸ выбранных MV и запросы
-----------------------------

4 лабораторная задача: Найти баланс(saldo) пользователя на текущий момент.
----------------------------------------------------------------------------------------------------------------------

1. MV для входящих транзакций от пользователей (Суммируем по user_id_in):
```sql
CREATE MATERIALIZED VIEW mvitko_411071.saldo_incoming_4
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY user_id
POPULATE 
AS
SELECT
    user_id_in AS user_id,
    sum(amount) AS sum_incoming_transaction
FROM mvitko_411071.transactions
GROUP BY user_id_in;
```
2. Создание распределенной таблицы для входящих транзакций из пункта 1:
```sql
CREATE TABLE mvitko_411071.distr_saldo_incoming_4
ON CLUSTER kube_clickhouse_cluster AS mvitko_411071.saldo_incoming_4
ENGINE = Distributed(kube_clickhouse_cluster, mvitko_411071,
saldo_incoming_4, xxHash64(user_id));
```
3. MV для исходящих транзакций от пользователей(суммируем по user_id_out):
```sql
CREATE MATERIALIZED VIEW mvitko_411071.saldo_outcoming_4
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY user_id
POPULATE 
AS
SELECT
    user_id_out AS user_id,
    sum(amount) AS sum_outcoming_transaction
FROM mvitko_411071.transactions
GROUP BY user_id_out;
```
4. Создание распределенной таблицы для исходящих транзакций из пункта 3:
```sql
CREATE TABLE mvitko_411071.distr_saldo_outcoming_4
ON CLUSTER kube_clickhouse_cluster AS mvitko_411071.saldo_outcoming_4
ENGINE = Distributed(kube_clickhouse_cluster, mvitko_411071,
saldo_outcoming_4, xxHash64(user_id));
```
5. Итоговое представление полученных данных, VIEW:

```sql
CREATE VIEW mvitko_411071.saldo_view ON CLUSTER kube_clickhouse_cluster
AS
SELECT
    user_id,
    sum(sum_incoming_transaction) AS sum_incoming_transaction,
    sum(sum_outcoming_transaction) AS sum_outcoming_transaction,
    (sum_incoming_transaction - sum_outcoming_transaction) AS saldo
FROM
(
    SELECT
        user_id,
        sum_incoming_transaction,
        0 AS sum_outcoming_transaction
    FROM
        mvitko_411071.distr_saldo_incoming_4
    UNION ALL
    SELECT
        user_id,
        0 AS sum_incoming_transaction,
        sum_outcoming_transaction
    FROM
        mvitko_411071.distr_saldo_outcoming_4
) AS comb
GROUP BY user_id
```
Пример запроса и результата:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/f12b54c5-5a4e-46b4-ab1a-11d6d605c6c8)


Лабораторная 3: Найти сумму всех входящих и исходящих транзакций на каждый месяц для каждого отдельного пользователя.
---------------------------------------------------------------------------------------------------------------------

1. MV и распределенная таблица для входящих транзакций:
```sql
CREATE MATERIALIZED VIEW mvitko_411071.incoming_transactions_3
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY (user_id, month)
POPULATE 
AS
SELECT
    user_id_in AS user_id,
    toYYYYMM(datetime) AS month,
    sum(amount) AS sum_incoming_transaction
FROM mvitko_411071.transactions
GROUP BY user_id_in, month;

CREATE TABLE mvitko_411071.distr_incoming_transactions_3
ON CLUSTER kube_clickhouse_cluster AS mvitko_411071.incoming_transactions_3
ENGINE = Distributed(kube_clickhouse_cluster, mvitko_411071,
incoming_transactions_3, xxHash64(user_id));
```

2. MV и распределенная таблица для исходящих транзакций:
```sql
CREATE MATERIALIZED VIEW mvitko_411071.outcoming_transactions_3
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY (user_id, month)
POPULATE 
AS
SELECT
    user_id_out AS user_id,
    toYYYYMM(datetime) AS month,
    sum(amount) AS sum_outcoming_transaction
FROM mvitko_411071.transactions
GROUP BY user_id_out, month;

CREATE TABLE mvitko_411071.distr_outcoming_transactions_3
ON CLUSTER kube_clickhouse_cluster AS mvitko_411071.outcoming_transactions_3
ENGINE = Distributed(kube_clickhouse_cluster, mvitko_411071,
outcoming_transactions_3, xxHash64(user_id));
```

3. Объединяющая таблица, VIEW:

```sql
CREATE VIEW mvitko_411071.summ_transactions_view ON CLUSTER kube_clickhouse_cluster
AS
SELECT
    user_id,
    month,
    sum(sum_incoming_transaction) AS sum_incoming_transaction,
    sum(sum_outcoming_transaction) AS sum_outcoming_transaction
FROM
(
    SELECT
        user_id,
        month,
        sum_incoming_transaction,
        0 AS sum_outcoming_transaction
    FROM
        mvitko_411071.distr_incoming_transactions_3
    UNION ALL
    SELECT
        user_id,
        month,
        0 AS sum_incoming_transaction,
        sum_outcoming_transaction
    FROM
        mvitko_411071.distr_outcoming_transactions_3
) AS comb
GROUP BY user_id, month;
```

Пример запроса и результат:

![image](https://github.com/MrShaller/MrShaller/assets/62774239/937d9a48-4582-4ebd-8ee6-98445b9d5878)
