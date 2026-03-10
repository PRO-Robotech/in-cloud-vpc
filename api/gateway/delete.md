# Gateway — delete

**Endpoint:** `POST /v1/gateways/delete`

## Пример запроса

```bash
curl 'api:9006/v1/gateways/delete' \
--header 'Content-Type: application/json' \
--data '{
  "gateways": [
    {
      "metadata": {
        "name": "prod-nat",
        "namespace": "tenant-a"
      }
    }
  ]
}'
```
