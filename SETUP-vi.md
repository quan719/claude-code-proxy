# Hướng dẫn cài đặt — Claude Code + Proxy + Advisor (Ubuntu)

Bộ này gồm 2 thứ:
- **Patch proxy**: cho phép advisor / security-review chạy được khi dùng gateway (proxy) thay vì `api.anthropic.com`.
- **Lệnh `/advisor`**: executer hỏi ý kiến một advisor độc lập (2 AI) trước khi trả lời.

---

## 1. Cài Claude Code (bản chính thức)

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Kiểm tra:
```bash
claude --version
```

## 2. Cấu hình proxy + key

Tạo / sửa file `~/.claude/settings.json`. Lưu ý:
- `ANTHROPIC_BASE_URL`: **dùng chung** (giống máy gốc).
- `ANTHROPIC_AUTH_TOKEN`: **key RIÊNG của bạn** — tự lấy trên gateway, KHÔNG dùng chung key người khác.
- Tên model phải dạng **dấu chấm** (`claude-opus-4.8`), không phải dấu gạch (`claude-opus-4-8`) — proxy chỉ nhận dạng dấu chấm.

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://zendigikey.shop",
    "ANTHROPIC_AUTH_TOKEN": "<KEY-RIENG-CUA-BAN>",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4.8",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4.6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4.5",
    "SECURITY_REVIEW_MODEL": "claude-opus-4.8",
    "SG_AGENTIC_MODEL": "claude-opus-4.8",
    "API_TIMEOUT_MS": "600000"
  },
  "model": "opus"
}
```

Kiểm tra proxy + key chạy được:
```bash
curl -sS -X POST "$ANTHROPIC_BASE_URL/v1/messages" \
  -H "x-api-key: $ANTHROPIC_AUTH_TOKEN" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4.8","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}' \
  -w "\n[HTTP %{http_code}]\n"
```
HTTP 200 = OK. HTTP 503 `model_not_found` = sai tên model (kiểm lại dấu chấm).

## 3. Lấy bản patch advisor + lệnh /advisor

```bash
git clone https://github.com/quan719/claude-code-proxy.git
cd claude-code-proxy
git checkout fix/agentic-proxy-base-url
```

### 3a. Cài lệnh /advisor
Lệnh slash chỉ chạy khi nằm trong `~/.claude/commands/`:
```bash
mkdir -p ~/.claude/commands
cp .claude/commands/advisor.md ~/.claude/commands/
```

### 3b. Áp patch proxy vào plugin security-guidance
Plugin advisor đọc file `llm.py`. Cần đảm bảo 2 điểm trong file đó:
1. Forward `ANTHROPIC_BASE_URL` xuống tiến trình con (đã có sẵn trong repo này).
2. Cài SDK để bật agentic review:
```bash
python3 -m pip install claude-agent-sdk
```

Đường dẫn plugin sau khi cài Claude Code thường ở:
```
~/.claude/plugins/marketplaces/<marketplace>/plugins/security-guidance/hooks/llm.py
```
So sánh với bản đã patch trong repo:
```
plugins/security-guidance/hooks/llm.py
```
Nếu file đã cài chưa có dòng `env["ANTHROPIC_BASE_URL"] = os.environ["ANTHROPIC_BASE_URL"]`, copy đè bản trong repo sang.

## 4. Dùng

- Khởi động: `claude`
- Quét/hỏi advisor 2-AI: gõ `/advisor <câu hỏi>`
- Advisor bảo mật tự động: chạy nền sau mỗi `git commit` / `git push` (chỉ hiện khi có lỗ hổng).

## Khắc phục sự cố

| Triệu chứng | Nguyên nhân | Sửa |
|---|---|---|
| Advisor im lặng, `403 / model_not_found` | Tên model dạng dấu gạch | Đổi sang `claude-opus-4.8` (dấu chấm) |
| `git diff error: filename too long` (WinError 206 / E2BIG) | Quá nhiều file untracked | Thêm thư mục thừa vào `.gitignore` |
| `agentic_review: SDK unavailable` | Chưa cài SDK | `pip install claude-agent-sdk` |
| `/advisor` không hiện | File chưa ở `~/.claude/commands/` | Copy `advisor.md` vào đó, khởi động lại |
