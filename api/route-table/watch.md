# RouteTable — watch

**Endpoint:** `POST /v1/route-tables/watch`

## Пример запроса

```bash
curl 'api:9006/v1/route-tables/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "95",
  "selectors": [
    {
      "fieldSelector": {
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

## Формат события

```json
{
  "type": "modify",
  "routeTables": [
    {
      "metadata": {
        "name": "private-rt",
        "namespace": "tenant-a",
        "uid": "d5e6f7a8-...",
        "resourceVersion": "96"
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
          },
          {
            "destinationCidrBlock": "10.1.0.0/16",
            "target": {
              "addressRef": {
                "name": "peering-ip",
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
