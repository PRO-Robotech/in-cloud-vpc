# Namespace — list

**Endpoint:** `POST /v1/namespaces/list`

## Пример запроса

```bash
curl 'api:9006/v1/namespaces/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "name": "my-namespace"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "resourceVersion": "10",
  "namespaces": [
    {
      "metadata": {
        "name": "my-namespace",
        "uid": "ns-a1b2c3d4-...",
        "creationTimestamp": "2026-03-01T12:00:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "displayName": "My Namespace",
        "description": "Namespace description"
      },
      "status": {
        "state": "active"
      }
    }
  ]
}
```
