This screenshot shows a portion of an **n8n workflow** centered around the error-handling path of an **HTTP Request** node.

| Component               | Description                                                                                                                                                           |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **HTTP Request2**       | An HTTP Request node configured to send a **POST** request to the Google Gemini API (`https://generativelang...`). It has two outputs: **Success** and **Error**.     |
| **Success Output**      | The upper output port labeled **Success**. It is not connected to any node in the visible portion of the workflow.                                                    |
| **Error Output**        | The lower output port labeled **Error**. It is connected to the next node in the workflow.                                                                            |
| **If9**                 | An **If** node that receives data from the Error output of the HTTP Request node. It evaluates a condition and provides two possible outputs: **true** and **false**. |
| **True Branch**         | The **true** output of the If node is connected to the **Wait3** node.                                                                                                |
| **False Branch**        | The **false** output is currently not connected to any node. A **+** icon is displayed, allowing another node to be added.                                            |
| **Wait3**               | A **Wait** node placed after the **true** branch of the If node.                                                                                                      |
| **Workflow Connection** | A connection exits the Wait node and loops back toward the left side of the workflow, connecting back near the HTTP Request node, forming a loop in the workflow.     |

### Workflow structure shown

```text
                Success
HTTP Request2 ------------>

                Error
HTTP Request2 ─────────► If9
                           │
                     ┌─────┴─────┐
                  true         false
                    │
                    ▼
                 Wait3
                    │
                    └───────────────┐
                                    │
                                    ▼
                           Back to HTTP Request2
```

### Visible elements

* **1 HTTP Request node** (`HTTP Request2`)
* **1 If node** (`If9`)
* **1 Wait node** (`Wait3`)
* A connection from the **Error** output of the HTTP Request node to the **If** node.
* A connection from the **true** output of the If node to the **Wait** node.
* A return connection from the **Wait** node back toward the **HTTP Request** node, creating a loop.
* The **Success** output of the HTTP Request node is visible but not connected within the displayed area.
* The **false** output of the If node is currently unconnected.
