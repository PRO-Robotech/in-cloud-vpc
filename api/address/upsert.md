# Address — upsert

**Endpoint:** `POST /v1/addresses/upsert`

Создание или обновление IP-адреса. Два типа:

- `private` — приватный IP; аллоцируется из CIDR Subnet (через `subnetRef`); IPAM назначает автоматически
- `public` — публичный IP (аналог AWS Elastic IP); резервируется из пула провайдера

Address — самостоятельный ресурс. Привязка к NetworkInterface выполняется через `AddressBinding`.

## Входные параметры

- `addresses[]` — массив ресурсов Address
- `addresses[].metadata.name` — уникальное имя Address в namespace
- `addresses[].metadata.namespace` — namespace (изоляция tenant'а)
- `addresses[].metadata.uid` — UUID; сервер генерирует при upsert; обязателен при update
- `addresses[].metadata.labels` — метки для labelSelector
- `addresses[].metadata.annotations` — произвольные аннотации
- `addresses[].spec.type` — тип адреса: `private` или `public`
- `addresses[].spec.subnetRef` — ссылка на Subnet: `{ name, namespace }` (только при `type: private`; IPAM аллоцирует IP из CIDR)
- `addresses[].spec.comment` — комментарий
- `addresses[].spec.description` — описание
- `addresses[].spec.displayName` — отображаемое имя

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `addresses[]` | да | Object[] | |
| `addresses[].metadata.name` | да | string | |
| `addresses[].metadata.namespace` | да | string | |
| `addresses[].metadata.uid` | при update | string | |
| `addresses[].metadata.labels` | нет | Object | |
| `addresses[].metadata.annotations` | нет | Object | |
| `addresses[].spec.type` | да | enum (`private` / `public`) | |
| `addresses[].spec.subnetRef` | да (при `type: private`) | Object | |
| `addresses[].spec.subnetRef.name` | да (при `type: private`) | string | |
| `addresses[].spec.subnetRef.namespace` | да (при `type: private`) | string | |
| `addresses[].spec.comment` | нет | string | |
| `addresses[].spec.description` | нет | string | |
| `addresses[].spec.displayName` | нет | string | |

## Ограничения

| Поле | Правило | Ошибка |
|------|---------|--------|
| `metadata.name` | обязательное; DNS-label, 1–63 символа, `^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$` | `400 invalid_name` |
| `metadata.namespace` | обязательное; тот же формат что `name` | `400 invalid_namespace` |
| `metadata.uid` | запрещён при upsert; обязателен при update; UUID v4 | `400 invalid_uid` |
| `metadata.labels` | ключ: `^[a-z0-9\-_.\/]{1,63}$`; значение: `^.{0,63}$` | `400 invalid_labels` |
| `metadata.annotations` | ключ: `^[a-z0-9\-_.\/]{1,253}$`; значение: без ограничений | `400 invalid_annotations` |
| `name` + `namespace` | уникальная пара (при upsert) | `409 already_exists` |
| `spec.type` | обязательное; `private` или `public` | `400 invalid_address_type` |
| `spec.type` | неизменяемое (immutable) | `400 immutable_field` |
| `spec.subnetRef` | обязательное при `type: private`; `{ name, namespace }`; Subnet должен существовать | `404 referenced_resource_not_found` |
| `spec.subnetRef` | запрещён при `type: public` | `400 subnet_not_allowed_for_public` |
| `spec.subnetRef` (private) | в подсети должны быть свободные IP | `409 subnet_exhausted` |
| `spec.subnetRef` | неизменяемое (immutable) | `400 immutable_field` |
| `spec.description` | макс. 256 символов | `400 invalid_description` |
| `spec.displayName` | макс. 128 символов | `400 invalid_display_name` |
| `spec.comment` | макс. 256 символов | `400 invalid_comment` |

## Выходные параметры

- `addresses[]` — массив Address
- `addresses[].metadata` — стандартные метаданные
- `addresses[].metadata.name` — имя Address
- `addresses[].metadata.namespace` — namespace
- `addresses[].metadata.uid` — UUID
- `addresses[].metadata.labels` — метки
- `addresses[].metadata.annotations` — аннотации
- `addresses[].metadata.creationTimestamp` — время создания (RFC 3339)
- `addresses[].metadata.resourceVersion` — версия ресурса
- `addresses[].spec.type` — тип адреса
- `addresses[].spec.subnetRef` — ссылка на Subnet (при `type: private`)
- `addresses[].spec.comment` — комментарий
- `addresses[].spec.description` — описание
- `addresses[].spec.displayName` — отображаемое имя
- `addresses[].status.ip` — аллоцированный IP-адрес (приватный или публичный)
- `addresses[].status.state` — состояние: `allocated`, `released`

| название | тип данных |
|----------|-----------|
| `addresses[]` | Object[] |
| `addresses[].metadata` | Object |
| `addresses[].metadata.name` | string |
| `addresses[].metadata.namespace` | string |
| `addresses[].metadata.uid` | string |
| `addresses[].metadata.labels` | Object |
| `addresses[].metadata.annotations` | Object |
| `addresses[].metadata.creationTimestamp` | string |
| `addresses[].metadata.resourceVersion` | string |
| `addresses[].spec.type` | enum |
| `addresses[].spec.subnetRef` | Object |
| `addresses[].spec.comment` | string |
| `addresses[].spec.description` | string |
| `addresses[].spec.displayName` | string |
| `addresses[].status.ip` | string |
| `addresses[].status.state` | enum |

## Пример запроса (приватный IP)

```bash
curl 'api:9006/v1/addresses/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "addresses": [
    {
      "metadata": {
        "name": "web-private-ip",
        "namespace": "tenant-a"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "web-subnet",
          "namespace": "tenant-a"
        },
        "description": "Private IP for web server"
      }
    }
  ]
}'
```

## Пример запроса (публичный IP)

```bash
curl 'api:9006/v1/addresses/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "addresses": [
    {
      "metadata": {
        "name": "web-eip-1",
        "namespace": "tenant-a",
        "labels": {
          "role": "web"
        }
      },
      "spec": {
        "type": "public",
        "description": "Elastic IP for web server"
      }
    }
  ]
}'
```

## Пример ответа (приватный IP)

```json
{
  "addresses": [
    {
      "metadata": {
        "name": "web-private-ip",
        "namespace": "tenant-a",
        "uid": "e2f3a4b5-...",
        "creationTimestamp": "2026-03-01T12:02:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "web-subnet",
          "namespace": "tenant-a"
        },
        "description": "Private IP for web server"
      },
      "status": {
        "ip": "10.0.1.12",
        "state": "allocated"
      }
    }
  ]
}
```

## Пример ответа (публичный IP)

```json
{
  "addresses": [
    {
      "metadata": {
        "name": "web-eip-1",
        "namespace": "tenant-a",
        "uid": "f1a2b3c4-...",
        "labels": {
          "role": "web"
        },
        "creationTimestamp": "2026-03-01T12:03:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "type": "public",
        "description": "Elastic IP for web server"
      },
      "status": {
        "ip": "203.0.113.42",
        "state": "allocated"
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании. Поля `spec.type` и `spec.subnetRef` являются неизменяемыми (immutable).
