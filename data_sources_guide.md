# Инструкция агента по источникам данных (Data Sources Guide)

Единая справка по источникам данных аналитической платформы (Greenplum, ClickHouse, MS SQL, Patroni, S3, Trino, внешние сервисы). Применяется к любой аналитической задаче: выбор/проверка таблиц, маппинг ключей, фильтры, часовые пояса и подводные камни.

Version: v2.2
Owner: аналитик (data sources)
Last updated: 2026-11-10
Sources: собрано из `E088` (PA-19784 docs, PA-19963, PA-20216, PA-20868, PA-20397, PA-20729, PA-21712, PA-21675), `E041 GraphHopper`, `smkt_db*`.

---

## 1. Подключение к БД (библиотеки)

| Среда | БД / хост | Библиотека | Заметки |
|------|-----------|-----------|---------|
| Greenplum | `greenplum-prod.dwh.ecom.tech:5433`, `gp_prd` (также Spb2/Moscow) | `smkt_db/smkt_greenplum`, `smkt_db2.gp.GreenplumFacade` | креды через `.env` (`GP_*`) или `credentials/creds.py` |
| ClickHouse | `os-002033006.srvr.smkt.corp:8123` (и `os-091002001`, `os-011002169`, `os-011002162`) | `smkt_db2.ch.ClickHouseFacade` | креды `CH_USER/CH_PWD` |
| MS SQL Server | `HW-011002222.samokat.io,1433`, база `DWH`, схема `[olap]` | отдельный клиент | — |
| Patroni (PostgreSQL) | `240.patroni.smkt.corp:5432`, база `store_track` | отдельный клиент | DSA чек-листы |
| S3 / Minio | `https://minio-ml-p01.samokat.io` | `smkt_clickhouse`/s3 | parquet-прогнозы |
| Trino / Presto | `lake.ecom.tech:443` | Trino API | Data Lake |

Приоритет настроек Greenplum: аргументы → env `GP_*` → `credentials/creds.py` → defaults.
Приоритет настроек ClickHouse: аргументы → env `CH_*` → `credentials/creds.py` → defaults (host/port).

---

## 2. Карта слоёв и ключи ЦФЗ (Dark Store)

Три схемы-слоя, у каждого свой тип ключа магазина. **Ключи между слоями нельзя смешивать напрямую.**

| Слой | Схемы | Примеры таблиц | Ключ ЦФЗ |
|------|-------|----------------|----------|
| Sandbox / витрины | `grp_sandbox` | `dm_order_extended`, `mincheck_overview__darkstore`, все `*_agg_*`, `tkaftaranov_*` | `ds_id` / `darkstore_id` (lowercase UUID) |
| Витрины роверов (PA-20651) | `grp_sandbox` | `rover_dashboard__agg_session` (почас. конверсии/сессии), `rover_dashboard__dau_agg` (DAU даркстора/день), `rover_dashboard__completion_time` (ср. время этапов роверных заказов) | `ds_id`, `part_dt` |
| ODS (снапшоты) | `grp_ods_*` | `store_snp`, `service_area_suspension_snp`, `order_snp`, `picker_snp`, `delivery_snp`, `worked_out_shift_snp` | `store_id` (UUID, **VARCHAR**; у `store_snp` — **UPPER** регистр) |
| EM (факты) | `grp_em` | `fct_shift`, `fct_shift_assignment`, `dim_store`, `dim_sku`, `fct_order` | `store_rk` (bigint) |
| Бизнес-слой BV | `grp_bv` | `c_order_cancelled`, `c_order` | `order_id` / `employee_profile_courier_rk` |

**Правила join ключей ЦФЗ:**
- ODS `store_snp.store_id` — UUID в **верхнем** регистре → всегда `lower(st.store_id) = ds.ds_id`.
- Sandbox `mincheck_overview__darkstore.ds_id` — UUID в нижнем регистре.
- EM-таблицы joinятся только по `store_rk` через `mincheck_overview__darkstore.store_rk` (мост Sandbox↔EM).
- SCD-справочник `grp_em.dim_store` (store_rk → store_id) — join **всегда с датой**: `date between valid_from_dt and valid_to_dt`.

---

## 3. Основные справочники (dim / магазины)

| Таблица | Схема | Назначение | Ключ |
|---------|-------|-----------|------|
| `mincheck_overview__darkstore` | `grp_sandbox` | Базовый список ЦФЗ: ds_id, store_rk, статус, тип, TZ, время рабочего окна | `ds_id` |
| `store_snp` | `grp_ods_aps3_sfdct` | Название, город, СВ, РТ | `lower(store_id)` |
| `dim_store` | `grp_em` | SCD-справочник store_rk→store_id, store_type_dk, city_dk, territory_nm, region_nm | `store_rk` (+дата) |
| `dim_sku` | `grp_em` | Товары: sku_uuid, sku_nm, категории | `sku_uuid` |
| `warehouse_snp`, `city_snp` | `grp_ods_dstwh` | Склады, города | `id` / `city_id` |
| `service_area_snp` | `grp_ods_es00_sam00` | Зоны глушения (tagId + categoryId → тип зоны) | `"id"` |

