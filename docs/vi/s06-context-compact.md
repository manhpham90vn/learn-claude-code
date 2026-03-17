# s06: Nén Context

`s01 > s02 > s03 > s04 > s05 > [ s06 ] | s07 > s08 > s09 > s10 > s11 > s12`

> *"Context sẽ đầy; bạn cần một cách để tạo không gian"* -- chiến lược nén ba lớp cho các session vô hạn.

## Vấn đề

Context window là hữu hạn. Một `read_file` trên file 1000 dòng tốn ~4000 tokens. Sau khi đọc 30 files và chạy 20 bash commands, bạn đạt 100,000+ tokens. Agent không thể làm việc trên các codebase lớn mà không có compression.

## Giải pháp

Ba lớp, tăng dần về mức độ aggressive:

```
Every turn:
+------------------+
| Tool call result |
+------------------+
        |
        v
[Layer 1: micro_compact]        (silent, every turn)
  Replace tool_result > 3 turns old
  with "[Previous: used {tool_name}]"
        |
        v
[Check: tokens > 50000?]
   |               |
   no              yes
   |               |
   v               v
continue    [Layer 2: auto_compact]
              Save transcript to .transcripts/
              LLM summarizes conversation.
              Replace all messages with [summary].
                    |
                    v
            [Layer 3: compact tool)
              Model calls compact explicitly.
              Same summarization as auto_compact.
```

## Cách nó hoạt động

1. **Layer 1 -- micro_compact**: Trước mỗi LLM call, replace old tool results với placeholders.

```python
def micro_compact(messages: list) -> list:
    tool_results = []
    for i, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for j, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((i, j, part))
    if len(tool_results) <= KEEP_RECENT:
        return messages
    for _, _, part in tool_results[:-KEEP_RECENT]:
        if len(part.get("content", "")) > 100:
            part["content"] = f"[Previous: used {tool_name}]"
    return messages
```

2. **Layer 2 -- auto_compact**: Khi tokens vượt ngưỡng, save full transcript xuống disk, sau đó ask LLM to summarize.

```python
def auto_compact(messages: list) -> list:
    # Save transcript for recovery
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    # LLM summarizes
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity..."
            + json.dumps(messages, default=str)[:80000]}],
        max_tokens=2000,
    )
    return [
        {"role": "user", "content": f"[Compressed]\n\n{response.content[0].text}"},
        {"role": "assistant", "content": "Understood. Continuing."},
    ]
```

3. **Layer 3 -- manual compact**: Tool `compact` trigger same summarization on demand.

4. Vòng lặp tích hợp cả ba:

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                        # Layer 1
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)       # Layer 2
        response = client.messages.create(...)
        # ... tool execution ...
        if manual_compact:
            messages[:] = auto_compact(messages)       # Layer 3
```

Transcripts preserve full history on disk. Nothing is truly lost -- chỉ là move ra khỏi active context.

## Thay đổi từ s05

| Component      | Before (s05)     | After (s06)                |
|----------------|------------------|----------------------------|
| Tools          | 5                | 5 (base + compact)         |
| Context mgmt   | None             | Three-layer compression    |
| Micro-compact  | None             | Old results -> placeholders|
| Auto-compact   | None             | Token threshold trigger    |
| Transcripts    | None             | Saved to .transcripts/     |

## Thử ngay

```sh
cd learn-claude-code
python agents/s06_context_compact.py
```

1. `Đọc tất cả các file Python trong thư mục agents/ từng file một` (xem micro-compact replace old results)
2. `Tiếp tục đọc files cho đến khi compression trigger tự động)
3. `Sử dụng tool compact để nén cuộc hội thoại thủ công`
