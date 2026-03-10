# Namespace — upsert

**Endpoint:** `POST /v1/namespaces/upsert`

Создание или обновление namespace — контейнера изоляции tenant'а. У Namespace нет поля `metadata.namespace` (он сам является контейнером). Уникальность — по `metadata.name`.

## Входные параметры

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `namespaces[]` | да | Object[] | |
| `namespaces[].metadata.name` | да | string | |
| `namespaces[].metadata.uid` | при update | string | |
| `namespaces[].metadata.labels` | нет | Object | |
| `namespaces[].metadata.annotations` | нет | Object | |
| `namespaces[].spec.displayName` | нет | string | |
| `namespaces[].spec.description` | нет | string | |

## Ограничения

| Поле | Правило | Ошибка |
|------|---------|--------|
| `metadata.name` | обязательное; DNS-label, 1–63 символа, `^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$` | `400 invalid_name` |
| `metadata.uid` | запрещён при upsert; обязателен при update; UUID v4 | `400 invalid_uid` |
| `metadata.labels` | ключ: `^[a-z0-9\-_.\/]{1,63}$`; значение: `^.{0,63}$` | `400 invalid_labels` |
| `metadata.annotations` | ключ: `^[a-z0-9\-_.\/]{1,253}$`; значение: без ограничений | `400 invalid_annotations` |
| `metadata.name` | уникальное (при upsert) | `409 already_exists` |
| `spec.description` | макс. 256 символов | `400 invalid_description` |
| `spec.displayName` | макс. 128 символов | `400 invalid_display_name` |

## Выходные параметры

| название | тип данных |
|----------|-----------|
| `namespaces[]` | Object[] |
| `namespaces[].metadata` | Object |
| `namespaces[].metadata.name` | string |
| `namespaces[].metadata.uid` | string |
| `namespaces[].metadata.labels` | Object |
| `namespaces[].metadata.annotations` | Object |
| `namespaces[].metadata.creationTimestamp` | string |
| `namespaces[].metadata.resourceVersion` | string |
| `namespaces[].spec.displayName` | string |
| `namespaces[].spec.description` | string |
| `namespaces[].status.state` | string |

## Пример запроса

```bash
curl 'api:9006/v1/namespaces/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "namespaces": [
    {
      "metadata": {
        "name": "my-namespace"
      },
      "spec": {
        "displayName": "My Namespace",
        "description": "Namespace description"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "namespaces": [
    {
      "metadata": {
        "name": "my-namespace",
        "uid": "ns-a1b2c3d4-...",
        "creationTimestamp": "2026-03-01T12:00:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "displayName": "My Namespace",
        "description": "Namespace description"
      },
      "status": {
        "state": "active"
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании.

## Переходы состояний

| Состояние | Описание |
|-----------|----------|
| `active` | Namespace активен, ресурсы внутри доступны |
