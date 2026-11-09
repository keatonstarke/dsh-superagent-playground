# Gemini 反代全流程：账号来源、凭证、项目与部署（2026-11）

> 状态：调研稿（v1）
> 采样时间：2026-11
> 前置：`docs/research/gemini-pro-rp-高调用量方案.md`（Pro 路径与成本）；`docs/research/gemini-3.8-flash-rp-落地方案.md`（Flash 落地）
> 本文回答三个问题：**账号从哪来、凭证怎么拿、反代怎么搭。** 目标是把 Antigravity 的 OAuth 额度变成 OpenAI/Claude/Gemini 兼容 API，用于 RP 等高调用量场景。
> 标注：`[官方]` 可查文档；`[项目]` 开源项目 README/源码；`[市场]` 公开报价；`[实测]` 社区复现；`[估算]` 本文推算。

---

## 0. 结论摘要

| 问题 | 结论 |
| --- | --- |
| 最稳的组合 | **官方 Google AI Pro（$19.99/月）+ CLIProxyAPI 单账号**，不碰号池；Argon 够用，成本可预测 |
| 最便宜的组合 | 成品号/车位：Gemini Pro 成品号 **¥5~¥59**、Ultra 车位 **¥250~¥290/月**；便宜但封号/售后风险最高 `[市场]` |
| 主流反代项目 | **CLIProxyAPI**（58k★，Go，:8317）、**AIClient2API**（8.9k★，Node/Docker，:3000）、**Antigravity-Manager**（32.9k★，Rust/GUI，:8045）、**antigravity-proxy**（极简 Python，:8877） |
| 账号单价 | 官方 Pro $19.99/月；PVA Gmail $0.46~$0.65；6 个月成品号 ¥59；短期成品号 ¥5~¥11；Ultra 车位 ¥250~¥290/月 `[市场]` |
| 单号产能 | 2,000~5,000 轮 RP/月（40k 输入、thinking low）`[估算]`；低于 2,196 轮/月时，3.8 Flash 官方 API 更便宜 |
| 封号 | 社区口径 5%~20%/月，灰度期更高；第三方 OAuth 路由有 **4 个账号被封的实证**（2026-11-05~06）`[实测]` |
| 全流程 | 选号 → 验收 → `agy` 授权 → 部署反代 → 配模型 → 压测/额度校准 → 运维替换，共 7 步 |
| 一句话 | **反代的技术门槛不高，真正的成本在"账号寿命"和"协议改版"；能买官方 API 的规模，不要用号池硬扛。** |

### 0.1 最小路径（单号，30 分钟）

```bash
# 1) 官方 AI Pro 账号 + Antigravity CLI 登录
curl -fsSL https://storage.googleapis.com/antigravity-releases/install.sh | bash
agy                      # 浏览器登录
agy models               # 确认有 gemini-4-argon-*

# 2) 下载 CLIProxyAPI，写最小配置
VERSION=8.0.61
mkdir -p /opt/cli-proxy-api && cd /opt/cli-proxy-api
curl -L "https://github.com/router-for-me/CLIProxyAPI/releases/download/v${VERSION}/CLIProxyAPI_${VERSION}_linux_amd64.tar.gz" | tar xz
cat > config.yaml <<'YAML'
config-version: 8
server:
  host: "127.0.0.1"
  port: 8317
management:
  allow-remote: false
  secret-key: "CHANGE_ME"
access:
  api-keys: ["sk-rp-CHANGE_ME"]
oauth:
  auth-dir: "~/.cli-proxy-api"
routing:
  strategy: "weighted-round-robin"
  session-affinity: true
YAML

# 3) 授权并启动
./cli-proxy-api -config config.yaml -antigravity-login -no-browser
./cli-proxy-api -config config.yaml &

# 4) 测试
curl -N http://127.0.0.1:8317/v1/chat/completions \
  -H "Authorization: Bearer sk-rp-CHANGE_ME" \
  -H "Content-Type: application/json" \
  -d '{"model":"gemini-4-argon-medium","stream":true,
       "messages":[{"role":"user","content":"用一句话描写雨夜。"}]}'
```

单号够用就停在这里；要多号/权重/会话粘性，继续 §6。

---

## 1. 反代是什么：四层架构

```
┌────────────────────────────────────────────────────────────┐
│ 4. 网关层   OpenAI/Claude/Gemini 兼容、鉴权、限流、路由、日志 │
├────────────────────────────────────────────────────────────┤
│ 3. 协议层   Antigravity v1internal ⇄ OpenAI/Claude/Gemini    │
├────────────────────────────────────────────────────────────┤
│ 2. 凭证层   OAuth access/refresh token、project id、cookie   │
├────────────────────────────────────────────────────────────┤
│ 1. 账号层   Google 账号 + AI Pro/Ultra 订阅 + 出口 IP         │
└────────────────────────────────────────────────────────────┘
```

**"反代"不是网络代理**，而是把官方客户端（Antigravity CLI/App）的私有协议翻译成标准 API。它依赖三件事：

1. 一个**有订阅**的 Google 账号（Argon 需要 AI Pro 付费订阅，不含试用）`[官方]`。
2. 从官方客户端拿到的 **OAuth 凭证**（access + refresh token）。
3. 一个能持续跟进上游改版的**协议适配层**（项目或自建）。

### 1.1 三种上游对比