**Атрибуты ЦФЗ из `mincheck_overview__darkstore`:**
- `timezone_offset_hr` — смещение TZ относительно UTC.
- `workday_start_lcl`, `workday_end_lcl` — рабочее окно (например 08:00–23:00; пересёк полуночи — обработать).
- `status` (`OPERATING`), `"type"` (`DARK_STORE`, `DARK_STORE_MAX`).
- `started_to_operate_dt` — дата начала работы (открытия) даркстора.
- `finished_to_operate_dt` — дата закрытия (NULL = всё ещё работает).
- Фильтр «даркстор работал весь период» (PA-20216): `status = 'OPERATING'` и `"type" in ('DARK_STORE','DARK_STORE_MAX')` и `started_to_operate_dt <= start_dt` и `coalesce(finished_to_operate_dt, now()) >= end_dt` (паттерн из PA-20262AB).

---

## 4. Факты: заказы, доставка, сборка

| Таблица | Схема | Назначение | Ключевые поля |
|---------|-------|-----------|---------------|
| `dm_order_extended` | `grp_sandbox` | Основная по заказам: время, статус, опоздания, GMV, клиент | `order_id`, `ds_id`, `created_utc_dttm`, `pick*/delivery_*_dttm`, `order_final_status`, `delivery_final_status`, `late_mnt`, `is_late`, `is_crit_late`, `gmv`, `log_strategy`, тайминги этапов: `waiting_for_picking_mnt`, `picking_mnt`, `waiting_for_delivery_mnt`, `delivery_mnt`, `c2d_mnt` (click-to-delivery), SLA: `client_sla`, `geo_sla`, `sla_mnt`, клиент: `customer_rk` / `customer_id` / `customer_uuid` |
| `order_snp` | `grp_ods_es00_orhst` | Снимок заказов (альтернатива). Состав заказа — в JSON-колонке `"orderItems"` (массив позиций, см. подводные камни ниже) | `"id"`, времена, `"showCaseTypeName"`, `"sourceName"`, `"orderItems"` |
| `order_history_snp` | ⚠ `grp_ods_orhst` — **схема устарела** | История статусов заказа. Всю схему `grp_ods_orhst` заменила `grp_ods_es00_orhst`: искать замену там (пока в ней только `order_snp`/`order_cdc`) и обновить эту строку гайда при нахождении источника | `order_id`, `store_id`, `order_status_id`, timestamps |
| `order_line_snp` | ⚠ `grp_ods_orhst` — **схема устарела, заменена** | Строки заказа: товары, цены, кол-во. Схему `grp_ods_orhst` заменила `grp_ods_es00_orhst`; состав заказа теперь хранится в JSON `"orderItems"` у `grp_ods_es00_orhst.order_snp` (структура — см. подводные камни `order_snp` ниже). Данные в старой таблице не обновляются (на 2026-11-02 свежесть только до 2026-10-25) — не использовать | `product_id` → `"productId"`, `price` → `"productPrice"`, `quantity` → повторение элементов с тем же `"productId"` |
| `delivery_snp` | `grp_ods_es00_dlvry` | Доставки | `delivery_id`, `"courierId"`, timestamps |
| `fulfillment_snp` (`fulfillment_cdc`) | `grp_ods_es00_flfmt` | Время fulfilment. Доставка ровером: `"orderProcessingFlag"` ilike `'%ROVER_DELIVERY%'` → `'ROVER'` иначе `'COURIER'`. Фолбэк ровер→курьер: поля `"orderId"`, `"createdDateTime"`, `"storeId"`, `"changeDateTime"` (сорт. по changeDateTime → первый/последний delivery_option). Альтернатива (паттерн PA-20651 `agg_order.sql`): роверные заказы = `grp_sandbox.dm_order_extended.delivery_option = 'ROVER'`; значения `delivery_option` (2026-07..10): `DELIVERY`, `PICK_UP`, `ROVER`, `DRONE`, `PICKUP_POINT` | этапы/тайминги |
| `picker_snp` | `grp_ods_es00_pickr` | Сборщики, состав заказа (`orderLines`, `pickingTasks`) | `order_id`, `worker_id`, JSONB `orderLines` |
| `task_audit_snp` | `grp_ods_pickr` | Аудит задач сборки | `task_id`, `picker_id`, time |
| `c_order_cancelled` | `grp_bv` | Источник отмен | `order_id`, `cancelled_reason_code` → `'client'`/`'service'` |
| `c_order` | `grp_bv` | Заказы курьеров | `order_id`, `employee_profile_courier_rk` |

