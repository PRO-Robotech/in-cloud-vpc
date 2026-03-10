# Gateway — upsert

**Endpoint:** `POST /v1/gateways/upsert`

Создание или обновление Gateway — NAT Gateway. Позволяет NI без публичного Address выходить в интернет через общий IP шлюза (SNAT). Gateway — самостоятельный ресурс без прямых ссылок на VPC/Subnet. Связь с VPC/Subnet определяется через RouteTable (маршрут `target.gatewayRef`) и RouteTableBinding.

## Входные параметры

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `gateways[]` | да | Object[] | |
| `gateways[].metadata.name` | да | string | |
| `gateways[].metadata.namespace` | да | string | |
| `gateways[].metadata.uid` | при update | string | |
| `gateways[].metadata.labels` | нет | Object | |
| `gateways[].metadata.annotations` | нет | Object | |
| `gateways[].spec.comment` | нет | string | |
| `gateways[].spec.description` | нет | string | |
| `gateways[].spec.displayName` | нет | string | |

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

> Gateway имеет единственный тип — NAT. Дополнительных spec-полей, кроме стандартных строковых, нет.

## Выходные параметры

| название | тип данных |
|----------|-----------|
| `gateways[]` | Object[] |
| `gateways[].metadata` | Object |
| `gateways[].metadata.name` | string |
| `gateways[].metadata.namespace` | string |
| `gateways[].metadata.uid` | string |
| `gateways[].metadata.labels` | Object |
| `gateways[].metadata.annotations` | Object |
| `gateways[].metadata.creationTimestamp` | string |
| `gateways[].metadata.resourceVersion` | string |
| `gateways[].spec.comment` | string |
| `gateways[].spec.description` | string |
| `gateways[].spec.displayName` | string |
| `gateways[].status.state` | enum |
| `gateways[].status.publicIp` | string |

## Пример запроса

```bash
curl 'api:9006/v1/gateways/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "gateways": [
    {
      "metadata": {
        "name": "prod-nat",
        "namespace": "tenant-a"
      },
      "spec": {
        "description": "NAT Gateway for private subnets"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "gateways": [
    {
      "metadata": {
        "name": "prod-nat",
        "namespace": "tenant-a",
        "uid": "c3d4e5f6-...",
        "creationTimestamp": "2026-03-01T12:02:30Z",
        "resourceVersion": "1"
      },
      "spec": {
        "description": "NAT Gateway for private subnets"
      },
      "status": {
        "state": "available",
        "publicIp": "203.0.113.10"
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании.
