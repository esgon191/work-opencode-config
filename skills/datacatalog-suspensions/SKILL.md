---
name: suspensions
description: Справочник источников данных по заглушкам (suspension) ЦФЗ: service_area_suspension_snp, service_area_snp, поля createdAt/canceledAt/finishesAt, sourceType, типы зон. Использовать при расчётах простоев и мониторинге глушения витрин.
---

# Заглушки (suspension) — спецтема для мониторинга простоев

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
