# Деплой на VPS (ARM64)

## Архитектура

- CI (`frontend` / `backend` / `ai`) пушит **multi-arch** образы в GHCR: `linux/amd64` + `linux/arm64`.
- На ARM VPS в `docker-compose.yaml` для app-сервисов указано `platform: linux/arm64` — Docker подтянет нужный слой из manifest list.
- После `git pull` в `~/deploy` выполняется `docker compose pull && docker compose up -d`.

## Проверка на сервере

```bash
cd ~/deploy
docker image inspect ghcr.io/ds-testkit/frontend:latest --format '{{.Os}}/{{.Architecture}}'
docker image inspect ghcr.io/ds-testkit/backend:latest --format '{{.Os}}/{{.Architecture}}'
docker image inspect ghcr.io/ds-testkit/ai:latest --format '{{.Os}}/{{.Architecture}}'
# Ожидается: linux/arm64

docker compose ps
docker compose logs backend --tail 80
docker compose logs frontend --tail 80
docker compose logs ai-service --tail 80
docker compose exec -T backend curl -sf http://localhost:8000/api/v1/health
docker compose exec -T ai-service wget -qO- http://localhost:8100/health
curl -sf https://velikoss.ru/api/ai/health
```

## AI service

- Образ: `ghcr.io/ds-testkit/ai`, порт внутри сети `8100`.
- Nginx: `https://velikoss.ru/api/ai/` → `ai-service` (SSE, `proxy_buffering off` в `nginx/nginx.conf`).
- `JWT_SECRET` в compose = `SECRET_KEY` бекенда (обязательно задать в `.env`).
- LLM: `AI_PROVIDER`, `AI_API_KEY`, `AI_MODEL`, `AI_BASE_URL`.
  - **VseGPT:** `AI_PROVIDER=openai`, `AI_BASE_URL=https://api.vsegpt.ru`, ключ и модель из личного кабинета VseGPT.
  - **Ollama:** `AI_PROVIDER=ollama`, `AI_BASE_URL=http://ollama:11434`.

### Uptime Kuma

В UI Kuma (https://velikoss.ru/status/) добавьте монитор:

| Поле | Значение |
|------|----------|
| Type | HTTP(s) |
| URL | `https://velikoss.ru/api/ai/health` |
| Interval | 60s |
| Keyword | `ok` |

Опционально readiness (если Ollama): `https://velikoss.ru/api/ai/ready`.

`POST /v1/chat` в Kuma **не** мониторить (нужен JWT).

Если архитектура `linux/amd64` на ARM VPS — контейнеры не стартуют (`exec format error`).

## Типичные причины «вечной загрузки»

1. **Бекенд не healthy** — миграции/БД: смотреть `docker compose logs backend`.
2. **API недоступен** — nginx проксирует `/api/v1/` на `backend:8000`; без работающего бекенда `fetchMe` зависал (теперь таймаут 30 с).
3. **Healthcheck фронта** — раньше использовался `wget` (нет в `node:alpine`); nginx мог не подниматься из-за `depends_on: service_healthy`.
4. **ORIGIN / X-Forwarded-*** — для SvelteKit за nginx заданы `ORIGIN`, `PROTOCOL_HEADER`, `HOST_HEADER` в compose.

## `.env` на сервере

Скопируйте из `.env.example`: `DB_*`, `SECRET_KEY`, `PUBLIC_ORIGIN`, `CORS_ORIGINS`, `AI_*`.

После первого деплоя ai: `docker compose pull ai-service && docker compose up -d ai-service nginx`.
