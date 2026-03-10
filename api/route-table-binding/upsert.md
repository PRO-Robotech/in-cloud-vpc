# RouteTableBinding — upsert

**Endpoint:** `POST /v1/route-table-bindings/upsert`

Создание или обновление RouteTableBinding — связи между RouteTable и Subnet или VPC. Самостоятельный ресурс. Binding на Subnet имеет приоритет над Binding на VPC. Если у Subnet есть собственный RouteTableBinding, VPC-level binding для него игнорируется.

## Входные параметры

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `routeTableBindings[]` | да | Object[] | |
| `routeTableBindings[].metadata.name` | да | string | |
| `routeTableBindings[].metadata.namespace` | да | string | |
| `routeTableBindings[].metadata.uid` | при update | string | |
| `routeTableBindings[].metadata.labels` | нет | Object | |
| `routeTableBindings[].metadata.annotations` | нет | Object | |
| `routeTableBindings[].spec.routeTableRef` | да | Object | |
| `routeTableBindings[].spec.routeTableRef.name` | да | string | |
| `routeTableBindings[].spec.routeTableRef.namespace` | да | string | |
| `routeTableBindings[].spec.target` | да | Object | |
| `routeTableBindings[].spec.target.type` | да | enum (`subnet` / `vpc`) | |
| `routeTableBindings[].spec.target.ref` | да | Object | |
| `routeTableBindings[].spec.target.ref.name` | да | string | |
| `routeTableBindings[].spec.target.ref.namespace` | да | string | |
| `routeTableBindings[].spec.comment` | нет | string | |
| `routeTableBindings[].spec.description` | нет | string | |
| `routeTableBindings[].spec.displayName` | нет | string | |

## Ограничения

| Поле | Правило | Ошибка |
|------|---------|--------|
| `metadata.name` | обязательное; DNS-label, 1–63 символа, `^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$` | `400 invalid_name` |
| `metadata.namespace` | обязательное; тот же формат что `name` | `400 invalid_namespace` |
| `metadata.uid` | запрещён при upsert; обязателен при update; UUID v4 | `400 invalid_uid` |
| `metadata.labels` | ключ: `^[a-z0-9\-_.\/]{1,63}$`; значение: `^.{0,63}$` | `400 invalid_labels` |
| `metadata.annotations` | ключ: `^[a-z0-9\-_.\/]{1,253}$`; значение: без ограничений | `400 invalid_annotations` |
| `name` + `namespace` | уникальная пара (при upsert) | `409 already_exists` |
| `spec.routeTableRef` | обязательное; `{ name, namespace }`; RouteTable должен существовать | `404 referenced_resource_not_found` |
| `spec.routeTableRef` | неизменяемое (immutable) | `400 immutable_field` |
| `spec.target` | обязательное; объект с `type` + `ref` | `400 invalid_target` |
| `spec.target.type` | обязательное; `subnet` или `vpc` | `400 invalid_target_type` |
| `spec.target.type` | неизменяемое (immutable) | `400 immutable_field` |
| `spec.target.ref` | обязательное; `{ name, namespace }`; целевой ресурс (Subnet или VPC) должен существовать | `404 referenced_resource_not_found` |
| `spec.target.ref` | неизменяемое (immutable) | `400 immutable_field` |
| `target` (type=subnet) | один RouteTableBinding на Subnet | `409 subnet_already_bound` |
| `target` (type=vpc) | один RouteTableBinding уровня VPC на VPC | `409 vpc_already_bound` |
| `spec.description` | макс. 256 символов | `400 invalid_description` |
| `spec.displayName` | макс. 128 символов | `400 invalid_display_name` |
| `spec.comment` | макс. 256 символов | `400 invalid_comment` |

## Выходные параметры

| название | тип данных |
|----------|-----------|
| `routeTableBindings[]` | Object[] |
| `routeTableBindings[].metadata` | Object |
| `routeTableBindings[].metadata.name` | string |
| `routeTableBindings[].metadata.namespace` | string |
| `routeTableBindings[].metadata.uid` | string |
| `routeTableBindings[].metadata.labels` | Object |
| `routeTableBindings[].metadata.annotations` | Object |
| `routeTableBindings[].metadata.creationTimestamp` | string |
| `routeTableBindings[].metadata.resourceVersion` | string |
| `routeTableBindings[].spec.routeTableRef` | Object |
| `routeTableBindings[].spec.target` | Object |
| `routeTableBindings[].spec.target.type` | enum |
| `routeTableBindings[].spec.target.ref` | Object |
| `routeTableBindings[].spec.comment` | string |
| `routeTableBindings[].spec.description` | string |
| `routeTableBindings[].spec.displayName` | string |

## Пример запроса (Привязка к Subnet)

```bash
curl 'api:9006/v1/route-table-bindings/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "routeTableBindings": [
    {
      "metadata": {
        "name": "web-subnet-rt-binding",
        "namespace": "tenant-a"
      },
      "spec": {
        "routeTableRef": {
          "name": "private-rt",
          "namespace": "tenant-a"
        },
        "target": {
          "type": "subnet",
          "ref": {
            "name": "web-subnet",
            "namespace": "tenant-a"
          }
        }
      }
    }
  ]
}'
```

## Пример запроса (Привязка к VPC — общая таблица маршрутов)

```bash
curl 'api:9006/v1/route-table-bindings/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "routeTableBindings": [
    {
      "metadata": {
        "name": "prod-vpc-default-rt",
        "namespace": "tenant-a"
      },
      "spec": {
        "routeTableRef": {
          "name": "private-rt",
          "namespace": "tenant-a"
        },
        "target": {
          "type": "vpc",
          "ref": {
            "name": "prod-vpc",
            "namespace": "tenant-a"
          }
        }
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "routeTableBindings": [
    {
      "metadata": {
        "name": "web-subnet-rt-binding",
        "namespace": "tenant-a",
        "uid": "f6a7b8c9-...",
        "creationTimestamp": "2026-03-01T12:05:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "routeTableRef": {
          "name": "private-rt",
          "namespace": "tenant-a"
        },
        "target": {
          "type": "subnet",
          "ref": {
            "name": "web-subnet",
            "namespace": "tenant-a"
          }
        }
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании. Поля `spec.routeTableRef`, `spec.target.type` и `spec.target.ref` являются неизменяемыми (immutable).
