# VPC — watch

**Endpoint:** `POST /v1/vpcs/watch`

## Пример запроса

```bash
curl 'api:9006/v1/vpcs/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "55",
  "selectors": [
    {
      "labelSelector": {
        "env": "prod"
      }
    }
  ]
}'
```

## Формат события

```json
{
  "type": "modify",
  "vpcs": [
    {
      "metadata": {
        "name": "prod-vpc",
        "namespace": "tenant-a",
        "uid": "a1b2c3d4-...",
        "resourceVersion": "56"
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
