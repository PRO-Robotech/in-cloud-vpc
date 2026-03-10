# NetworkInterface — delete

**Endpoint:** `POST /v1/network-interfaces/delete`

## Пример запроса

```bash
curl 'api:9006/v1/network-interfaces/delete' \
--header 'Content-Type: application/json' \
--data '{
  "networkInterfaces": [
    {
      "metadata": {
        "name": "web-eni-1",
        "namespace": "tenant-a"
      }
    }
  ]
}'
```
