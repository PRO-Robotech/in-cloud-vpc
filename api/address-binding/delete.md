# AddressBinding — delete

**Endpoint:** `POST /v1/address-bindings/delete`

## Пример запроса

```bash
curl 'api:9006/v1/address-bindings/delete' \
--header 'Content-Type: application/json' \
--data '{
  "addressBindings": [
    {
      "metadata": {
        "name": "web-eni-1-private",
        "namespace": "tenant-a"
      }
    }
  ]
}'
```
