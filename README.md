# Gemini API 国内调用失败怎么办？401、403、429 与超时排查教程

接入 Gemini 时，最让开发者头疼的往往是：代码已经写好，但请求一直失败。

有人遇到 `401 Unauthorized`，有人返回 `429 Too Many Requests`，也有人在本地测试成功，部署到服务器后却不断超时。

**Gemini API 国内调用失败，应该先检查什么？**

建议按这个顺序排查：

1. 确认请求发送到了正确的 API 地址。
2. 确认 API Key 与当前服务平台匹配。
3. 确认模型 ID、调用权限和账户额度。
4. 根据状态码与错误详情检查参数、限流或服务异常。
5. 如果没有收到 HTTP 响应，再检查网络连接和超时配置。

本文以 **OpenAI Chat Completions 兼容中转接口** 为例，整理 Gemini API 调用失败的常见原因和处理方法。

**大模型 API 中转服务入口：**

> 平台地址：<https://quanzil.com>

> 平台地址：<https://quanzil.net>

具体地址、模型支持、错误格式和计费规则，请以平台后台与开发文档为准。

---

## 先判断：接口报错，还是请求没有连接成功

这两种情况的排查方向不同。

### 收到了 HTTP 状态码

例如：

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
429 Too Many Requests
502 Bad Gateway
```

这通常说明某个服务端或网关已经返回响应。

接下来应检查状态码、响应正文和请求配置。响应也可能来自代理或网关，因此还需要确认请求是否到达了预期的 API 服务。

### 没有收到 HTTP 响应

例如：

```text
Connection timed out
Connection refused
Name resolution failed
TLS handshake failed
```

这类问题优先检查：

- 域名是否正确
- DNS 是否能够解析
- 服务器能否访问目标地址
- 代理配置是否正确
- TLS 证书与系统时间是否正常
- 连接超时是否设置过短

如果连接阶段就失败，修改提示词或更换生成参数通常无法解决。

---

## Gemini API 常见错误对照表

| 错误或状态码 | 常见原因 | 优先检查 |
| --- | --- | --- |
| 400 | 请求格式、参数或输入限制问题 | 请求体与模型支持范围 |
| 401 | 鉴权失败 | API Key、请求头、服务地址 |
| 403 | 权限或访问策略限制 | 模型权限、账户状态、服务要求 |
| 404 | 路径或资源不存在 | Base URL、接口路径、模型 ID |
| 429 | 限流或配额不足 | 错误类型、并发、账户额度 |
| 500、502、503 | 服务端、网关或上游异常 | 服务状态、错误详情、有限重试 |
| 连接超时 | 无法及时建立连接 | 网络、域名、代理 |
| 读取超时 | 等待响应数据过久 | 服务状态、任务长度、读取超时 |

不同平台可能对同一种问题返回不同状态码，**错误正文通常比单独的状态码更有参考价值**。

---

## 返回 401：检查 API Key 和鉴权配置

`401 Unauthorized` 通常表示请求没有通过鉴权。

### API Key 是否属于当前平台

调用中转接口时，应使用该中转平台签发的 Key。

例如：

- 中转平台 Key 应搭配该平台的接口地址。
- 官方服务凭据应按对应官方服务的文档使用。
- 其他平台的 Key 通常不能直接用于当前平台。

API Key 和请求地址不匹配，是常见的接入错误。

### 请求头是否正确

本文使用的兼容接口通常通过以下请求头鉴权：

```text
Authorization: Bearer YOUR_API_KEY
```

检查是否遗漏 `Bearer`，以及 Key 是否包含多余空格或换行。

使用 OpenAI SDK 时，通常只需要正确设置 `api_key`，由 SDK 处理对应请求头。

### 环境变量是否成功加载

不要直接打印完整 Key，可以只检查变量是否存在：

```python
import os

api_key = os.getenv("AI_API_KEY")

