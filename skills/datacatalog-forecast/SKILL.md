---
name: forecast
description: Справочник источников данных по прогнозам: staff_prediction_snp/cdc, prediction_2_snp/cdc (почасовые JSON time0h..time32h, predictionType), request_snp, витрины ttoktamysov (назначенные/курьеро-часы), daablinov, S3 parquet, Pickerton API. Использовать при работе с прогнозами численности персонала, заказов и курьеро-часов.
---

# Прогноз

| Таблица | Схема | Назначение | Ключ |
|---------|-------|-----------|------|
| `staff_prediction_cdc` / `staff_prediction_snp` | `grp_ods_es00_preds` | Прогноз численности персонала по ролям (PICKER, DELIVERYMAN) по часам | `"darkstoreId"` + `"date"` (текст!) + `role` |
| `prediction_2_snp` / `prediction_2_cdc` | `grp_ods_es00_preds` | ES `shifts.predictions.event-streaming.prediction.2` (Смены): прогнозы по сотрудникам/заказам ЦФЗ, авто и ручные версии | `"predictionId"`; гранула: `"workspaceId"` + `"predictionDate"` + `"predictionType"` |
| `request_snp` / `request_cdc` | `grp_ods_es00_shfeh` | ES Смен: логи заявок на доп часы сборки (`requestSlots`, `status`, `confirmationType`, `approverId`) | `"requestId"` |
| `ttoktamysov_store_assigned_picker_assigns_contrib_cdc` | `grp_sandbox` | Назначенные часы сборщиков (`status='assigned'`, `userRole='picker'`) с разложением длительности по источникам вклада: `model_/coord_/transfer_/other_contributed_hours` (дока ANLT, pageId 4475250809) | `assignment_id` (1 строка = 1 назначение) |
| `ttoktamysov_prediction_and_assigned_hours` | `grp_sandbox` | Почасовые курьеро-часы курьеров (DELIVERYMAN) в ЦФЗ: прогноз, назначения, ИД, факт. Только DARK_STORE в рабочие часы, данные с 2026-01-01, UTC → локальное TZ ЦФЗ (дока ANLT, pageId 4524583813) | `ds_id` + `dt` (локальная дата прогноза) + `hour` (1–24, `24` = 00:00–01:00 той же даты) |
| `forecast__agg_picking_task` (+`_partition_pa17494`) | `grp_sandbox` | Агрегаты задач сборки по часам | по часам |
| `daablinov__order_prognosis__prediction_avail_agg` / `_by_hours` | `grp_sandbox` | Прогноз заказов по дням / часам | `warehouse_id`, `forecast_date/hour`, `value` |
| S3 `all_warehouses.parquet` / `warehouses.parquet` | minio | Прогноз заказов (файл) | — |
| Pickerton forecast API (внешний) | `pickerton-ai-agent-picker-forecast.app.kube02019.internal` | Прогноз сборки и курьеров — чистый прогноз модели, **без ручных корректировок** (в отличие от `staff_prediction`/`prediction_2`, где manual уже вмешан). Доступы у команды прогноза (T040) | — |

**Подводные камни `staff_prediction_cdc` / `staff_prediction_snp`:**
- `"date"` — текст (yyyy-mm-dd), нужен `::date`.
- Прогноз — в JSON-колонках `"orderValue"`, `"autoStaffValue"`, `"submittedStaffValue"` с ключами `time1h`…`time24h`.
- Значения в JSON — **плоские числа** (`{"time1h": 0, "time2h": 3, ...}`), не объекты (проверено на данных).
- Час **24** → `date + 1 день`, `hour = 0`.
- Берётся последняя версия — `max(ce_version)` оконной функцией (snp: `distinct on` по `coalesce("updatedAt", "submittedAt"::timestamp, "createdAt")`).
- Сумма всех часовых ключей = человеко-часы роли за день.

**Подводные камни `ttoktamysov_prediction_and_assigned_hours`:**
- Строка есть только там, где есть прогноз; часы ограничены режимом работы ЦФЗ на дату (версия `dim_store` по `valid_from/to_dt`).
- Сборка: прогноз — `grp_ods_es00_preds.staff_prediction_snp` (DELIVERYMAN, последняя версия по `"submittedStaffPredictionVersion"`/`"updatedAt"`/`ce_version`); назначения — `grp_ods_es00_shfts.shift_assignment_snp` (`status='assigned'`); ИД — `grp_ods_es00_shfts.shift_schedule_snp` + `grp_ods_es00_wrkmg.user_cdc` (слот целиком внутри WORKING-периода той же привязки); факт — `grp_ods_es00_shfts.worked_out_shift_snp` (`itemType='assignment'`). Все интервалы в UTC → TZ ЦФЗ, часы = сумма длительности пересечений, а не кол-во курьеров.
- Метрика E042 «недостижимый дефицит» (укомплект): по ИД — `max(прогноз − ИД, 0)`; с учётом назначений — `max(прогноз − max(ИД, назначения), 0)`; знаменатель — `max(прогноз, 0)`; NULL → 0; числитель/знаменатель до 0,01 ч на стор×день, итог = сумма числ. / сумма знам.

