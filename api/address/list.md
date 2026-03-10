# Address — list

**Endpoint:** `POST /v1/addresses/list`

## Пример запроса

```bash
curl 'api:9006/v1/addresses/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-a",
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

> `refs` с `resType: "Subnet"` — найти все Address, привязанные к конкретной подсети.

## Пример ответа

```json
{
  "resourceVersion": "70",
  "addresses": [
    {
      "metadata": {
        "name": "web-private-ip",
        "namespace": "tenant-a",
        "uid": "e2f3a4b5-...",
        "creationTimestamp": "2026-03-01T12:02:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "web-subnet",
          "namespace": "tenant-a"
        },
        "description": "Private IP for web server"
      },
      "status": {
        "ip": "10.0.1.12",
        "state": "allocated"
      }
    }
  ]
}
```
