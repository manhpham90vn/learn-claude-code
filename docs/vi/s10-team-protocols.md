# s10: Team Protocols

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > [ s10 ] s11 > s12`

> *"Đồng đội cần các quy tắc communication chung"* -- một request-response pattern điều khiển mọi negotiation.

## Vấn đề

Trong s09, teammates làm việc và communicate nhưng thiếu coordination có cấu trúc:

**Shutdown**: Killing một thread để lại các file nửa viết và config.json stale. Bạn cần một handshake: lead requests, teammate approves (finish và exit) hoặc rejects (keep working).

**Plan approval**: Khi lead nói "refactor auth module," teammate bắt đầu ngay. Với các thay đổi high-risk, lead nên review plan trước.

Cả hai chia sẻ cùng cấu trúc: một bên gửi request với một unique ID, bên kia respond reference ID đó.

## Giải pháp

```
Shutdown Protocol            Plan Approval Protocol
==================           ======================

Lead             Teammate    Teammate           Lead
  |                 |           |                 |
  |--shutdown_req-->|           |--plan_req------>|
  | {req_id:"abc"}  |           | {req_id:"xyz"}  |
  |                 |           |                 |
  |<--shutdown_resp-|           |<--plan_resp-----|
  | {req_id:"abc",  |           | {req_id:"xyz",  |
  |  approve:true}  |           |  approve:true}  |

Shared FSM:
  [pending] --approve--> [approved]
  [pending] --reject---> [rejected]

Trackers:
  shutdown_requests = {req_id: {target, status}}
  plan_requests     = {req_id: {from, plan, status}}
```

## Cách nó hoạt động

1. Lead initiates shutdown bằng cách generate một request_id và gửi qua inbox.

```python
shutdown_requests = {}

def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
    return f"Shutdown request {req_id} sent (status: pending)"
```

2. Teammate receives request và responds với approve/reject.

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
    BUS.send(sender, "lead", args.get("reason", ""),
             "shutdown_response",
             {"request_id": req_id, "approve": approve})
```

3. Plan approval follows identical pattern. Teammate submits a plan (generating request_id), lead reviews (referencing same request_id).

```python
plan_requests = {}

def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback,
             "plan_approval_response",
             {"request_id": request_id, "approve": approve})
```

Một FSM, hai applications. Cùng `pending -> approved | rejected` state machine xử lý bất kỳ request-response protocol nào.

## Thay đổi từ s09

| Component      | Before (s09)     | After (s10)                  |
|----------------|------------------|------------------------------|
| Tools          | 9                | 12 (+shutdown_req/resp +plan)|
| Shutdown       | Natural exit only| Request-response handshake   |
| Plan gating    | None             | Submit/review with approval  |
| Correlation    | None             | request_id per request       |
| FSM            | None             | pending -> approved/rejected |

## Thử ngay

```sh
cd learn-claude-code
python agents/s10_team_protocols.py
```

1. `Spawn alice là một coder. Sau đó request cô ấy shutdown.`
2. `Liệt kê teammates để xem status của alice sau khi shutdown được approve`
3. `Spawn bob với một risky refactoring task. Review và reject plan của anh ấy.`
4. `Spawn charlie, yêu anh ấy submit một plan, sau đó approve nó.`
5. Gõ `/team` để monitor statuses