| 上游 | 凭证 | 可用模型 | 稳定性 | 适合 |
| --- | --- | --- | --- | --- |
| Antigravity OAuth | `~/.gemini/antigravity-cli/antigravity-oauth-token` | Argon、3.1 Pro、3.8 Flash、Claude | 中（协议会改） | 本文主路线 |
| Gemini Web Cookie | `__Secure-1PSID` / `__Secure-1PSIDTS` | Web 端 Gemini | 低（Cookie 短命） | 临时/实验 |
| Gemini API Key | AI Studio Key | Flash/Pro 按官方 | 高 | 生产/可预测 |

---

## 2. 账号来源

### 2.1 官方订阅（最稳）

| 计划 | 价格 | Antigravity 可用模型 | 备注 |
| --- | --- | --- | --- |
| Free | $0 | 3.8 Flash | 周配额；Argon/3.1 Pro 不可用 |
| Google AI Plus | $4.99 | 3.8 Flash | 2026-11-13 起失去 Gemini Pro 模型 `[市场]` |
| Google AI Pro | $19.99/月，$199.99/年 | **Argon**、3.8 Flash、Claude 5.5 | Argon 仅付费订阅，不含试用 `[官方]` |
| Google AI Ultra 5x | $99.99/月 | 上述 + 3.7/3.6 Flash | 5 小时 + 周配额更高 |
| Google AI Ultra 20x | $199.99/月 | 上述 + **3.1 Pro** | 单位配额最优，优先纵向扩 |
| Enterprise | 合同 | 3.1 Pro 等 | 适合团队/合规 |

**建议**：个人/小团队先买 1 个 AI Pro；要 3.1 Pro 或更高配额再上 Ultra 20x。不要为了省 $20 去买来源不明的成品号——后面替换成本更高。

### 2.2 自建注册（可复制流程）

**准备**：

| 项 | 选择 | 成本 |
| --- | --- | --- |
| 浏览器 | Camoufox / Patchright / 指纹浏览器（一号一 profile） | 免费~$10/月 |
| 出口 IP | 移动代理或静态住宅，一号一 IP，地区与号码一致 | $3.99/GB 或 $7.99/IP/月 `[市场]` |
| 手机号 | 非 VoIP，一号一码，保留 30 天可复用 | $0.85/次，$7.60/30 天 `[市场]` |
| 恢复邮箱 | 独立域名/邮箱，不与主号关联 | 忽略不计 |
| 支付 | 虚拟卡/地区礼品卡/Google Play | 视地区 |

**步骤**：

1. 新建浏览器 profile：时区、语言、UA、屏幕、WebRTC 与代理地区一致；关闭 WebRTC 泄漏，DNS 走代理。
2. 打开 `accounts.google.com/signup`，用真实感姓名、成年生日、唯一强密码；不要用批量生成的规律字符串。
3. 手机验证：用一次性号码；失败换号，不要连续重试同一号码。
4. 设置恢复邮箱 + TOTP 2FA；记录 **账号/密码/恢复邮箱/2FA 密钥/手机号/注册 IP/日期**。
5. **养号 3~7 天**：正常浏览 YouTube/Gmail/搜索，不要当天登录 Gemini 或 Antigravity。
6. 订阅 AI Pro：优先本地支付方式；虚拟卡失败不要连续试，换卡或换区。
7. 登录 Antigravity CLI，确认 `agy models` 里能看到 Argon；跑 20 轮标准 RP。
8. 入池：标记账号、订阅到期日、IP、配额、健康分。

**成本与成功率** `[估算]`：

- 单号首次成本：SMS $0.85 + 代理 $4~$8 + 订阅 $19.99 ≈ **$25~$29**。
- 注册成功率：30%~60%，取决于 IP 信誉、号码质量、浏览器指纹；被要求额外验证的号直接弃用。
- 时间：熟练后 10~15 分钟/号；批量时按 20 分钟/号排。

#### 2.2.1 半自动化注册脚本要点

批量注册不要一上来写全自动；先做"半自动 + 人工过验证"：

1. 一号一 `browser_context`（Playwright/Patchright/Camoufox persistent context），`storage_state` 单独保存。
2. 代理在 context 层设置（`proxy={"server":..., "username":..., "password":...}`），确保 DNS/WebRTC 不泄漏。
3. 身份用模板生成：姓名/生日/密码，避免规律；同一模板不要跨号复用。
4. 表单填完后等待短信：调用接码平台 API 取码，超时 120s 换号；被拒的号码标记黑名单。
5. 遇到 reCAPTCHA/设备验证 → 转人工，不要硬刚；硬刚会拉高整个 IP 段的风险。
6. 成功后立即设置恢复邮箱 + TOTP，保存 `storage_state` 与凭证。
7. 养号任务单独排队：3~7 天后再订阅/登录 Antigravity；不要注册当天就跑 RP。
8. 失败样本记录：IP、号码、UA、失败原因，用来淘汰供应商。

> 不建议把注册、订阅、反代放在同一台机器/同一 IP；账号层和调用层分开，封号时影响面更小。

### 2.3 成品号/车位（便宜，但风险最高）

| 类型 | 价格 `[市场]` | 交付 | 风险 |
| --- | --- | --- | --- |
| PVA Gmail | $0.46~$0.65 | 账号+密码+辅助邮箱 | 不含 AI Pro；可能被找回 |
| Gemini Pro 成品号（6~12 个月） | ¥5.15~¥11 | 账密/首登/JSON | 多为活动/像素机号；封号无售后 |
| Gemini Pro 独立号（6 个月） | ¥59 | 独立账号+2FA | 首登 1 小时质保；不保证 Antigravity |
| Ultra 车位（家庭组） | ¥250~¥290/月 | 加入家庭组 | 共享/拼车违反条款风险；额度独立但可被踢 |
| 官方 Pro | $19.99/月 | 自己的账号 | 最稳 |

