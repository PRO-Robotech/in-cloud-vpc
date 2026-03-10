# Instance — upsert

**Endpoint:** `POST /v1/instances/upsert`

Создание или обновление Instance — виртуальной машины с привязкой к гипервизору. Instance ссылается на NI через `spec.networkInterfacesRef`. Agent watch'ит Instance со своим `hvRef`, разрешает `networkInterfacesRef` → NI → Subnet → VPC и конфигурирует data plane.

## Входные параметры

| название | обязательность | тип данных | значение по умолчанию |
|----------|---------------|------------|----------------------|
| `instances[]` | да | Object[] | |
| `instances[].metadata.name` | да | string | |
| `instances[].metadata.namespace` | да | string | |
| `instances[].metadata.uid` | при update | string | |
| `instances[].metadata.labels` | нет | Object | |
| `instances[].metadata.annotations` | нет | Object | |
| `instances[].spec.hvRef` | да | string | |
| `instances[].spec.networkInterfacesRef` | да | Object[] | |
| `instances[].spec.networkInterfacesRef[].name` | да | string | |
| `instances[].spec.networkInterfacesRef[].namespace` | да | string | |
| `instances[].spec.comment` | нет | string | |
| `instances[].spec.description` | нет | string | |
| `instances[].spec.displayName` | нет | string | |

## Ограничения

| Поле | Правило | Ошибка |
|------|---------|--------|
| `metadata.name` | обязательное; DNS-label, 1–63 символа, `^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$` | `400 invalid_name` |
| `metadata.namespace` | обязательное; тот же формат что `name` | `400 invalid_namespace` |
| `metadata.uid` | запрещён при upsert; обязателен при update; UUID v4 | `400 invalid_uid` |
| `metadata.labels` | ключ: `^[a-z0-9\-_.\/]{1,63}$`; значение: `^.{0,63}$` | `400 invalid_labels` |
| `metadata.annotations` | ключ: `^[a-z0-9\-_.\/]{1,253}$`; значение: без ограничений | `400 invalid_annotations` |
| `name` + `namespace` | уникальная пара (при upsert) | `409 already_exists` |
| `spec.hvRef` | обязательное; имя гипервизора | `400 invalid_hv_ref` |
| `spec.hvRef` | **мутабельное** — меняется при live migration | — |
| `spec.networkInterfacesRef` | обязательное; массив `{name, namespace}`; NI должен существовать | `404 referenced_resource_not_found` |
| `spec.networkInterfacesRef` | **мутабельное** — attach/detach NI | — |
| `spec.description` | макс. 256 символов | `400 invalid_description` |
| `spec.displayName` | макс. 128 символов | `400 invalid_display_name` |
| `spec.comment` | макс. 256 символов | `400 invalid_comment` |

> `spec.hvRef` и `spec.networkInterfacesRef` — обязательные spec-поля. Оба **мутабельные**: при live migration Scheduler или оператор меняет `hvRef` на новый HV; `networkInterfacesRef` позволяет attach/detach NI. Agent на целевом HV получает watch-событие, разрешает `networkInterfacesRef` → NI → Subnet → VPC и применяет конфигурацию.

## Выходные параметры

| название | тип данных |
|----------|-----------|
| `instances[]` | Object[] |
| `instances[].metadata` | Object |
| `instances[].metadata.name` | string |
| `instances[].metadata.namespace` | string |
| `instances[].metadata.uid` | string |
| `instances[].metadata.labels` | Object |
| `instances[].metadata.annotations` | Object |
| `instances[].metadata.creationTimestamp` | string |
| `instances[].metadata.resourceVersion` | string |
| `instances[].spec.hvRef` | string |
| `instances[].spec.networkInterfacesRef` | Object[] |
| `instances[].spec.comment` | string |
| `instances[].spec.description` | string |
| `instances[].spec.displayName` | string |
| `instances[].status.state` | enum |

## Состояния (status.state)

| Состояние | Описание |
|-----------|----------|
| `pending` | Создан, Agent ещё не подтвердил |
| `running` | Agent применил конфигурацию, VM работает |
| `stopped` | VM остановлена |
| `terminated` | VM завершена, ресурс можно удалить |

### Переходы состояний

```
pending ──Agent applied──▶ running
running ──shutdown──▶ stopped
running ──hvRef changed──▶ pending (на новом HV)
stopped ──start──▶ pending → running
running ──terminate──▶ terminated
```

## Пример запроса

```bash
curl 'api:9006/v1/instances/upsert' \
--header 'Content-Type: application/json' \
--data '{
  "instances": [
    {
      "metadata": {
        "name": "web-server-1",
        "namespace": "tenant-a",
        "labels": {
          "role": "web",
          "tier": "frontend"
        }
      },
      "spec": {
        "hvRef": "hv1",
        "networkInterfacesRef": [
          {
            "name": "web-eni-1",
            "namespace": "tenant-a"
          }
        ],
        "description": "Production web server",
        "displayName": "Web Server 1"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "instances": [
    {
      "metadata": {
        "name": "web-server-1",
        "namespace": "tenant-a",
        "uid": "a1b2c3d4-...",
        "labels": {
          "role": "web",
          "tier": "frontend"
        },
        "creationTimestamp": "2026-03-01T12:03:30Z",
        "resourceVersion": "1"
      },
      "spec": {
        "hvRef": "hv1",
        "networkInterfacesRef": [
          {
            "name": "web-eni-1",
            "namespace": "tenant-a"
          }
        ],
        "description": "Production web server",
        "displayName": "Web Server 1"
      },
      "status": {
        "state": "pending"
      }
    }
  ]
}
```

> **Update:** для обновления существующего ресурса необходимо указать `metadata.uid`, полученный при создании. Поля `spec.hvRef` и `spec.networkInterfacesRef` являются мутабельными — `hvRef` меняется при live migration, `networkInterfacesRef` позволяет attach/detach NI.
