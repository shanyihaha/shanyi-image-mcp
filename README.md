# 闪艺画图 MCP

把 [闪艺](https://shanyiapi.com) 的图像接口包装成 MCP server，让 Claude Code / Codex 等 MCP 客户端直接生图、改图、批处理、多图参考。

使用 `gpt-image-2` / `gpt-image-2-pro` 模型（边长 ≥1600 自动切 pro）。

---

## 功能

| Tool | 说明 |
|---|---|
| `image_generate` | 文生图。支持 1K / 2K / 4K；2K/4K 自动切 pro，强制 `n=1` |
| `image_edit` | 单图参考/编辑。1K 走 `/v1/images/edits`（支持 alpha mask）；2K 走 `reference_image` |
| `image_batch_edit` | 多张图逐张同指令处理（仅 1K）；non-pro 5 并发，pro 串行 |
| `image_multi_reference` | 2-10 张参考图融合成 1 张新图 |
| `server_info` | 查看 base URL、模型、size 规则、重试策略、安全约束 |

第一次使用前，让 LLM 调一次 `server_info`，可以看到当前运行时配置和可用能力。

---

## 一键安装

### Windows（PowerShell）

```powershell
# 1. 解压收到的 shanyi-image-mcp.zip
# 2. 进入解压后的目录
cd C:\path\to\shanyi-image-mcp

# 3. 跑一键安装（交互输入闪艺 sk-key + 输出目录）
python install.py
```

### macOS / Linux

```bash
unzip shanyi-image-mcp.zip
cd shanyi-image-mcp
python install.py
```

脚本会：

1. 检查 Python >= 3.10（没装去 https://www.python.org/downloads/ 下）
2. 安装依赖（mcp / httpx / Pillow）
3. 交互配置闪艺 API key、输出目录
4. 写入 `~/.claude.json` 和 `~/.codex/config.toml`（已有配置自动备份）
5. 启动 server 做一次 initialize 握手验证

非交互安装：

```powershell
# Windows
$env:SHANYI_API_KEY="sk-..."; python install.py --yes
```

```bash
# macOS / Linux
SHANYI_API_KEY=sk-... \
SHANYI_SAVE_DIR=~/Pictures/shanyi-image-out \
python install.py --yes
```

常用选项：

```bash
python install.py --no-codex
python install.py --no-claude
python install.py --mirror tsinghua
python install.py --baseurl https://shanyiapi.com
```

安装完成后重启 Claude Code / Codex，让 LLM 调 `server_info` 验证。

---

## Size 规则

- W/H 必须是 8 的倍数
- W/H 必须在 256 到 4096 范围内
- 1K 福利档可能被代理处理到约 1.57MP
- 2K/4K 自动切 `gpt-image-2-pro`
- 2K/4K 强制 `n=1` 并加跨进程锁，避免多个 MCP 同时打爆 pro 队列

推荐 size：

| 档位 | 推荐值 |
|---|---|
| 1K | `1024x1024`, `1280x720`, `720x1280`, `1024x1536`, `1536x1024` |
| 2K | `2048x2048`, `2048x1152`, `1152x2048` |
| 4K | `3840x2160`, `2160x3840` |

---

## 环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `SHANYI_API_KEY` | 空 | 闪艺 API token |
| `SHANYI_BASEURL` | `https://shanyiapi.com` | 闪艺 base URL |
| `SHANYI_MODEL` | `gpt-image-2` | 默认模型 |
| `SHANYI_SAVE_DIR` | `~/Pictures/shanyi-image-out` | 默认输出目录 |
| `SHANYI_SAVE_DIR_ROOT` | 同输出目录 | 输出安全根目录（沙箱） |
| `SHANYI_USE_SHELL_PROXY` | `0` | 设为 `1` 才让 httpx 读取 shell 代理 |

---

## 手动配置

Claude Code (`~/.claude.json`):

```json
{
  "mcpServers": {
    "shanyi-image": {
      "command": "/path/to/python",
      "args": ["/absolute/path/to/shanyi-image-mcp/server.py"],
      "env": {
        "SHANYI_API_KEY": "sk-...",
        "SHANYI_SAVE_DIR": "/Users/you/Pictures/shanyi-image-out",
        "SHANYI_SAVE_DIR_ROOT": "/Users/you/Pictures/shanyi-image-out"
      }
    }
  }
}
```

Codex (`~/.codex/config.toml`):

```toml
[mcp_servers.shanyi-image]
command = "/path/to/python"
args = ["/absolute/path/to/shanyi-image-mcp/server.py"]

[mcp_servers.shanyi-image.env]
SHANYI_API_KEY = "sk-..."
SHANYI_SAVE_DIR = "/Users/you/Pictures/shanyi-image-out"
SHANYI_SAVE_DIR_ROOT = "/Users/you/Pictures/shanyi-image-out"
```

---

## 重试与限流

- **1K**：失败 → 4s + jitter → 重试 → 8s + jitter → 重试（共 3 次尝试）；网络层异常额外免费重试 1 次
- **2K/4K**：双层锁（进程内 asyncio.Semaphore(1) + 跨进程文件锁 `~/.cache/shanyi-image/bigsize.lock`），可恢复 5xx 60s 后重试 1 次；Cloudflare 524 fail fast 不重试
- **4K 图生图 / 多图参考**已禁用（origin 处理 > 120s 撞 CF 上限）

---

## 安全约束

- 所有输出强制落在 `SHANYI_SAVE_DIR_ROOT` 之下，传出根目录的路径会被拒
- 输入图按 magic bytes 校验为 PNG/JPEG/WebP/GIF；非图片立即拒
- 单输入图 ≤ 4MB；多图参考累计 ≤ 8MB
- baseurl 锁在启动期 env，运行期 tool 不接受参数（防 key 外泄到攻击者 host）
- basename 仅允许 `[A-Za-z0-9_-.]`，禁含 `/` `..` 和路径分量