**验号要点**：

- 价格明显低于 $20/月的"Pro 年号"，基本是活动号/盗刷号/共享号；随时归零。
- 要求卖家提供：订阅截图、到期日、是否含 Antigravity、是否可改 2FA、首登质保。
- 到手先改密码/2FA/恢复邮箱；不要继续用卖家给的辅助邮箱。
- 先跑 20 轮 RP + `agy models` + `/usage`，通过再付款/确认。

### 2.4 账号验收清单

- [ ] 干净 IP 登录成功，无 "Verify your account"
- [ ] `one.google.com` 显示 AI Pro/Ultra 有效，到期日正确
- [ ] `agy` 登录成功，`agy models` 含目标模型（Argon / 3.1 Pro）
- [ ] `/usage` 显示 5 小时与周配额 > 0
- [ ] 20 轮标准 RP 成功率 > 90%，无 403
- [ ] token 1 小时后能自动刷新
- [ ] 2FA/恢复邮箱已改为自己控制
- [ ] 记录到账号库（加密），标记来源/成本/到期日

### 2.5 支付与地区

- **官方订阅**：外币信用卡、Google Play 余额、地区礼品卡。国内 VISA 通过 Google Play 订阅是社区可行路径之一 `[实测]`。
- **虚拟卡**：Nobepay/Depay 等；失败率高，账单地址/地区要与账号一致。
- **区域定价**：不同国家价格不同；跨区支付/跨区 IP 容易触发风控。
- **家庭组**：Ultra 可 6 人家庭组，成员额度独立；但共享/拼车属于条款灰区，且"不接受反代"是社区共识 `[市场]`。

### 2.6 封号与售后

常见封禁信号 `[实测]`：

| 信号 | 含义 | 处理 |
| --- | --- | --- |
| `403 This service has been disabled in this account for violation of Terms of Service` | 判定违反条款（常见于第三方 OAuth 路由） | 弃号/申诉，不要原地重试 |
| `403 Verify your account to continue` | 需要身份/手机验证 | 人工验证或弃号 |
| `401 invalid_grant` | refresh token 失效 | 重新 `agy` 授权 |
| 连续 `429 RESOURCE_EXHAUSTED` | 配额耗尽 | 冷却/换号 |
| `404 model not found` | 订阅档位不支持该模型 | 换模型/升级订阅 |

**替换节奏**：按 5%~20%/月损耗备号；灰度期（新号前 3 天）单独池，成功率 > 90% 才进主池。

---

## 3. 凭证：从 `agy` 到 token

### 3.1 安装与登录 `[官方/项目]`

```bash
# Linux/macOS 安装 Antigravity CLI
curl -fsSL https://storage.googleapis.com/antigravity-releases/install.sh | bash

# 交互登录（浏览器 OAuth）
agy

# 验证模型列表
agy models
```

登录后凭证在：

```
~/.gemini/antigravity-cli/antigravity-oauth-token
```

字段（JSON）：

```json
{
  "access_token": "ya29....",
  "refresh_token": "1//...",
  "expiry": "2026-11-09T13:00:00.000000Z",
  "token_type": "Bearer"
}
```

> 不同版本字段可能略有差异；以实际文件为准。AIClient2API 使用 `configs/oauth_creds.json`，Antigravity-Manager 使用 `~/.antigravity_tools/gui_config.json`，本质都是同一套 OAuth 凭证。

### 3.2 刷新 token

```bash
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

client_id=<Antigravity CLI 内置 client_id>
client_secret=<项目源码中的 installed-app secret>
refresh_token=<refresh_token>
grant_type=refresh_token
```

- 提前 50 分钟刷新（`REFRESH_SKEW=3000s`）`[项目]`。
- 刷新失败 → 重新 `agy` 登录；不要用同一个 refresh token 高频重试。
- client_id/secret 是官方 CLI 内置的 installed-app 凭证，开源项目里公开可查；不要把它当成用户密钥。

### 3.3 发现 project id

Antigravity 请求必须带 `project`。用 `loadCodeAssist` 拿：

```bash
POST https://cloudcode-pa.googleapis.com/v1internal:loadCodeAssist
Authorization: Bearer <access_token>
Content-Type: application/json
Client-Metadata: {"ideType":"ANTIGRAVITY","platform":"MACOS","pluginType":"GEMINI"}

{}
```

返回里取 `cloudaicompanionProject`；没有则调 `onboardUser`（LRO）再取。`[项目]`

> 有的实现用空 body 成功，有的带 `{"metadata":{"ideType":"ANTIGRAVITY"}}`；以实际返回为准。

### 3.4 凭证库设计

```
auths/
  antigravity-alice@gmail.com.json
  antigravity-bob@gmail.com.json
```

每个文件：

```json
{
  "email": "alice@gmail.com",
  "access_token": "...",
  "refresh_token": "...",
  "expiry": "2026-11-09T13:00:00Z",
  "project_id": "cloudaicompanion-...",
  "plan": "pro",
  "proxy": "http://user:pass@ip:port",
  "weight": 1,
  "status": "active",
  "last_used": "2026-11-09T12:00:00Z"
}
```

