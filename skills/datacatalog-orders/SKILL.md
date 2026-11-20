---
name: orders
description: Справочник источников данных по заказам, доставке и сборке: dm_order_extended, order_snp (JSON "orderItems", дедуп версий, партиционирование по dataflow_dttm), order_line_snp (устарел), delivery_snp, fulfillment_snp, picker_snp, отмены c_order_cancelled. Использовать при расчётах заказов, таймингов этапов, SLA, GMV, отмен и роверных заказов.
---

# Факты: заказы, доставка, сборка

| Таблица | Схема | Назначение | Ключевые поля |
|---------|-------|-----------|---------------|
| `dm_order_extended` | `grp_sandbox` | Основная по заказам: время, статус, опоздания, GMV, клиент | `order_id`, `ds_id`, `created_utc_dttm`, `pick*/delivery_*_dttm`, `order_final_status`, `delivery_final_status`, `late_mnt`, `is_late`, `is_crit_late`, `gmv`, `log_strategy`, тайминги этапов: `waiting_for_picking_mnt`, `picking_mnt`, `waiting_for_delivery_mnt`, `delivery_mnt`, `c2d_mnt` (click-to-delivery), SLA: `client_sla`, `geo_sla`, `sla_mnt`, клиент: `customer_rk` / `customer_id` / `customer_uuid` |
| `order_snp` | `grp_ods_es00_orhst` | Снимок заказов (альтернатива). Состав заказа — в JSON-колонке `"orderItems"` (массив позиций, см. подводные камни ниже) | `"id"`, времена, `"showCaseTypeName"`, `"sourceName"`, `"orderItems"` |
| `order_history_snp` | ⚠ `grp_ods_orhst` — **схема устарела** | История статусов заказа. Всю схему `grp_ods_orhst` заменила `grp_ods_es00_orhst`: искать замену там (пока в ней только `order_snp`/`order_cdc`) и обновить эту строку гайда при нахождении источника | `order_id`, `store_id`, `order_status_id`, timestamps |
| `order_line_snp` | ⚠ `grp_ods_orhst` — **схема устарела, заменена** | Строки заказа: товары, цены, кол-во. Схему `grp_ods_orhst` заменила `grp_ods_es00_orhst`; состав заказа теперь хранится в JSON `"orderItems"` у `grp_ods_es00_orhst.order_snp` (структура — см. подводные камни `order_snp` ниже). Данные в старой таблице не обновляются (на 2026-11-02 свежесть только до 2026-10-25) — не использовать | `product_id` → `"productId"`, `price` → `"productPrice"`, `quantity` → повторение элементов с тем же `"productId"` |
| `delivery_snp` | `grp_ods_es00_dlvry` | Доставки | `delivery_id`, `"courierId"`, timestamps |
| `fulfillment_snp` (`fulfillment_cdc`) | `grp_ods_es00_flfmt` | Время fulfilment. Доставка ровером: `"orderProcessingFlag"` ilike `'%ROVER_DELIVERY%'` → `'ROVER'` иначе `'COURIER'`. Фолбэк ровер→курьер: поля `"orderId"`, `"createdDateTime"`, `"storeId"`, `"changeDateTime"` (сорт. по changeDateTime → первый/последний delivery_option). Альтернатива: роверные заказы = `grp_sandbox.dm_order_extended.delivery_option = 'ROVER'`; значения `delivery_option` (2026-07..10): `DELIVERY`, `PICK_UP`, `ROVER`, `DRONE`, `PICKUP_POINT` | этапы/тайминги |
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
- `"comboSetsInfo"` (array) — есть ровно у `'COMBO_SET'`-позиций (счётчики сходятся 1:1); внутренняя структура элементов массива ⚠ пока не вскрыта.
- Поля `"quantity"` **нет** — количество = число элементов массива с тем же `"productId"` (+ в составе `"comboSetsInfo"` для комбо).
- `order_snp` хранит **несколько версий заказа** — строка на каждый статус (воронка CREATED→…→DELIVERED/CANCELLED; за 2026-11-02: ~841К заказов → ~4.8М строк). Перед агрегатами дедуп до последней версии: `distinct on (id) ... order by id, ce_version desc, ce_time desc` — иначе позиции/суммы задвоятся.
- Связка с витриной: `dm_order_extended.order_id` = `order_snp."id"`, 1:1 (проверено 2026-10-01: 863 236 = 863 236).
- ⚠ **Партиционирование `order_snp` — по `dataflow_dttm`** (время загрузки снапшота), НЕ по `"createdDateTime"` (проверено через `pg_partitions`/`pg_partition.paratts`, `pg_get_partkeydef` в этой версии GP нет; 2026-11-06). Range-партиции месячные с границей на 14-е число (`_prt_20260714` = `[2026-06-14, 2026-07-14)` — имя = END), в 2023 — дневные; всего ~202.
- ⚠ Фильтр по `"createdDateTime"` (даже константным диапазоном) партиции **не отсекает** — сканируется вся история (на периоде 4 мес таймаут). Отсечение даёт только фильтр по ключу: `dataflow_dttm >= timestamp 'период_с' and dataflow_dttm < timestamp 'период_до' + interval '7 days'` (запас на поздние CE-версии заказов конца периода; строки грузятся после создания заказа). EXPLAIN: 202 партиций → 5, cost скана 194M → 35M.
- Названий/категорий товаров в JSON нет — только `"productId"` → мапить на `grp_em.dim_sku.sku_uuid`.
- Данных о курьерах/сборщиках в JSON нет (только товарный состав заказа).
- Партиции: range по `dataflow_dttm` (см. блок подводных камней выше). Фильтр даты задавать константой; подзапрос `max("createdDateTime")` по всей таблице тяжёлый (таймауты). `"createdDateTime"::date`-фильтр и фильтр по самой `"createdDateTime"` партиции не режут — нужен `dataflow_dttm`.
- Свежесть снапшота с лагом: на утро 2026-11-03 доступны данные только до 2026-11-02 21:00.

**Отмены — источники:**
- `cancelled_by_user`, `refusedbycustomer` → `'client'`.
- всё остальное → `'service'`.
- Источник отмен «курьер» **не выделяется** (нет данных), `cancel_courier_cnt = 0`.

**Типовые фильтры заказов:** `log_strategy = 'DIRECT'`, `order_final_status is not null`, `delivery_final_status <> 'IN_PROGRESS'`.
