# s12: Worktree + Task Isolation

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > [ s12 ]`

> *"Mỗi người làm việc trong thư mục của mình, không can thiệp"* -- tasks quản lý mục tiêu, worktrees quản lý thư mục, bind bằng ID.

## Vấn đề

Đến s11, agents có thể claim và hoàn thành tasks tự chủ. Nhưng mọi task chạy trong một thư mục chia sẻ. Hai agents refactoring các modules khác nhau cùng lúc sẽ va chạm: agent A edit `config.py`, agent B edit `config.py`, unstaged changes trộn lẫn, và không ai có thể roll back sạch sẽ.

Task board track *cái gì cần làm* nhưng không có ý kiến về *ở đâu làm*. Fix: cho mỗi task git worktree directory riêng. Tasks quản lý mục tiêu, worktrees quản lý execution context. Bind chúng bằng task ID.

## Giải pháp

```
Control plane (.tasks/)             Execution plane (.worktrees/)
+------------------+                +------------------------+
| task_1.json      |                | auth-refactor/         |
|   status: in_progress  <------>   branch: wt/auth-refactor
|   worktree: "auth-refactor"   |   task_id: 1             |
+------------------+                +------------------------+
| task_2.json      |                | ui-login/              |
|   status: pending    <------>     branch: wt/ui-login
|   worktree: "ui-login"       |   task_id: 2             |
+------------------+                +------------------------+
                                    |
                          index.json (worktree registry)
                          events.jsonl (lifecycle log)

State machines:
  Task:     pending -> in_progress -> completed
  Worktree: absent  -> active      -> removed | kept
```

## Cách nó hoạt động

1. **Tạo một task.** Persist goal trước.

```python
TASKS.create("Implement auth refactor")
# -> .tasks/task_1.json  status=pending  worktree=""
```

2. **Tạo một worktree và bind vào task.** Passing `task_id` tự động advance task thành `in_progress`.

```python
WORKTREES.create("auth-refactor", task_id=1)
# -> git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
# -> index.json gets new entry, task_1.json gets worktree="auth-refactor"
```

Binding viết state vào cả hai bên:

```python
def bind_worktree(self, task_id, worktree):
    task = self._load(task_id)
    task["worktree"] = worktree
    if task["status"] == "pending":
        task["status"] = "in_progress"
    self._save(task)
```

3. **Chạy commands trong worktree.** `cwd` trỏ đến thư mục isolated.

```python
subprocess.run(command, shell=True, cwd=worktree_path,
               capture_output=True, text=True, timeout=300)
```

4. **Đóng lại.** Hai lựa chọn:
   - `worktree_keep(name)` -- preserve thư mục cho sau.
   - `worktree_remove(name, complete_task=True)` -- remove thưu mục, complete bound task, emit event. Một call xử lý teardown + completion.

```python
def remove(self, name, force=False, complete_task=False):
    self._run_git(["worktree", "remove", wt["path"]])
    if complete_task and wt.get("task_id") is not None:
        self.tasks.update(wt["task_id"], status="completed")
        self.tasks.unbind_worktree(wt["task_id"])
        self.events.emit("task.completed", ...)
```

5. **Event stream.** Mỗi lifecycle step emit đến `.worktrees/events.jsonl`:

```json
{
  "event": "worktree.remove.after",
  "task": {"id": 1, "status": "completed"},
  "worktree": {"name": "auth-refactor", "status": "removed"},
  "ts": 1730000000
}
```

Events emitted: `worktree.create.before/after/failed`, `worktree.remove.before/after/failed`, `worktree.keep`, `task.completed`.

Sau một crash, state reconstruct từ `.tasks/` + `.worktrees/index.json` trên disk. Conversation memory là volatile; file state là durable.

## Thay đổi từ s11

| Component          | Before (s11)               | After (s12)                                  |
|--------------------|----------------------------|----------------------------------------------|
| Coordination       | Task board (owner/status)  | Task board + explicit worktree binding       |
| Execution scope    | Shared directory           | Task-scoped isolated directory               |
| Recoverability     | Task status only           | Task status + worktree index                 |
| Teardown           | Task completion            | Task completion + explicit keep/remove       |
| Lifecycle visibility | Implicit in logs         | Explicit events in `.worktrees/events.jsonl` |

## Thử ngay

```sh
cd learn-claude-code
python agents/s12_worktree_task_isolation.py
```

1. `Tạo tasks cho backend auth và frontend login page, sau đó liệt kê tasks.`
2. `Tạo worktree "auth-refactor" cho task 1, sau đó bind task 2 vào worktree mới "ui-login".`
3. `Chạy "git status --short" trong worktree "auth-refactor".`
4. `Giữ worktree "ui-login", sau đó liệt kê worktrees và inspect events.`
5. `Remove worktree "auth-refactor" với complete_task=true, sau đó liệt kê tasks/worktrees/events.`
