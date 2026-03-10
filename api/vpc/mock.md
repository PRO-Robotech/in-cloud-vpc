# VPC — PoC mock

Данные для PoC-сценария из HV-SETUP.md.

## Upsert

**Запрос:**

```bash
curl -s 'http://api:9006/v1/vpcs/upsert' \
-H 'Content-Type: application/json' \
-d '{
  "vpcs": [
    {
      "metadata": {
        "name": "vpc-1",
        "namespace": "tenant-poc"
      },
      "spec": {
        "description": "VPC for web tier",
        "displayName": "VPC 1"
      }
    },
    {
      "metadata": {
        "name": "vpc-2",
        "namespace": "tenant-poc"
      },
      "spec": {
        "description": "VPC for DB tier",
        "displayName": "VPC 2"
      }
    }
  ]
}'
```

**Ответ:**

```json
{
  "vpcs": [
    {
      "metadata": {
        "name": "vpc-1",
        "namespace": "tenant-poc",
        "uid": "11111111-1111-1111-1111-111111111111",
        "creationTimestamp": "2026-03-09T10:00:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "description": "VPC for web tier",
        "displayName": "VPC 1"
      },
      "status": {
        "vni": 100001
      }
    },
    {
      "metadata": {
        "name": "vpc-2",
        "namespace": "tenant-poc",
        "uid": "11111111-1111-1111-1111-222222222222",
        "creationTimestamp": "2026-03-09T10:00:00Z",
        "resourceVersion": "1"
      },
      "spec": {
        "description": "VPC for DB tier",
        "displayName": "VPC 2"
      },
      "status": {
        "vni": 100002
      }
    }
  ]
}
```

## List

**Запрос:**

```bash
curl -s 'http://api:9006/v1/vpcs/list' \
-H 'Content-Type: application/json' \
-d '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-poc"
      }
    }
  ]
}'
```

**Ответ:**

```json
{
  "resourceVersion": "8",
  "vpcs": [
    {
      "metadata": {
        "name": "vpc-1",
        "namespace": "tenant-poc",
        "uid": "11111111-1111-1111-1111-111111111111",
        "resourceVersion": "1"
      },
      "spec": {
        "description": "VPC for web tier",
        "displayName": "VPC 1"
      },
      "status": {
        "vni": 100001
      }
    },
    {
      "metadata": {
        "name": "vpc-2",
        "namespace": "tenant-poc",
        "uid": "11111111-1111-1111-1111-222222222222",
        "resourceVersion": "1"
      },
      "spec": {
        "description": "VPC for DB tier",
        "displayName": "VPC 2"
      },
      "status": {
        "vni": 100002
      }
    }
  ]
}
```

## Delete

```bash
curl -s 'http://api:9006/v1/vpcs/delete' \
-H 'Content-Type: application/json' \
-d '{
  "vpcs": [
    {
      "metadata": {
        "name": "vpc-1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "vpc-2",
        "namespace": "tenant-poc"
      }
    }
  ]
}'
```
