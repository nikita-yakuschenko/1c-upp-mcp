# Развёртывание MCP 1C RAG на Dokploy

Кратко: проект — это **Docker Compose** из нескольких сервисов (Qdrant, Embedding Service, Loader, MCP Server, MCP Inspector). В Dokploy его поднимают как приложение типа **Docker Compose**.

## Что нужно учесть в Dokploy

1. **Тип приложения** — Docker Compose (не одиночный Dockerfile).
2. **Сеть** — для доступа через домены используется Traefik; либо подключаете сервисы к `dokploy-network`, либо включаете **Isolated Deployments** (тогда сеть создаётся автоматически).
3. **Данные** — при автодеплое репозиторий каждый раз клонируется заново, поэтому каталоги вида `./qdrant_storage` и `./embeddings/models` будут пустыми после деплоя. Нужны **named volumes** или каталог `../files` (см. [документацию Dokploy](https://docs.dokploy.com/docs/core/docker-compose)).
4. **Имена контейнеров** — в Dokploy не задавайте `container_name` у сервисов (логи и метрики работают корректнее без них).

Ниже — пошаговый запуск и вариант `docker-compose` под Dokploy.

---

## Пошаговый запуск на Dokploy

### 1. Репозиторий

Укажите в Dokploy URL вашего репозитория, например: `https://github.com/nikita-yakuschenko/1c-upp-mcp.git`.

### 2. Новое приложение Docker Compose

1. В Dokploy: **Project** → **Add Service** → тип **Compose**.
2. **Compose type**: **Docker Compose** (не Stack).
3. **Repository**: URL вашего репозитория.
4. **Branch**: например `main`.
5. **Compose path**:  
   - если используете готовый файл для Dokploy — укажите `./docker-compose.dokploy.yml`;  
   - если правите основной — `./docker-compose.yml`.

### 3. Сохранение данных (обязательно)

В репозитории при деплое делается `git clone`, поэтому пути вида `./qdrant_storage` и `./embeddings/models` после следующего деплоя станут пустыми. В Dokploy нужно использовать:

- **Named volumes** (как в приложенном `docker-compose.dokploy.yml`), **или**
- Bind mount в каталог `../files` (например `../files/qdrant_storage`, `../files/embedding-models`), созданный через **Advanced → Mounts**.

Рекомендуется вариант с **named volumes** (как в `docker-compose.dokploy.yml`): данные переживут деплои и бэкапы через Dokploy будут доступны.

### 4. Домены и доступ снаружи

**Вариант A — без доменов (только порты)**  
Ничего не настраиваете в Dokploy. После деплоя сервисы доступны по портам хоста (см. README), если у контейнеров указаны `ports`. Подходит для теста или доступа по IP.

**Вариант B — домены через Dokploy (рекомендуется)**  
Домен: **`module.team`**. Для каждого сервиса задайте поддомен во вкладке **Domains**:

- **Loader** → `loader.module.team`
- **MCP Server** → `mcp.module.team`
- **Qdrant** → `qdrant.module.team`
- **MCP Inspector** → `inspector.module.team`

**DNS:** у регистратора/хостинга домена `module.team` создайте для каждого поддомена A-запись на IP сервера с Dokploy. Либо одну wildcard: `*.module.team` → A → IP сервера (тогда все поддомены сразу ведут на этот сервер, Traefik разведёт их по сервисам).

1. В приложении откройте вкладку **Domains**.  
2. Нажмите **Add Domain**.  
3. Выберите **сервис** (`loader`, `mcp-server`, `mcp-inspector`, `qdrant`) и укажите поддомен (например для MCP Server — `mcp.module.team`).  
4. Dokploy сам проставит Traefik labels и сеть.  
5. Для HTTPS: в настройках домена выберите SSL (например Let's Encrypt).

Если используете **Isolated Deployments**, сеть для Traefik создаётся автоматически, вручную прописывать `dokploy-network` в compose не обязательно.

**Вариант C — ручная настройка Traefik**  
Если нужен полный контроль, добавьте всем сервисам, которые должны быть снаружи, сеть `dokploy-network` и [Traefik labels](https://docs.dokploy.com/docs/core/docker-compose/domains) (в т.ч. `traefik.http.services.<name>.loadbalancer.server.port=...`). Для большинства сценариев достаточно варианта B.

### 5. Переменные окружения

В Dokploy: вкладка **Environment** у приложения. Можно задать, например:

- `COLLECTION_NAME` — имя коллекции в Qdrant (по умолчанию `1c_rag`).
- При необходимости: `EMBEDDING_SERVICE_URL`, `QDRANT_HOST`, `QDRANT_PORT` (внутри compose они уже указывают на сервисы по именам: `http://embedding-service:5000`, `qdrant`, `6333`).

В `docker-compose.dokploy.yml` переменные уже проброшены из окружения через `${...}` там, где это нужно.

### 6. Деплой

1. Сохраните настройки приложения.  
2. Нажмите **Deploy**.  
3. Дождитесь сборки и запуска (первый раз дольше из‑за образа эмбеддингов и модели).  
4. Если настраивали домены — подождите ~10 секунд, пока Traefik получит сертификаты.

### 7. Проверка

При поддоменах **`*.module.team`** (см. п. 4):

- **Loader**: `https://loader.module.team` (или порт 8501).  
- **MCP Server**: `https://mcp.module.team/mcp` — этот URL указывать в Cursor/IDE для MCP.  
- **Qdrant**: `https://qdrant.module.team/dashboard` (или порт 6333).  
- **MCP Inspector**: `https://inspector.module.team` (или порт 6274).

Дальше — по README: выгрузка структуры 1С (EPF), загрузка ZIP в Loader, настройка MCP в Cursor/IDE.

---

## Файл docker-compose.dokploy.yml

В корне репозитория добавлен файл **`docker-compose.dokploy.yml`** — вариант compose, подготовленный под Dokploy:

- Убраны `container_name`.
- Для данных Qdrant и кэша моделей эмбеддингов используются **named volumes**, чтобы данные не терялись при повторных деплоях.
- Все сервисы подключены к одной внутренней сети; при использовании **Domains** в Dokploy или **Isolated Deployments** маршрутизация через Traefik настраивается через UI.

В Dokploy в поле **Compose path** укажите: `./docker-compose.dokploy.yml`.

---

## Полезные ссылки

- [Docker Compose в Dokploy](https://docs.dokploy.com/docs/core/docker-compose)
- [Domains для Docker Compose](https://docs.dokploy.com/docs/core/docker-compose/domains)
- [Isolated Deployments](https://docs.dokploy.com/docs/core/docker-compose/utilities)
- [Volumes и бэкапы](https://docs.dokploy.com/docs/core/docker-compose#method-2-docker-named-volumes)