- 文件权限 `600`；整库用 age/sops 加密；备份离线。
- 不要把 token 写进日志/前端；管理面板只显示邮箱尾号与配额。
- 一号一文件，便于 CLIProxyAPI/AIClient2API 直接读取。

### 3.5 Web Cookie / API Key（备选）

**Web Cookie**：`__Secure-1PSID` + `__Secure-1PSIDTS`，用 `gemini-webapi` 自动刷新；Chromium 的 Device Bound Session 会让 Cookie 几小时失效，建议 Firefox 取 Cookie `[项目]`。只适合临时实验，不适合 24/7 高调用量。

**API Key**：AI Studio Key + `gemini-balance` 做多 Key 轮询；稳定、可预测，但没有"白嫖"空间。

---

## 4. 协议层：Antigravity v1internal

### 4.1 端点与请求头 `[项目]`

| 项 | 值 |
| --- | --- |
| Base URL | `https://cloudcode-pa.googleapis.com`（备用 `https://daily-cloudcode-pa.googleapis.com`） |
| 方法 | `POST /v1internal:loadCodeAssist`、`:onboardUser`、`:generateContent`、`:streamGenerateContent?alt=sse` |
| Authorization | `Bearer <access_token>` |
| Content-Type | `application/json` |
| User-Agent | `antigravity/cli/1.4.2 darwin/arm64`（Argon 必须 CLI UA；版本会变） |
| Client-Metadata | `{"ideType":"ANTIGRAVITY","platform":"MACOS","pluginType":"GEMINI"}` |
| X-Goog-Api-Client | `google-cloud-sdk vscode_cloudshelleditor/0.1` |

> UA / metadata 是上游识别客户端的关键；用 IDE 的 UA 可能拿不到 Argon（Argon 只在 App/CLI/IDE 扩展）`[项目]`。

### 4.2 请求 envelope

```json
{
  "project": "cloudaicompanion-xxxx",
  "model": "gemini-3.8-flash-low",
  "requestType": "agent",
  "userAgent": "antigravity",
  "requestId": "agent-<uuid>",
  "request": {
    "contents": [
      {"role": "user", "parts": [{"text": "..."}]}
    ],
    "systemInstruction": {"parts": [{"text": "..."}]},
    "generationConfig": {
      "thinkingConfig": {"thinkingLevel": "low", "includeThoughts": false},
      "maxOutputTokens": 1600
    }
  }
}
```

模型名（上游）：

- `gemini-4-argon-medium` / `gemini-4-argon-high`
- `gemini-3.1-pro-low` / `gemini-3.1-pro-high`
- `gemini-3.8-flash-low` / `-medium` / `-high`
- `gemini-3.6-flash-*`
- `claude-sonnet-4-6`、`claude-opus-4-6-thinking`

### 4.3 SSE 响应

标准 Gemini 形状：

```json
{"candidates":[{"content":{"parts":[{"text":"..."}]},"finishReason":"STOP"}],"usageMetadata":{...}}
```

翻译成 OpenAI：

```json
{"choices":[{"delta":{"content":"..."},"index":0,"finish_reason":null}]}
```

### 4.4 错误码

| HTTP | 上游信息 | 原因 | 处理 |
| --- | --- | --- | --- |
| 400 | `INVALID_ARGUMENT` | payload/模型/项目不匹配 | 检查 envelope、模型名、project |
| 401 | `UNAUTHENTICATED` | access token 过期 | 刷新；失败则重新授权 |
| 403 | `PERMISSION_DENIED` / ToS | 无权限/封号 | 换号；ToS 号弃用 |
| 404 | `NOT_FOUND` | 模型不在订阅档位 | 换模型/升级 |
| 429 | `RESOURCE_EXHAUSTED` | 5h/周配额或 RPM | 冷却、换号、退避 |
| 500/503 | 上游 | 容量 | 退避重试/换 base URL |

### 4.5 最小 Python 反代骨架

> 仅示意协议转换；生产请用 §5 的项目，别自己维护边界情况。

