# AddressBinding — list

**Endpoint:** `POST /v1/address-bindings/list`

## Пример запроса

```bash
curl 'api:9006/v1/address-bindings/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-a",
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

> `refs` позволяет найти все AddressBinding, связанные с конкретным NetworkInterface или Address.

## Пример ответа

```json
{
  "resourceVersion": "85",
  "addressBindings": [
    {
      "metadata": {
        "name": "web-eni-1-private",
        "namespace": "tenant-a",
        "uid": "a9b8c7d6-...",
        "creationTimestamp": "2026-03-01T12:04:30Z",
        "resourceVersion": "1"
      },
      "spec": {
        "addressRef": {
          "name": "web-private-ip",
          "namespace": "tenant-a"
        },
        "networkInterfaceRef": {
          "name": "web-eni-1",
          "namespace": "tenant-a"
        }
      }
    },
    {
      "metadata": {
        "name": "web-eni-1-public",
        "namespace": "tenant-a",
        "uid": "b0c9d8e7-...",
        "creationTimestamp": "2026-03-01T12:05:00Z",
        "resourceVersion": "1"
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
