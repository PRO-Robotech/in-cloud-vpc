# Instance — delete

**Endpoint:** `POST /v1/instances/delete`

## Пример запроса

```bash
curl 'api:9006/v1/instances/delete' \
--header 'Content-Type: application/json' \
--data '{
  "instances": [
    {
      "metadata": {
        "name": "web-server-1",
        "namespace": "tenant-a"
      }
    }
  ]
}'
```
