---
name: darkstores
description: Справочник источников данных по ЦФЗ (дарксторам): карта слоёв Greenplum (grp_sandbox, grp_ods_*, grp_em, grp_bv), ключи ds_id/store_id/store_rk и правила join между слоями, справочники mincheck_overview__darkstore, store_snp, dim_store, dim_sku и атрибуты ЦФЗ. Использовать при выборе таблиц, маппинге ключей магазинов, фильтрах по дарксторам и SCD-справочникам.
---

# Карта слоёв и ключи ЦФЗ (Dark Store)

Слои, у каждого свой тип ключа магазина. **Ключи между слоями нельзя смешивать напрямую.**

| Слой | Схемы | Примеры таблиц | Ключ ЦФЗ |
|------|-------|----------------|----------|
| Sandbox / витрины | `grp_sandbox` | `dm_order_extended`, `mincheck_overview__darkstore`, все `*_agg_*`, `tkaftaranov_*` | `ds_id` / `darkstore_id` (lowercase UUID) |
| Витрины роверов | `grp_sandbox` | `rover_dashboard__agg_session` (почас. конверсии/сессии), `rover_dashboard__dau_agg` (DAU даркстора/день), `rover_dashboard__completion_time` (ср. время этапов роверных заказов) | `ds_id`, `part_dt` |
| ODS (снапшоты) | `grp_ods_*` | `store_snp`, `service_area_suspension_snp`, `order_snp`, `picker_snp`, `delivery_snp`, `worked_out_shift_snp` | `store_id` (UUID, **VARCHAR**; у `store_snp` — **UPPER** регистр) |
| EM (факты) | `grp_em` | `fct_shift`, `fct_shift_assignment`, `dim_store`, `dim_sku`, `fct_order` | `store_rk` (bigint) |
| Бизнес-слой BV | `grp_bv` | `c_order_cancelled`, `c_order` | `order_id` / `employee_profile_courier_rk` |

**Правила join ключей ЦФЗ:**
- ODS `store_snp.store_id` — UUID в **верхнем** регистре → всегда `lower(st.store_id) = ds.ds_id`.
- Sandbox `mincheck_overview__darkstore.ds_id` — UUID в нижнем регистре.
- EM-таблицы joinятся только по `store_rk` через `mincheck_overview__darkstore.store_rk` (мост Sandbox↔EM).
- SCD-справочник `grp_em.dim_store` (store_rk → store_id) — join **всегда с датой**: `date between valid_from_dt and valid_to_dt`.

---

# Основные справочники (dim / магазины)

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
- Фильтр «даркстор работал весь период»: `status = 'OPERATING'` и `"type" in ('DARK_STORE','DARK_STORE_MAX')` и `started_to_operate_dt <= start_dt` и `coalesce(finished_to_operate_dt, now()) >= end_dt`.
