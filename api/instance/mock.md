# Instance — PoC mock

Данные для PoC-сценария из API-MOCK.md.

## Upsert

Четыре Instance — каждый ссылается на свой NI через `spec.networkInterfacesRef`. Instance создаётся после того, как сеть полностью готова (NI в `available`).

**Запрос:**

```bash
curl -s 'http://api:9006/v1/instances/upsert' \
-H 'Content-Type: application/json' \
-d '{
  "instances": [
    {
      "metadata": {
        "name": "instance-a1",
        "namespace": "tenant-poc",
        "labels": {
          "vpc": "vpc-1"
        }
      },
      "spec": {
        "hvRef": "hv1",
        "displayName": "Web server 1",
        "networkInterfacesRef": [
          { "name": "eni-a1", "namespace": "tenant-poc" }
        ]
      }
    },
    {
      "metadata": {
        "name": "instance-b1",
        "namespace": "tenant-poc",
        "labels": {
          "vpc": "vpc-2"
        }
      },
      "spec": {
        "hvRef": "hv1",
        "displayName": "DB server 1",
        "networkInterfacesRef": [
          { "name": "eni-b1", "namespace": "tenant-poc" }
        ]
      }
    },
    {
      "metadata": {
        "name": "instance-a2",
        "namespace": "tenant-poc",
        "labels": {
          "vpc": "vpc-1"
        }
      },
      "spec": {
        "hvRef": "hv2",
        "displayName": "Web server 2",
        "networkInterfacesRef": [
          { "name": "eni-a2", "namespace": "tenant-poc" }
        ]
      }
    },
    {
      "metadata": {
        "name": "instance-b2",
        "namespace": "tenant-poc",
        "labels": {
          "vpc": "vpc-2"
        }
      },
      "spec": {
        "hvRef": "hv2",
        "displayName": "DB server 2",
        "networkInterfacesRef": [
          { "name": "eni-b2", "namespace": "tenant-poc" }
        ]
      }
    }
  ]
}'
```

**Ответ:**

```json
{
  "instances": [
    {
      "metadata": {
        "name": "instance-a1",
        "namespace": "tenant-poc",
        "uid": "44444444-1111-1111-1111-aaa111111111",
        "labels": {
          "vpc": "vpc-1"
        },
        "creationTimestamp": "2026-03-09T10:00:06Z",
        "resourceVersion": "7"
      },
      "spec": {
        "hvRef": "hv1",
        "displayName": "Web server 1",
        "networkInterfacesRef": [
          { "name": "eni-a1", "namespace": "tenant-poc" }
        ]
      },
      "status": {
        "state": "pending"
      }
    },
    {
      "metadata": {
        "name": "instance-b1",
        "namespace": "tenant-poc",
        "uid": "44444444-1111-1111-1111-bbb111111111",
        "labels": {
          "vpc": "vpc-2"
        },
        "creationTimestamp": "2026-03-09T10:00:06Z",
        "resourceVersion": "7"
      },
      "spec": {
        "hvRef": "hv1",
        "displayName": "DB server 1",
        "networkInterfacesRef": [
          { "name": "eni-b1", "namespace": "tenant-poc" }
        ]
      },
      "status": {
        "state": "pending"
      }
    },
    {
      "metadata": {
        "name": "instance-a2",
        "namespace": "tenant-poc",
        "uid": "44444444-1111-1111-1111-aaa222222222",
        "labels": {
          "vpc": "vpc-1"
        },
        "creationTimestamp": "2026-03-09T10:00:06Z",
        "resourceVersion": "7"
      },
      "spec": {
        "hvRef": "hv2",
        "displayName": "Web server 2",
        "networkInterfacesRef": [
          { "name": "eni-a2", "namespace": "tenant-poc" }
        ]
      },
      "status": {
        "state": "pending"
      }
    },
    {
      "metadata": {
        "name": "instance-b2",
        "namespace": "tenant-poc",
        "uid": "44444444-1111-1111-1111-bbb222222222",
        "labels": {
          "vpc": "vpc-2"
        },
        "creationTimestamp": "2026-03-09T10:00:06Z",
        "resourceVersion": "7"
      },
      "spec": {
        "hvRef": "hv2",
        "displayName": "DB server 2",
        "networkInterfacesRef": [
          { "name": "eni-b2", "namespace": "tenant-poc" }
        ]
      },
      "status": {
        "state": "pending"
      }
    }
  ]
}
```

## List (Instance на HV1)

```bash
curl -s 'http://api:9006/v1/instances/list' \
-H 'Content-Type: application/json' \
-d '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-poc",
        "hvRef": "hv1"
      }
    }
  ]
}'
```

**Ответ:**

```json
{
  "resourceVersion": "8",
  "instances": [
    {
      "metadata": {
        "name": "instance-a1",
        "namespace": "tenant-poc",
        "uid": "44444444-1111-1111-1111-aaa111111111",
        "labels": {
          "vpc": "vpc-1"
        },
        "resourceVersion": "7"
      },
      "spec": {
        "hvRef": "hv1",
        "displayName": "Web server 1",
        "networkInterfacesRef": [
          { "name": "eni-a1", "namespace": "tenant-poc" }
        ]
      },
      "status": {
        "state": "pending"
      }
    },
    {
      "metadata": {
        "name": "instance-b1",
        "namespace": "tenant-poc",
        "uid": "44444444-1111-1111-1111-bbb111111111",
        "labels": {
          "vpc": "vpc-2"
        },
        "resourceVersion": "7"
      },
      "spec": {
        "hvRef": "hv1",
        "displayName": "DB server 1",
        "networkInterfacesRef": [
          { "name": "eni-b1", "namespace": "tenant-poc" }
        ]
      },
      "status": {
        "state": "pending"
      }
    }
  ]
}
```

## Delete

```bash
curl -s 'http://api:9006/v1/instances/delete' \
-H 'Content-Type: application/json' \
-d '{
  "instances": [
    {
      "metadata": {
        "name": "instance-a1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "instance-a2",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "instance-b1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "instance-b2",
        "namespace": "tenant-poc"
      }
    }
  ]
}'
```
