# RouteTable — list

**Endpoint:** `POST /v1/route-tables/list`

## Пример запроса

```bash
curl 'api:9006/v1/route-tables/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-a",
        "refs": [
          {
            "name": "prod-nat",
            "resType": "Gateway"
          }
        ]
      }
    }
  ]
}'
```

> `refs` позволяет найти все RouteTable, содержащие маршруты через конкретный Gateway или Address.

## Пример ответа

```json
{
  "resourceVersion": "95",
  "routeTables": [
    {
      "metadata": {
        "name": "private-rt",
        "namespace": "tenant-a",
        "uid": "d5e6f7a8-...",
        "creationTimestamp": "2026-03-01T12:03:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "routes": [
          {
            "destinationCidrBlock": "0.0.0.0/0",
            "target": {
              "gatewayRef": {
                "name": "prod-nat",
                "namespace": "tenant-a"
              }
            }
          }
        ],
        "description": "Route table for private subnets"
      }
    }
  ]
}
```
