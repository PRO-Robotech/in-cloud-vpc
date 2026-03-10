# Subnet — PoC mock

Данные для PoC-сценария из HV-SETUP.md.

## Upsert

**Запрос:**

```bash
curl -s 'http://api:9006/v1/subnets/upsert' \
-H 'Content-Type: application/json' \
-d '{
  "subnets": [
    {
      "metadata": {
        "name": "subnet-1",
        "namespace": "tenant-poc"
      },
      "spec": {
        "cidrBlock": "10.0.0.0/24",
        "vpcRef": {
          "name": "vpc-1",
          "namespace": "tenant-poc"
        },
        "displayName": "Subnet VPC-1"
      }
    },
    {
      "metadata": {
        "name": "subnet-2",
        "namespace": "tenant-poc"
      },
      "spec": {
        "cidrBlock": "10.0.0.0/24",
        "vpcRef": {
          "name": "vpc-2",
          "namespace": "tenant-poc"
        },
        "displayName": "Subnet VPC-2"
      }
    }
  ]
}'
```

**Ответ:**

```json
{
  "subnets": [
    {
      "metadata": {
        "name": "subnet-1",
        "namespace": "tenant-poc",
        "uid": "22222222-1111-1111-1111-111111111111",
        "creationTimestamp": "2026-03-09T10:00:01Z",
        "resourceVersion": "2"
      },
      "spec": {
        "cidrBlock": "10.0.0.0/24",
        "vpcRef": {
          "name": "vpc-1",
          "namespace": "tenant-poc"
        },
        "displayName": "Subnet VPC-1"
      },
      "status": {
        "availableIpCount": 251,
        "vni": 100001
      }
    },
    {
      "metadata": {
        "name": "subnet-2",
        "namespace": "tenant-poc",
        "uid": "22222222-1111-1111-1111-222222222222",
        "creationTimestamp": "2026-03-09T10:00:01Z",
        "resourceVersion": "2"
      },
      "spec": {
        "cidrBlock": "10.0.0.0/24",
        "vpcRef": {
          "name": "vpc-2",
          "namespace": "tenant-poc"
        },
        "displayName": "Subnet VPC-2"
      },
      "status": {
        "availableIpCount": 251,
        "vni": 100002
      }
    }
  ]
}
```

## Delete

```bash
curl -s 'http://api:9006/v1/subnets/delete' \
-H 'Content-Type: application/json' \
-d '{
  "subnets": [
    {
      "metadata": {
        "name": "subnet-1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "subnet-2",
        "namespace": "tenant-poc"
      }
    }
  ]
}'
```
