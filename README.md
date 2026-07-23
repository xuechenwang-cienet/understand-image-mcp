# understand_image MCP

一个调用任意 OpenAI 兼容多模态大模型来理解图片内容的 MCP server。

## 功能

- 工具 `understand_image`，两个必填参数：
  - `image_source`：图片地址，支持 HTTP/HTTPS URL 或本地文件路径
  - `prompt`：向模型提出的问题或指令
- 可选参数：`max_tokens`（默认 1024）、`temperature`（默认 0.2）
- 返回模型的文本回答

## 支持的模型

任何提供 OpenAI 兼容 `chat/completions` 接口的多模态模型，例如：

| 提供方 | LLM_API_URL | LLM_MODEL_NAME |
|--------|-------------|----------------|
| OpenAI | `https://api.openai.com/v1` | `gpt-4o` / `gpt-4o-mini` |
| 通义千问 | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `qwen-vl-max` / `qwen2.5-vl-72b-instruct` |
| DeepSeek | `https://api.deepseek.com/v1` | `deepseek-vision` |
| Ollama（本地） | `http://localhost:11434/v1` | `llama3.2-vision` |
| vLLM / LM Studio | `http://localhost:1234/v1` | 对应模型名 |

## 安装

需要 Python 3.10+。任选一种方式：

**A. uv（推荐，自动装依赖）**

```bash
cd understand-image-mcp
uv run --with mcp --with httpx --with pydantic python3 -c "import server; print('ok')"
```

**B. venv**

```bash
cd understand-image-mcp
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

## 配置

复制 `.env.example` 为 `.env` 并填入你的配置，或直接通过环境变量设置：

```bash
export LLM_API_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
export LLM_MODEL_NAME=qwen-vl-max
export LLM_API_KEY=sk-xxxx
```

三个必填变量缺一不可：`LLM_API_URL`、`LLM_MODEL_NAME`、`LLM_API_KEY`。
本地无需鉴权的服务（如 Ollama）可将 key 设为任意非空占位符。

可选变量：

| 变量 | 默认 | 说明 |
|------|------|------|
| `LLM_FETCH_REMOTE` | `true` | 远程 URL 是否在本地抓取后转 base64 再发给模型。默认开启——这是**阿里云 Token Plan 等不支持远程 URL 的提供商所必需**的；对 OpenAI 等能自行抓图的服务仍可正常工作。设为 `false` 则把 URL 原样传给模型。 |
| `LLM_IMAGE_MAX_BYTES` | `26214400` (25MB) | 远程/本地图片最大字节数，超出则报错。 |

## 在 opencode 中使用

编辑 `~/.config/opencode/opencode.json`（或项目内 `opencode.json`），加入：

```json
{
  "mcp": {
    "understand_image": {
      "type": "local",
      "command": [
        "uv", "run", "--with", "mcp", "--with", "httpx", "--with", "pydantic",
        "python3", "/ABSOLUTE/PATH/TO/understand-image-mcp/server.py"
      ],
      "environment": {
        "LLM_API_URL": "https://dashscope.aliyuncs.com/compatible-mode/v1",
        "LLM_MODEL_NAME": "qwen-vl-max",
        "LLM_API_KEY": "sk-xxxx"
      },
      "enabled": true
    }
  }
}
```

> `command` 里用 server.py 的**绝对路径**最稳妥（数组形式天然支持路径含空格）。
> 也可改用 venv 里的解释器：`["/path/to/.venv/bin/python", "/abs/path/to/server.py"]`。

## 测试

用 MCP Inspector 快速验证：

```bash
uv run --with mcp --with httpx --with pydantic mcp run server.py
# 或
npx @modelcontextprotocol/inspector uv run --with mcp --with httpx --with pydantic python3 server.py
```

直接调用一次（绕过 MCP，验证 API 配置）：

```bash
LLM_API_URL=https://dashscope.aliyuncs.com/compatible-mode/v1 \
LLM_MODEL_NAME=qwen-vl-max \
LLM_API_KEY=sk-xxxx \
uv run --with mcp --with httpx --with pydantic python3 -c "
import asyncio, server
async def t():
    print(await server.understand_image(server.UnderstandImageInput(
        image_source='https://upload.wikimedia.org/wikipedia/commons/4/47/PNG_transparency_demonstration_1.png',
        prompt='How many dice are in this image?')))
asyncio.run(t())
"
```

## 工作机制

- 本地文件路径 → 读取后 base64 编码为 data URI 传入
- 远程 URL（默认 `LLM_FETCH_REMOTE=true`）→ 在本地用 httpx 抓取（跟随重定向）→ base64 编码为 data URI 传入。这样**阿里云 Token Plan 等不自行抓图的端点也能用远程 URL**
- 通过 `mimetypes` 推断图片 MIME 类型，无法识别时默认 `image/jpeg`
- 请求体为标准 OpenAI 多模态格式：`content` 数组里同时放 `text` 和 `image_url`
- 端点自动拼接为 `{LLM_API_URL}/chat/completions`（容忍尾部斜杠、已含后缀）

> 阿里云百炼 Token Plan 的 OpenAI 兼容端点是
> `https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`，
> 与其 Anthropic 兼容端点 `/apps/anthropic/v1` **共用同一把 API Key**，
> 支持视觉理解的模型有 `qwen3.7-plus`、`qwen3.6-plus`、`qwen3.6-flash`、`qwen3.8-max-preview`（`qwen3.7-max` 不支持图像）。

## 错误处理

所有错误都返回可操作的 `"Error: ..."` 字符串，包括：缺少配置、文件不存在、
连接失败、超时、HTTP 状态码（401/403/404/429）及响应片段。
