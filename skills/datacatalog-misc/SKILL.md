---
name: misc
description: Прочие источники данных: MS SQL DWH.[olap] (излишки, списания, склады, атрибуты ЦФЗ), компенсации и жалобы ФРОВ (compensation_snp), DSA чек-листы (Patroni store_track), loader, события приложения ssa_eds.events_app_d, order_economy, GraphHopper. Использовать для смежных задач вне основных слоёв Greenplum.
---

# Прочее / специфичные источники

| Таблица / Источник | Где | Описание |
|--------------------|-----|----------|
| `t_posting_surplus`, `t_write_off`, `t_order_line`, `t_product` | MS SQL `DWH.[olap]` | Излишки, списания, GMV, товары (пересчёты) |
| `v_new_warehouse`, `v_warehouses` | MS SQL | Справочники складов (DARK_STORE / OPERATING) |
| `[ext].[ЦФЗ_Атрибуты]`, `[ext].[ЦФЗ_Территории]` | MS SQL | Атрибуты/территории ЦФЗ для сплитования |
| `compensation_snp`, `compensation_type_snp` | `grp_ods_es00_operc` | Компенсации / жалобы ФРОВ. **Отличие:** компенсация — любой факт возмещения (товар/деньги, разбор до `"compensatedItems"`); жалоба — компенсация с флагом отсутствия фрода `"compensationFraudCheck"` ~ `NOT_FRAUD` (доверенные жалобы), считается через общую поставку компенсаций. Жалоба привязана к заказу через `"orderId"`. Ключевые поля: `"createdAt"` (время жалобы), `"compensationFraudCheck"` (фильтр NOT_FRAUD), `"complimentCompensationReasonLinks"` (json-массив причин), `status` (`fulfilled`) | `"orderId"`, `compensation_reason_snp.id` |
| `compensation_reason_snp` | `grp_ods_es00_operc` | Справочник причин компенсаций/жалоб (join по `id` → `name`) | `id`, `name` |
| `audit_checklist_*`, `images` | Patroni `store_track` | DSA чек-листы Сборщика |
| `loader.robocheck_parameter`, `loader.cybercheck_parameter`, `loader.sam_darkstores`, `loader.order_fulfillment`, `loader.cybercheck_simulation_order_event` | loader | Параметры алгоритмов, справочники |
| `ssa_eds.events_app_d` | SSA (ClickHouse) | События мобильного приложения. Ключи/атрибуция: `dictGet('dict.dpb_showcase', 'store_id', user_properties_showcase_id)` → получение `ds_id` из showcase; `topK(client_id)[1]` → атрибуция клиента по сессии. Ровер-события: `Cart - Cart - Successful Order` (+ `user_properties_delivery_type` = `with_rover`/`with_courier` → `paid_w_rover`/`paid_w_courier`), `Cart - Cart - View` (+ `toggle_delivery_type != ''` → `seen_cart_w_rover_toggle`), `Cart - RoverToggle - Click` → `rovertoggle_click`, `Cart - RoverToggle - Error View` (+ `error_code` = `rover_undeliver`/`rover_busy` → `seen_rover_undeliver`/`seen_rover_busy`), `Cart - Cart - Rover Onboarding View` → `seen_rover_onbording`. Фильтр `type not ilike '%push%'`. `device_time` — unix epoch, требует приведения (toDateTime) к читаемому виду | `part_dt`, `user_properties_showcase_id`, `client_id`, `session_id` |
| `order_economy` | `grp_ssa_finance` | Экономика заказа по строкам PL: CP1/CP2. Данные на уровне позиции заказа (`store_id` + `pl_item_*_txt`), не на уровне заказа. Ключи CP1/CP2 — наборы `pl_item_2_txt` (+ `pl_item_1_txt in ('Операционные расходы ЦФЗ')`), сумма по `pl_by_order_amt` | `store_id`, `created_dttm`, `pl_item_1_txt`, `pl_item_2_txt`, `pl_item_3_txt`, `pl_by_order_amt`, `city_dk` |
| GraphHopper (внешний) | внешний API | Расчёт расстояний/времени по координатам |
