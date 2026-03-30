# OpenAI 自动注册

> ⚠️ 当前不可用，等待后续更新。

## 功能亮点
- 支持三种邮箱提供商：`LuckMail`、`TempMail.lol` 与 `GPTMail`，可手动指定，也可自动回退。
- 支持自动上传到 Sub2API，优先使用全局 Admin API Key（`x-api-key`），无需先登录拿 bearer。
- 支持 **Sub2API 失效账号清理**：全量拉取账号后，在本地精确匹配 `extra.privacy_mode == "training_set_failed"`，再执行删除，不依赖服务端 privacy 过滤结果。
- 支持 **Sub2API dry-run**：先打印命中账号列表，不删除，避免误删。
- 支持 CPA 自动上号（上传 token）与失效账号清理（探测 401/用量超阈值后删除）。
- 支持统一的 `--prune-local`：当 Sub2API / CPA 任一上传成功后，自动删除本地 token 文件和 `tokens/accounts.txt` 里的对应账号行。

## 环境准备
基础功能（注册 / Sub2API 上传 / Sub2API 清理）只需要：

```bash
cd openai-register
uv sync
```

如果要启用 **CPA 清理功能**（`--cpa-clean`，依赖 `aiohttp`），再安装可选依赖：

```bash
cd openai-register
uv sync --extra cpa
```

## 运行示例
```bash
cd openai-register
uv run python openai_register.py --once
```

指定邮箱提供商：
```bash
uv run python openai_register.py --mail-provider luckmail --once
uv run python openai_register.py --mail-provider tempmail --once
uv run python openai_register.py --mail-provider gptmail --once
```

使用 LuckMail（Outlook 接码）前先配置环境变量：
```bash
export LUCKMAIL_BASE_URL="https://your-luckmail-domain.com"
export LUCKMAIL_API_KEY="YOUR_LUCKMAIL_API_KEY"
# 可选
export LUCKMAIL_API_SECRET="YOUR_LUCKMAIL_API_SECRET"
export LUCKMAIL_USE_HMAC="false"
export LUCKMAIL_PROJECT_CODE="openai"
export LUCKMAIL_EMAIL_TYPE="ms_graph"
export LUCKMAIL_DOMAIN="outlook.com"
export LUCKMAIL_ORDER_TIMEOUT="180"
export LUCKMAIL_POLL_INTERVAL="6"

uv run python openai_register.py --mail-provider luckmail --once
```

启用 Sub2API 自动上传示例（推荐用全局 Admin API Key）：
```bash
cd openai-register
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-admin-api-key YOUR_SUB2API_ADMIN_API_KEY \
  --sub2api-group-ids 2 \
  --sub2api-upload --prune-local \
  --once
```

也可直接用环境变量：
```bash
export SUB2API_BASE_URL="http://203.0.113.10:8080"
export SUB2API_ADMIN_API_KEY="YOUR_SUB2API_ADMIN_API_KEY"
export SUB2API_GROUP_IDS="2"
export AUTO_UPLOAD_SUB2API="true"
uv run python openai_register.py --once
```

使用邮箱密码登录获取 bearer token 的认证方式：
```bash
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-email admin@example.com \
  --sub2api-password 'your_admin_password' \
  --sub2api-upload --prune-local \
  --once
```

先 dry-run 查看 Sub2API 中 `training_set_failed` 的账号，不删除：
```bash
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-admin-api-key YOUR_SUB2API_ADMIN_API_KEY \
  --sub2api-clean-training-set-failed \
  --sub2api-clean-only \
  --sub2api-clean-dry-run \
  --once
```

确认命中名单后，执行真实删除：
```bash
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-admin-api-key YOUR_SUB2API_ADMIN_API_KEY \
  --sub2api-clean-training-set-failed \
  --sub2api-clean-only \
  --once
```

> 说明：Sub2API 清理现在会先 **全量分页拉取账号**，再在本地精确匹配 `extra.privacy_mode == "training_set_failed"`，只有命中的账号才会删除。

启用 CPA 上传 + 清理 + 本地同步示例（如果还要用 `--cpa-clean`，需先 `uv sync --extra cpa`）：
```bash
cd openai-register
uv run python openai_register.py \
  --cpa-base-url http://203.0.113.10:8317 \
  --cpa-token YOUR_CPA_LOGIN_PASSWORD \
  --cpa-upload --cpa-clean --prune-local \
  --cpa-workers 1 --cpa-timeout 12 --cpa-retries 1 --cpa-used-threshold 95 \
  --cpa-target-count 300 --sleep-min 5 --sleep-max 30
```

