# Namespace — PoC mock

Данные для PoC-сценария из HV-SETUP.md.

## Upsert

**Запрос:**

```bash
curl -s 'http://api:9006/v1/namespaces/upsert' \
-H 'Content-Type: application/json' \
-d '{
  "namespaces": [
    {
      "metadata": {
        "name": "tenant-poc"
      },
      "spec": {
        "displayName": "PoC Tenant",
        "description": "Namespace for proof-of-concept"
      }
    }
  ]
}'
```

**Ответ:**

```json
{
  "namespaces": [
    {
      "metadata": {
        "name": "tenant-poc",
        "uid": "ns-00000000-0000-0000-0000-000000000001",
        "creationTimestamp": "2026-03-01T10:00:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "displayName": "PoC Tenant",
        "description": "Namespace for proof-of-concept"
      },
      "status": {
        "state": "active"
      }
    }
  ]
}
```

## Delete

```bash
curl -s 'http://api:9006/v1/namespaces/delete' \
-H 'Content-Type: application/json' \
-d '{
  "namespaces": [
    {
      "metadata": {
        "name": "tenant-poc"
      }
    }
  ]
}'
```
