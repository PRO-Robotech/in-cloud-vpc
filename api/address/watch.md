# Address — watch

**Endpoint:** `POST /v1/addresses/watch`

## Пример запроса

```bash
curl 'api:9006/v1/addresses/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "70",
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
  "addresses": [
    {
      "metadata": {
        "name": "web-private-ip-2",
        "namespace": "tenant-a",
        "uid": "f3a4b5c6-...",
        "resourceVersion": "71"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "web-subnet",
          "namespace": "tenant-a"
        }
      },
      "status": {
        "ip": "10.0.1.13",
        "state": "allocated"
      }
    }
  ]
}
```