```python
import json, time, uuid
from datetime import datetime
import httpx
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

TOKEN_FILE = "/home/rp/.gemini/antigravity-cli/antigravity-oauth-token"
CLIENT_ID = "<antigravity-cli-client-id>"
CLIENT_SECRET = "<antigravity-cli-client-secret>"
BASE = "https://cloudcode-pa.googleapis.com"
UA = "antigravity/cli/1.4.2 darwin/arm64"
META = '{"ideType":"ANTIGRAVITY","platform":"MACOS","pluginType":"GEMINI"}'
app = FastAPI()
state = {"project": None, "token": None, "exp": 0}

def read_token():
    return json.load(open(TOKEN_FILE))

def access_token():
    now = time.time()
    if state["token"] and state["exp"] - now > 120:
        return state["token"]
    tok = read_token()
    exp = datetime.fromisoformat(tok["expiry"].replace("Z", "+00:00")).timestamp()
    if tok.get("access_token") and exp - now > 120:
        state.update(token=tok["access_token"], exp=exp)
        return state["token"]
    r = httpx.post("https://oauth2.googleapis.com/token", data={
        "client_id": CLIENT_ID, "client_secret": CLIENT_SECRET,
        "refresh_token": tok["refresh_token"], "grant_type": "refresh_token",
    })
    r.raise_for_status()
    data = r.json()
    state.update(token=data["access_token"], exp=now + data.get("expires_in", 3600))
    return state["token"]

def headers():
    return {"Authorization": f"Bearer {access_token()}", "Content-Type": "application/json",
            "User-Agent": UA, "Client-Metadata": META}

def project_id():
    if state["project"]:
        return state["project"]
    r = httpx.post(f"{BASE}/v1internal:loadCodeAssist", headers=headers(), json={})
    r.raise_for_status()
    state["project"] = r.json()["cloudaicompanionProject"]
    return state["project"]

def to_gemini(messages):
    contents, system = [], []
    for m in messages:
        if m["role"] == "system":
            system.append(m["content"])
        else:
            contents.append({"role": "user" if m["role"] == "user" else "model",
                             "parts": [{"text": m["content"]}]})
    si = {"parts": [{"text": "\n\n".join(system)}]} if system else None
    return contents, si

@app.post("/v1/chat/completions")
async def chat(body: dict):
    contents, si = to_gemini(body["messages"])
    envelope = {
        "project": project_id(), "model": body["model"], "requestType": "agent",
        "userAgent": "antigravity", "requestId": f"agent-{uuid.uuid4().hex}",
        "request": {"contents": contents, "generationConfig": {
            "thinkingConfig": {"thinkingLevel": "low", "includeThoughts": False},
            "maxOutputTokens": body.get("max_tokens", 1600)}},
    }
    if si:
        envelope["request"]["systemInstruction"] = si

    async def stream():
        async with httpx.AsyncClient(timeout=None) as client:
            async with client.stream("POST", f"{BASE}/v1internal:streamGenerateContent?alt=sse",
                                     headers=headers(), json=envelope) as r:
                r.raise_for_status()
                async for line in r.aiter_lines():
                    if not line.startswith("data:"):
                        continue
                    chunk = json.loads(line[5:].strip())
                    for cand in chunk.get("candidates", []):
                        for part in cand.get("content", {}).get("parts", []):
                            if part.get("text"):
                                yield "data: " + json.dumps(
                                    {"choices": [{"delta": {"content": part["text"]}, "index": 0}]},
                                    ensure_ascii=False) + "\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(stream(), media_type="text/event-stream")
```

**骨架缺什么**：工具调用、多模态、usage、finish_reason、重试/退避、账号池、鉴权、日志脱敏。生产直接用 §5 的项目。

---

## 5. 反代项目选型

| 项目 | 语言/部署 | 默认入口 | 账号模型 | 协议 | 适合 | 注意 |
| --- | --- | --- | --- | --- | --- | --- |
| **CLIProxyAPI**（58k★） | Go，二进制/Docker | `:8317` | 多 OAuth 账号、权重、会话粘性 | OpenAI / Gemini / Claude / Codex | 生产号池 | 功能多，配置略重 |
| **AIClient2API**（8.9k★） | Node/Docker | `:3000`（OAuth 8086） | Provider Pool、轮询/故障转移 | OpenAI / Claude / Gemini | 想要 Web UI 的小团队 | 需要挂多个回调端口 |
| **Antigravity-Manager**（32.9k★） | Rust/Tauri + Docker | `:8045` | 账号切换、配额面板 | OpenAI / Anthropic | 桌面/单机多号 | GUI 为主，自动化弱 |
| **antigravity-proxy**（9★） | Python stdlib | `:8877` | 单 CLI token | OpenAI + Gemini 原生 | 单号极简 | 无鉴权/无池；仅映射 3.1 Pro/3.5 Flash |
| **gemini-balance**（5.8k★） | Python | `:8000` | 官方 API Key 池 | OpenAI / Gemini | 官方 Key 轮询 | 不接 OAuth 号 |
| **OmniRoute**（85.8k★） | 多语言网关 | 自定义 | 多 provider | 多协议 | 多模型聚合 | 太宽，非 Antigravity 专用 |
| **Gemini-API**（3.8k★） | Python | SDK | Web Cookie | 自有 | 临时实验 | Cookie 短命 |

### 5.1 推荐组合

- **单号/个人**：`antigravity-proxy` 或 CLIProxyAPI 单账号；绑定 `127.0.0.1`。
- **3~20 号/小团队**：CLIProxyAPI + 静态住宅 IP + 权重轮询 + 会话粘性；AIClient2API 做可视化管理。
- **50+ 号/产品**：不建议；用官方 API/Vertex。若必须，CLIProxyAPI + 专职工信 + 自动替换 + 法务评估。
- **只要 3.8 Flash**：直接官方 API（$0.0091/轮），比号池简单。

---

## 6. 部署全流程（CLIProxyAPI）

### 6.1 服务器

- 2C2G 起，Ubuntu 22.04/24.04；磁盘 20GB。
- 出口：与账号注册地一致；一号一静态住宅/移动 IP（项目支持 per-credential `proxy-url`）。
- 端口：只开 SSH；`8317` 绑 `127.0.0.1`，通过 Caddy/Nginx 或 SSH 隧道对外。

### 6.2 安装

```bash
VERSION=8.0.61
curl -L -o /tmp/cpa.tar.gz \
  "https://github.com/router-for-me/CLIProxyAPI/releases/download/v${VERSION}/CLIProxyAPI_${VERSION}_linux_amd64.tar.gz"
mkdir -p /opt/cli-proxy-api && tar xzf /tmp/cpa.tar.gz -C /opt/cli-proxy-api
cp /opt/cli-proxy-api/config.example.yaml /opt/cli-proxy-api/config.yaml
/opt/cli-proxy-api/cli-proxy-api --help

# 建服务用户并授权（后面 OAuth 登录要用同一用户）
sudo useradd -r -m -d /var/lib/cpaproxy cpaproxy || true
sudo chown -R cpaproxy:cpaproxy /opt/cli-proxy-api /var/lib/cpaproxy
```

