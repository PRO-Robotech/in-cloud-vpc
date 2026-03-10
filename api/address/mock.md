# Address — PoC mock

Данные для PoC-сценария из HV-SETUP.md.

Четыре приватных адреса — по одному на каждый NI. IPAM аллоцирует IP из CIDR Subnet'а.

## Upsert

**Запрос:**

```bash
curl -s 'http://api:9006/v1/addresses/upsert' \
-H 'Content-Type: application/json' \
-d '{
  "addresses": [
    {
      "metadata": {
        "name": "addr-a1",
        "namespace": "tenant-poc"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "addr-b1",
        "namespace": "tenant-poc"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "addr-a2",
        "namespace": "tenant-poc"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "addr-b2",
        "namespace": "tenant-poc"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        }
      }
    }
  ]
}'
```

**Ответ:**

```json
{
  "addresses": [
    {
      "metadata": {
        "name": "addr-a1",
        "namespace": "tenant-poc",
        "uid": "66666666-1111-1111-1111-aaa111111111",
        "creationTimestamp": "2026-03-09T10:00:04Z",
        "resourceVersion": "5"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        }
      },
      "status": {
        "ip": "10.0.0.1",
        "state": "allocated"
      }
    },
    {
      "metadata": {
        "name": "addr-b1",
        "namespace": "tenant-poc",
        "uid": "66666666-1111-1111-1111-bbb111111111",
        "creationTimestamp": "2026-03-09T10:00:04Z",
        "resourceVersion": "5"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        }
      },
      "status": {
        "ip": "10.0.0.3",
        "state": "allocated"
      }
    },
    {
      "metadata": {
        "name": "addr-a2",
        "namespace": "tenant-poc",
        "uid": "66666666-1111-1111-1111-aaa222222222",
        "creationTimestamp": "2026-03-09T10:00:04Z",
        "resourceVersion": "5"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        }
      },
      "status": {
        "ip": "10.0.0.2",
        "state": "allocated"
      }
    },
    {
      "metadata": {
        "name": "addr-b2",
        "namespace": "tenant-poc",
        "uid": "66666666-1111-1111-1111-bbb222222222",
        "creationTimestamp": "2026-03-09T10:00:04Z",
        "resourceVersion": "5"
      },
      "spec": {
        "type": "private",
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        }
      },
      "status": {
        "ip": "10.0.0.4",
        "state": "allocated"
      }
    }
  ]
}
```

## Delete

```bash
curl -s 'http://api:9006/v1/addresses/delete' \
-H 'Content-Type: application/json' \
-d '{
  "addresses": [
    {
      "metadata": {
        "name": "addr-a1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "addr-b1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "addr-a2",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "addr-b2",
        "namespace": "tenant-poc"
      }
    }
  ]
}'
```