**Подводные камни `order_snp` / структура `"orderItems"` (полный перебор ключей за 2026-11-02: 31,05 млн позиций):**
- `"orderItems"` — JSON-**массив**, каждый элемент = одна единица товара; поля camelCase → парсинг: `"orderItems"::jsonb` + `jsonb_array_elements(...)`. Названия полей в SQL обрамлять `"..."` (регистр!).
- Обязательные поля элемента (есть в 100% позиций): `"productId"` (string, UUID товара), `"orderItemId"` (string, UUID строки), `"orderItemType"` (string), `"productPrice"` (number, копейки: 18900 = 189.00), `"finalDiscountPrice"`, `"totalDiscountedPrice"`, `"productDiscountedPrice"` (number, итоговые цены), `"fitting"` (boolean), скидки: `"comboDiscount"`, `"loyaltyAmount"`, `"promoXYDiscount"`, `"showcaseDiscount"`, `"bonusMachineDiscount"`, `"specialDeliveryDiscount"`, `"subscriptionDiscountAmount"` (number).
- `"orderItemType"` (2026-11-02): `'PRODUCT'` (~97.5%), `'COMBO_SET'` (~2.5%), `'CUTLERY'` (редко, приборы). Фильтровать товарные позиции → `"orderItemType" = 'PRODUCT'`.
- `"salesModelType"` — опционально: `null` (~91%) / `'IMPULSE_1P'` (~9%, импульсные товары).
- Опциональные поля: `"showcaseDiscountType"` (`'CVM'`, ~38% позиций), `"identificationMark"` (код маркировки GS1, `\x1d`-разделитель, ~7%), `"identificationMarkRequestId"`, `"identificationMarkRequestTimestamp"` (epoch ms), `"identificationMarkLocalModuleDatabaseVersion"`, `"identificationMarkLocalModuleInstanceId"` (редко, <0.1%).
- `"comboSetsInfo"` (array) — есть ровно у `'COMBO_SET'`-позиций (счётчики сходятся 1:1); внутренняя структура элементов массива ⚠ пока не вскрыта (обрывы MCP-соединения).
- Поля `"quantity"` **нет** — количество = число элементов массива с тем же `"productId"` (+ в составе `"comboSetsInfo"` для комбо).
- `order_snp` хранит **несколько версий заказа** — строка на каждый статус (воронка CREATED→…→DELIVERED/CANCELLED; за 2026-11-02: ~841К заказов → ~4.8М строк). Перед агрегатами дедуп до последней версии: `distinct on (id) ... order by id, ce_version desc, ce_time desc` — иначе позиции/суммы задвоятся (PA-21516).
- Связка с витриной: `dm_order_extended.order_id` = `order_snp."id"`, 1:1 (проверено 2026-10-01: 863 236 = 863 236, PA-21516).
- ⚠ **Партиционирование `order_snp` — по `dataflow_dttm`** (время загрузки снапшота), НЕ по `"createdDateTime"` (проверено через `pg_partitions`/`pg_partition.paratts`, `pg_get_partkeydef` в этой версии GP нет; PA-21516, 2026-11-06). Range-партиции месячные с границей на 14-е число (`_prt_20260714` = `[2026-06-14, 2026-07-14)` — имя = END), в 2023 — дневные; всего ~202.
- ⚠ Фильтр по `"createdDateTime"` (даже константным диапазоном) партиции **не отсекает** — сканируется вся история (на периоде 4 мес таймаут). Отсечение даёт только фильтр по ключу: `dataflow_dttm >= timestamp 'период_с' and dataflow_dttm < timestamp 'период_до' + interval '7 days'` (запас на поздние CE-версии заказов конца периода; строки грузятся после создания заказа). EXPLAIN: 202 партиций → 5, cost скана 194M → 35M (PA-21516).
- Названий/категорий товаров в JSON нет — только `"productId"` → мапить на `grp_em.dim_sku.sku_uuid`.
- Данных о курьерах/сборщиках в JSON нет (только товарный состав заказа).
- Партиции: range по `dataflow_dttm` (см. блок подводных камней выше). Фильтр даты задавать константой; подзапрос `max("createdDateTime")` по всей таблице тяжёлый (таймауты). `"createdDateTime"::date`-фильтр и фильтр по самой `"createdDateTime"` партиции не режут — нужен `dataflow_dttm`.
- Свежесть снапшота с лагом: на утро 2026-11-03 доступны данные только до 2026-11-02 21:00.

**Отмены — источники:**
- `cancelled_by_user`, `refusedbycustomer` → `'client'`.
- всё остальное → `'service'`.
- Источник отмен «курьер» **не выделяется** (нет данных), `cancel_courier_cnt = 0`.

**Типовые фильтры заказов:** `log_strategy = 'DIRECT'`, `order_final_status is not null`, `delivery_final_status <> 'IN_PROGRESS'`.

---

## 5. Факты: смены курьеров и сборщиков

| Таблица | Схема | Назначение | Ключ |
|---------|-------|-----------|------|
| `fct_shift_assignment` | `grp_em` | План смен (SCD) | `store_rk`, `employee_profile_rk`, `valid_from/to_dttm` |
| `fct_shift` | `grp_em` | Факт смен | `store_rk`, `employee_profile_rk` |
| `worked_out_shift_snp` | `grp_ods_shftb` | Отработанные смены | `shift_id`, `worker_id`, `hours` |
| `shift_log_snp` | `grp_ods_shftb` | Логи смен | `shift_id`, `store_id`, `start_time`, `end_time` |
| `forecast__agg_picker_shift` | `grp_sandbox` | Агрегаты смен сборщиков | по часам |
| `shift_schedule_d` | `ssa_shifts_shifts` (ClickHouse SMKT) | Предпочтительный график смен сотрудника (запланированные интервалы): слоты `[time_slot_started_at, time_slot_ended_at)` в UTC (Zulu); CDC: `type` (created/deleted), `version` (PA-20729) | `user_id` (UUID); дедуп-ключ `(user_id, time_slot_started_at, time_slot_ended_at)` по max(version) |
| `shift_assignment_d` | `ssa_shifts_shifts` (ClickHouse SMKT) | Назначения на смены (SSA): `workspace_id` (ЦФЗ), `user_role` (deliveryman/picker), `status` (assigned/canceled), `workspace_type` (DARKSTORE) (PA-20729) | `assignment_id`, дедуп по max(version); ЦФЗ пользователя = workspace_id последнего `assigned` назначения по `updated_at` |

