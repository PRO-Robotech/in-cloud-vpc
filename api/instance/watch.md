# Instance — watch

**Endpoint:** `POST /v1/instances/watch`

## Пример запроса

```bash
curl 'api:9006/v1/instances/watch' \
--header 'Content-Type: application/json' \
--data '{
  "resourceVersion": "75",
  "selectors": [
    {
      "fieldSelector": {
        "hvRef": "hv1"
      }
    }
  ]
}'
```

## Формат события

```json
{
  "type": "add",
  "instances": [
    {
      "metadata": {
        "name": "db-server-1",
        "namespace": "tenant-a",
        "uid": "b2c3d4e5-...",
        "resourceVersion": "76"
      },
      "spec": {
        "hvRef": "hv1",
        "networkInterfacesRef": [
          {
            "name": "db-eni-1",
            "namespace": "tenant-a"
          }
        ],
        "displayName": "DB Server 1"
      },
      "status": {
        "state": "pending"
      }
    }
  ]
}
```
