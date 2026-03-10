# Namespace — delete

**Endpoint:** `POST /v1/namespaces/delete`

## Пример запроса

```bash
curl 'api:9006/v1/namespaces/delete' \
--header 'Content-Type: application/json' \
--data '{
  "namespaces": [
    {
      "metadata": {
        "name": "my-namespace"
      }
    }
  ]
}'
```
