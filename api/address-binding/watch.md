# AddressBinding — watch

**Endpoint:** `POST /v1/address-bindings/watch`

## Пример запроса

```bash
curl 'api:9006/v1/address-bindings/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "85",
  "selectors": [
    {
      "fieldSelector": {
        "refs": [
          {
            "name": "web-eni-1",
            "resType": "NetworkInterface"
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
  "type": "deleted",
  "addressBindings": [
    {
      "metadata": {
        "name": "web-eni-1-public",
        "namespace": "tenant-a",
        "uid": "b0c9d8e7-...",
        "resourceVersion": "86"
      },
      "spec": {
        "addressRef": {
          "name": "web-eip-1",
          "namespace": "tenant-a"
        },
        "networkInterfaceRef": {
          "name": "web-eni-1",
          "namespace": "tenant-a"
        }
      }
    }
  ]
}
```
