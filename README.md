# CloudToLocal · 双模式 AI 网关

> 把 **DeepSeek 免费网页版** + **本地 Ollama** 统一成你本机的 OpenAI 兼容接口。
> **纯标准库、零第三方依赖、低内存、流式转发不缓冲。**

---

## 能解决什么

| 痛点 | CloudToLocal 的方案 |
| --- | --- |
| WorkBuddy / Cherry Studio / Continue 等只认 OpenAI 接口，但你想白嫖 DeepSeek 网页免费版 | 网关把网页版内部接口伪装成 `/v1/chat/completions`，客户端零改动 |
| 有些客户端（Open WebUI、Continue）只认 Ollama 原生接口 | 网关同时实现 Ollama 原生 REST，把云端模型"变成"本地模型 |
| 网页版没有原生工具调用（function calling） | 网关 5 层鲁棒解析，把文本工具调用还原成标准 `tool_calls` |
| 网页免费版单对话有轮次/长度上限，超限就回收 | 自动开新对话 + 同步上下文 + 无感重试 |
| 重启/退出后旧会话堆积在账号里 | 启动即清理自建会话，绝不碰你手动创建的对话 |

---

## 两种模式

```
                    ┌─────────────────────────────────────┐
   任意 OpenAI 客户端 ──►  http://127.0.0.1:8787/v1  ──►  CloudToLocal 网关
   (WorkBuddy / 脚本)    (或 OLLAMA_HOST=127.0.0.1:8787)   │
                                                         │ 路由判断 model 名
                                                         ▼
                          ┌──────────────────────┬──────────────────────┐
                          │ 本地 Ollama (11434)  │  DeepSeek 网页版(云端) │
                          │ 离线 / 不耗额度       │  免登录令牌(浏览器提取) │
                          └──────────────────────┴──────────────────────┘
```

1. **云端白嫖（免费）**：本机 Chrome / Edge 登录过 `chat.deepseek.com` 即可。工具自动从浏览器
   Local Storage 读取 `userToken`，无需手动复制。推理在云端，本机不加载权重。
   模型名：`deepseek-chat` / `deepseek-reasoner` / `deepseek-search`。
2. **本地模型（Ollama）**：请求的 `model` 名在本地 Ollama 已安装时直接走本地推理。
   可通过 `POST /api/pull` 一键拉取。

**路由规则**：同一个 `/v1/chat/completions`，`model` 在本地 Ollama 列表 → 本地；否则 → 云端。

---

## 快速开始

### 0. 依赖

- Python 3.9+（仅标准库，无需 `pip install`）
- Node.js（可选，PoW 求解；缺失时回退纯 Python）
- Ollama（可选，仅本地模型需要）

### 1. 启动

```bash
# 用浏览器先登录一次 chat.deepseek.com（用于提取云端令牌）

python cloudtolocal.py
# 自定义端口：
CL_PORT=9000 python cloudtolocal.py
```

启动后监听 `http://127.0.0.1:8787`。

### 2. 接入任意 OpenAI 客户端

把客户端的 `base_url` 指向 `http://127.0.0.1:8787/v1`，`api_key` 填任意值（默认 `sk-cloudtolocal`）。

或用 Ollama 原生模式（适合只认 Ollama 的客户端）：

```bat
:: ollama_bridge.bat 已设好 OLLAMA_HOST
set OLLAMA_HOST=127.0.0.1:8787
ollama list          :: 能看到 deepseek-chat 等"本地模型"
ollama run deepseek-chat
```

### 3. 测试

```bash
curl http://127.0.0.1:8787/v1/models
curl http://127.0.0.1:8787/health
```

---

## 配置

复制 `config.example.json` 为 `config.json` 后填写（**所有字段均可选**）：

