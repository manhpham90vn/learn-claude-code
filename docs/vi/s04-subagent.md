# s04: Subagent

`s01 > s02 > s03 > [ s04 ] s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Chia nhỏ các task lớn; mỗi subtask có một context sạch"* -- subagents sử dụng messages[] độc lập, giữ cuộc hội thoại chính sạch sẽ.

## Vấn đề

Khi agent làm việc, messages array của nó tăng lên. Mỗi file read, mỗi bash output đều ở trong context vĩnh viễn. "Testing framework nào project này sử dụng?" có thể cần đọc 5 file, nhưng parent chỉ cần câu trả lời: "pytest."

## Giải pháp

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- fresh
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Parent context stays clean. Subagent context is discarded.
```

## Cách nó hoạt động

1. Parent có tool `task`. Child có tất cả base tools trừ `task` (không recursive spawning).

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task",
     "description": "Spawn a subagent with fresh context.",
     "input_schema": {
         "type": "object",
         "properties": {"prompt": {"type": "string"}},
         "required": ["prompt"],
     }},
]
```

2. Subagent bắt đầu với `messages=[]` và chạy vòng lặp của riêng nó. Chỉ text cuối cùng trả về parent.

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant",
                             "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({"type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(
        b.text for b in response.content if hasattr(b, "text")
    ) or "(no summary)"
```

Toàn bộ message history của child (có thể 30+ tool calls) bị discard. Parent nhận một đoạn tóm tắt như một `tool_result` bình thường.

## Thay đổi từ s03

| Component      | Before (s03)     | After (s04)               |
|----------------|------------------|---------------------------|
| Tools          | 5                | 5 (base) + task (parent)  |
| Context        | Single shared    | Parent + child isolation  |
| Subagent       | None             | `run_subagent()` function |
| Return value   | N/A              | Summary text only         |

## Thử ngay

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

1. `Sử dụng subtask để tìm testing framework nào project này sử dụng`
2. `Delegate: đọc tất cả các file .py và tóm tắt mỗi file làm gì`
3. `Sử dụng task để tạo một module mới, sau đó verify từ đây`
