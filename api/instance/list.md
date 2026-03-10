# Instance — list

**Endpoint:** `POST /v1/instances/list`

## Пример запроса

```bash
curl 'api:9006/v1/instances/list' \
--header 'Content-Type: application/json' \
--data '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-a",
        "hvRef": "hv1"
      }
    }
  ]
}'
```

## Пример ответа

```json
{
  "resourceVersion": "75",
  "instances": [
    {
      "metadata": {
        "name": "web-server-1",
        "namespace": "tenant-a",
        "uid": "a1b2c3d4-...",
        "labels": {
          "role": "web",
          "tier": "frontend"
        },
        "creationTimestamp": "2026-03-01T12:03:30Z",
        "resourceVersion": "2"
      },
      "spec": {
        "hvRef": "hv1",
        "networkInterfacesRef": [
          {
            "name": "web-eni-1",
            "namespace": "tenant-a"
          }
        ],
        "description": "Production web server",
        "displayName": "Web Server 1"
      },
      "status": {
        "state": "running"
      }
    }
  ]
}
```