**Подводные камни `prediction_2_snp` (док ES: E042.A06.S11 shifts.predictions.event-streaming.prediction.2):**
- `"autoHourlyValue"` / `"manualHourlyValue"` — почасовые объекты `time0h`…`time32h`, каждый час — **вложенный объект** `{"value": N}` или null (в snp хранится как текст: `"time8h":"{\"value\":3}"`). Парсинг: `jsonb_each_text(...)` → `(value::json ->> 'value')::numeric`.
- `time24h`…`time32h` — часы **следующих суток** (перетекание смены ЦФЗ; time25h = 01:00 следующих суток). Сумма time0h..time32h = весь рабочий день ЦФЗ.
- `"predictionDate"` — дата **начала работы ЦФЗ** (не календарная дата закрытия).
- `predictionType` (значения из доки): `SMKT_DELIVERYMEN_STAFF_VALUE` — число курьеров в час; `SMKT_ORDERS_VALUE` — число заказов в час; `SMKT_PICKERS_STAFF_VALUE` — число сборщиков в час; `SMKT_PICKERS_HOUR_LOAD_VALUE` — **нагрузка на сборщика в час (НЕ человеко-часы!)**; `SMKT_ACTUAL_/ASSIGNED_DELIVERYMEN_STAFF_VALUE`, `SMKT_PICKERS_EXTRA_STAFF_VALUE`, `SMKT_KITCHEN_OPERATORS_STAFF_VALUE`, `SMKT_DARKSTORE_CRITS_/CRIT_PARAMS_DELIVERYMAN` (в data — вложенные словари, не числа); `SBL_SD_/SHUTTLE_STAFF_VALUE`, `SBL_SD_STOP_VALUE` — КС/шаттл, дневная гранула.
- Сумма `.value` по часам `SMKT_*_STAFF_VALUE` = человеко-часы роли; сверяется с `staff_prediction_snp` (auto↔auto, submitted↔manual) — на данных совпадает.
- Актуальная версия — `distinct on ("workspaceId", "predictionDate", "predictionType") ... order by dataflow_dttm desc`.
- `snp_delete_flg = 0` фильтровать обязательно (в обеих таблицах).
- `"manualHourlyValue"` для `SMKT_PICKERS_HOUR_LOAD_VALUE` не заполняется (null).
- Внутренняя структура часовых объектов подтверждена на данных (сен–окт 2026, сплит-дарксторы, актуальные версии): только два варианта — `{"value": N}` у всех типов и `{"critsByCourierCount": {...}}` у `SMKT_DARKSTORE_CRITS_DELIVERYMAN` (только auto); прочих ключей нет, скаляров нет.
- Заполненность manual по типам (сен–окт 2026): `SMKT_PICKERS_/DELIVERYMEN_/KITCHEN_OPERATORS_STAFF_VALUE` — manual = зеркало auto (счётчики часов совпадают 1:1); `SMKT_ASSIGNED_/ACTUAL_DELIVERYMEN_STAFF_VALUE` — manual частично; `SMKT_PICKERS_EXTRA_STAFF_VALUE` — **только manual** (auto нет); у `SMKT_ORDERS_VALUE`, `SMKT_PICKERS_HOUR_LOAD_VALUE`, `SMKT_DARKSTORE_CRITS_DELIVERYMAN` manual нет. `SMKT_DARKSTORE_CRIT_PARAMS_DELIVERYMAN` в данных за период не встречается.
- Витрина `ttoktamysov_store_assigned_picker_assigns_contrib_cdc`: ручные от авто часов отделяются **по эмиттеру версии назначения в CDC** (`shift_assignment_cdc.issuerId`). Классификация источника версии: `'model'` — жёстко зашитый UUID сервиса-модели `f90b7cd3-836c-4897-afdf-e4bfc16238db`; `'coord'` — `issuerId` есть в `grp_em.dim_employee_profile` (`employee_role_txt ilike '%coordinator%'`); `'transfer'` — `issuerId` = `transferId` из `grp_ods_shftb.shift_assignment_metadata_snp`; иначе `'other'`. Часы атрибутируются по дельтам интервалов между версиями (расширение `time_start`/`time_end` версии относит добавленный сегмент на источник версии; первый interval — целиком первой версии), сегменты клипаются к финальному интервалу. Сумма четырёх `*_contributed_hours` = `last_assignment_duration_hours`. Время — локальное TZ даркстора; назначения до 2026-01-01 отрезаны.
