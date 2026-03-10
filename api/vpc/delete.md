# VPC — delete

**Endpoint:** `POST /v1/vpcs/delete`

## Пример запроса

```bash
curl 'api:9006/v1/vpcs/delete' \
--header 'Content-Type: application/json' \
--data '{
  "vpcs": [
    {
      "metadata": {
        "name": "prod-vpc",
        "namespace": "tenant-a"
      }
    }
  ]
}'
```
