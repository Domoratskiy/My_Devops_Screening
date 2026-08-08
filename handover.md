Incident handover

**Impact:** 12% error rate, p95 latency выросла до 2.8s, пользователи получают 502 Bad Gateway. Downtime ~3 минуты.

**Причина:** В релизе `a81f9c2` увеличена `WEB_CONCURRENCY` с 2 до 16. API запустил 16 воркеров, переполнил память и был убит OOM Killer (exit 137).

**Mitigation:** Откатил `WEB_CONCURRENCY` до `2` и перезапустил контейнер. Сервис восстановлен за 1.5 минуты.

**Changes:** Исправлен compose (resource limits, healthcheck, ротация логов, убран privileged и секреты, фиксированы версии образов). Исправлен CI (тесты перед деплоем, immutable tags, проверка после деплоя, ручной rollback).

**Evidence:** `curl -s -o /dev/null -w "%{http_code}" http://localhost/health` возвращает `200`.

**оставшиеся риски:** Диск заполнен на 92% — логи до ротации занимают 11 GB.

**следующее действие команды:** Очистить старые логи (`docker system prune -f`), добавить мониторинг диска и алертинг на OOM.