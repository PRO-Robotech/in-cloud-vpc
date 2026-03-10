# NetworkInterface — upsert

**Endpoint:** `POST /v1/network-interfaces/upsert`

Создание или обновление виртуального сетевого интерфейса (VNIC). Создаётся независимо. Приватный и публичный IP назначаются через ресурсы `Address` + `AddressBinding`.

**Переходы состояний:**

```
CREATED ──AddressBinding(private Address)──▶ AVAILABLE
```

## Входные параметры

- `networkInterfaces[]` — массив ресурсов NetworkInterface
- `networkInterfaces[].metadata.name` — уникальное имя NetworkInterface в namespace
- `networkInterfaces[].metadata.namespace` — namespace (изоляция tenant'а)
- `networkInterfaces[].metadata.uid` — UUID; сервер генерирует при upsert; обязателен при update
- `networkInterfaces[].metadata.labels` — метки для labelSelector
- `networkInterfaces[].metadata.annotations` — произвольные аннотации
- `networkInterfaces[].spec.subnetRef` — ссылка на Subnet: `{ name, namespace }`
- `networkInterfaces[].spec.comment` — комментарий
- `networkInterfaces[].spec.description` — описание
- `networkInterfaces[].spec.displayName` — отображаемое имя

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `networkInterfaces[]` | да | Object[] | |
| `networkInterfaces[].metadata.name` | да | string | |
| `networkInterfaces[].metadata.namespace` | да | string | |
| `networkInterfaces[].metadata.uid` | при update | string | |
| `networkInterfaces[].metadata.labels` | нет | Object | |
| `networkInterfaces[].metadata.annotations` | нет | Object | |
| `networkInterfaces[].spec.subnetRef` | да | Object | |
| `networkInterfaces[].spec.subnetRef.name` | да | string | |
| `networkInterfaces[].spec.subnetRef.namespace` | да | string | |
| `networkInterfaces[].spec.comment` | нет | string | |
| `networkInterfaces[].spec.description` | нет | string | |
| `networkInterfaces[].spec.displayName` | нет | string | |

## Ограничения

| Поле | Правило | Ошибка |
|------|---------|--------|
| `metadata.name` | обязательное; DNS-label, 1–63 символа, `^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$` | `400 invalid_name` |
| `metadata.namespace` | обязательное; тот же формат что `name` | `400 invalid_namespace` |
| `metadata.uid` | запрещён при upsert; обязателен при update; UUID v4 | `400 invalid_uid` |
| `metadata.labels` | ключ: `^[a-z0-9\-_.\/]{1,63}$`; значение: `^.{0,63}$` | `400 invalid_labels` |
| `metadata.annotations` | ключ: `^[a-z0-9\-_.\/]{1,253}$`; значение: без ограничений | `400 invalid_annotations` |
| `name` + `namespace` | уникальная пара (при upsert) | `409 already_exists` |
| `spec.subnetRef` | обязательное; `{ name, namespace }`; Subnet должен существовать | `404 referenced_resource_not_found` |
| `spec.subnetRef` | неизменяемое (immutable) | `400 immutable_field` |
| `spec.description` | макс. 256 символов | `400 invalid_description` |
| `spec.displayName` | макс. 128 символов | `400 invalid_display_name` |
| `spec.comment` | макс. 256 символов | `400 invalid_comment` |

## Выходные параметры

- `networkInterfaces[]` — массив NetworkInterface
- `networkInterfaces[].metadata` — стандартные метаданные
- `networkInterfaces[].metadata.name` — имя NetworkInterface
- `networkInterfaces[].metadata.namespace` — namespace
- `networkInterfaces[].metadata.uid` — UUID
- `networkInterfaces[].metadata.labels` — метки
- `networkInterfaces[].metadata.annotations` — аннотации
- `networkInterfaces[].metadata.creationTimestamp` — время создания (RFC 3339)
- `networkInterfaces[].metadata.resourceVersion` — версия ресурса
- `networkInterfaces[].spec.subnetRef` — ссылка на Subnet
- `networkInterfaces[].spec.comment` — комментарий
- `networkInterfaces[].spec.description` — описание
- `networkInterfaces[].spec.displayName` — отображаемое имя
- `networkInterfaces[].status.state` — состояние: `created`, `available`
- `networkInterfaces[].status.privateIpAddress` — приватный IP (из AddressBinding с private Address)
- `networkInterfaces[].status.macAddress` — MAC-адрес (генерируется при привязке private Address)
- `networkInterfaces[].status.publicIp` — публичный IP (из AddressBinding с public Address, NAT-маппинг)
- `networkInterfaces[].status.subnetRef` — ссылка на Subnet: `{ name, namespace }` (копия `spec.subnetRef`)
- `networkInterfaces[].status.vpcRef` — ссылка на VPC: `{ name, namespace }` (из Subnet.spec.vpcRef)

| название | тип данных |
|----------|-----------|
| `networkInterfaces[]` | Object[] |
| `networkInterfaces[].metadata` | Object |
| `networkInterfaces[].metadata.name` | string |
| `networkInterfaces[].metadata.namespace` | string |
| `networkInterfaces[].metadata.uid` | string |
| `networkInterfaces[].metadata.labels` | Object |
| `networkInterfaces[].metadata.annotations` | Object |
| `networkInterfaces[].metadata.creationTimestamp` | string |
| `networkInterfaces[].metadata.resourceVersion` | string |
| `networkInterfaces[].spec.subnetRef` | Object |
| `networkInterfaces[].spec.comment` | string |
| `networkInterfaces[].spec.description` | string |
| `networkInterfaces[].spec.displayName` | string |
| `networkInterfaces[].status.state` | enum |
| `networkInterfaces[].status.privateIpAddress` | string |
| `networkInterfaces[].status.macAddress` | string |
| `networkInterfaces[].status.publicIp` | string |
| `networkInterfaces[].status.subnetRef` | Object |
| `networkInterfaces[].status.vpcRef` | Object |

## Пример запроса

```bash
curl 'api:9006/v1/network-interfaces/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "networkInterfaces": [
    {
      "metadata": {
        "name": "web-eni-1",
        "namespace": "tenant-a",
        "labels": {
          "role": "web"
        }
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-a"
        },
        "description": "Primary ENI for web-server-1",
        "displayName": "Web ENI 1"
      }
    }
  ]
}'
```

## Пример ответа

Состояние `created` — ресурс существует, но ещё не привязан ни к одному Address.

```json
{
  "networkInterfaces": [
    {
      "metadata": {
        "name": "web-eni-1",
        "namespace": "tenant-a",
        "uid": "e5f6a7b8-...",
        "labels": {
          "role": "web"
        },
        "creationTimestamp": "2026-03-01T12:04:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-a"
        },
        "description": "Primary ENI for web-server-1",
        "displayName": "Web ENI 1"
      },
      "status": {
        "state": "created",
        "privateIpAddress": "",
        "macAddress": "",
        "publicIp": "",
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-a"
        },
        "vpcRef": null
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании. Поле `spec.subnetRef` является неизменяемым (immutable).

> **Переход состояний:** после создания `AddressBinding` с приватным Address, Status Controller обновляет NI — заполняет `privateIpAddress`, `macAddress`, `subnetRef`, `vpcRef` и переводит `status.state` из `created` в `available`.