### 6.3 `config.yaml` 最小配置

```yaml
config-version: 8

server:
  host: "127.0.0.1"
  port: 8317

management:
  allow-remote: false
  secret-key: "CHANGE_ME_MANAGEMENT_KEY"

access:
  api-keys:
    - "sk-rp-CHANGE_ME"

oauth:
  auth-dir: "~/.cli-proxy-api"

routing:
  strategy: "weighted-round-robin"   # round-robin / weighted-round-robin / fill-first
  session-affinity: true             # 同一会话粘同一账号，保缓存/口吻
  session-affinity-ttl: "1h"
  retry:
    request-retry: 3
    max-retry-interval: 30
```

### 6.4 添加账号

```bash
# 交互 OAuth；-no-browser 会打印 URL，适合无桌面服务器
# 注意：用运行服务的同一用户执行，否则凭证会落到 root 的 home
sudo -u cpaproxy /opt/cli-proxy-api/cli-proxy-api \
  -config /opt/cli-proxy-api/config.yaml -antigravity-login -no-browser

# 授权完成后，凭证落到 auth-dir（默认 ~/.cli-proxy-api）
ls -l /var/lib/cpaproxy/.cli-proxy-api/
```

> 也可以把 `oauth.auth-dir` 写成绝对路径（如 `/var/lib/cpaproxy/.cli-proxy-api`），避免 home 展开不一致。

多账号重复执行；每个账号的 OAuth 回调端口可用 `-oauth-callback-port` 覆盖。

### 6.5 启动与 systemd

```ini
# /etc/systemd/system/cli-proxy-api.service
[Unit]
Description=CLIProxyAPI
After=network-online.target

[Service]
User=cpaproxy
WorkingDirectory=/opt/cli-proxy-api
ExecStart=/opt/cli-proxy-api/cli-proxy-api -config /opt/cli-proxy-api/config.yaml
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cli-proxy-api
sudo journalctl -u cli-proxy-api -f
```

### 6.6 测试

```bash
KEY=sk-rp-CHANGE_ME

curl -s http://127.0.0.1:8317/v1/models \
  -H "Authorization: Bearer $KEY" | jq

curl -N http://127.0.0.1:8317/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-3.8-flash-low",
    "stream": true,
    "messages": [{"role":"user","content":"用一句话描写雨夜的港口。"}]
  }'
```

### 6.7 客户端接入

| 客户端 | Base URL | Key | 模型 |
| --- | --- | --- | --- |
| OpenAI SDK | `http://127.0.0.1:8317/v1` | `sk-rp-...` | `gemini-3.8-flash-low` |
| SillyTavern | 同上（Chat Completion / Custom） | 同上 | `gemini-4-argon-medium` 或 `gemini-3.8-flash-low` |
| Claude Code | `http://127.0.0.1:8317`（Anthropic 路由） | 同上 | `claude-sonnet-4-6` |
| Gemini SDK | `http://127.0.0.1:8317`（Gemini 路由） | 同上 | `gemini-3.1-pro-high` |

### 6.8 号池与路由

- **权重**：按账号剩余配额/历史成功率给 `weight`（1~1,000,000）；新号先 1，稳定后升。
- **会话粘性**：`session-affinity: true`，同一 `conversation_id` 固定账号，避免口吻漂移、提高缓存命中。
- **冷却**：429/5xx 自动冷却；401 触发刷新；403 标记 banned。
- **配额采集**：管理 API/`/usage` 定时拉 5h/周剩余，写 Redis；低于 20% 降权。
- **故障转移**：`request-retry: 3` + 多账号；不要无限重试同一号。

### 6.9 安全加固

```bash
# 只监听 localhost；远程用 Caddy + TLS + IP allowlist
```

```caddyfile
rp.example.com {
    reverse_proxy 127.0.0.1:8317
    @blocked not remote_ip 203.0.113.0/24
    respond @blocked 403
}
```

- 管理 API `secret-key` 必须改；`allow-remote: false`。
- 客户端 Key 一人一把；定期轮换；日志不记录 prompt/正文。
- 凭证库加密；容器/主机最小权限；禁止把 `8317` 裸奔公网。
- 备份 `~/.cli-proxy-api` 与 `config.yaml`；升级前快照。

### 6.10 升级/回滚

```bash
# 新版本先解压到 /opt/cli-proxy-api-next，改 systemd ExecStart 灰度
# 验证 /v1/models + 20 轮 RP + 配额刷新后，再切正式
```

---

## 7. AIClient2API 路线（Web UI）

```bash
docker run -d --name aiclient2api \
  -p 3000:3000 -p 8086:8086 -p 1455:1455 -p 56121:56121 -p 19876-19880:19876-19880 \
  --restart=always \
  -v /srv/aiclient2api/configs:/app/configs \
  -v /srv/aiclient2api/plugins:/app/src/plugins-user \
  -v /srv/aiclient2api/.private:/app/.private \
  justlikemaki/aiclient-2-api
```

- Web UI：`http://<host>:3000`；Antigravity OAuth 回调端口 `8086`。
- 授权：Provider Pools → Antigravity → Generate Authorization → Google 登录。
- Provider Pool：`configs/provider_pools.json`，可多账号轮询/故障转移。
- 端点：`http://<host>:3000/gemini-antigravity/v1/chat/completions`；Key 在 Web UI 配置。
- 模型：`gemini-4-argon-medium`、`gemini-3.1-pro-high`、`gemini-3.8-flash-low` 等。
- 适合：不想写 YAML、要可视化管理；缺点：回调端口多、容器较重。