```jsonc
{
  "host": "127.0.0.1",
  "port": 8787,
  "api_key": "sk-cloudtolocal",
  "token": "",            // 手动填云端令牌（不填则自动从浏览器提取）
  "cookies": "",
  "ollama_enabled": true,
  "ollama_base_url": "http://localhost:11434",
  "ollama_native_api": true,
  "max_turns_per_conv": 30,
  "max_conv_chars": 400000,
  "sync_context": true,
  "delete_old_sessions": true,
  "dedupe_window": 45,
  "rate_limit_max_retries": 3,
  "rate_limit_backoff": [8, 16, 32],
  "cleanup_on_start": true
}
```

字段说明见文件内 `_comment`。

---

## 工具调用（Agent / Function Calling）

DeepSeek 网页版没有原生 function calling，模型把工具调用以文本标记输出。网关用 **5 层鲁棒解析**（命中即返回）：

1. `<<TOOL_CALLS>>` 包起来的 JSON（**数组或单对象**均可，自动修尾逗号）
2. DeepSeek 原生标记 `<｜tool_call_name｜>/<｜tool_call_args｜>`（全角/半角兼容）
3. ` ```toolcall ` / ` ```json ` 代码围栏
4. `<tool_call>{...}</tool_call>` XML 风格（Qwen 系）
5. 全文 JSON 扫描兜底（抗嵌套截断；支持 `{"function": {...}}` 嵌套形式）

解析结果还原成标准 OpenAI `tool_calls`（`finish_reason="tool_calls"`）。带 `tools` 时网关会缓冲整段回复再判定，**原始标记在输出前被强制剔除，绝不泄漏给用户**；若解析失败，原始输出会落盘到 `debug_tool_raw.log` 便于诊断。

---

## 管理端点

| 方法 / 路径 | 说明 |
| --- | --- |
| `GET /health` | 看会话状态、创建数 |
| `POST /v1/conversation/reset` | 手动开新对话 |
| `POST /v1/conversation/cleanup` | 清理网关自建的残留对话 |
| `GET /v1/conversations/all` | 列出账号下全部对话（含手动创建的） |
| `POST /v1/conversation/wipe` | 清理网关自建会话后重建一个干净对话（**绝不删你的手动对话**） |

> `wipe` 需带确认体：`{"confirm":"WIPE_ALL_MY_CONVERSATIONS"}`。

兼容性：`/v1/chat/completions`、`/chat/completions`、`/v1/responses`、`/v1` 均可用；请求体支持
chunked / gzip / deflate / 无 `Content-Length`；流式显式 `Connection: close`。

---

## 会话与上下文

- 云端请求自动复用同一 DeepSeek 对话（`parent_message_id` 串接上下文），不每次新建。
- 单对话达到 `max_turns_per_conv` 或服务端拒绝超长时，自动开新对话并记录到 `conversations.log`。
- 轮换时自动删除上游旧对话，避免账号堆积空会话。
- 每次启动先清理上次运行残留的自建会话（`cleanup_on_start` 可关）。
- **上下文裁剪**：新对话首轮只注入 `system` + 最近 `first_turns_keep` 轮对话 + 内部历史摘要，
  更早内容由历史摘要补齐，避免客户端全量历史（实测 130K+ 字符）撑爆单对话预算导致死循环。

---

## 项目结构

```
cloudtolocal.py      # 主程序（单文件、标准库、约 3.4K 行）
config.example.json  # 配置样例（复制为 config.json 填写）
pow_solver.js        # Node.js PoW 求解（缺失时回退纯 Python）
start.bat            # Windows 启动脚本
ollama_bridge.bat    # 设好 OLLAMA_HOST 后启动 Ollama 客户端的脚本
install_ollama.bat   # 本地模型一键安装引导
```

---

## 免责声明

本工具为个人学习用途。云端路径调用的是 DeepSeek 网页免费版**内部**接口，可能违反其服务条款，
存在账号风控 / 限流风险；凭证随会话过期，失效后重新登录浏览器即可。请**合规、克制**使用。
本地 Ollama 路径为完全离线的自有模型推理，无此风险。

---

## License

[MIT](./LICENSE) © 2026 wxrjd321
