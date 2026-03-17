# s07: Hệ thống Task

`s01 > s02 > s03 > s04 > s05 > s06 | [ s07 ] s08 > s09 > s10 > s11 > s12`

> *"Chia nhỏ các mục tiêu lớn thành các task nhỏ, sắp xếp chúng, lưu vào disk"* -- một task graph dựa trên file với dependencies, đặt nền tảng cho sự hợp tác multi-agent.

## Vấn đề

TodoManager của s03 là một flat checklist trong memory: không có ordering, không có dependencies, không có status ngoài done-or-not. Các mục tiêu thực có cấu trúc -- task B phụ thuộc task A, tasks C và D có thể chạy song song, task E đợi cả C và D.

Không có relationships rõ ràng, agent không thể biết cái gì đã sẵn sàng, cái gì bị block, hay cái gì có thể chạy đồng thời. Và vì list chỉ tồn tại trong memory, context compression (s06) xóa sạch nó.

## Giải pháp

Promote checklist thành một **task graph** được lưu vào disk. Mỗi task là một JSON file với status, dependencies (`blockedBy`), và dependents (`blocks`). Graph trả lời ba câu hỏi tại bất kỳ thời điểm nào:

- **Cái gì sẵn sàng?** -- tasks với status `pending` và `blockedBy` rỗng.
- **Cái gì bị block?** -- tasks đang đợi các dependencies chưa hoàn thành.
- **Cái gì đã done?** -- tasks `completed`, việc hoàn thành tự động unblock dependents.

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

Task graph (DAG):
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+    +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

Ordering:     task 1 must finish before 2 and 3
Parallelism:  tasks 2 and 3 can run at the same time
Dependencies: task 4 waits for both 2 and 3
Status:       pending -> in_progress -> completed
```

Task graph này trở thành xương sống coordination cho mọi thứ sau s07: background execution (s08), multi-agent teams (s09+), và worktree isolation (s12) đều đọc và viết vào cùng cấu trúc này.

## Cách nó hoạt động

1. **TaskManager**: một JSON file cho mỗi task, CRUD với dependency graph.

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def create(self, subject, description=""):
        task = {"id": self._next_id, "subject": subject,
                "status": "pending", "blockedBy": [],
                "blocks": [], "owner": ""}
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

2. **Dependency resolution**: hoàn thành một task xóa ID của nó khỏi `blockedBy` list của mọi task khác, tự động unblock dependents.

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

3. **Status + dependency wiring**: `update` xử lý transitions và dependency edges.

```python
def update(self, task_id, status=None,
           add_blocked_by=None, add_blocks=None):
    task = self._load(task_id)
    if status:
        task["status"] = status
        if status == "completed":
            self._clear_dependency(task_id)
    self._save(task)
```

4. Bốn task tools đi vào dispatch map.

```python
TOOL_HANDLERS = {
    # ...base tools...
    "task_create": lambda **kw: TASKS.create(kw["subject"]),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status")),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

Từ s07 trở đi, task graph là mặc định cho multi-step work. Todo của s03 vẫn còn cho các checklist đơn giản trong một session.

## Thay đổi từ s06

| Component | Before (s06) | After (s07) |
|---|---|---|
| Tools | 5 | 8 (`task_create/update/list/get`) |
| Planning model | Flat checklist (in-memory) | Task graph with dependencies (on disk) |
| Relationships | None | `blockedBy` + `blocks` edges |
| Status tracking | Done or not | `pending` -> `in_progress` -> `completed` |
| Persistence | Lost on compression | Survives compression and restarts |

## Thử ngay

```sh
cd learn-claude-code
python agents/s07_task_system.py
```

1. `Tạo 3 tasks: "Setup project", "Write code", "Write tests". Làm chúng phụ thuộc nhau theo thứ tự.`
2. `Liệt kê tất cả tasks và hiển thị dependency graph`
3. `Hoàn thành task 1 sau đó liệt kê tasks để xem task 2 được unblock`
4. `Tạo một task board cho refactoring: parse -> transform -> emit -> test, trong đó transform và emit có thể chạy song song sau parse`
