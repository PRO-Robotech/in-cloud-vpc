# Gateway — watch

**Endpoint:** `POST /v1/gateways/watch`

## Пример запроса

```bash
curl 'api:9006/v1/gateways/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "90"
}'
```

## Формат события

```json
{
  "type": "modify",
  "gateways": [
    {
      "metadata": {
        "name": "prod-nat",
        "namespace": "tenant-a",
        "uid": "c3d4e5f6-...",
        "resourceVersion": "91"
      },
      "spec": {
        "description": "NAT Gateway for private subnets"
      },
      "status": {
        "state": "available",
        "publicIp": "203.0.113.10"
      }
    }
  ]
}
```
