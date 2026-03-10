# AddressBinding — PoC mock

Данные для PoC-сценария из HV-SETUP.md.

Четыре привязки Address ↔ NI. После создания AddressBinding Status Controller обновляет NI — заполняет IP, MAC, переводит в `available`.

## Upsert

**Запрос:**

```bash
curl -s 'http://api:9006/v1/address-bindings/upsert' \
-H 'Content-Type: application/json' \
-d '{
  "addressBindings": [
    {
      "metadata": {
        "name": "bind-eni-a1",
        "namespace": "tenant-poc"
      },
      "spec": {
        "addressRef": {
          "name": "addr-a1",
          "namespace": "tenant-poc"
        },
        "networkInterfaceRef": {
          "name": "eni-a1",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "bind-eni-b1",
        "namespace": "tenant-poc"
      },
      "spec": {
        "addressRef": {
          "name": "addr-b1",
          "namespace": "tenant-poc"
        },
        "networkInterfaceRef": {
          "name": "eni-b1",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "bind-eni-a2",
        "namespace": "tenant-poc"
      },
      "spec": {
        "addressRef": {
          "name": "addr-a2",
          "namespace": "tenant-poc"
        },
        "networkInterfaceRef": {
          "name": "eni-a2",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "bind-eni-b2",
        "namespace": "tenant-poc"
      },
      "spec": {
        "addressRef": {
          "name": "addr-b2",
          "namespace": "tenant-poc"
        },
        "networkInterfaceRef": {
          "name": "eni-b2",
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
  "addressBindings": [
    {
      "metadata": {
        "name": "bind-eni-a1",
        "namespace": "tenant-poc",
        "uid": "77777777-1111-1111-1111-aaa111111111",
        "creationTimestamp": "2026-03-09T10:00:05Z",
        "resourceVersion": "6"
      },
      "spec": {
        "addressRef": {
          "name": "addr-a1",
          "namespace": "tenant-poc"
        },
        "networkInterfaceRef": {
          "name": "eni-a1",
          "namespace": "tenant-poc"
        }
      },
      "status": {
        "state": "bound"
      }
    },
    {
      "metadata": {
        "name": "bind-eni-b1",
        "namespace": "tenant-poc",
        "uid": "77777777-1111-1111-1111-bbb111111111",
        "creationTimestamp": "2026-03-09T10:00:05Z",
        "resourceVersion": "6"
      },
      "spec": {
        "addressRef": {
          "name": "addr-b1",
          "namespace": "tenant-poc"
        },
        "networkInterfaceRef": {
          "name": "eni-b1",
          "namespace": "tenant-poc"
        }
      },
      "status": {
        "state": "bound"
      }
    },
    {
      "metadata": {
        "name": "bind-eni-a2",
        "namespace": "tenant-poc",
        "uid": "77777777-1111-1111-1111-aaa222222222",
        "creationTimestamp": "2026-03-09T10:00:05Z",
        "resourceVersion": "6"
      },
      "spec": {
        "addressRef": {
          "name": "addr-a2",
          "namespace": "tenant-poc"
        },
        "networkInterfaceRef": {
          "name": "eni-a2",
          "namespace": "tenant-poc"
        }
      },
      "status": {
        "state": "bound"
      }
    },
    {
      "metadata": {
        "name": "bind-eni-b2",
        "namespace": "tenant-poc",
        "uid": "77777777-1111-1111-1111-bbb222222222",
        "creationTimestamp": "2026-03-09T10:00:05Z",
        "resourceVersion": "6"
      },
      "spec": {
        "addressRef": {
          "name": "addr-b2",
          "namespace": "tenant-poc"
        },
        "networkInterfaceRef": {
          "name": "eni-b2",
          "namespace": "tenant-poc"
        }
      },
      "status": {
        "state": "bound"
      }
    }
  ]
}
```

## Delete

```bash
curl -s 'http://api:9006/v1/address-bindings/delete' \
-H 'Content-Type: application/json' \
-d '{
  "addressBindings": [
    {
      "metadata": {
        "name": "bind-eni-a1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "bind-eni-b1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "bind-eni-a2",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "bind-eni-b2",
        "namespace": "tenant-poc"
      }
    }
  ]
}'
```