print("API Key 已配置：", bool(api_key))
```

如果本地能调用、服务器返回 401，优先确认部署环境是否配置了相同的变量名称，以及应用是否已经重新加载配置。

---

## 返回 403：检查模型权限与访问要求

`403 Forbidden` 通常表示请求受到权限或访问策略限制。

可能原因包括：

- 当前 Key 没有目标模型的调用权限
- 账户状态受限
- 平台启用了 IP 白名单
- 当前请求不符合服务访问要求
- 某个功能需要单独开通

建议先查看错误详情，再到平台后台确认权限配置。

如果使用官方服务，还需要确认其当前支持地区、账户条件和使用要求。使用中转服务时，则应按该平台的服务说明处理。

不要把所有 403 都当作网络问题，也不要通过反复重试来处理明确的权限限制。

---

## 返回 404：检查 Base URL 与接口路径

Gemini API 调用返回 `404`，经常与地址配置有关。

假设平台文档提供的完整聊天接口为：

```text
https://your-api-domain.com/v1/chat/completions
```

那么两种调用方式通常需要不同配置：

| 调用方式 | 通常填写的地址 |
| --- | --- |
| cURL、Python requests | 完整聊天接口地址 |
| OpenAI SDK | 平台提供的 Base URL，例如以 `/v1` 结尾的基础地址 |

以上域名仅用于说明结构，需要替换成真实 API 地址。

如果把完整聊天路径填进 SDK 的 Base URL，可能导致路径重复拼接；如果直接发送 HTTP 请求时只填基础地址，也可能请求到错误路径。

此外，还需要检查模型 ID。部分平台会对不存在的模型返回 404，另一些平台可能返回 400。

相关接入说明可以查看：

- [开发文档](/docs.html)
- [Gemini API 国内调用教程：接口地址、API Key 与 Python 示例](/blog/gemini-api/gemini-api-china-call.html)

---

## 返回 400：先退回最小请求

`400 Bad Request` 通常表示服务不接受当前请求。

常见原因包括：

- JSON 格式错误
- 缺少必要字段
- 消息结构不符合要求
- 使用了模型不支持的参数
- 输入超过限制
- 图片或工具调用格式错误

排查时，先只保留 `model` 和一条文本消息：

```json
{
  "model": "替换为平台支持的Gemini模型ID",
  "messages": [
    {
      "role": "user",
      "content": "请回复：接口测试成功。"
    }
  ]
}
```

如果最小请求成功，再逐个加入其他配置。

例如，可以依次测试：

1. 多轮消息。
2. 输出长度限制。
3. 流式输出。
4. 图片输入。
5. 工具调用或结构化输出。

这样更容易确定是哪一个参数或功能导致失败。

**OpenAI 格式兼容，不代表所有模型支持完全相同的参数。** 应以当前模型和平台文档为准。

---

## 返回 429：先区分限流和额度不足

`429 Too Many Requests` 可能表示请求频率过高，也可能与账户配额有关。

### 如果是频率或并发限制

可以采取以下措施：

- 降低同时发出的请求数量
- 使用队列处理批量任务
- 对重复请求增加缓存
- 设置有限次数的退避重试
- 如果响应包含 `Retry-After`，按平台说明处理等待时间

不要在失败后立即发起大量重试，否则可能进一步触发限流。

### 如果是余额或配额不足

需要检查：

- 账户余额
- Key 的额度上限
- 模型调用配额
- 平台是否设置了日用量或月用量限制

这类问题通常需要处理账户额度，持续重试无法补充余额。

如果使用的 SDK 自带重试，业务层再增加重试时，应控制总尝试次数，避免请求数量成倍增加。

---

## 请求超时：本地能用，服务器不能用怎么办

国内开发者接入 Gemini API 时，经常遇到环境差异：

- 本地测试正常，服务器连接失败
- 开发环境正常，容器中无法访问
- 普通问答正常，长文任务超时
- 非流式请求正常，流式连接中断

### 在实际运行环境中测试

本地能够访问，不代表服务器具有相同的网络条件。

应在应用实际运行的服务器或容器中检查：

- 环境变量是否完整
- 域名解析是否正常
- 出站 HTTPS 请求是否允许
- 是否配置了不同代理
- 防火墙或平台网络规则是否限制访问

### 区分连接超时和读取超时

Python `requests` 可以分别设置：

```python
timeout=(10, 90)
```

其中：

- `10` 表示连接超时。
- `90` 表示读取超时。

读取超时主要限制等待响应数据的时间，并不是整个请求的严格总时长上限。

对于持续连接失败的问题，应先检查网络；对于复杂任务等待过久的问题，再结合服务状态和业务要求调整超时。

### 检查请求规模

长文档、较多历史消息或较长输出，都可能增加处理时间。

可以先用短消息验证接口，再逐步增加输入。如果短请求正常、长请求反复失败，应检查输入限制、任务耗时和平台配置。

---

## 返回 500、502 或 503：如何处理临时服务异常

这类错误可能来自中转服务、网关或上游模型服务。

建议：

1. 查看平台服务状态或公告。
2. 保留状态码、错误类型和请求 ID。
3. 对可重试的临时错误设置有限重试。
4. 避免在故障期间大量并发重试。
5. 对关键业务设计失败提示或备用模型流程。

对于生成内容的请求，如果客户端超时或收到异常，不能一概认为上游完全没有处理。

再次调用可能产生新的生成结果，也可能增加费用。因此，正式项目应控制重试次数，并结合调用记录核对实际情况。

---

## 一个用于排查 Gemini API 的 Python 示例

下面的脚本只发送一条简短消息，方便检查基础配置。

先安装依赖：

```bash
python -m pip install requests
```

在 macOS 或 Linux 中设置环境变量：

```bash
export AI_API_KEY="替换为你的中转平台API Key"
export AI_CHAT_URL="替换为平台提供的完整聊天接口地址"
export GEMINI_MODEL="替换为平台支持的Gemini模型ID"
```

创建 `gemini_check.py`：

```python
import os
import requests

