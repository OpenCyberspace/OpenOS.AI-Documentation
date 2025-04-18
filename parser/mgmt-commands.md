# Management commands execution:

Management commands can be executed using parser action API, the management commands are sent to the internal services like either cluster controller or to the various internal services of the block via the cluster controller gateway. Parser can be used to perform initial template validation in case of management commands execution.

---

## Executing Management Commands via Parser

The Parser supports executing predefined management commands on either **blocks** or **clusters**. This is handled through the `executeMgmtCommand` action.

### Target Scope

| **Target** | **Required Field** | **Supported Services**                                   |
|------------|--------------------|-----------------------------------------------------------|
| Block      | `blockId`          | `instances`, `executor`, `autoscaler`, `health`          |
| Cluster    | `clusterId`        | `stability_checker`                                      |

Either `blockId` or `clusterId` must be provided depending on the target.

---

### Required Fields in Request

| **Field**     | **Type** | **Required** | **Description**                                                              |
|---------------|----------|--------------|------------------------------------------------------------------------------|
| `blockId`     | string   | Yes*         | ID of the target block (if block-level command).                            |
| `clusterId`   | string   | Yes*         | ID of the target cluster (if cluster-level command).                        |
| `service`     | string   | Yes          | Target service component (e.g., `executor`, `stability_checker`).           |
| `mgmtCommand` | string   | Yes          | The command to execute.          |
| `mgmtData`    | object   | No           | Additional data required by the command.                                    |

\* Either `blockId` or `clusterId` must be set — not both.

---

### Example Payload (Block-Level Command)

```json
{
  "header": {
    "templateUri": "Parser/V1",
    "parameters": {}
  },
  "body": {
    "spec": {
      "values": {
        "blockId": "block-xyz-001",
        "service": "executor",
        "mgmtCommand": "restart",
        "mgmtData": {
          "force": true
        }
      }
    }
  }
}
```

---

### Example Payload (Cluster-Level Command)

```json
{
  "header": {
    "templateUri": "Parser/V1",
    "parameters": {}
  },
  "body": {
    "spec": {
      "values": {
        "clusterId": "cluster-west-vision-001",
        "service": "stability_checker",
        "mgmtCommand": {},
        "mgmtData": {}
      }
    }
  }
}
```

---

### `curl` Command

```bash
curl -X POST http://<parser-host>:<port>/api/executeMgmtCommand \
  -H "Content-Type: application/json" \
  -d '{
    "header": {
      "templateUri": "Parser/V1",
      "parameters": {}
    },
    "body": {
      "spec": {
        "values": {
          "blockId": "block-xyz-001",
          "service": "executor",
          "mgmtCommand": "",
          "mgmtData": {}
        }
      }
    }
  }'
```

Replace `blockId` with `clusterId` and `service` accordingly for cluster-level commands.
