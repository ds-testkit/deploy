# Деплой на VPS (ARM64)

## Архитектура

- CI (`frontend` / `backend`) пушит образы **только** `linux/arm64` в GHCR.
- На сервере в `docker-compose.yaml` для app-сервисов указано `platform: linux/arm64`.
- После `git pull` в `~/deploy` выполняется `docker compose pull && docker compose up -d`.

## Проверка на сервере

```bash
cd ~/deploy
docker image inspect ghcr.io/ds-testkit/frontend:latest --format '{{.Os}}/{{.Architecture}}'
docker image inspect ghcr.io/ds-testkit/backend:latest --format '{{.Os}}/{{.Architecture}}'
# Ожидается: linux/arm64

docker compose ps
docker compose logs backend --tail 80
docker compose logs frontend --tail 80
docker compose exec -T backend curl -sf http://localhost:8000/api/v1/health
```

Если архитектура `linux/amd64` на ARM VPS — контейнеры не стартуют (`exec format error`).

## Типичные причины «вечной загрузки»

1. **Бекенд не healthy** — миграции/БД: смотреть `docker compose logs backend`.
2. **API недоступен** — nginx проксирует `/api/v1/` на `backend:8000`; без работающего бекенда `fetchMe` зависал (теперь таймаут 30 с).
3. **Healthcheck фронта** — раньше использовался `wget` (нет в `node:alpine`); nginx мог не подниматься из-за `depends_on: service_healthy`.
4. **ORIGIN / X-Forwarded-*** — для SvelteKit за nginx заданы `ORIGIN`, `PROTOCOL_HEADER`, `HOST_HEADER` в compose.

## `.env` на сервере

Скопируйте из `.env.example`: `DB_*`, `PUBLIC_ORIGIN`, `CORS_ORIGINS`.
