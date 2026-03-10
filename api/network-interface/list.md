# NetworkInterface — list

**Endpoint:** `POST /v1/network-interfaces/list`

## Пример запроса

```bash
curl 'api:9006/v1/network-interfaces/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-a",
        "refs": [
          {
            "name": "web-subnet",
            "resType": "Subnet"
          }
        ]
      },
      "labelSelector": {
        "role": "web"
      }
    }
  ]
}'
```

> `refs` для NetworkInterface фильтрует по `spec.subnetRef` / `status.vpcRef`.

## Пример ответа

```json
{
  "resourceVersion": "80",
  "networkInterfaces": [
    {
      "metadata": {
        "name": "web-eni-1",
        "namespace": "tenant-a",
        "uid": "e5f6a7b8-...",
        "labels": {
          "role": "web"
        },
        "creationTimestamp": "2026-03-01T12:04:00Z",
        "resourceVersion": "3"
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