**Antigravity-Manager 路线**：

```bash
docker run -d --name antigravity-manager \
  -p 8045:8045 \
  -e API_KEY=sk-antigravity \
  -e WEB_PASSWORD=change-me \
  -v ~/.antigravity_tools:/root/.antigravity_tools \
  lbjlaq/antigravity-manager:latest
```

- Web UI：`http://<host>:8045`；API：`http://<host>:8045/v1`。
- 账号管理/配额面板在 GUI；适合桌面单机多号，不适合无头批量。

---

## 8. RP 接入

### 8.1 SillyTavern / OpenAI 兼容客户端

| 项 | 值 |
| --- | --- |
| API | Chat Completion → Custom (OpenAI-compatible) |
| Base URL | `http://127.0.0.1:8317/v1` |
| API Key | `sk-rp-...` |
| Model | `gemini-4-argon-medium`（Pro）/ `gemini-3.8-flash-low`（便宜）/ `gemini-3.1-pro-high`（Ultra）；以 `/v1/models` 实际列表为准 |
| Streaming | 开 |
| Max tokens | 1,200~1,800 |
| Temperature | **不要传**（2026-07 起废弃） |
| Context | 20k~30k；每 15 轮摘要 |

### 8.2 反代层的 RP 适配

- **模型别名即 thinking 档**：OpenAI 协议没有 `thinking_level`，用 `-low/-medium/-high` 后缀选择；RP 用 `-low`。
- **system prompt**：放 `messages[0]` 的 system；反代会转成 `systemInstruction`；长 canon 放前面，便于上游缓存。
- **会话 ID**：同一角色/会话固定 `conversation_id`，配合 `session-affinity` 固定账号。
- **超时**：流式首 token 20s、总 300s；长正文分段。
- **重试**：429 退避 + 换号；403 不重试；空流重试一次。
- **采样参数**：在反代层丢弃 `temperature` / `top_p` / `top_k`（2026-07 起废弃）；SillyTavern 侧即使发送也不会传给上游。

### 8.3 质量

- Argon 定位是 coding/agent，RP 文风需自测；3.8 Flash 的 RP 证据见 `gemini-3.8-flash-rp-落地方案.md`。
- 账本/摘要/反全知/复读检测照前两篇执行；反代只解决"通道"，不解决"写法"。

---

## 9. 运维

### 9.1 指标

| 指标 | 采集 | 告警 |
| --- | --- | --- |
| 账号可用率 | 每分钟 1 token 探活 | < 90% |
| 401/403/429 | 反代日志 | 403 > 1%/5min |
| 配额剩余 | `/usage` / 管理 API | < 20% |
| 首 token | 反代埋点 | P95 > 5s |
| 成功率 | 2xx / 总请求 | < 95% |
| 单轮成本 | token × 账号摊销 | 超预算 |

### 9.2 日常

- 每天：看 403/429、配额、账号到期日。
- 每周：token 刷新演练、备份、模型列表变更。
- 每月：成本复盘、替换损耗、协议升级（上游改版）。

### 9.3 封号替换 runbook

1. 403/401 连续失败 → 标记 `banned`，摘流量。
2. 备份该号凭证/日志（脱敏）。
3. 补号：官方订阅/成品号/自建；先灰度 20 轮。
4. 授权 `agy` → 导入反代 → 权重 1 → 观察 24h。
5. 稳定后升权重；旧号删除/申诉。

---

## 10. 成本与产能

### 10.1 单号成本 `[估算/市场]`

| 类型 | 月成本 | 预期 RP 轮/月 | 单轮成本 | 风险 |
| --- | --- | --- | --- | --- |
| 官方 AI Pro | $19.99 | 2,000~5,000 | $0.004~$0.010 | 低 |
| 官方 Ultra 20x | $199.99 | 40,000~100,000 | $0.002~$0.005 | 低 |
| 6 个月成品号 | ¥59（≈$8.3） | 2,000~4,000 | $0.002~$0.004 | 高（封号/找回） |
| 短期成品号 | ¥5~¥11 | 1,000~3,000 | <$0.001 | 极高（无售后） |
| Ultra 车位 | ¥250~¥290 | 20,000~50,000 | $0.005~$0.015 | 中高（拼车/踢人） |
| 3.8 Flash API | 按量 | 无限 | $0.0091 | 低 |

### 10.2 盈亏平衡

```
Pro 订阅 vs 3.8 Flash API：19.99 / 0.0091 ≈ 2,196 轮/月
Ultra 20x vs 3.8 Flash API：199.99 / 0.0091 ≈ 21,977 轮/月
Pro 订阅 vs 3.1 Pro API：19.99 / 0.026 ≈ 769 轮/月
```

- 账号月产低于平衡点 → 直接用 API。
- 账号月产高于平衡点 3 倍以上，才值得承担号池运维。

### 10.3 规模示例

| 日轮次 | 月轮次 | 需要 Pro 号（3k/月） | 需要 Ultra 20x（60k/月） | API 3.8 Flash |
| --- | --- | --- | --- | --- |
| 1,000 | 30k | 10 | 1 | $272 |
| 20,000 | 600k | 200 | 10 | $5,445 |
| 200,000 | 6M | 2,000 | 100 | $54,450 |

