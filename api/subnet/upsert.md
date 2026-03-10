# Subnet — upsert

**Endpoint:** `POST /v1/subnets/upsert`

Создание или обновление Subnet — сети с адресным пространством. Привязка к VPC задаётся через `spec.vpcRef` — обязательное, неизменяемое поле.

## Входные параметры

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `subnets[]` | да | Object[] | |
| `subnets[].metadata.name` | да | string | |
| `subnets[].metadata.namespace` | да | string | |
| `subnets[].metadata.uid` | при update | string | |
| `subnets[].metadata.labels` | нет | Object | |
| `subnets[].metadata.annotations` | нет | Object | |
| `subnets[].spec.cidrBlock` | да | string | |
| `subnets[].spec.vpcRef` | да | Object | |
| `subnets[].spec.vpcRef.name` | да | string | |
| `subnets[].spec.vpcRef.namespace` | да | string | |
| `subnets[].spec.comment` | нет | string | |
| `subnets[].spec.description` | нет | string | |
| `subnets[].spec.displayName` | нет | string | |

## Ограничения

| Поле | Правило | Ошибка |
|------|---------|--------|
| `metadata.name` | обязательное; DNS-label, 1–63 символа, `^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$` | `400 invalid_name` |
| `metadata.namespace` | обязательное; тот же формат что `name` | `400 invalid_namespace` |
| `metadata.uid` | запрещён при upsert; обязателен при update; UUID v4 | `400 invalid_uid` |
| `metadata.labels` | ключ: `^[a-z0-9\-_.\/]{1,63}$`; значение: `^.{0,63}$` | `400 invalid_labels` |
| `metadata.annotations` | ключ: `^[a-z0-9\-_.\/]{1,253}$`; значение: без ограничений | `400 invalid_annotations` |
| `name` + `namespace` | уникальная пара (при upsert) | `409 already_exists` |
| `spec.cidrBlock` | обязательное; валидный IPv4 CIDR; префикс `/16` – `/28` | `400 invalid_cidr_block` |
| `spec.cidrBlock` | неизменяемое (immutable) | `400 immutable_field` |
| `spec.vpcRef` | обязательное; `{ name, namespace }`; VPC должен существовать | `404 referenced_resource_not_found` |
| `spec.vpcRef` | неизменяемое (immutable) — Subnet не перемещается между VPC | `400 immutable_field` |
| CIDR Subnet | не должен пересекаться с другими Subnet в том же VPC | `409 cidr_overlap` |
| `spec.description` | макс. 256 символов | `400 invalid_description` |
| `spec.displayName` | макс. 128 символов | `400 invalid_display_name` |
| `spec.comment` | макс. 256 символов | `400 invalid_comment` |

## Выходные параметры

| название | тип данных |
|----------|-----------|
| `subnets[]` | Object[] |
| `subnets[].metadata` | Object |
| `subnets[].metadata.name` | string |
| `subnets[].metadata.namespace` | string |
| `subnets[].metadata.uid` | string |
| `subnets[].metadata.labels` | Object |
| `subnets[].metadata.annotations` | Object |
| `subnets[].metadata.creationTimestamp` | string |
| `subnets[].metadata.resourceVersion` | string |
| `subnets[].spec.cidrBlock` | string |
| `subnets[].spec.vpcRef` | Object |
| `subnets[].spec.comment` | string |
| `subnets[].spec.description` | string |
| `subnets[].spec.displayName` | string |
| `subnets[].status.availableIpCount` | int |
| `subnets[].status.vni` | int |

## Пример запроса

```bash
curl 'api:9006/v1/subnets/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "subnets": [
    {
      "metadata": {
        "name": "web-subnet",
        "namespace": "tenant-a",
        "labels": {
          "tier": "web"
        }
      },
      "spec": {
        "cidrBlock": "10.0.1.0/24",
        "vpcRef": {
          "name": "prod-vpc",
          "namespace": "tenant-a"
        },
        "displayName": "Web tier subnet"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "subnets": [
    {
      "metadata": {
        "name": "web-subnet",
        "namespace": "tenant-a",
        "uid": "b2c3d4e5-...",
        "labels": {
          "tier": "web"
        },
        "creationTimestamp": "2026-03-01T12:01:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "cidrBlock": "10.0.1.0/24",
        "vpcRef": {
          "name": "prod-vpc",
          "namespace": "tenant-a"
        },
        "displayName": "Web tier subnet"
      },
      "status": {
        "availableIpCount": 251,
        "vni": 100001
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании. Поля `spec.cidrBlock` и `spec.vpcRef` являются неизменяемыми (immutable).
