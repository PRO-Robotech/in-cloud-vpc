# RouteTableBinding — watch

**Endpoint:** `POST /v1/route-table-bindings/watch`

## Пример запроса

```bash
curl 'api:9006/v1/route-table-bindings/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "100",
  "selectors": [
    {
      "fieldSelector": {
        "refs": [
          {
            "name": "web-subnet",
            "resType": "Subnet"
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
  "type": "add",
  "routeTableBindings": [
    {
      "metadata": {
        "name": "web-subnet-rt-binding",
        "namespace": "tenant-a",
        "uid": "f6a7b8c9-...",
        "resourceVersion": "101"
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
    }
  ]
}
```