names = ["AI_API_KEY", "AI_CHAT_URL", "GEMINI_MODEL"]
missing = [name for name in names if not os.getenv(name)]

if missing:
    raise RuntimeError(
        "缺少环境变量：" + ", ".join(missing)
    )

try:
    response = requests.post(
        os.environ["AI_CHAT_URL"],
        headers={
            "Authorization": (
                f"Bearer {os.environ['AI_API_KEY']}"
            ),
            "Content-Type": "application/json",
        },
        json={
            "model": os.environ["GEMINI_MODEL"],
            "messages": [
                {
                    "role": "user",
                    "content": "请回复：接口测试成功。",
                }
            ],
        },
        timeout=(10, 90),
    )

    print("HTTP 状态码：", response.status_code)

    request_id = (
        response.headers.get("x-request-id")
        or response.headers.get("request-id")
    )

    if request_id:
        print("请求 ID：", request_id)

    try:
        data = response.json()
    except ValueError:
        print("响应不是有效 JSON，请检查接口地址和网关响应。")
    else:
        if response.ok:
            choices = data.get("choices", [])
            if choices:
                print(choices[0].get("message", {}).get("content"))
            else:
                print("响应中没有 choices，请确认接口返回格式。")
        else:
            error = data.get("error", {})
            if isinstance(error, dict):
                print("错误类型：", error.get("type"))
                print("错误代码：", error.get("code"))
                print("错误信息：", error.get("message"))
            else:
                print("请按平台文档检查错误响应。")

except requests.exceptions.ConnectTimeout:
    print("连接超时：检查网络、域名和代理配置。")

except requests.exceptions.ReadTimeout:
    print("读取超时：检查服务状态、任务长度和超时设置。")

except requests.exceptions.ConnectionError:
    print("连接失败：检查 DNS、TLS、网络和接口地址。")

except requests.exceptions.RequestException as exc:
    print("请求异常：", type(exc).__name__)
```

运行：

```bash
python gemini_check.py
```

脚本没有自动重试，方便观察单次请求的原始结果。

错误信息可能包含请求相关内容。分享日志前，应检查并删除 Key、个人信息或业务敏感数据。

---

## 排查 Gemini API 时，建议保留哪些信息

如果需要联系平台技术支持，提供下面的信息通常更方便定位：

- 报错发生的时间与时区
- 调用的模型 ID
- HTTP 状态码
- 错误类型与错误代码
- 平台返回的请求 ID
- SDK 或依赖版本
- 是否使用流式输出
- 本地与服务器是否都能复现

一般不需要提供完整 API Key。

如果需要展示请求体，可以用一条不包含业务数据的测试消息复现问题。

---

## 常见问题

### Gemini API 国内调用失败，一定是网络原因吗？

不一定。

鉴权失败、模型权限不足、接口路径错误、参数不兼容和额度不足，都可能导致失败。

先判断是否收到 HTTP 响应，再根据错误详情排查。

### 换一个 Gemini 模型就能解决吗？

如果问题来自模型不存在、权限不足或参数不支持，换成可用模型可能有帮助。

如果是 Key 错误、接口地址错误或网络连接失败，换模型通常不能解决。

### 为什么调用一次成功，下一次就返回 429？

可能触发了短时间请求限制、并发限制或 token 用量限制，也可能是账户额度发生变化。

需要结合错误正文和调用记录判断。

### 流式请求中途断开，要重新生成吗？

先检查客户端、后端、代理层和平台的连接情况。

重新生成可能产生不同内容和额外费用。业务上可以提示用户重试，或按平台支持情况设计恢复流程。

### 为什么 API 调用成功，但没有文本内容？

响应可能包含工具调用、其他输出类型或模型的特殊结束情况，也可能与兼容响应格式有关。

应检查完整的响应结构和结束原因，而不是只读取一个文本字段。

---

## 按顺序解决 Gemini API 调用问题

排查 Gemini API 国内调用失败时，建议从最小请求开始：

1. 确认 Key、接口地址和模型 ID。
2. 只发送一条简短文本消息。
3. 根据状态码与错误详情定位问题。
4. 基础调用成功后，逐步加入业务参数。
5. 上线前补充超时、限流、日志和有限重试。

大模型 API 中转接口可以帮助统一基础调用方式。实际可用性还需要在自己的部署环境中测试，并结合平台文档确认功能与限制。

**服务与接入信息：**

> <https://quanzil.com>

> <https://quanzil.net>

