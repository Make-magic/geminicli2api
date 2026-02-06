# Gemini CLI 转 API 代理 (geminicli2api)

这是一个基于 FastAPI 的代理服务器，它将 Gemini CLI 工具转换为兼容 OpenAI 和原生 Gemini API 的端点。这使您可以通过熟悉的 OpenAI API 接口或直接的 Gemini API 调用来利用 Google 的免费 Gemini API 配额。

## 🚀 功能特性

- **OpenAI 兼容 API**：完美替代 OpenAI 的聊天完成 (chat completions) API
- **原生 Gemini API**：直接代理 Google 的 Gemini API
- **流式传输支持**：两种 API 格式均支持实时流式响应
- **多模态支持**：支持文本和图像输入
- **身份验证**：多种验证方式（Bearer 令牌、Basic 认证、API 密钥）
- **Google 搜索增强 (Grounding)**：使用 `-search` 模型后缀启用 Google 搜索以获得基于事实的回答。
- **思考/推理控制**：通过 `-nothinking` 和 `-maxthinking` 模型后缀控制 Gemini 的思考过程。
- **Docker 就绪**：容器化设计，易于部署
- **Hugging Face Spaces**：已准备好部署到 Hugging Face Spaces

## 🔧 环境变量

### 必填项
- `GEMINI_AUTH_PASSWORD`: API 访问的认证密码

### 可选凭证来源 (任选其一)
- `GEMINI_CREDENTIALS`: 包含 Google OAuth 凭证的 JSON 字符串
- `GOOGLE_APPLICATION_CREDENTIALS`: Google OAuth 凭证文件的路径
- `GOOGLE_CLOUD_PROJECT`: Google Cloud 项目 ID
- `GEMINI_PROJECT_ID`: 备选项目 ID 变量

### 凭证 JSON 示例
```json
{
  "client_id": "your-client-id",
  "client_secret": "your-client-secret", 
  "token": "your-access-token",
  "refresh_token": "your-refresh-token",
  "scopes": ["https://www.googleapis.com/auth/cloud-platform"],
  "token_uri": "https://oauth2.googleapis.com/token"
}
```

## 📡 API 端点

### OpenAI 兼容端点
- `POST /v1/chat/completions` - 聊天完成 (支持流式和非流式)
- `GET /v1/models` - 列出可用模型

### 原生 Gemini 端点  
- `GET /v1beta/models` - 列出 Gemini 模型
- `POST /v1beta/models/{model}:generateContent` - 生成内容
- `POST /v1beta/models/{model}:streamGenerateContent` - 流式生成内容
- 所有其他 Gemini API 端点均会被代理

### 实用工具端点
- `GET /health` - 用于容器编排的健康检查

## 🔐 身份验证

API 支持多种身份验证方法：

1. **Bearer 令牌**: `Authorization: Bearer YOUR_PASSWORD`
2. **Basic 认证**: `Authorization: Basic base64(username:YOUR_PASSWORD)`
3. **查询参数**: `?key=YOUR_PASSWORD`
4. **Google 请求头**: `x-goog-api-key: YOUR_PASSWORD`

## 🐳 Docker 使用方法

```bash
# 构建镜像
docker build -t geminicli2api .

# 在默认端口 8888 上运行 (兼容模式)
docker run -p 8888:8888 \
  -e GEMINI_AUTH_PASSWORD=your_password \
  -e GEMINI_CREDENTIALS='{"client_id":"...","token":"..."}' \
  -e PORT=8888 \
  geminicli2api

# 在端口 7860 上运行 (Hugging Face 兼容)
docker run -p 7860:7860 \
  -e GEMINI_AUTH_PASSWORD=your_password \
  -e GEMINI_CREDENTIALS='{"client_id":"...","token":"..."}' \
  -e PORT=7860 \
  geminicli2api
```

### Docker Compose

```bash
# 默认设置 (端口 8888)
docker-compose up -d

# Hugging Face 设置 (端口 7860)
docker-compose --profile hf up -d geminicli2api-hf
```

## 🤗 Hugging Face Spaces

本项目已配置为支持 Hugging Face Spaces 部署：

1. Fork 本仓库
2. 在 Hugging Face 上创建一个新的 Space
3. 连接你的仓库
4. 在 Space 设置中设置所需的环境变量：
   - `GEMINI_AUTH_PASSWORD`
   - `GEMINI_CREDENTIALS` (或其他凭证来源)

Space 将使用包含的 Dockerfile 自动构建和部署。

## 📝 OpenAI API 示例

```python
import openai

# 配置客户端使用你的代理
client = openai.OpenAI(
    base_url="http://localhost:8888/v1",  # 如果是 HF 则使用 7860
    api_key="your_password"  # 你的 GEMINI_AUTH_PASSWORD
)

# 像普通的 OpenAI API 一样使用
response = client.chat.completions.create(
    model="gemini-2.5-pro-maxthinking",
    messages=[
        {"role": "user", "content": "用简单的术语解释相对论。"}
    ],
    stream=True
)

# 将推理过程与最终答案分开
for chunk in response:
    if chunk.choices[0].delta.reasoning_content:
        print(f"Thinking: {chunk.choices[0].delta.reasoning_content}")
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

## 🔧 原生 Gemini API 示例

```python
import requests

headers = {
    "Authorization": "Bearer your_password",
    "Content-Type": "application/json"
}

data = {
    "contents": [
        {
            "role": "user",
            "parts": [{"text": "用简单的术语解释相对论。"}]
        }
    ],
    "thinkingConfig": {
        "thinkingBudget": 32768,
        "includeThoughts": True
    }
}

response = requests.post(
    "http://localhost:8888/v1beta/models/gemini-2.5-pro:generateContent",  # 如果是 HF 则使用 7860
    headers=headers,
    json=data
)

print(response.json())
```

## 🎯 支持的模型

### 基础模型

- `gemini-3.0-pro-preview`
- `gemini-3.0-flash-preview`
- `gemini-2.5-pro`
- `gemini-2.5-flash`
- `gemini-1.5-pro`
- `gemini-1.5-flash`
- `gemini-1.0-pro`

### 模型变体
代理会自动为 `gemini-2.5-pro` 和 `gemini-2.5-flash` 模型创建以下变体：

- **`-search`**: 在模型名称后附加 `-search` 以启用 Google 搜索增强。
  - 示例: `gemini-2.5-pro-search`
- **`-nothinking`**: 附加 `-nothinking` 以最小化推理步骤。
  - 示例: `gemini-2.5-flash-nothinking`
- **`-maxthinking`**: 附加 `-maxthinking` 以最大化推理预算。
  - 示例: `gemini-2.5-pro-maxthinking`

## 📄 许可证

MIT 许可证 - 详情请参阅 LICENSE 文件。

## 🤝 贡献

欢迎贡献！请随时提交 Pull Request。
