# s03: TodoWrite

`s01 > s02 > [ s03 ] s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Một agent không có kế hoạch sẽ trôi dạt"* -- liệt kê các bước trước, sau đó thực thi.

## Vấn đề

Với các task nhiều bước, model bị mất track. Nó lặp lại công việc, bỏ qua bước, hoặc đi lang thang. Các cuộc hội thoại dài làm điều này tệ hơn -- system prompt mờ dần khi tool results填充 context. Một refactoring 10 bước có thể hoàn thành bước 1-3, sau đó model bắt đầu tự biên vì nó quên bước 4-10.

## Giải pháp

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | TodoManager state     |
              | [ ] task A            |
              | [>] task B  <- doing  |
              | [x] task C            |
              +-----------------------+
                          |
              if rounds_since_todo >= 3:
                inject <reminder> into tool_result
```

## Cách nó hoạt động

1. TodoManager lưu trữ các items với statuses. Chỉ một item có thể `in_progress` tại một thời điểm.

```python
class TodoManager:
    def update(self, items: list) -> str:
        validated, in_progress_count = [], 0
        for item in items:
            status = item.get("status", "pending")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item["id"], "text": item["text"],
                              "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress")
        self.items = validated
        return self.render()
```

2. Tool `todo` vào dispatch map như bất kỳ tool nào khác.

```python
TOOL_HANDLERS = {
    # ...base tools...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

3. Một nag reminder inject một nudge nếu model đi 3+ rounds mà không gọi `todo`.

```python
if rounds_since_todo >= 3 and messages:
    last = messages[-1]
    if last["role"] == "user" and isinstance(last.get("content"), list):
        last["content"].insert(0, {
            "type": "text",
            "text": "<reminder>Update your todos.</reminder>",
        })
```

Ràng buộc "một in_progress tại một thời điểm" buộc focus tuần tự. Nag reminder tạo ra trách nhiệm.

## Thay đổi từ s02

| Component      | Before (s02)     | After (s03)                |
|----------------|------------------|----------------------------|
| Tools          | 4                | 5 (+todo)                  |
| Planning       | None             | TodoManager with statuses  |
| Nag injection  | None             | `<reminder>` after 3 rounds|
| Agent loop     | Simple dispatch  | + rounds_since_todo counter|

## Thử ngay

```sh
cd learn-claude-code
python agents/s03_todo_write.py
```

1. `Refactor file hello.py: thêm type hints, docstrings, và một main guard`
2. `Tạo một Python package với __init__.py, utils.py, và tests/test_utils.py`
3. `Review tất cả các file Python và sửa bất kỳ vấn đề style nào`