**Подводные камни `ssa_shifts_shifts` (PA-20729):**
- Под `_d` — `ReplicatedReplacingMergeTree` (`*_rr`) с мёржами не по расписанию: последняя версия собирается вручную — `argMax(поле, version)` + `group by <ключ>`; `FINAL` на Distributed не использовать.
- `shift_schedule_d.type = 'deleted'` отсекать **после** дедупа (HAVING), а не в WHERE.
- Аликас агрегата может перекрыть колонку в WHERE (`argMax(created_at, version) as created_at` → WHERE `created_at` резолвится в агрегат, ILLEGAL_AGGREGATION): выходной алиас называть иначе (`last_created_at`).
- План «как за день до смены» (PA-20729): последняя версия слота с `created_at < toStartOfDay(time_slot_started_at)` (граница суток, не окно 24ч); слот без версий до своего дня выпадает. Развёртка слота на целые часы в CH — `array join range(hours_cnt)`, hours_cnt = `dateDiff('hour', toStartOfHour(start), toStartOfHour(end)) + if(end != toStartOfHour(end), 1, 0)`.
- `user_id`: в `shift_schedule_d` — тип UUID, в `shift_assignment_d` — String → join через `toString(s.user_id) = a.user_id` (формат совпадает, проверено: ~61% пользователей графика имеют назначения).
- `user_id` — персональные данные: наружу не выводить; обезличивание — порядковый номер пользователя внутри ЦФЗ (`dense_rank() over (partition by workspace_id order by user_id)` — стабилен между запусками).
- В `shift_schedule_d` нет ЦФЗ и роли — фильтр сборщиков и ds брать из `shift_assignment_d` по `user_id`.

**Фильтры плана/факта смен:**
- План `fct_shift_assignment`: `shift_assignment_status_dk = 'assigned'`, `delete_flg = 0`, `shift_employee_role_dk = 'deliveryman'`, дата в окне SCD.
- Факт `fct_shift`: `delete_flg = 0`, `shift_employee_role_dk = 'deliveryman'` (курьеры) / `'picker'` (сборщики).
- Факт `fct_shift`: начало смены — `created_at_dttm`, конец — `end_at_dttm` (оба UTC).

**Часовой пояс:** смены хранятся в UTC → перевод в локальное время ЦФЗ:
```sql
+ interval '1 hour' * s.timezone_offset_hr
```
Развёртка смены по часам — `cross join lateral generate_series(date_trunc('hour', start_local), date_trunc('hour', end_local), interval '1 hour')`; минуты в часе = пересечение `[start_local, end_local]` с `[bucket, bucket+1h)`.

---

## 6. Прогноз

| Таблица | Схема | Назначение | Ключ |
|---------|-------|-----------|------|
| `staff_prediction_cdc` / `staff_prediction_snp` | `grp_ods_es00_preds` | Прогноз численности персонала по ролям (PICKER, DELIVERYMAN) по часам | `"darkstoreId"` + `"date"` (текст!) + `role` |
| `prediction_2_snp` / `prediction_2_cdc` | `grp_ods_es00_preds` | ES `shifts.predictions.event-streaming.prediction.2` (Смены): прогнозы по сотрудникам/заказам ЦФЗ, авто и ручные версии (PA-20397) | `"predictionId"`; гранула: `"workspaceId"` + `"predictionDate"` + `"predictionType"` |
| `request_snp` / `request_cdc` | `grp_ods_es00_shfeh` | ES Смен: логи заявок на доп часы сборки (`requestSlots`, `status`, `confirmationType`, `approverId`) (PA-20397) | `"requestId"` |
| `ttoktamysov_store_assigned_picker_assigns_contrib_cdc` | `grp_sandbox` | Назначенные часы сборщиков (`status='assigned'`, `userRole='picker'`) с разложением длительности по источникам вклада: `model_/coord_/transfer_/other_contributed_hours` (дока ANLT, pageId 4475250809) (PA-20397) | `assignment_id` (1 строка = 1 назначение) |
| `ttoktamysov_prediction_and_assigned_hours` | `grp_sandbox` | Почасовые курьеро-часы курьеров (DELIVERYMAN) в ЦФЗ: прогноз, назначения, ИД, факт. Только DARK_STORE в рабочие часы, данные с 2026-01-01, UTC → локальное TZ ЦФЗ (дока ANLT, pageId 4524583813) (PA-21712) | `ds_id` + `dt` (локальная дата прогноза) + `hour` (1–24, `24` = 00:00–01:00 той же даты) |
| `forecast__agg_picking_task` (+`_partition_pa17494`) | `grp_sandbox` | Агрегаты задач сборки по часам | по часам |
| `daablinov__order_prognosis__prediction_avail_agg` / `_by_hours` | `grp_sandbox` | Прогноз заказов по дням / часам | `warehouse_id`, `forecast_date/hour`, `value` |
| S3 `all_warehouses.parquet` / `warehouses.parquet` | minio | Прогноз заказов (файл) | — |
| Pickerton forecast API (внешний) | `pickerton-ai-agent-picker-forecast.app.kube02019.internal` | Прогноз сборки и курьеров — чистый прогноз модели, **без ручных корректировок** (в отличие от `staff_prediction`/`prediction_2`, где manual уже вмешан). Доступы у команды прогноза (T040) (PA-20397) | — |

