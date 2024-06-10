ClickHouse Lab Report
---------------------

Витко Михаил J4140


Table Schemas
-------------

1. Входная таблица mvitko_411071.transactions:

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

