# RouteTable — upsert

**Endpoint:** `POST /v1/route-tables/upsert`

Создание или обновление RouteTable — правил маршрутизации внутри VPC. Определяет, куда направлять трафик из Subnet. RouteTable — самостоятельный ресурс без привязки к Subnet/VPC. Привязка выполняется через `RouteTableBinding`.

## Входные параметры

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `routeTables[]` | да | Object[] | |
| `routeTables[].metadata.name` | да | string | |
| `routeTables[].metadata.namespace` | да | string | |
| `routeTables[].metadata.uid` | при update | string | |
| `routeTables[].metadata.labels` | нет | Object | |
| `routeTables[].metadata.annotations` | нет | Object | |
| `routeTables[].spec.routes[]` | нет | Object[] | |
| `routeTables[].spec.routes[].destinationCidrBlock` | да | string | |
| `routeTables[].spec.routes[].target` | да | Object | |
| `routeTables[].spec.routes[].target.gatewayRef` | oneOf | Object | |
| `routeTables[].spec.routes[].target.gatewayRef.name` | условно | string | |
| `routeTables[].spec.routes[].target.gatewayRef.namespace` | условно | string | |
| `routeTables[].spec.routes[].target.addressRef` | oneOf | Object | |
| `routeTables[].spec.routes[].target.addressRef.name` | условно | string | |
| `routeTables[].spec.routes[].target.addressRef.namespace` | условно | string | |
| `routeTables[].spec.comment` | нет | string | |
| `routeTables[].spec.description` | нет | string | |
| `routeTables[].spec.displayName` | нет | string | |

## Ограничения

| Поле | Правило | Ошибка |
|------|---------|--------|
| `metadata.name` | обязательное; DNS-label, 1–63 символа, `^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$` | `400 invalid_name` |
| `metadata.namespace` | обязательное; тот же формат что `name` | `400 invalid_namespace` |
| `metadata.uid` | запрещён при upsert; обязателен при update; UUID v4 | `400 invalid_uid` |
| `metadata.labels` | ключ: `^[a-z0-9\-_.\/]{1,63}$`; значение: `^.{0,63}$` | `400 invalid_labels` |
| `metadata.annotations` | ключ: `^[a-z0-9\-_.\/]{1,253}$`; значение: без ограничений | `400 invalid_annotations` |
| `name` + `namespace` | уникальная пара (при upsert) | `409 already_exists` |
| `spec.routes[].destinationCidrBlock` | обязательное; валидный IPv4 CIDR или `0.0.0.0/0` | `400 invalid_destination_cidr` |
| `spec.routes[].target` | обязательное; ровно одно из: `gatewayRef`, `addressRef` | `400 invalid_route_target` |
| `spec.routes[].target.gatewayRef` | если указан — `{ name, namespace }`; Gateway должен существовать | `404 referenced_resource_not_found` |
| `spec.routes[].target.addressRef` | если указан — `{ name, namespace }`; Address должен существовать | `404 referenced_resource_not_found` |
| `spec.routes[]` | `destinationCidrBlock` уникален в массиве (нет дублей) | `400 duplicate_destination` |
| `spec.description` | макс. 256 символов | `400 invalid_description` |
| `spec.displayName` | макс. 128 символов | `400 invalid_display_name` |
| `spec.comment` | макс. 256 символов | `400 invalid_comment` |

## Выходные параметры

| название | тип данных |
|----------|-----------|
| `routeTables[]` | Object[] |
| `routeTables[].metadata` | Object |
| `routeTables[].metadata.name` | string |
| `routeTables[].metadata.namespace` | string |
| `routeTables[].metadata.uid` | string |
| `routeTables[].metadata.labels` | Object |
| `routeTables[].metadata.annotations` | Object |
| `routeTables[].metadata.creationTimestamp` | string |
| `routeTables[].metadata.resourceVersion` | string |
| `routeTables[].spec.routes[]` | Object[] |
| `routeTables[].spec.routes[].destinationCidrBlock` | string |
| `routeTables[].spec.routes[].target` | Object |
| `routeTables[].spec.comment` | string |
| `routeTables[].spec.description` | string |
| `routeTables[].spec.displayName` | string |

## Пример запроса

```bash
curl 'api:9006/v1/route-tables/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "routeTables": [
    {
      "metadata": {
        "name": "private-rt",
        "namespace": "tenant-a"
      },
      "spec": {
        "routes": [
          {
            "destinationCidrBlock": "0.0.0.0/0",
            "target": {
              "gatewayRef": {
                "name": "prod-nat",
                "namespace": "tenant-a"
              }
            }
          }
        ],
        "description": "Route table for private subnets"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "routeTables": [
    {
      "metadata": {
        "name": "private-rt",
        "namespace": "tenant-a",
        "uid": "d5e6f7a8-...",
        "creationTimestamp": "2026-03-01T12:03:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "routes": [
          {
            "destinationCidrBlock": "0.0.0.0/0",
            "target": {
              "gatewayRef": {
                "name": "prod-nat",
                "namespace": "tenant-a"
              }
            }
          }
        ],
        "description": "Route table for private subnets"
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании.
