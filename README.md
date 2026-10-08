# goit-rdb-hw-07 — Часові ряди та напівструктуровані дані у задачах ML

Домашнє завдання до Теми 7 курсу «Реляційні бази даних». Notebook `hw7_timeseries_jsonb.ipynb` будує відтворюваний pipeline у PostgreSQL (через `pgserver` у Google Colab): time-series ML-ознаки на даних NYC Taxi і JSONB event-log з GIN- та expression-індексами.

## Джерело даних

**NYC TLC Yellow Taxi Trip Records:** <https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page>

* Файли `yellow_tripdata_2024-02.parquet` і `yellow_tripdata_2024-03.parquet` notebook завантажує сам із `https://d37ci6vzurychx.cloudfront.net/trip-data/`. Parquet у репозиторій не додається.
* Беруться дві місячні партиції (60 днів), щоб ознака `trips_lag_30d` мала достатньо історії. З кожного місяця береться випадкова вибірка 250 000 поїздок (`random_state=42`), разом 500 000.
* Колонки: `tpep_pickup_datetime`, `tpep_dropoff_datetime`, `PULocationID`, `DOLocationID`, `trip_distance`, `fare_amount`, `passenger_count`.
* Очищення: поїздки поза місяцем файлу відкидаються; лишаються рядки з `0 < fare_amount < 500`, `0 < trip_distance < 100` і заповненим `PULocationID`.
* **Fallback:** якщо завантаження не вдається, notebook автоматично будує stub-schema (60 днів × 5 zones = 300 рядків, фіксована дата `2024-03-31`). Яке джерело використано, видно у змінній `DATA_SOURCE` в output notebook (`nyc_tlc` або `stub`). Примусово увімкнути stub можна через `USE_REAL_DATA = False`.

**Event-log** `t7_hw_events` синтетичний і генерується в SQL через `generate_series`: 50 000 подій, 1 000 користувачів, 60 днів, фіксована опорна дата.

## Що реалізовано

### Time-series features (`t7_hw_taxi_daily` → `v_t7_hw_taxi_features`)

Daily-агрегат на рівні `(pickup_zone_id, ts_day)`; `ts_day DATE` — локальний день NYC.

| Ознака | Window | Frame |
|---|---|---|
| `trips_lag_1d`, `trips_lag_7d`, `trips_lag_30d` | `w_pid` | `LAG` за `ORDER BY ts_day` |
| `rolling_avg_fare_7d` | `w7` | `RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW` |
| `rolling_std_fare_30d` | `w30` | `RANGE BETWEEN INTERVAL '29 days' PRECEDING AND CURRENT ROW` |
| `cumulative_trips`, `expanding_avg_trips` | `we` | `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |

Усі чотири вікна мають `PARTITION BY pickup_zone_id`; notebook перевіряє це `assert`-ом по `pg_get_viewdef`. Додатково в notebook є:

* демонстрація різниці ROWS і RANGE на днях із пропусками;
* приклад «витоку» LAG між zone, якщо прибрати `PARTITION BY`;
* діагностика пропусків у календарі;
* приклад `TIMESTAMP` проти `TIMESTAMPTZ` через перехід на літній час (DST).

### JSONB

* 5 event-типів із різними схемами payload: `page_view`, `click`, `add_to_cart`, `checkout`, `recommendation_shown`.
* Запити: containment `@>`, key existence `?` / `?|` / `?&`, JSONPath через `jsonb_path_query`, `jsonb_path_exists` і `@?`.
* `EXPLAIN (ANALYZE, BUFFERS)`:
  * `@>` до і після `GIN (payload jsonb_path_ops)`;
  * `payload ->> 'device' = 'mobile'` до і після expression B-tree index;
  * додатково: `jsonb_ops` проти `jsonb_path_ops` для оператора `?` і вимірювання write-cost індексів.
* `t7_hw_checkouts_norm` — нормалізована таблиця для checkout-подій. Той самий аналітичний запит виконується у JSONB- і normalized-версії.
* Bonus: MongoDB aggregation pipeline як pseudocode.
* Таблиця метрик EXPLAIN збирається автоматично. Після неї — Reflection.

## Як запустити

1. Відкрийте `hw7_timeseries_jsonb.ipynb` у Google Colab.
2. Оберіть **Runtime → Restart session and run all**. Перша клітинка встановлює `pgserver psycopg2-binary sqlalchemy pandas pyarrow requests`; завантаження даних займає кілька хвилин.
3. Остання клітинка виводить чекліст ✅ і завершується `assert`, якщо якийсь пункт не виконано.

Для локального запуску з наявним PostgreSQL можна задати змінну середовища `HW7_PG_URI` (наприклад, `postgresql+psycopg2://user:pass@localhost:5432/db`). Тоді `pgserver` не використовується.

Усі службові об'єкти мають префікс `t7_hw_` і створюються через `DROP ... IF EXISTS`, тож повторний запуск відтворюваний.