**Подводные камни `staff_prediction_cdc` / `staff_prediction_snp`:**
- `"date"` — текст (yyyy-mm-dd), нужен `::date`.
- Прогноз — в JSON-колонках `"orderValue"`, `"autoStaffValue"`, `"submittedStaffValue"` с ключами `time1h`…`time24h`.
- Значения в JSON — **плоские числа** (`{"time1h": 0, "time2h": 3, ...}`), не объекты (проверено на данных, PA-20397).
- Час **24** → `date + 1 день`, `hour = 0`.
- Берётся последняя версия — `max(ce_version)` оконной функцией (snp: `distinct on` по `coalesce("updatedAt", "submittedAt"::timestamp, "createdAt")`).
- Сумма всех часовых ключей = человеко-часы роли за день.

**Подводные камни `ttoktamysov_prediction_and_assigned_hours` (PA-21712):**
- Строка есть только там, где есть прогноз; часы ограничены режимом работы ЦФЗ на дату (версия `dim_store` по `valid_from/to_dt`).
- Сборка: прогноз — `grp_ods_es00_preds.staff_prediction_snp` (DELIVERYMAN, последняя версия по `"submittedStaffPredictionVersion"`/`"updatedAt"`/`ce_version`); назначения — `grp_ods_es00_shfts.shift_assignment_snp` (`status='assigned'`); ИД — `grp_ods_es00_shfts.shift_schedule_snp` + `grp_ods_es00_wrkmg.user_cdc` (слот целиком внутри WORKING-периода той же привязки); факт — `grp_ods_es00_shfts.worked_out_shift_snp` (`itemType='assignment'`). Все интервалы в UTC → TZ ЦФЗ, часы = сумма длительности пересечений, а не кол-во курьеров.
- Метрика E042 «недостижимый дефицит» (укомплект): по ИД — `max(прогноз − ИД, 0)`; с учётом назначений — `max(прогноз − max(ИД, назначения), 0)`; знаменатель — `max(прогноз, 0)`; NULL → 0; числитель/знаменатель до 0,01 ч на стор×день, итог = сумма числ. / сумма знам. Объединённый скрипт за произвольный период: `E088 Технологии управления нагрузкой и эффективностью дарксторов/T042_Смены/PA-21712 Недостижимый дефицит курьеров/e042_недостижимый_дефицит_за_произвольный_период.sql`.

