# RouteTableBinding — delete

**Endpoint:** `POST /v1/route-table-bindings/delete`

## Пример запроса

```bash
curl 'api:9006/v1/route-table-bindings/delete' \
--header 'Content-Type: application/json' \
--data '{
  "routeTableBindings": [
    {
      "metadata": {
        "name": "web-subnet-rt-binding",
        "namespace": "tenant-a"
      }
    }
  ]
}'
```
