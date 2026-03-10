# VPC — upsert

**Endpoint:** `POST /v1/vpcs/upsert`

Создание или обновление VPC — области изоляции. VPC не содержит адресов — это чистый isolation domain. Адресные пространства определяются Subnet'ами, которые подключаются к VPC.

## Входные параметры

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `vpcs[]` | да | Object[] | |
| `vpcs[].metadata.name` | да | string | |
| `vpcs[].metadata.namespace` | да | string | |
| `vpcs[].metadata.uid` | при update | string | |
| `vpcs[].metadata.labels` | нет | Object | |
| `vpcs[].metadata.annotations` | нет | Object | |
| `vpcs[].spec.comment` | нет | string | |
| `vpcs[].spec.description` | нет | string | |
| `vpcs[].spec.displayName` | нет | string | |

## Ограничения

| Поле | Правило | Ошибка |
|------|---------|--------|
| `metadata.name` | обязательное; DNS-label, 1–63 символа, `^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$` | `400 invalid_name` |
| `metadata.namespace` | обязательное; тот же формат что `name` | `400 invalid_namespace` |
| `metadata.uid` | запрещён при upsert; обязателен при update; UUID v4 | `400 invalid_uid` |
| `metadata.labels` | ключ: `^[a-z0-9\-_.\/]{1,63}$`; значение: `^.{0,63}$` | `400 invalid_labels` |
| `metadata.annotations` | ключ: `^[a-z0-9\-_.\/]{1,253}$`; значение: без ограничений | `400 invalid_annotations` |
| `name` + `namespace` | уникальная пара (при upsert) | `409 already_exists` |
| `spec.description` | макс. 256 символов | `400 invalid_description` |
| `spec.displayName` | макс. 128 символов | `400 invalid_display_name` |
| `spec.comment` | макс. 256 символов | `400 invalid_comment` |

> Нет обязательных spec-полей помимо стандартных строковых.

## Выходные параметры

| название | тип данных |
|----------|-----------|
| `vpcs[]` | Object[] |
| `vpcs[].metadata` | Object |
| `vpcs[].metadata.name` | string |
| `vpcs[].metadata.namespace` | string |
| `vpcs[].metadata.uid` | string |
| `vpcs[].metadata.labels` | Object |
| `vpcs[].metadata.annotations` | Object |
| `vpcs[].metadata.creationTimestamp` | string |
| `vpcs[].metadata.resourceVersion` | string |
| `vpcs[].spec.comment` | string |
| `vpcs[].spec.description` | string |
| `vpcs[].spec.displayName` | string |
| `vpcs[].status.vni` | int |

## Пример запроса

```bash
curl 'api:9006/v1/vpcs/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "vpcs": [
    {
      "metadata": {
        "name": "prod-vpc",
        "namespace": "tenant-a",
        "labels": {
          "env": "prod"
        }
      },
      "spec": {
        "description": "Production VPC",
        "displayName": "Prod VPC"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "vpcs": [
    {
      "metadata": {
        "name": "prod-vpc",
        "namespace": "tenant-a",
        "uid": "a1b2c3d4-...",
        "labels": {
          "env": "prod"
        },
        "creationTimestamp": "2026-03-01T12:00:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "description": "Production VPC",
        "displayName": "Prod VPC"
      },
      "status": {
        "vni": 100001
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании.
