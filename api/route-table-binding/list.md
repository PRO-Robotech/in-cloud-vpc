# RouteTableBinding — list

**Endpoint:** `POST /v1/route-table-bindings/list`

## Пример запроса

```bash
curl 'api:9006/v1/route-table-bindings/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-a",
        "refs": [
          {
            "name": "private-rt",
            "resType": "RouteTable"
          }
        ]
      }
    }
  ]
}'
```

> `refs` позволяет найти все RouteTableBinding, связанные с конкретным RouteTable, Subnet или VPC.

## Пример ответа

```json
{
  "resourceVersion": "100",
  "routeTableBindings": [
    {
      "metadata": {
        "name": "web-subnet-rt-binding",
        "namespace": "tenant-a",
        "uid": "f6a7b8c9-...",
        "creationTimestamp": "2026-03-01T12:05:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "routeTableRef": {
          "name": "private-rt",
          "namespace": "tenant-a"
        },
        "target": {
          "type": "subnet",
          "ref": {
            "name": "web-subnet",
            "namespace": "tenant-a"
          }
        }
      }
    },
    {
      "metadata": {
        "name": "prod-vpc-default-rt",
        "namespace": "tenant-a",
        "uid": "a7b8c9d0-...",
        "creationTimestamp": "2026-03-01T12:05:30Z",
        "resourceVersion": "1"
      },
      "spec": {
        "routeTableRef": {
          "name": "private-rt",
          "namespace": "tenant-a"
        },
        "target": {
          "type": "vpc",
          "ref": {
            "name": "prod-vpc",
            "namespace": "tenant-a"
          }
        }
      }
    }
  ]
}
```
