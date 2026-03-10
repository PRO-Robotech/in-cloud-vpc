# Namespace — watch

**Endpoint:** `POST /v1/namespaces/watch`

## Пример запроса

```bash
curl 'api:9006/v1/namespaces/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "10",
  "selectors": [
    {
      "fieldSelector": {
        "name": "my-namespace"
      }
    }
  ]
}'
```

## Формат события

```json
{
  "type": "modify",
  "namespaces": [
    {
      "metadata": {
        "name": "my-namespace",
        "uid": "ns-a1b2c3d4-...",
        "resourceVersion": "11"
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
