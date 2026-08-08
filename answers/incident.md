# Incident analysis

## Первые пять команд

1. `docker compose ps` — гипотеза: проверяем состояние контейнеров, чтобы увидеть, какой сервис перезапускается. Ожидаемый сигнал: `api` в статусе `Restarting` — проблема именно в приложении.

2. `docker inspect "$(docker compose ps -q api)" --format 'OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}} RestartCount={{.RestartCount}}'` — гипотеза: контейнер убит из-за нехватки памяти (OOM). Ожидаемый сигнал: `OOMKilled=true ExitCode=137 RestartCount=7` — прямое подтверждение OOM.

3. `free -m` — гипотеза: проверяем доступную память на хосте. Ожидаемый сигнал: `available: 180` — памяти критически мало, OOM неизбежен.

4. `docker logs --tail=100 $(docker compose ps -q api)` — гипотеза: в логах приложения видны ошибки до падения. Ожидаемый сигнал: сообщения о нехватке памяти или ошибки Node.js.

5. `df -h /var/lib/docker/containers` — гипотеза: проверяем, не переполнен ли диск логами (вторичный риск). Ожидаемый сигнал: использование > 90% — диск переполнен.

## Основная причина

В релизе `a81f9c2` изменена переменная `WEB_CONCURRENCY` с `2` на `16` в `docker-compose.prod.yml`. Это привело к тому, что Node.js запустил 16 воркеров вместо 2. Каждый воркер потребляет ~150-200 MB памяти, итого ~2.9 GB только на воркеры. Суммарное потребление превысило 4 GB → OOM Killer убил процесс Node.js (exit 137).

**Evidence:**
- `OOMKilled=true ExitCode=137 RestartCount=7` в inspect контейнера
- `free -m` показывает 180 MB свободной памяти
- `git diff` показывает изменение `WEB_CONCURRENCY: "2"` → `"16"`

## Вторичные риски

1. **Disk full:** `/var/lib/docker/containers` занимает 11 GB, диск использован на 92% — нет ротации логов
2. **Security:** Пароль БД в открытом виде в compose-файле
3. **No resource limits:** Контейнер может съесть всю память хоста
4. **No swap:** Отсутствие swap ускоряет наступление OOM
5. **DB exposed:** Порт 5432 открыт наружу
6. **Privileged mode:** Контейнер запущен с привилегиями root

## Первое mitigation

**Немедленный rollback** — переключить `WEB_CONCURRENCY` обратно на `2`:

```bash
sed -i 's/WEB_CONCURRENCY: "16"/WEB_CONCURRENCY: "2"/' /opt/shop/docker-compose.prod.yml
docker compose -f /opt/shop/docker-compose.prod.yml up -d --no-deps api