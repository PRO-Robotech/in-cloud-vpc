# AddressBinding — upsert

**Endpoint:** `POST /v1/address-bindings/upsert`

Создание или обновление связи между Address и NetworkInterface. При привязке приватного Address NI получает IP и MAC, переходит из `created` → `available`. При привязке публичного Address создаётся 1:1 NAT-маппинг.

## Входные параметры

- `addressBindings[]` — массив ресурсов AddressBinding
- `addressBindings[].metadata.name` — уникальное имя AddressBinding в namespace
- `addressBindings[].metadata.namespace` — namespace (изоляция tenant'а)
- `addressBindings[].metadata.uid` — UUID; сервер генерирует при upsert; обязателен при update
- `addressBindings[].metadata.labels` — метки для labelSelector
- `addressBindings[].metadata.annotations` — произвольные аннотации
- `addressBindings[].spec.addressRef` — ссылка на Address: `{ name, namespace }`
- `addressBindings[].spec.networkInterfaceRef` — ссылка на NetworkInterface: `{ name, namespace }`
- `addressBindings[].spec.comment` — комментарий
- `addressBindings[].spec.description` — описание
- `addressBindings[].spec.displayName` — отображаемое имя

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `addressBindings[]` | да | Object[] | |
| `addressBindings[].metadata.name` | да | string | |
| `addressBindings[].metadata.namespace` | да | string | |
| `addressBindings[].metadata.uid` | при update | string | |
| `addressBindings[].metadata.labels` | нет | Object | |
| `addressBindings[].metadata.annotations` | нет | Object | |
| `addressBindings[].spec.addressRef` | да | Object | |
| `addressBindings[].spec.addressRef.name` | да | string | |
| `addressBindings[].spec.addressRef.namespace` | да | string | |
| `addressBindings[].spec.networkInterfaceRef` | да | Object | |
| `addressBindings[].spec.networkInterfaceRef.name` | да | string | |
| `addressBindings[].spec.networkInterfaceRef.namespace` | да | string | |
| `addressBindings[].spec.comment` | нет | string | |
| `addressBindings[].spec.description` | нет | string | |
| `addressBindings[].spec.displayName` | нет | string | |

## Ограничения

| Поле | Правило | Ошибка |
|------|---------|--------|
| `metadata.name` | обязательное; DNS-label, 1–63 символа, `^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$` | `400 invalid_name` |
| `metadata.namespace` | обязательное; тот же формат что `name` | `400 invalid_namespace` |
| `metadata.uid` | запрещён при upsert; обязателен при update; UUID v4 | `400 invalid_uid` |
| `metadata.labels` | ключ: `^[a-z0-9\-_.\/]{1,63}$`; значение: `^.{0,63}$` | `400 invalid_labels` |
| `metadata.annotations` | ключ: `^[a-z0-9\-_.\/]{1,253}$`; значение: без ограничений | `400 invalid_annotations` |
| `name` + `namespace` | уникальная пара (при upsert) | `409 already_exists` |
| `spec.addressRef` | обязательное; `{ name, namespace }`; Address должен существовать | `404 referenced_resource_not_found` |
| `spec.addressRef` | Address не должен быть уже привязан к другому NI | `409 already_bound` |
| `spec.addressRef` | неизменяемое (immutable) | `400 immutable_field` |
| `spec.networkInterfaceRef` | обязательное; `{ name, namespace }`; NI должен существовать | `404 referenced_resource_not_found` |
| `spec.networkInterfaceRef` | неизменяемое (immutable) | `400 immutable_field` |
| `spec.networkInterfaceRef` (private Address) | у NI не должно быть уже привязанного private Address | `409 already_bound` |
| `spec.networkInterfaceRef` (public Address) | NI должен быть в состоянии `available` (уже имеет private Address) | `400 network_interface_not_available` |
| `spec.description` | макс. 256 символов | `400 invalid_description` |
| `spec.displayName` | макс. 128 символов | `400 invalid_display_name` |
| `spec.comment` | макс. 256 символов | `400 invalid_comment` |

## Выходные параметры

- `addressBindings[]` — массив AddressBinding
- `addressBindings[].metadata` — стандартные метаданные
- `addressBindings[].metadata.name` — имя AddressBinding
- `addressBindings[].metadata.namespace` — namespace
- `addressBindings[].metadata.uid` — UUID
- `addressBindings[].metadata.labels` — метки
- `addressBindings[].metadata.annotations` — аннотации
- `addressBindings[].metadata.creationTimestamp` — время создания (RFC 3339)
- `addressBindings[].metadata.resourceVersion` — версия ресурса
- `addressBindings[].spec.addressRef` — ссылка на Address
- `addressBindings[].spec.networkInterfaceRef` — ссылка на NetworkInterface
- `addressBindings[].spec.comment` — комментарий
- `addressBindings[].spec.description` — описание
- `addressBindings[].spec.displayName` — отображаемое имя

| название | тип данных |
|----------|-----------|
| `addressBindings[]` | Object[] |
| `addressBindings[].metadata` | Object |
| `addressBindings[].metadata.name` | string |
| `addressBindings[].metadata.namespace` | string |
| `addressBindings[].metadata.uid` | string |
| `addressBindings[].metadata.labels` | Object |
| `addressBindings[].metadata.annotations` | Object |
| `addressBindings[].metadata.creationTimestamp` | string |
| `addressBindings[].metadata.resourceVersion` | string |
| `addressBindings[].spec.addressRef` | Object |
| `addressBindings[].spec.networkInterfaceRef` | Object |
| `addressBindings[].spec.comment` | string |
| `addressBindings[].spec.description` | string |
| `addressBindings[].spec.displayName` | string |

## Пример запроса (привязка приватного IP к NI)

```bash
curl 'api:9006/v1/address-bindings/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "addressBindings": [
    {
      "metadata": {
        "name": "web-eni-1-private",
        "namespace": "tenant-a"
      },
      "spec": {
        "addressRef": {
          "name": "web-private-ip",
          "namespace": "tenant-a"
        },
        "networkInterfaceRef": {
          "name": "web-eni-1",
          "namespace": "tenant-a"
        }
      }
    }
  ]
}'
```

## Пример запроса (привязка публичного IP к NI — NAT-маппинг)

```bash
curl 'api:9006/v1/address-bindings/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "addressBindings": [
    {
      "metadata": {
        "name": "web-eni-1-public",
        "namespace": "tenant-a"
      },
      "spec": {
        "addressRef": {
          "name": "web-eip-1",
          "namespace": "tenant-a"
        },
        "networkInterfaceRef": {
          "name": "web-eni-1",
          "namespace": "tenant-a"
        }
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "addressBindings": [
    {
      "metadata": {
        "name": "web-eni-1-private",
        "namespace": "tenant-a",
        "uid": "a9b8c7d6-...",
        "creationTimestamp": "2026-03-01T12:04:30Z",
        "resourceVersion": "1"
      },
      "spec": {
        "addressRef": {
          "name": "web-private-ip",
          "namespace": "tenant-a"
        },
        "networkInterfaceRef": {
          "name": "web-eni-1",
          "namespace": "tenant-a"
        }
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании. Поля `spec.addressRef` и `spec.networkInterfaceRef` являются неизменяемыми (immutable).

> **Побочный эффект:** после создания AddressBinding с приватным Address, Status Controller обновляет NI — заполняет `privateIpAddress`, `macAddress`, `subnetRef`, `vpcRef` и переводит `status.state` из `created` в `available`.