**Подводные камни `prediction_2_snp` (док ES: E042.A06.S11 shifts.predictions.event-streaming.prediction.2):**
- `"autoHourlyValue"` / `"manualHourlyValue"` — почасовые объекты `time0h`…`time32h`, каждый час — **вложенный объект** `{"value": N}` или null (в snp хранится как текст: `"time8h":"{\"value\":3}"`). Парсинг: `jsonb_each_text(...)` → `(value::json ->> 'value')::numeric`.
- `time24h`…`time32h` — часы **следующих суток** (перетекание смены ЦФЗ; time25h = 01:00 следующих суток). Сумма time0h..time32h = весь рабочий день ЦФЗ.
- `"predictionDate"` — дата **начала работы ЦФЗ** (не календарная дата закрытия).
- `predictionType` (значения из доки): `SMKT_DELIVERYMEN_STAFF_VALUE` — число курьеров в час; `SMKT_ORDERS_VALUE` — число заказов в час; `SMKT_PICKERS_STAFF_VALUE` — число сборщиков в час; `SMKT_PICKERS_HOUR_LOAD_VALUE` — **нагрузка на сборщика в час (НЕ человеко-часы!)**; `SMKT_ACTUAL_/ASSIGNED_DELIVERYMEN_STAFF_VALUE`, `SMKT_PICKERS_EXTRA_STAFF_VALUE`, `SMKT_KITCHEN_OPERATORS_STAFF_VALUE`, `SMKT_DARKSTORE_CRITS_/CRIT_PARAMS_DELIVERYMAN` (в data — вложенные словари, не числа); `SBL_SD_/SHUTTLE_STAFF_VALUE`, `SBL_SD_STOP_VALUE` — КС/шаттл, дневная гранула.
- Сумма `.value` по часам `SMKT_*_STAFF_VALUE` = человеко-часы роли; сверяется с `staff_prediction_snp` (auto↔auto, submitted↔manual) — на данных совпадает (PA-20397).
- Актуальная версия — `distinct on ("workspaceId", "predictionDate", "predictionType") ... order by dataflow_dttm desc`.
- `snp_delete_flg = 0` фильтровать обязательно (в обеих таблицах).
- `"manualHourlyValue"` для `SMKT_PICKERS_HOUR_LOAD_VALUE` не заполняется (null).
- Внутренняя структура часовых объектов подтверждена на данных (PA-20397, сен–окт 2026, сплит-дарксторы, актуальные версии): только два варианта — `{"value": N}` у всех типов и `{"critsByCourierCount": {...}}` у `SMKT_DARKSTORE_CRITS_DELIVERYMAN` (только auto); прочих ключей нет, скаляров нет.
- Заполненность manual по типам (сен–окт 2026): `SMKT_PICKERS_/DELIVERYMEN_/KITCHEN_OPERATORS_STAFF_VALUE` — manual = зеркало auto (счётчики часов совпадают 1:1); `SMKT_ASSIGNED_/ACTUAL_DELIVERYMEN_STAFF_VALUE` — manual частично; `SMKT_PICKERS_EXTRA_STAFF_VALUE` — **только manual** (auto нет); у `SMKT_ORDERS_VALUE`, `SMKT_PICKERS_HOUR_LOAD_VALUE`, `SMKT_DARKSTORE_CRITS_DELIVERYMAN` manual нет. `SMKT_DARKSTORE_CRIT_PARAMS_DELIVERYMAN` в данных за период не встречается.
- Витрина `ttoktamysov_store_assigned_picker_assigns_contrib_cdc`: ручные от авто часов отделяются **по эмиттеру версии назначения в CDC** (`shift_assignment_cdc.issuerId`). Классификация источника версии: `'model'` — жёстко зашитый UUID сервиса-модели `f90b7cd3-836c-4897-afdf-e4bfc16238db`; `'coord'` — `issuerId` есть в `grp_em.dim_employee_profile` (`employee_role_txt ilike '%coordinator%'`); `'transfer'` — `issuerId` = `transferId` из `grp_ods_shftb.shift_assignment_metadata_snp`; иначе `'other'`. Часы атрибутируются по дельтам интервалов между версиями (расширение `time_start`/`time_end` версии относит добавленный сегмент на источник версии; первый interval — целиком первой версии), сегменты клипаются к финальному интервалу. Сумма четырёх `*_contributed_hours` = `last_assignment_duration_hours`. Время — локальное TZ даркстора; назначения до 2026-01-01 отрезаны.

---

## 7. Заглушки (suspension) — спецтема для мониторинга простоев

| Таблица | Схема | Назначение | Ключ |
|---------|-------|-----------|------|
| `service_area_suspension_snp` | `grp_ods_es00_ndast` | Заглушки | `"storeId"`, `"serviceAreaId"` |
| `service_area_snp` | `grp_ods_es00_sam00` | Типы зон (tagId + categoryId) | `"id"` |

**Поля заглушки:**
- Время начала — `"createdAt"`, фактическое завершение — `coalesce("canceledAt","finishesAt")`; длительность = разница.
- `"sourceType"` (источник): `ALERT` (авто), `WDS` (ручная, Web Dark Store), `MOBILE_NDA` (ручная, «Управляй Про»).
- Причина — `"reason"::json->>'name'`.
- Фильтр: `coalesce("canceledAt","finishesAt") > "createdAt"` (длительность > 0).

**Тип зоны** = комбинация `tagId` + `categoryId` из `service_area_snp`:
- `vitrina_fast`, `vitrina_oblast`, `vitrina_oblast_poshire`, `vitrina_rover`.

**Часовой пояс:** `"createdAt"` в UTC → `+ interval '1 hour' * timezone_offset_hr`.

---

## 8. Прочее / специфичные источники