> 200 个号已经是"运维号池"而不是"配置一下"；2000 个号不建议。规模上来后，API/Vertex 更省心。

---

## 11. 风险与合规

| 风险 | 说明 | 缓解 |
| --- | --- | --- |
| 条款风险 | Google 条款限制账号买卖/共享、非官方客户端自动化、规避配额；可随时封禁 | 只用自己的号；不对外转售；关键业务用官方 API |
| 封号 | 第三方 OAuth 路由已有 4 号封禁实证 `[实测]`；403 ToS/Verify | 官方客户端；一号一 IP；灰度；备号 |
| 凭证泄露 | token 可读写账号；日志/前端泄露 | 加密、最小权限、脱敏 |
| 数据隐私 | RP 内容经反代；共享池可能被看 | 自建、不过第三方池、日志不存正文 |
| 供应链 | 成品号可能来自盗刷/活动滥用；被找回 | 改密/2FA；不买明显低价号 |
| 协议改版 | UA/端点/字段随客户端更新 | 订阅项目 release；灰度；保留 API 回退 |
| 支付 | 虚拟卡失败/chargeback | 地区匹配；备用卡；不频繁试卡 |

**对外产品**：用官方 API/Vertex；反代只做内部/自用/实验。把"随时归零"写进设计：灰度、冗余、API 兜底。

---

## 12. 待验证

| 项 | 方法 | 优先级 |
| --- | --- | --- |
| 单号真实 RP 月产能 | 连续 7 天跑 40k/900 轮，记录 `/usage` | 高 |
| 封号率与 IP 类型的关系 | 静态住宅 vs 移动 vs 机房，各 10 号 30 天 | 高 |
| 成品号存活周期 | 买 20 个 ¥5~¥11 号，记录首登/7 天/30 天存活 | 高 |
| CLIProxyAPI 权重路由效果 | 10 号池，对比轮询/加权 | 中 |
| 显式缓存/会话粘性的命中率 | 同会话固定账号 vs 轮询 | 中 |
| 上游 UA/字段变更频率 | 每周 diff 客户端 release + 项目 issue | 中 |
| Ultra 车位实际配额 | 车位号跑 `/usage` 一周 | 中 |

---

## 附录 A. 命令清单

```bash
# 1. 安装/登录 Antigravity CLI
curl -fsSL https://storage.googleapis.com/antigravity-releases/install.sh | bash
agy
agy models

# 2. 部署 CLIProxyAPI
VERSION=8.0.61
curl -L -o /tmp/cpa.tar.gz \
  "https://github.com/router-for-me/CLIProxyAPI/releases/download/v${VERSION}/CLIProxyAPI_${VERSION}_linux_amd64.tar.gz"
mkdir -p /opt/cli-proxy-api && tar xzf /tmp/cpa.tar.gz -C /opt/cli-proxy-api
cp /opt/cli-proxy-api/config.example.yaml /opt/cli-proxy-api/config.yaml
/opt/cli-proxy-api/cli-proxy-api -config /opt/cli-proxy-api/config.yaml -antigravity-login -no-browser
systemctl enable --now cli-proxy-api

# 3. 测试
curl -s http://127.0.0.1:8317/v1/models -H "Authorization: Bearer $KEY" | jq
curl -N http://127.0.0.1:8317/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"gemini-3.8-flash-low","stream":true,
       "messages":[{"role":"user","content":"用一句话描写雨夜。"}]}'
```

## 附录 B. 账号库模板

```json
{
  "email": "rp01@example.com",
  "plan": "pro",
  "expires": "2026-12-09",
  "source": "official",
  "cost_usd": 19.99,
  "proxy": "http://user:pass@198.51.100.10:8080",
  "project_id": "cloudaicompanion-xxxx",
  "auth_file": "auths/antigravity-rp01.json",
  "weight": 1,
  "status": "active",
  "notes": "2026-11-09 首登，20 轮通过"
}
```

## 附录 C. 来源

- Antigravity 计划/模型：<https://antigravity.google/docs/plans>、<https://antigravity.google/docs/models>
- Antigravity CLI headless：<https://antigravity.google/docs/cli/headless>
- CLIProxyAPI：<https://github.com/router-for-me/CLIProxyAPI>、`config.example.yaml`
- AIClient2API：<https://github.com/justlovemaki/AIClient2API>
- Antigravity-Manager：<https://github.com/lbjlaq/Antigravity-Manager>
- antigravity-proxy（极简）：<https://github.com/usamashehab/antigravity-proxy>
- gemini-balance：<https://github.com/snailyp/gemini-balance>
- Gemini-API（Web Cookie）：<https://github.com/HanaokaYuzu/Gemini-API>
- OmniRoute 封号 issue：<https://github.com/cheaper-inference/OmniRoute/issues/19266>
- 账号市场：<https://priceai.cc/products/gemini-pro-year>、<https://accsmarket.com/en/catalog/gmail>、<https://www.woaimaihao.com/gemini-pro-account>
- SMS/代理价格：<https://voidmob.com/sms/platforms/google>、<https://proxy302.com/pricing/>
- Ultra 车位：<https://global.v2ex.co/t/1198520>
- Google AI Pro 价格：<https://www.activepieces.com/blog/google-ai-pro-subscription-price-and-limits-in-2026>
- AI Plus 失去 Pro：<https://www.notebookcheck.net/Google-AI-Plus-loses-Gemini-Pro-models-on-November-13.1445681.0.html>
