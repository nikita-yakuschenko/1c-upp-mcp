# 1C UPP MCP — MCP-сервер с RAG по структуре конфигурации 1С

Проект создан на основании разработок **Сергея Филкина**. Лицензия и условия использования — см. [LICENSE](LICENSE) в корне репозитория.

---

Набор сервисов для работы с векторной базой по документации конфигурации 1С: загрузка выгрузки, эмбеддинги, RAG-поиск и MCP для Cursor и других IDE.

**Состав:**

1. **Embedding Service** — генерация векторных представлений
2. **Loader** — веб-интерфейс загрузки архива выгрузки из 1С в Qdrant
3. **MCP Server** — ответы на вопросы по конфигурации через RAG
4. **Qdrant** — векторная БД
5. **MCP Inspector** — отладка/проверка MCP

Обработка для выгрузки структуры из 1С: `ПолучитьТекстСтруктурыКонфигурацииФайлами.epf`.

## Запуск (Docker Compose)

```bash
chmod +x start.sh && chmod +x stop.sh
./start.sh
# или: docker-compose up --build
```

Остановка: `./stop.sh` или `docker-compose down`.

## Доступные сервисы

| Сервис        | URL (локально)              |
|---------------|-----------------------------|
| Loader        | http://localhost:8501       |
| Embedding API | http://localhost:5000        |
| Qdrant        | http://localhost:6333/dashboard |
| MCP Server    | http://localhost:8000/mcp   |
| MCP Inspector | http://localhost:6274       |

## Как пользоваться

1. Запустить сервисы: `./start.sh`
2. Выгрузить структуру конфигурации из 1С через обработку `ПолучитьТекстСтруктурыКонфигурацииФайлами.epf`
3. Открыть Loader (порт 8501), загрузить ZIP с markdown и `objects.csv`, нажать «Начать обработку»
4. Подключить MCP в Cursor/IDE — в URL указать **хост или домен** (например при размещении на домене: `https://mcp.module.team/mcp`, локально: `http://localhost:8000/mcp`).

**Подключение в Cursor** (`.cursor/mcp.json`):

```json
{
  "servers": {
    "1c-upp-mcp-server": {
      "headers": { "x-collection-name": "имя_коллекции_в_qdrant" },
      "url": "https://<хост или домен>/mcp"
    }
  }
}
```

- **url** — хост или домен MCP-сервера (при домене без порта: `https://mcp.example.com/mcp`; локально: `http://localhost:8000/mcp`).
- **x-collection-name** — имя коллекции в Qdrant (по умолчанию `1c_rag`).

## Развёртывание на Dokploy

Пошаговая инструкция: [DOKPLOY.md](DOKPLOY.md).

## Переменные окружения

- `EMBEDDING_SERVICE_URL` — URL сервиса эмбеддингов
- `QDRANT_HOST`, `QDRANT_PORT` — хост и порт Qdrant
- `COLLECTION_NAME` — имя коллекции в Qdrant (по умолчанию `1c_rag`)
- `ROW_BATCH_SIZE`, `EMBEDDING_BATCH_SIZE` — размеры батчей

## Структура репозитория

```
├── embeddings/     # Сервис эмбеддингов
├── loader/         # Веб-загрузчик
├── mcp/            # MCP RAG-сервер
├── inspector/      # MCP Inspector
├── article/        # Статья
├── docker-compose.yml
├── docker-compose.dokploy.yml
├── start.sh / stop.sh
├── DOKPLOY.md
└── LICENSE
```

## Логи

```bash
docker-compose logs -f loader
docker-compose logs -f mcp-server
docker-compose logs -f
```
