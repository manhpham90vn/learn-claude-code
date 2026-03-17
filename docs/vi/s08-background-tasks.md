# s08: Background Tasks

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > [ s08 ] s09 > s10 > s11 > s12`

> *"Chạy các thao tác chậm ở background; agent tiếp tục suy nghĩ"* -- daemon threads chạy commands, inject notifications khi hoàn thành.

## Vấn đề

Một số commands mất phút: `npm install`, `pytest`, `docker build`. Với blocking loop, model ngồi đợi nhàn rỗi. Nếu user hỏi "install dependencies và trong khi đó tạo file config," agent làm chúng tuần tự, không phải song song.

## Giải pháp

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | subprocess runs |
| ...             |        | ...             |
| [LLM call] <---+------- | enqueue(result) |
|  ^drain queue   |        +-----------------+
+-----------------+

Timeline:
Agent --[spawn A]--[spawn B]--[other work]----
             |          |
             v          v
          [A runs]   [B runs]      (parallel)
             |          |
             +-- results injected before next LLM call --+
```

## Cách nó hoạt động

1. BackgroundManager track tasks với một thread-safe notification queue.

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}
        self._notification_queue = []
        self._lock = threading.Lock()
```

2. `run()` start một daemon thread và return ngay lập tức.

```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True)
    thread.start()
    return f"Background task {task_id} started"
```

3. Khi subprocess hoàn thành, kết quả của nó đi vào notification queue.

```python
def _execute(self, task_id, command):
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300)
        output = (r.stdout + r.stderr).strip()[:50000]
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id, "result": output[:500]})
```

4. Agent loop drains notifications trước mỗi LLM call.

```python
def agent_loop(messages: list):
    while True:
        notifs = BG.drain_notifications()
        if notifs:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['result']}" for n in notifs)
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n"
                           f"</background-results>"})
            messages.append({"role": "assistant",
                "content": "Noted background results."})
        response = client.messages.create(...)
```

Loop giữ single-threaded. Chỉ subprocess I/O là parallelized.

## Thay đổi từ s07

| Component      | Before (s07)     | After (s08)                |
|----------------|------------------|----------------------------|
| Tools          | 8                | 6 (base + background_run + check)|
| Execution      | Blocking only    | Blocking + background threads|
| Notification   | None             | Queue drained per loop     |
| Concurrency    | None             | Daemon threads             |

## Thử ngay

```sh
cd learn-claude-code
python agents/s08_background_tasks.py
```

1. `Chạy "sleep 5 && echo done" ở background, sau đó tạo một file trong khi nó đang chạy`
2. `Start 3 background tasks: "sleep 2", "sleep 4", "sleep 6". Kiểm tra status của chúng.`
3. `Chạy pytest ở background và tiếp tục làm việc khác`