**CPA 参数填写说明：**
- `--cpa-token`：这里填的就是 **CPA 后台登录密码**，不是别的 API key。
- `--cpa-base-url`：只需要填写到 **协议 + IP/域名 + 端口**，不要带后台页面路径。
- 正确示例：`http://203.0.113.10:8317`
- 错误示例：`http://203.0.113.10:8317/management.html#/`
- `--cpa-target-count`：CPA 目标有效 token 数，达到后循环模式会随机休眠并继续检查。

## 参数说明
- `--proxy`：可选，HTTP/S 代理地址。
- `--mail-provider`：邮箱提供商，可选 `auto` / `luckmail` / `gptmail` / `tempmail`，默认 `auto`。`auto` 会优先尝试 LuckMail，再回退到 TempMail.lol 和 GPTMail。
- `--luckmail-base-url` / `--luckmail-api-key`：启用 LuckMail 必填；也支持同名环境变量 `LUCKMAIL_BASE_URL` / `LUCKMAIL_API_KEY`。
- `--luckmail-api-secret` / `--luckmail-use-hmac`：LuckMail 可选 HMAC 鉴权配置。
- `--luckmail-project-code` / `--luckmail-email-type` / `--luckmail-domain`：LuckMail 创单参数，默认分别是 `openai` / `ms_graph` / `outlook.com`。
- `--luckmail-order-timeout` / `--luckmail-poll-interval`：LuckMail 轮询超时与轮询间隔。
- `--once`：只跑一轮；不加则循环运行。
- `--sleep-min` / `--sleep-max`：循环模式下两轮之间的随机等待秒数。
- `--sub2api-base-url`：Sub2API 地址，只填写到 `协议 + IP/域名 + 端口`。
- `--sub2api-admin-api-key`：Sub2API 全局管理员 API Key，请求头走 `x-api-key`，这是推荐方式。
- `--sub2api-bearer`：兼容旧方式的 Bearer token。
- `--sub2api-email` / `--sub2api-password`：兼容旧方式，401 时可自动登录换 bearer。
- `--sub2api-group-ids`：上传后绑定的分组 ID，逗号分隔，默认 `2`。
- `--sub2api-upload`：注册成功后自动上传到 Sub2API；也可用环境变量 `AUTO_UPLOAD_SUB2API=true`。
- `--sub2api-clean-training-set-failed`：清理 Sub2API 中 `extra.privacy_mode == "training_set_failed"` 的账号。
- `--sub2api-clean-only`：只执行 Sub2API 清理，不跑注册流程。
- `--sub2api-clean-dry-run`：只列出本地精确匹配命中的账号，不执行删除。
- `--prune-local`：当 Sub2API / CPA 任一上传成功后，删除本地 token 文件和 `tokens/accounts.txt` 中对应账号行。
- `--cpa-base-url`：CPA 管理地址，只填写到 `协议 + IP/域名 + 端口`，不要带 `/management.html#/` 这类后台页面路径。
- `--cpa-token`：这里填写的就是 CPA 后台登录密码。可直接用参数传入，也可用环境变量 `CPA_TOKEN` 覆盖。
- `--cpa-workers` / `--cpa-timeout` / `--cpa-retries` / `--cpa-used-threshold`：CPA 清理探测并发、超时、重试、用量判定阈值（默认 95）。
- `--cpa-upload`：注册成功后把 token 文件上传到 CPA；这一项本身不需要额外依赖。
- `--cpa-clean`：注册成功后探测并删除 CPA 中失效/用量超阈值的账号（需要可选依赖 `aiohttp`，也就是 `uv sync --extra cpa`）。
- `--cpa-target-count`：CPA 目标有效 token 数（默认 300）。

## 输出位置
- 账号密码：`tokens/accounts.txt`（格式：`email----password`）。
- Token JSON：`tokens/token_<email>_<timestamp>.json`。

> 注意：`tokens/` 目录已加入 `.gitignore`，默认不会提交到仓库。

## 注意
- 需能访问 `https://auth.openai.com`；代理地区尽量避开 CN/HK。
- 若某个临时邮箱提供商不稳定，可切换 `--mail-provider gptmail` 或 `--mail-provider tempmail` 单独测试。
- 如果使用 `--mail-provider auto`，会优先尝试 `TempMail.lol`，失败后自动回退到 `GPTMail`。
.openai.com`；代理地区尽量避开 CN/HK。
- 若某个临时邮箱提供商不稳定，可切换 `--mail-provider gptmail` 或 `--mail-provider tempmail` 单独测试。
- 如果使用 `--mail-provider auto`，会优先尝试 `TempMail.lol`，失败后自动回退到 `GPTMail`。