| Таблица / Источник | Где | Описание |
|--------------------|-----|----------|
| `t_posting_surplus`, `t_write_off`, `t_order_line`, `t_product` | MS SQL `DWH.[olap]` | Излишки, списания, GMV, товары (пересчёты) |
| `v_new_warehouse`, `v_warehouses` | MS SQL | Справочники складов (DARK_STORE / OPERATING) |
| `[ext].[ЦФЗ_Атрибуты]`, `[ext].[ЦФЗ_Территории]` | MS SQL | Атрибуты/территории ЦФЗ для сплитования |
| `compensation_snp`, `compensation_type_snp` | `grp_ods_es00_operc` | Компенсации / жалобы ФРОВ. **Отличие:** компенсация — любой факт возмещения (товар/деньги, разбор до `"compensatedItems"`); жалоба — компенсация с флагом отсутствия фрода `"compensationFraudCheck"` ~ `NOT_FRAUD` (доверенные жалобы), считается через общую поставку компенсаций. Жалоба привязана к заказу через `"orderId"`. Ключевые поля (PA-20216): `"createdAt"` (время жалобы), `"compensationFraudCheck"` (фильтр NOT_FRAUD), `"complimentCompensationReasonLinks"` (json-массив причин), `status` (`fulfilled`). Скрипты: PA-16891 (Компенсации ФРОВ), PA-20216 (жалобы/доли), PA-18509/PA-17995/PA-18720 (ФРОВ-аналитика) | `"orderId"`, `compensation_reason_snp.id` |
| `compensation_reason_snp` | `grp_ods_es00_operc` | Справочник причин компенсаций/жалоб (join по `id` → `name`) | `id`, `name` |
| `audit_checklist_*`, `images` | Patroni `store_track` | DSA чек-листы Сборщика |
| `loader.robocheck_parameter`, `loader.cybercheck_parameter`, `loader.sam_darkstores`, `loader.order_fulfillment`, `loader.cybercheck_simulation_order_event` | loader | Параметры алгоритмов, справочники |
| `ssa_eds.events_app_d` | SSA | События мобильного приложения (PA-20651). Ключи/атрибуция: `dictGet('dict.dpb_showcase', 'store_id', user_properties_showcase_id)` → получение `ds_id` из showcase; `topK(client_id)[1]` → атрибуция клиента по сессии. Ровер-события: `Cart - Cart - Successful Order` (+ `user_properties_delivery_type` = `with_rover`/`with_courier` → `paid_w_rover`/`paid_w_courier`), `Cart - Cart - View` (+ `toggle_delivery_type != ''` → `seen_cart_w_rover_toggle`), `Cart - RoverToggle - Click` → `rovertoggle_click`, `Cart - RoverToggle - Error View` (+ `error_code` = `rover_undeliver`/`rover_busy` → `seen_rover_undeliver`/`seen_rover_busy`), `Cart - Cart - Rover Onboarding View` → `seen_rover_onbording`. Фильтр `type not ilike '%push%'`. `device_time` — unix epoch, требует приведения (toDateTime) к читаемому виду | `part_dt`, `user_properties_showcase_id`, `client_id`, `session_id` |
| `order_economy` | `grp_ssa_finance` | Экономика заказа по строкам PL (PA-20868): CP1/CP2. Данные на уровне позиции заказа (`store_id` + `pl_item_*_txt`), не на уровне заказа. Ключи CP1/CP2 — наборы `pl_item_2_txt` (+ `pl_item_1_txt in ('Операционные расходы ЦФЗ')`), сумма по `pl_by_order_amt` | `store_id`, `created_dttm`, `pl_item_1_txt`, `pl_item_2_txt`, `pl_item_3_txt`, `pl_by_order_amt`, `city_dk` |
| GraphHopper (внешний) | `E041/.../Расчетное расстояние из Graphhopper` | Расчёт расстояний/времени; входные координаты из `graphhopper_input.sql`, API через `script.py` (Python ≥3.13) |

---

## 9. Общие правила и подводные камни

### Регистр значений (проверять через `SELECT DISTINCT`, не экстраполировать)
| Поле | Значения | Регистр |
|------|----------|---------|
| `log_strategy` | `'DIRECT'` | upper |
| `cancelled_reason_code` | `'cancelled_by_user'`, `'refusedbycustomer'` | lower |
| `shift_employee_role_dk` (fct_shift) | `'deliveryman'`, `'picker'`; также встречаются `'goods_manager'`, `'darkstore_admin'`, `'kitchen_operator'`, `'senior_kitchen_operator'`, `'counter'`, `'revisor_counter'` (проверено по данным за 2026-09+, PA-20729) | lower |
| `shift_assignment_status_dk` | `'assigned'` | lower |
| `"sourceType"` (заглушки) | `'WDS'`, `'MOBILE_NDA'`, `'ALERT'` | upper/mixed |
| `user_role` (ssa_shifts_shifts) | `'deliveryman'`, `'picker'` | lower |
| `status` (ssa_shifts_shifts.shift_assignment_d) | `'assigned'`, `'canceled'` | lower |
| `type` (ssa_shifts_shifts.shift_schedule_d) | `'created'`, `'deleted'` ('updated' есть в схеме, в данных не встречался) | lower |
| `workspace_type` (ssa_shifts_shifts) | `'DARKSTORE'` | upper |

### Часовые пояса
| Данные | В источнике | Преобразование |
|--------|-------------|----------------|
| Заказы `created_lcl_dttm` | ⚠ не использовать (битый/нулевой) | переводить `created_utc_dttm` в локальное через джойн TZ: `date_trunc('hour', created_utc_dttm) + interval '1 hour' * ds.timezone_offset_hr` (join `mincheck_overview__darkstore` по `ds_id`) |
| Смены (план/факт) | UTC | `+ interval '1 hour' * timezone_offset_hr` |
| `ssa_shifts_shifts` (график/назначения смен, DateTime64) | UTC (Zulu) | при необходимости переводить в локальное TZ ЦФЗ (PA-20729) |
| Заглушки `"createdAt"` | UTC | `+ interval '1 hour' * timezone_offset_hr` |
| Прогноз `staff_prediction_cdc` | абстрактное (не привязано к TZ) | без изменений |
| `ssa_eds.events_app_d.device_time` | unix epoch | `toDateTime(device_time)` для читаемого вида; агрегация `dthr` через `toTimeZone(toStartOfHour(session_start_dttm), 'Europe/Moscow')` (PA-20651) |

