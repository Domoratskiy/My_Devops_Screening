**AI Tool:** ChatGPT (OpenAI GPT-4)

**Parts used:** Структуризация ответов, генерация YAML конфигураций, расчет memory budget.

**Verification:** Все значения ресурсов пересчитаны вручную на основе данных о потреблении Node.js воркеров (~180 MB/воркер) и total memory 4 GB.

**Rejected AI suggestion:** AI предложил использовать `docker system prune -af` для очистки диска. Я отклонил — это может удалить текущие образы и сломать rollback.

**Why:** В production агрессивные prune-команды опасны без подтверждения.