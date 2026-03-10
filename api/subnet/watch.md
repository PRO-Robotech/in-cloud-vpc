# Subnet — watch

**Endpoint:** `POST /v1/subnets/watch`

## Пример запроса

```bash
curl 'api:9006/v1/subnets/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "60",
  "selectors": [
    {
      "labelSelector": {
        "tier": "web"
      }
    }
  ]
}'
```

## Формат события

```json
{
  "type": "modify",
  "subnets": [
    {
      "metadata": {
        "name": "web-subnet",
        "namespace": "tenant-a",
        "uid": "b2c3d4e5-...",
        "resourceVersion": "61"
      },
      "spec": {
        "cidrBlock": "10.0.1.0/24",
        "vpcRef": {
          "name": "prod-vpc",
          "namespace": "tenant-a"
        },
        "displayName": "Web tier subnet"
      },
      "status": {
        "availableIpCount": 250,
        "vni": 100001
      }
    }
  ]
}
```