### Частые ошибки
1. Прямой join EM-таблиц (`store_rk`) с ODS/sandbox (`store_id/ds_id`) без моста → **перед join мапить** через `mincheck_overview__darkstore.store_rk` или `dim_store`.
2. Забыть `lower()` на `store_snp.store_id` (UPPER-регистр) → не сджойнится с `ds_id`.
3. `staff_prediction_cdc."date"` как текст без `::date`.
4. Не проверить регистр строковой константы (upper/lower) — сверять по `SELECT DISTINCT`.
5. Между источником и витриной расходится часовой пояс — всегда явно указывать `timezone_offset_hr`.
6. GMV из `dm_order_extended` — метрика на уровне заказа (`order_id`). При джойне к строкам PL-уровня (`order_economy`) нельзя суммировать напрямую → задвоится по числу позиций. Агрегировать `sum(gmv)` в отдельном CTE по `(ds_id, created_lcl_dttm::date)` и джойнить к агрегату по `(store_id, order_date)` (PA-20868).
7. Схема `grp_ods_orhst` **устарела** (заменена на `grp_ods_es00_orhst`) — не использовать для новых задач; замена строк заказа найдена (`order_snp."orderItems"`), замена `order_history_snp` пока не найдена, см. пометки ⚠ в разделе 4.
8. `order_snp` — несколько версий на заказ (строка на статус): без дедупа `max(ce_version)` позиции и суммы задваиваются. Период фильтровать по `"createdDateTime"` + обязательно по `dataflow_dttm` (ключ партиционирования, запас +7 дней) — иначе скан всей истории и таймаут.
9. `dm_order_extended.ds_type` ненадёжен: NULL у ~39 млн строк по всей таблице, в т.ч. у реальных ЦФЗ (проверено PA-21675, 2026-11-09). Тип объекта брать из `mincheck_overview__darkstore."type"` по `ds_id` (`DARK_STORE`/`DARK_STORE_MAX` = даркстор, `KITCHEN` = кухня); `ds_id` в справочнике уникален — join без задвоений. `delivery_option` для «только доставка»: `in ('DELIVERY','ROVER','DRONE')`, самовывоз = `PICK_UP`/`PICKUP_POINT`.

---

## 10. Права на витрину (owner / grp_sandbox_w)

После создания или перезаливки любой витрины в `grp_sandbox` обязательно передать владение сервисной роли:

```sql
alter table grp_sandbox.my_mart owner to grp_sandbox_w;
```

**Почему это важно:**
- Подключение Airflow (DAG-и, которые читают/обновляют витрину) выполняется от сервисного аккаунта с ролью `grp_sandbox_w`. Если владелец таблицы — личный логин аналитика (`tkaftaranov_*`, `sbolshakov_*`), таблица не будет доступна Airflow: DAG упадёт с `permission denied`/отсутствием прав при чтении или записи.
- `owner to grp_sandbox_w` делается **сразу после** `create table`/`insert`/`delete`, до вызова `set_expire_date`. Только владелец может менять права и ставить срок хранения.
- Роль `grp_sandbox_w` сама делает таблицу доступной для чтения остальным (`GRANT SELECT TO PUBLIC`/членам роли) — это типовой шаблон во всех DAG-скриптах и DDL витрин (см. `PA-20651`, `PA-19784`).

**Типовой порядок при обновлении витрины (сырой SQL):**
```sql
alter table grp_sandbox.my_mart owner to grp_sandbox_w;
select public.set_expire_date('grp_sandbox.my_mart', (now() + interval '28 days')::date);
```

---

## 11. Срок хранения витрины (expire_date)

Витрины в `grp_sandbox` имеют срок хранения (горизонт, до которого таблицу не вычищают). Он задаётся функцией `public.set_expire_date(schema.table, date, comment?)`. При создании/обновлении витрины срок нужно обязательно выставлять, иначе таблицу может удалить автоочистка.

**Правила:**
- Витрина регулярно перезаливается DAG-ем → продлевать горизонт на **`(now() + interval '28 days')::date`** на каждом прогоне (rolling horizon), сразу после `alter table ... owner to grp_sandbox_w`.
- Витрина создаётся разово/вручную или требует длительного хранения → фиксированная дата + комментарий с задачей/аппрувером. Дата ставится больше текущей (напр. `2028-12-31`), а не `5999-01-01`; `5999-01-01` — только для «вечных» таблиц, создаваемых через `smkt_db*` с аппрувером.
- `set_expire_date` вызывается через `select public.set_expire_date(...)` и к статусу MVCC не привязан — выполняется как обычный DDL.

**Пример продления (сырой SQL):**
```sql
alter table grp_sandbox.my_mart owner to grp_sandbox_w;
select public.set_expire_date('grp_sandbox.my_mart', (now() + interval '28 days')::date);
```

---

## 12. Обязательные проверки перед выводом/передачей данных

- [ ] Определены схема и слой каждой таблицы; ключи ЦФЗ смаплены корректно.
- [ ] Часовой пояс каждого источника учтён и указан в методе.
- [ ] Строковые константы проверены на регистр.
- [ ] Для каждои метрики явны: числитель/знаменатель, популяция, фильтры, grain по времени.
- [ ] Отмены «курьер» (`cancel_courier_cnt`) — если источник отсутствует, явно отметить `0`/NULL.
- [ ] Если источник/поле не найден — явно пометить как `⚠ не найдено` и предложить прокси, а не молча подставлять.
- [ ] При переносе в отчёт/витрину — `coalesce` для отсутствующих значений, защита от задвоения (`least`/`distinct`).