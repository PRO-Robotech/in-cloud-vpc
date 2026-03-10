# VPC — list

**Endpoint:** `POST /v1/vpcs/list`

## Пример запроса

```bash
curl 'api:9006/v1/vpcs/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "name": "prod-vpc",
        "namespace": "tenant-a"
      },
      "labelSelector": {
        "env": "prod"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "resourceVersion": "55",
  "vpcs": [
    {
      "metadata": {
        "name": "prod-vpc",
        "namespace": "tenant-a",
        "uid": "a1b2c3d4-...",
        "labels": {
          "env": "prod"
        },
        "creationTimestamp": "2026-03-01T12:00:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "description": "Production VPC",
        "displayName": "Prod VPC"
      },
      "status": {
        "vni": 100001
      }
    }
  ]
}
```
