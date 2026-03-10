# Gateway — list

**Endpoint:** `POST /v1/gateways/list`

## Пример запроса

```bash
curl 'api:9006/v1/gateways/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-a"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "resourceVersion": "90",
  "gateways": [
    {
      "metadata": {
        "name": "prod-nat",
        "namespace": "tenant-a",
        "uid": "c3d4e5f6-...",
        "creationTimestamp": "2026-03-01T12:02:30Z",
        "resourceVersion": "1"
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
