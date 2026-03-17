# s01: Vòng lặp Agent

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Một vòng lặp & Bash là tất cả những gì bạn cần"* -- một tool + một vòng lặp = một agent.

## Vấn đề

Một mô hình ngôn ngữ có thể suy luận về code, nhưng nó không thể *chạm* vào thế giới thực -- không thể đọc file, chạy test, hay kiểm tra lỗi. Không có vòng lặp, mỗi lần gọi tool đều yêu cầu bạn thủ công copy-paste kết quả trở lại. Bạn trở thành vòng lặp.

## Giải pháp

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> |  Tool   |
| prompt |      |       |      | execute |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                    (loop until stop_reason != "tool_use")
```

Một điều kiện thoát kiểm soát toàn bộ luồng. Vòng lặp chạy cho đến khi model ngừng gọi tools.

## Cách nó hoạt động

1. User prompt trở thành message đầu tiên.

```python
messages.append({"role": "user", "content": query})
```

2. Gửi messages + tool definitions đến LLM.

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

3. Append assistant response. Kiểm tra `stop_reason` -- nếu model không gọi tool, chúng ta done.

```python
messages.append({"role": "assistant", "content": response.content})
if response.stop_reason != "tool_use":
    return
```

4. Thực thi mỗi tool call, thu thập kết quả, append như một user message. Quay lại bước 2.

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
messages.append({"role": "user", "content": results})
```

Assemble thành một function:

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

Đó là toàn bộ agent trong chưa đến 30 dòng. Mọi thứ khác trong khóa học này được xây dựng lên trên -- mà không thay đổi vòng lặp.

## Thay đổi từ trước đến nay

| Component     | Before     | After                          |
|---------------|------------|--------------------------------|
| Agent loop    | (none)     | `while True` + stop_reason     |
| Tools         | (none)     | `bash` (một tool)              |
| Messages      | (none)     | Accumulating list              |
| Control flow  | (none)     | `stop_reason != "tool_use"`    |

## Thử ngay

```sh
cd learn-claude-code
python agents/s01_agent_loop.py
```

1. `Tạo một file tên là hello.py in ra "Hello, World!"`
2. `Liệt kê tất cả các file Python trong thư mục này`
3. `Branch git hiện tại là gì?`
4. `Tạo một thư mục tên test_output và viết 3 file vào trong đó`
