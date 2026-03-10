# NetworkInterface — watch

**Endpoint:** `POST /v1/network-interfaces/watch`

## Пример запроса

```bash
curl 'api:9006/v1/network-interfaces/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "80",
  "selectors": [
    {
      "labelSelector": {
        "role": "web"
      }
    }
  ]
}'
```

## Формат события

```json
{
  "type": "modify",
  "networkInterfaces": [
    {
      "metadata": {
        "name": "web-eni-1",
        "namespace": "tenant-a",
        "uid": "e5f6a7b8-...",
        "resourceVersion": "81"
      },
      "spec": {
        "subnetRef": {
          "name": "web-subnet",
          "namespace": "tenant-a"
        },
        "description": "Primary ENI for web-server-1",
        "displayName": "Web ENI 1"
      },
      "status": {
        "state": "available",
        "privateIpAddress": "10.0.1.12",
        "macAddress": "02:42:0a:00:01:0c",
        "publicIp": "203.0.113.42",
        "subnetRef": {
          "name": "web-subnet",
          "namespace": "tenant-a"
        },
        "vpcRef": {
          "name": "prod-vpc",
          "namespace": "tenant-a"
        }
      }
    }
  ]
}
```
