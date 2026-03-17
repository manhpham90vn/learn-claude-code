# s11: Autonomous Agents

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > [ s11 ] s12`

> *"Đồng đội quét board và tự claim tasks"* -- không cần lead assign từng cái.

## Vấn đề

Trong s09-s10, teammates chỉ làm việc khi được told explicitly. Lead phải spawn mỗi cái với một prompt cụ thể. 10 unclaimed tasks trên board? Lead assign từng cái thủ công. Không scale.

True autonomy: teammates tự quét task board, claim unclaimed tasks, làm việc trên chúng, sau đó tìm thêm.

Một điểm tinh tế: sau context compression (s06), agent có thể quên mình là ai. Identity re-injection sửa cái này.

## Giải pháp

```
Teammate lifecycle with idle cycle:

+-------+
| spawn |
+---+---+
    |
    v
+-------+   tool_use     +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use (or idle tool called)
    v
+--------+
|  IDLE  |  poll every 5s for up to 60s
+---+----+
    |
    +---> check inbox --> message? ----------> WORK
    |
    +---> scan .tasks/ --> unclaimed? -------> claim -> WORK
    |
    +---> 60s timeout ----------------------> SHUTDOWN

Identity re-injection after compression:
  if len(messages) <= 3:
    messages.insert(0, identity_block)
```

## Cách nó hoạt động

1. Teammate loop có hai phases: WORK và IDLE. Khi LLM ngừng gọi tools (hoặc gọi `idle`), teammate enters IDLE.

```python
def _loop(self, name, role, prompt):
    while True:
        # -- WORK PHASE --
        messages = [{"role": "user", "content": prompt}]
        for _ in range(50):
            response = client.messages.create(...)
            if response.stop_reason != "tool_use":
                break
            # execute tools...
            if idle_requested:
                break

        # -- IDLE PHASE --
        self._set_status(name, "idle")
        resume = self._idle_poll(name, messages)
        if not resume:
            self._set_status(name, "shutdown")
            return
        self._set_status(name, "working")
```

2. Idle phase polls inbox và task board trong một loop.

```python
def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 60s / 5s = 12
        time.sleep(POLL_INTERVAL)
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            return True
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append({"role": "user",
                "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: "
                           f"{unclaimed[0]['subject']}</auto-claimed>"})
            return True
    return False  # timeout -> shutdown
```

3. Task board scanning: find pending, unowned, unblocked tasks.

```python
def scan_unclaimed_tasks() -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

4. Identity re-injection: khi context quá ngắn (compression happened), insert một identity block.

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}. Continue your work.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

## Thay đổi từ s10

| Component      | Before (s10)     | After (s11)                |
|----------------|------------------|----------------------------|
| Tools          | 12               | 14 (+idle, +claim_task)    |
| Autonomy       | Lead-directed    | Self-organizing            |
| Idle phase     | None             | Poll inbox + task board    |
| Task claiming  | Manual only      | Auto-claim unclaimed tasks |
| Identity       | System prompt    | + re-injection after compress|
| Timeout        | None             | 60s idle -> auto shutdown  |

## Thử ngay

```sh
cd learn-claude-code
python agents/s11_autonomous_agents.py
```

1. `Tạo 3 tasks trên board, sau đó spawn alice và bob. Xem họ tự động claim.`
2. `Spawn một coder teammate và để nó tìm việc từ task board`
3. `Tạo các tasks với dependencies. Xem teammates tôn trọng thứ tự blocked.`
4. Gõ `/tasks` để xem task board với owners
5. Gõ `/team` để monitor ai đang làm việc vs idle
