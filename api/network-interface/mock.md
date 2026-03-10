# NetworkInterface — PoC mock

Данные для PoC-сценария из HV-SETUP.md.

Четыре NI — по одному на каждый будущий Instance. NI содержит только `spec.subnetRef` — привязка к Instance происходит на стороне Instance через `spec.networkInterfacesRef`.

## Upsert

**Запрос:**

```bash
curl -s 'http://api:9006/v1/network-interfaces/upsert' \
-H 'Content-Type: application/json' \
-d '{
  "networkInterfaces": [
    {
      "metadata": {
        "name": "eni-a1",
        "namespace": "tenant-poc"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for web server 1"
      }
    },
    {
      "metadata": {
        "name": "eni-b1",
        "namespace": "tenant-poc"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for DB server 1"
      }
    },
    {
      "metadata": {
        "name": "eni-a2",
        "namespace": "tenant-poc"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for web server 2"
      }
    },
    {
      "metadata": {
        "name": "eni-b2",
        "namespace": "tenant-poc"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for DB server 2"
      }
    }
  ]
}'
```

**Ответ:**

API создаёт NI в состоянии `created`. IP и MAC пока не назначены — они появятся после создания Address и AddressBinding.

```json
{
  "networkInterfaces": [
    {
      "metadata": {
        "name": "eni-a1",
        "namespace": "tenant-poc",
        "uid": "55555555-1111-1111-1111-aaa111111111",
        "creationTimestamp": "2026-03-09T10:00:03Z",
        "resourceVersion": "4"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for web server 1"
      },
      "status": {
        "state": "created",
        "privateIpAddress": "",
        "macAddress": "",
        "publicIp": "",
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "vpcRef": {
          "name": "vpc-1",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "eni-b1",
        "namespace": "tenant-poc",
        "uid": "55555555-1111-1111-1111-bbb111111111",
        "creationTimestamp": "2026-03-09T10:00:03Z",
        "resourceVersion": "4"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for DB server 1"
      },
      "status": {
        "state": "created",
        "privateIpAddress": "",
        "macAddress": "",
        "publicIp": "",
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        },
        "vpcRef": {
          "name": "vpc-2",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "eni-a2",
        "namespace": "tenant-poc",
        "uid": "55555555-1111-1111-1111-aaa222222222",
        "creationTimestamp": "2026-03-09T10:00:03Z",
        "resourceVersion": "4"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for web server 2"
      },
      "status": {
        "state": "created",
        "privateIpAddress": "",
        "macAddress": "",
        "publicIp": "",
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "vpcRef": {
          "name": "vpc-1",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "eni-b2",
        "namespace": "tenant-poc",
        "uid": "55555555-1111-1111-1111-bbb222222222",
        "creationTimestamp": "2026-03-09T10:00:03Z",
        "resourceVersion": "4"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for DB server 2"
      },
      "status": {
        "state": "created",
        "privateIpAddress": "",
        "macAddress": "",
        "publicIp": "",
        "subnetRef": {
          "name": "subnet-2",
          "namespace": "tenant-poc"
        },
        "vpcRef": {
          "name": "vpc-2",
          "namespace": "tenant-poc"
        }
      }
    }
  ]
}
```

## Побочный эффект: NI → available

После создания AddressBinding Status Controller обновляет NI: заполняет `privateIpAddress`, `macAddress`, `subnetRef`, `vpcRef` и переводит `status.state` из `created` в `available`.

**Пример: NI `eni-a1` после обновления Status Controller'ом:**

```json
{
  "metadata": {
    "name": "eni-a1",
    "namespace": "tenant-poc",
    "uid": "55555555-1111-1111-1111-aaa111111111",
    "creationTimestamp": "2026-03-09T10:00:03Z",
    "resourceVersion": "8"
  },
  "spec": {
    "subnetRef": {
      "name": "subnet-1",
      "namespace": "tenant-poc"
    },
    "displayName": "ENI for web server 1"
  },
  "status": {
    "state": "available",
    "privateIpAddress": "10.0.0.1",
    "macAddress": "02:42:0a:00:00:01",
    "publicIp": "",
    "subnetRef": {
      "name": "subnet-1",
      "namespace": "tenant-poc"
    },
    "vpcRef": {
      "name": "vpc-1",
      "namespace": "tenant-poc"
    }
  }
}
```

### Agent workflow

Agent при получении watch-события Instance (instance-a1 как пример):

```
Agent watch'ит Instance с hvRef = "hv1":
  Instance instance-a1 → spec.networkInterfacesRef → [eni-a1]
  → NI eni-a1 → status.state = available
  → status.privateIpAddress = 10.0.0.1
  → status.macAddress = 02:42:0a:00:00:01
  → spec.subnetRef → subnet-1 → spec.cidrBlock = 10.0.0.0/24, spec.vpcRef → vpc-1
  → VPC vpc-1 → status.vni = 100001

Desired state:
  VRF vrf-vpc-1 table 100       (если нет — создать)
  VXLAN vxlan-100001 id 100001  (если нет — создать)
  Bridge br-100001              (если нет — создать)
  veth eni-a1-h / eni-a1-g
    MAC 02:42:0a:00:00:01, IP 10.0.0.1/24
```

Agent на HV1 обрабатывает `instance-a1` и `instance-b1`. Agent на HV2 — `instance-a2` и `instance-b2`. Каждый Agent фильтрует Instance по `spec.hvRef` и разрешает `networkInterfacesRef` для получения сетевой конфигурации.

## List

**Запрос (NI в VPC-1 по status.vpcRef):**

```bash
curl -s 'http://api:9006/v1/network-interfaces/list' \
-H 'Content-Type: application/json' \
-d '{
  "selectors": [
    {
      "fieldSelector": {
        "namespace": "tenant-poc",
        "refs": [
          {
            "name": "vpc-1",
            "resType": "VPC"
          }
        ]
      }
    }
  ]
}'
```

**Ответ:**

```json
{
  "resourceVersion": "8",
  "networkInterfaces": [
    {
      "metadata": {
        "name": "eni-a1",
        "namespace": "tenant-poc",
        "uid": "55555555-1111-1111-1111-aaa111111111",
        "resourceVersion": "8"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for web server 1"
      },
      "status": {
        "state": "available",
        "privateIpAddress": "10.0.0.1",
        "macAddress": "02:42:0a:00:00:01",
        "publicIp": "",
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "vpcRef": {
          "name": "vpc-1",
          "namespace": "tenant-poc"
        }
      }
    },
    {
      "metadata": {
        "name": "eni-a2",
        "namespace": "tenant-poc",
        "uid": "55555555-1111-1111-1111-aaa222222222",
        "resourceVersion": "8"
      },
      "spec": {
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "displayName": "ENI for web server 2"
      },
      "status": {
        "state": "available",
        "privateIpAddress": "10.0.0.2",
        "macAddress": "02:42:0a:00:00:02",
        "publicIp": "",
        "subnetRef": {
          "name": "subnet-1",
          "namespace": "tenant-poc"
        },
        "vpcRef": {
          "name": "vpc-1",
          "namespace": "tenant-poc"
        }
      }
    }
  ]
}
```

## Delete

```bash
curl -s 'http://api:9006/v1/network-interfaces/delete' \
-H 'Content-Type: application/json' \
-d '{
  "networkInterfaces": [
    {
      "metadata": {
        "name": "eni-a1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "eni-b1",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "eni-a2",
        "namespace": "tenant-poc"
      }
    },
    {
      "metadata": {
        "name": "eni-b2",
        "namespace": "tenant-poc"
      }
    }
  ]
}'
```
