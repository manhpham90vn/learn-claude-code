[English](./README.md) | [中文](./README-zh.md) | [日本語](./README-ja.md) | [Tiếng Việt](./README-vi.md)

# Học Claude Code -- Làm chủ Engineering Khung sườn (Harness) cho các Agent Thực sự

## Mô hình (Model) CHÍNH LÀ Agent

Trước khi nói về code, hãy làm rõ một điều.
**Một agent là một mô hình. Không phải framework. Không phải chuỗi prompt. Không phải quy trình kéo-thả.**

### Agent LÀ Gì
Một agent là một mạng nơ-ron được huấn luyện qua hàng tỷ cập nhật gradient để nhận thức môi trường, suy luận mục tiêu và hành động để đạt được chúng. Con người cũng là một agent dựa trên mạng nơ-ron sinh học. Khi DeepMind, OpenAI hay Anthropic nói về "agent", họ nói về: **một mô hình đã học được cách hành động.**

Các minh chứng lịch sử:
- **2013 -- DeepMind DQN chơi Atari:** Mạng nơ-ron duy nhất học từ pixel và điểm số để chơi xuất sắc 49 game Atari.
- **2019 -- OpenAI Five chinh phục Dota 2:** Các mô hình học nhóm và chiến thuật qua việc tự chơi (self-play) đánh bại cả nhà vô địch thế giới OG.
- **2019 -- DeepMind AlphaStar:** Đạt Grandmaster trong StarCraft II thông qua mô hình học, không phải kịch bản.
- **2019 -- Tencent Jueyu:** Thống trị Honor of Kings, đánh bại game thủ chuyên nghiệp.
- **2024-2025 -- LLM agents:** Claude, GPT, Gemini được sử dụng như coding agent. Kiến trúc luôn giống nhau: mô hình được huấn luyện nhận diện và hành động trong một môi trường.

**"Agent" không bao giờ là mã xung quanh. Agent luôn là mô hình.**

### Agent KHÔNG LÀ Gì
Các hệ thống kết nối prompt chằng chịt, kéo-thả quy trình, hay các nhánh if-else phức tạp không tạo nên agent. Chúng chỉ là những "kịch bản shell bị hoang tưởng". Trí thông minh là do học được, không phải do lập trình tạo ra. GOFAI (quy tắc logic thuần túy) đã thất bại từ lâu, và nay đang được ngụy trang lại dưới vỏ bọc LLM.

### Sự chuyển đổi tư duy: Phát triển Agent sang Phát triển Khung sườn (Harness)
"Phát triển agent" thực sự chỉ có hai nghĩa:
**1. Huấn luyện mô hình:** Điều chỉnh qua RLHF, fine-tuning với quy trình dữ liệu thực tế (do DeepMind, OpenAI, Anthropic thực hiện).
**2. Xây dựng Harness:** Viết mã tạo môi trường sống cho mô hình. Đây là điều hầu hết kỹ sư làm và là trọng tâm của repo này.

```
Harness = Công cụ (Tools) + Kiến thức + Quan sát + Giao diện Hành động + Phân quyền
```
Mô hình ra quyết định, Harness thực thi. Mô hình là tài xế, Harness là chiếc xe.
Harness cho lập trình viên là IDE, Terminal và File system. Harness cho kỹ sư nông nghiệp là cảm biến, điều khiển tưới tiêu.

### Kỹ sư Harness Thực Sự Làm Gì
- **Tạo công cụ:** Cấp cho agent "đôi tay" (bash, read, write...).
- **Cung cấp kiến thức:** Tài liệu, API specs, yêu cầu. Tải theo nhu cầu (on-demand), không nạp trước toàn bộ.
- **Quản lý ngữ cảnh:** Phân tách thành Agent con (subagents) và nén ngữ cảnh để tránh quá tải.
- **Kiểm soát phân quyền:** Đặt ranh giới an toàn, kiểm duyệt hoạt động phá hủy.
- **Thu thập dữ liệu:** Các quy trình hành động thành công là tín hiệu huấn luyện chất lượng để làm mô hình tốt hơn.

**Xây dựng Harness tốt, mô hình sẽ lo phần còn lại.**

### Tại sao là Claude Code -- Bậc thầy Harness Engineering
Claude Code là hệ thống phân tách môi trường tốt nhất hiện nay: nó không áp đặt mô hình phải đi theo cây quyết định cứng ngắc, cài đặt giới hạn an toàn và rồi lui ra khỏi luồng điều khiển của mô hình.

```
Claude Code = 1 vòng lặp agent + công cụ nâng cao + tải skill linh hoạt
          + nén ngữ cảnh + các subagent + hệ quản lý công việc (tasks)
          + hợp tác nhóm + làm việc song song (worktree isolation) + quản trị quyền.
```

