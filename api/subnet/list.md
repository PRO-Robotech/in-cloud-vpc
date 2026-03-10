# Subnet — list

**Endpoint:** `POST /v1/subnets/list`

## Пример запроса

```bash
curl 'api:9006/v1/subnets/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-a"
      },
      "labelSelector": {
        "tier": "web"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "resourceVersion": "60",
  "subnets": [
    {
      "metadata": {
        "name": "web-subnet",
        "namespace": "tenant-a",
        "uid": "b2c3d4e5-...",
        "labels": {
          "tier": "web"
        },
        "creationTimestamp": "2026-03-01T12:01:00Z",
        "resourceVersion": "1"
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
        "availableIpCount": 251,
        "vni": 100001
      }
    }
  ]
}
```
