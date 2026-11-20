---
name: couriers
description: Справочник источников данных по сменам курьеров и сборщиков: fct_shift, fct_shift_assignment, worked_out_shift_snp, ClickHouse ssa_shifts_shifts (shift_schedule_d, shift_assignment_d) с дедупом по argMax/version. Использовать при расчётах плана/факта смен, почасовых развёрток и графиков сотрудников.
---

# Факты: смены курьеров и сборщиков

| Таблица | Схема | Назначение | Ключ |
|---------|-------|-----------|------|
| `fct_shift_assignment` | `grp_em` | План смен (SCD) | `store_rk`, `employee_profile_rk`, `valid_from/to_dttm` |
| `fct_shift` | `grp_em` | Факт смен | `store_rk`, `employee_profile_rk` |
| `worked_out_shift_snp` | `grp_ods_shftb` | Отработанные смены | `shift_id`, `worker_id`, `hours` |
| `shift_log_snp` | `grp_ods_shftb` | Логи смен | `shift_id`, `store_id`, `start_time`, `end_time` |
| `forecast__agg_picker_shift` | `grp_sandbox` | Агрегаты смен сборщиков | по часам |
| `shift_schedule_d` | `ssa_shifts_shifts` (ClickHouse SMKT) | Предпочтительный график смен сотрудника (запланированные интервалы): слоты `[time_slot_started_at, time_slot_ended_at)` в UTC (Zulu); CDC: `type` (created/deleted), `version` | `user_id` (UUID); дедуп-ключ `(user_id, time_slot_started_at, time_slot_ended_at)` по max(version) |
| `shift_assignment_d` | `ssa_shifts_shifts` (ClickHouse SMKT) | Назначения на смены (SSA): `workspace_id` (ЦФЗ), `user_role` (deliveryman/picker), `status` (assigned/canceled), `workspace_type` (DARKSTORE) | `assignment_id`, дедуп по max(version); ЦФЗ пользователя = workspace_id последнего `assigned` назначения по `updated_at` |

**Подводные камни `ssa_shifts_shifts`:**
- Под `_d` — `ReplicatedReplacingMergeTree` (`*_rr`) с мёржами не по расписанию: последняя версия собирается вручную — `argMax(поле, version)` + `group by <ключ>`; `FINAL` на Distributed не использовать.
- `shift_schedule_d.type = 'deleted'` отсекать **после** дедупа (HAVING), а не в WHERE.
- Аликас агрегата может перекрыть колонку в WHERE (`argMax(created_at, version) as created_at` → WHERE `created_at` резолвится в агрегат, ILLEGAL_AGGREGATION): выходной алиас называть иначе (`last_created_at`).
- План «как за день до смены»: последняя версия слота с `created_at < toStartOfDay(time_slot_started_at)` (граница суток, не окно 24ч); слот без версий до своего дня выпадает. Развёртка слота на целые часы в CH — `array join range(hours_cnt)`, hours_cnt = `dateDiff('hour', toStartOfHour(start), toStartOfHour(end)) + if(end != toStartOfHour(end), 1, 0)`.
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
