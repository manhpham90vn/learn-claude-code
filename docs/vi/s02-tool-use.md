# s02: Sử dụng Tool

`s01 > [ s02 ] s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Thêm một tool có nghĩa là thêm một handler"* -- vòng lặp giữ nguyên; tools mới đăng ký vào dispatch map.

## Vấn đề

Chỉ với `bash`, agent shell out cho mọi thứ. `cat` cắt không predictable, `sed` fail với ký tự đặc biệt, và mỗi bash call là một bề mặt bảo mật không ràng buộc. Các tool chuyên dụng như `read_file` và `write_file` cho phép bạn enforce path sandboxing ở cấp độ tool.

Insight quan trọng: thêm tools không yêu cầu thay đổi vòng lặp.

## Giải pháp

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+

Dispatch map là một dict: {tool_name: handler_function}.
Một lookup thay thế bất kỳ chuỗi if/elif nào.
```

## Cách nó hoạt động

1. Mỗi tool có một handler function. Path sandboxing ngăn workspace escape.

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path

def run_read(path: str, limit: int = None) -> str:
    text = safe_path(path).read_text()
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit]
    return "\n".join(lines)[:50000]
```

2. Dispatch map link tool names đến handlers.

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"],
                                        kw["new_text"]),
}
```

3. Trong vòng lặp, lookup handler theo name. Loop body giữ nguyên từ s01.

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler \
            else f"Unknown tool: {block.name}"
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

Thêm một tool = thêm một handler + thêm một schema entry. Vòng lặp không bao giờ thay đổi.

## Thay đổi từ s01

| Component      | Before (s01)       | After (s02)                |
|----------------|--------------------|----------------------------|
| Tools          | 1 (bash only)      | 4 (bash, read, write, edit)|
| Dispatch       | Hardcoded bash call | `TOOL_HANDLERS` dict       |
| Path safety    | None               | `safe_path()` sandbox      |
| Agent loop     | Unchanged          | Unchanged                  |

## Thử ngay

```sh
cd learn-claude-code
python agents/s02_tool_use.py
```

1. `Đọc file requirements.txt`
2. `Tạo một file tên greet.py với một hàm greet(name)`
3. `Edit greet.py để thêm docstring vào hàm`
4. `Đọc greet.py để xác nhận edit đã hoạt động`
