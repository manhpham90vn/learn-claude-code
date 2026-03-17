# s05: Kỹ năng

`s01 > s02 > s03 > s04 > [ s05 ] s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Load knowledge khi bạn cần, không phải trước"* -- inject qua tool_result, không phải system prompt.

## Vấn đề

Bạn muốn agent follow domain-specific workflows: git conventions, testing patterns, code review checklists. Đặt mọi thứ trong system prompt lãng phí tokens cho các skills không dùng. 10 skills x 2000 tokens mỗi cái = 20,000 tokens, hầu hết không liên quan đến bất kỳ task nào.

## Giải pháp

```
System prompt (Layer 1 -- always present):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 tokens/skill
|   - test: Testing best practices     |
+--------------------------------------+

When model calls load_skill("git"):
+--------------------------------------+
| tool_result (Layer 2 -- on demand):  |
| <skill name="git">                   |
|   Full git workflow instructions...  |  ~2000 tokens
|   Step 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

Layer 1: skill *names* trong system prompt (rẻ). Layer 2: full *body* qua tool_result (theo yêu cầu).

## Cách nó hoạt động

1. Mỗi skill là một directory chứa `SKILL.md` với YAML frontmatter.

```
skills/
  pdf/
    SKILL.md       # ---\n name: pdf\n description: Process PDF files\n ---\n ...
  code-review/
    SKILL.md       # ---\n name: code-review\n description: Review code\n ---\n ...
```

2. SkillLoader scan cho `SKILL.md` files, sử dụng directory name làm skill identifier.

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills = {}
        for f in sorted(skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}

    def get_descriptions(self) -> str:
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

3. Layer 1 đi vào system prompt. Layer 2 chỉ là một tool handler khác.

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

TOOL_HANDLERS = {
    # ...base tools...
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

Model học những skills nào tồn tại (rẻ) và load chúng khi cần (đắt).

## Thay đổi từ s04

| Component      | Before (s04)     | After (s05)                |
|----------------|------------------|----------------------------|
| Tools          | 5 (base + task)  | 5 (base + load_skill)      |
| System prompt  | Static string    | + skill descriptions       |
| Knowledge      | None             | skills/\*/SKILL.md files   |
| Injection      | None             | Two-layer (system + result)|

## Thử ngay

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

1. `Những skills nào có sẵn?`
2. `Load skill agent-builder và làm theo hướng dẫn của nó`
3. `Tôi cần làm một code review -- load skill liên quan trước`
4. `Build một MCP server sử dụng skill mcp-builder`