## Tầm nhìn: Thế giới với các Real Agent
Những mô hình học được ở đây có thể áp dụng cho mọi hệ thống cần nhận thức và đánh giá để ra quyết định (Agent khách sạn, Bệnh viện, Quản lý tài sản...).
Hãy bắt đầu lấp đầy công xưởng, bệnh viện. Thế giới sẽ thay đổi.
**Bash là tất cả những gì bạn cần. Những Agent đích thực là tất cả những gì thế giới cần.**

---

```
        MÔ HÌNH AGENT (THE AGENT PATTERN)
        =================
    Người dùng --> tin nhắn[] --> LLM --> phản hồi
                                      |
                         Lý do dừng == "tool_use"?
                        /                        \
                      Có                         Không
                       |                           |
                  Thực thi công cụ              Trả văn bản
                  Cập nhật kết quả           
                  Vòng lặp chạy lại -----> tin nhắn[]
```

**12 bài học lũy tiến (s01 - s12) đi từ cấu trúc căn bản đến tự động cô lập phức tạp:**
> **s01** &nbsp; *"Một vòng lặp & Bash là tất cả những gì bạn cần"* 
> **s02** &nbsp; *"Thêm một công cụ là thêm một hàm xử lý (handler)"* 
> **s03** &nbsp; *"Agent vô tổ chức sẽ mất phương hướng"* 
> **s04** &nbsp; *"Chia nhỏ công việc; xử lý trong môi trường độc lập"* 
> **s05** &nbsp; *"Cung cấp trí tuệ nhưng phải phân tải khi cần"* 
> **s06** &nbsp; *"Ngữ cảnh bị đầy; phải cần hệ nén thông tin 3 lớp"* 
> **s07** &nbsp; *"Sơ đồ tiến trình quy trình phụ thuộc và lưu đĩa"* 
> **s08** &nbsp; *"Các tác vụ nặng chạy ngầm, không làm phiền Agent tư duy"* 
> **s09** &nbsp; *"Lớn quá thì bàn giao đồng đội qua luồng IM Box"* 
> **s10** &nbsp; *"Hợp tác chung thì phải cùng một giao thức hiểu nhau FSM"* 
> **s11** &nbsp; *"Tự đi giành việc, không cần đợi sai bảo"* 
> **s12** &nbsp; *"Luồng đa rẽ nhánh xử lý độc lập cô lập không va chạm"* 

## Mô hình Cốt lõi (The Core Pattern)
```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = TOOL_HANDLERS[block.name](**block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

## Khóa Học Nhanh (Quick Start)
```sh
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
cp .env.example .env   # Sửa .env và điền ANTHROPIC_API_KEY

python agents/s01_agent_loop.py       # Bắt đầu
python agents/s12_worktree_task_isolation.py  # Đích cuối 12 bài
python agents/s_full.py               # Kết hợp toàn bộ
```
Dành cho trình duyệt (Giao diện Interactive Next.js):
```sh
cd web && npm install && npm run dev   # http://localhost:3000
```

## Chặng Đường Tiếp Theo - Tích hợp thực tiễn

### Kode Agent CLI - Open-Source Coding Agent CLI
> `npm i -g @shareai-lab/kode`
Công cụ gõ lệnh đa năng được thiết kế từ gốc, sẵn sàng thay thế.
Tham khảo: [shareAI-lab/Kode-cli](https://github.com/shareAI-lab/Kode-cli)

### Kode Agent SDK - Nhúng Agent vào App của Bạn
Phiên bản thu gọn cho những dự án lớn khi thiết kế kiến trúc phân cấp chìm.
Tham khảo: [shareAI-lab/Kode-agent-sdk](https://github.com/shareAI-lab/Kode-agent-sdk)

## Hệ Thống Anh Em Song Sinh: `claw0`
Sê ri `learn-claude-code` là dạy bạn thiết kế *Agent gọi là chạy (on-demand)*. 
Repo **[claw0](https://github.com/shareAI-lab/claw0)** dạy bạn cách biến Agent thành *Trợ lý AI luôn thức 24/7 (always-on)* thông qua: Hearbeats (nhịp gõ), Cron (hẹn giờ), IM Channels (Telegram, Slack...), Context memory và Soul (Tính cách).
```
claw agent = agent core + heartbeat + cron + IM chat + memory + soul
```

## Giới Thiệu
<img width="260" src="https://github.com/user-attachments/assets/fe8b852b-97da-4061-a467-9694906b5edf" /><br>

Quét Wechat để theo dõi thêm, hoặc follow X/Twitter: [shareAI-Lab](https://x.com/baicai003)

## Giấy Phép
MIT License.

---
**Bash là tất cả những gì bạn cần. Những Agent đích thực là tất cả những gì thế giới cần.**
