# Address — delete

**Endpoint:** `POST /v1/addresses/delete`

## Пример запроса

```bash
curl 'api:9006/v1/addresses/delete' \
--header 'Content-Type: application/json' \
--data '{
  "addresses": [
    {
      "metadata": {
        "name": "web-private-ip",
        "namespace": "tenant-a"
      }
    }
  ]
}'
```
