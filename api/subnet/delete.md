# Subnet — delete

**Endpoint:** `POST /v1/subnets/delete`

## Пример запроса

```bash
curl 'api:9006/v1/subnets/delete' \
--header 'Content-Type: application/json' \
--data '{
  "subnets": [
    {
      "metadata": {
        "name": "web-subnet",
        "namespace": "tenant-a"
      }
    }
  ]
}'
```
