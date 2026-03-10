# RouteTable — delete

**Endpoint:** `POST /v1/route-tables/delete`

## Пример запроса

```bash
curl 'api:9006/v1/route-tables/delete' \
--header 'Content-Type: application/json' \
--data '{
  "routeTables": [
    {
      "metadata": {
        "name": "private-rt",
        "namespace": "tenant-a"
      }
    }
  ]
}'
```
