---
name: kfcrazy4-auto-post
description: 每日自动为小红书账号 KFCrazy4 生成疯狂星期四梗文 + 配图 + 发布全流程。当用户说"发一篇疯狂星期四"、"今天发文"、"自动发小红书"、"跑一遍 SOP" 或任何涉及 KFCrazy4 账号运营的指令时触发。
agent_created: true
---

# KFCrazy4 自动发文 SOP

为小红书账号 **KFCrazy4**（专注疯狂星期四梗文化）提供全自动内容创作与发布流程。

## 基础设施

| 资源 | 位置 | 用途 |
|------|------|------|
| `references/content-style-guide.md` | skill 内置 | KFCrazy4 账号内容风格规范、段子模板、标签库。**每次执行前加载** |
| `~/xiaohongshu-mcp/xiaohongshu-mcp-darwin-amd64` | 用户目录 | 小红书 MCP 服务二进制 |
| `~/xiaohongshu-mcp/start-mcp.sh` | 用户目录 | MCP 服务启动脚本（自动 cd 到正确目录） |
| `~/xiaohongshu-mcp/cookies.json` | 用户目录 | 小红书登录凭证（扫码一次永久有效） |
| `~/xiaohongshu-mcp/xhs_mcp_client.py` | 用户目录 | MCP HTTP 协议客户端工具库 |
| MCP 服务地址 | `http://127.0.0.1:18060/mcp` | Streamable HTTP MCP 端点 |

## 依赖技能/能力

| 技能/能力 | 用途 | 如何调用 |
|-----------|------|----------|
| 多模态内容生成 | 文生图生成配图 | 加载后调用 `buddy-cloud.py image` |
| `kfcrazy4-auto-post` 本 skill | 整体编排 | 本 skill 自行完成 |

## 核心机制：MCP 服务管理

> 小红书 MCP 服务（xiaohongshu-mcp）以 HTTP 协议运行在 `127.0.0.1:18060`，WorkBuddy 无法直接将其作为内置 MCP 工具注册。需要通过 **Python HTTP 客户端** 直接调用其 MCP 协议端点。

### MCP HTTP 协议调用方式

MCP 服务使用 Streamable HTTP 协议，调用步骤如下：

1. **初始化会话**：向 `http://127.0.0.1:18060/mcp` 发送 `initialize` 请求
   - Accept 头必须同时包含 `application/json` 和 `text/event-stream`
2. **获取 SessionId**：从初始化响应的 `Mcp-Session-Id` 头获取会话 ID
3. **发送初始化通知**：发送 `notifications/initialized`
4. **调用工具**：所有后续请求带上 `Mcp-Session-Id` 头，使用 `tools/call`

示例 Python 代码：

```python
import requests
H = {'Content-Type': 'application/json', 'Accept': 'application/json, text/event-stream'}

# 初始化
r = requests.post("http://127.0.0.1:18060/mcp", json={
    "jsonrpc": "2.0", "method": "initialize",
    "params": {"protocolVersion": "2025-06-18", "capabilities": {}, "clientInfo": {"name": "kfcrazy4-bot", "version": "1.0"}},
    "id": 1
}, headers=H)
sid = r.headers.get("Mcp-Session-Id", "")
H2 = {**H, "Mcp-Session-Id": sid}

# 通知
requests.post("http://127.0.0.1:18060/mcp", json={
    "jsonrpc": "2.0", "method": "notifications/initialized", "params": {}
}, headers=H2)

# 检查登录
r = requests.post("http://127.0.0.1:18060/mcp", json={
    "jsonrpc": "2.0", "method": "tools/call",
    "params": {"name": "check_login_status", "arguments": {}},
    "id": 2
}, headers=H2)

# 发布内容
r = requests.post("http://127.0.0.1:18060/mcp", json={
    "jsonrpc": "2.0", "method": "tools/call",
    "params": {"name": "publish_content", "arguments": {
        "title": "标题（20字内）",
        "content": "正文（1000字内）",
        "images": ["/path/to/image.png"],
        "tags": ["标签1", "标签2"]
    }},
    "id": 3
}, headers=H2, timeout=180)
```

### 可用 MCP 工具列表

| 工具名 | 功能 |
|--------|------|
| `check_login_status` | 检查登录状态 |
| `publish_content` | **发布图文笔记**（核心工具） |
| `publish_with_video` | 发布视频笔记 |
| `get_login_qrcode` | 获取登录二维码 |
| `delete_cookies` | 删除登录凭证 |
| `search_feeds` | 搜索内容 |
| `list_feeds` | 获取首页推荐 |
| `get_feed_detail` | 获取帖子详情 |
| `like_feed` | 点赞/取消点赞 |
| `favorite_feed` | 收藏/取消收藏 |
| `post_comment_to_feed` | 发表评论 |
| `reply_comment_in_feed` | 回复评论 |
| `user_profile` | 获取用户主页 |

## 完整工作流

### 阶段 0：MCP 服务检查与启动

1. 检查 MCP 服务是否在运行：
   ```bash
   lsof -i:18060
   ```
2. 如果未运行，启动服务：
   ```bash
   nohup ~/xiaohongshu-mcp/start-mcp.sh > ~/xiaohongshu-mcp/mcp.log 2>&1 &
   ```
3. 等待 3 秒确认服务启动成功

### 阶段 1：登录检查

通过 MCP HTTP 协议调用 `check_login_status`：
- **未登录** → 通过 `get_login_qrcode` 获取二维码，引导用户用小红书 App 扫码
- **已登录** → 继续

### 阶段 2【新增】：网上调研 — 抓取优质梗参考

> 此阶段确保文案质量有据可依，不再凭空生成

1. **搜索最新疯狂星期四梗内容**：
   ```text
   WebSearch 搜索关键词（每次随机选1-2个）：
   - "疯狂星期四 段子"
   - "V我50 文案 最新"
   - "疯狂星期四 神反转"
   - "疯四 抽象文案"
   - "KFC疯狂星期四 搞笑"
   ```

2. **抓取优质参考**：从搜索结果中选择 1-2 个内容最丰富的来源页面，用 WebFetch 获取详细内容

3. **分析流行模式**：提取当前最火的段子风格和写法，重点关注：
   - 故事铺垫方式（如何开头）
   - 细节描写（什么让故事真实）
   - 反转技巧（如何过渡到V50）
   - 情感渲染手法

4. **将调研结果作为上下文**：保存关键参考信息到临时文件，供阶段 3 使用

### 阶段 3：生成优质段子文案（优化版）

> 加载 `references/content-style-guide.md` 获取完整的内容风格规范
> **结合阶段 2 的调研结果**，参考当前热门段子的写法

按以下步骤生成 **1 条** 高质量疯狂星期四段子：

1. **随机选题**：从以下 16 个选题中随机选 **1 个**：
   - 玉皇大帝、间谍007、中彩票、考古古墓、CT扫描、秦始皇、外星人、穿越重生、阎王爷、考试、程序员、面试、算命先生、ChatGPT、霸道总裁、医生

2. **参考优质模板写段子**：使用「情感铺垫 → 突然反转」结构，模仿调研中找到的热门段子风格来写。

   **必须遵守的质量标准**：
   | 要求 | 说明 |
   |------|------|
   | 有故事铺垫 | 不能直接说"我要V50"，要有情景 setup（3-5句话铺垫） |
   | 细节真实 | 加入具体数字、场景、对话等细节 |
   | 有情感弧线 | 先制造情绪（深情/悲催/离谱/认真），再反转 |
   | 反转有力 | V50收尾要出人意料，与前面形成反差 |
   | 正文长度 | **300-500 字**（太短缺乏感染力） |

   **不好的例子（太短太直白）**：
   > 我是秦始皇，V我50助我东山再起
   
   **好的例子（有细节有反转）**：
   > 我最近越来越期待夜晚了，因为白天都没什么机会能和你说话，只能憋到晚上和你说句晚安。但你可别小看这两个字，它可包含着我今天清晨见到的阳光，中午看到的白云，傍晚遇见的微风……说了这么多，你都听得到吗？其实我在说今天是肯德基疯狂星期四，v我50，抚慰我支离破碎的心。

3. **标题**：`🤡 疯 狂 星 期 四 之 [选题]` — **不超过 20 字**

4. **标签**：选 **5-6 个**（固定标签 3-4 个 + 题材相关 1-2 个）

5. **提取配图 Prompt**：从段子中提取核心场景，按 content-style-guide.md 的规范构建

### 阶段 4：生成配图（基于单条段子）

> 加载「多模态内容生成」skill

1. 获取云服务认证：调用 `connect_cloud_service`
2. 使用阶段 3 提取的**配图 Prompt** 生成图片：
   ```bash
   echo -n "<tempToken>" | python3 <buddy_multimodal_dir>/scripts/buddy-cloud.py image "<配图Prompt>" --resolution 768:1024 --token-stdin
   ```
3. 下载到 `kfcrazy4/` 目录，使用无中文的英文文件名

### 阶段 5：保存 MD 文件

将完整图文方案保存为 `kfcrazy4/YYYY-MM-DD-[选题英文名].md`

### 阶段 6：发布到小红书

> ⚠️ **关键约束**：`publish_content` 的 `images` 参数是**必填字段**。图片生成失败必须重试。

通过 MCP HTTP 协议调用 `publish_content`：

| 参数 | 限制 |
|------|------|
| `title` | 最多 **20 字** |
| `content` | 最多 **1000 字** |
| `images` | **必填**，本地绝对路径，不含中文 |
| `tags` | **5-6 个** |

```python
publish_content(
    title="🤡 疯 狂 星 期 四 之 秦始皇",
    content="（正文300-500字，有铺垫有反转）",
    images=["/Users/.../kfcrazy4/qinshihuang.png"],
    tags=["疯狂星期四","V我50","发疯文学","肯德基","搞笑日常","秦始皇"]
)
```

发布后检查返回结果：
- `error` → 根据错误信息处理
- `result` → 发布成功，回填 MD 状态

### 阶段 7：交付结果

调用 `deliver_attachments` 将 MD 文件和配图交付给用户。

## 自动化配置

首次使用时，为用户配置每日自动化。

```json
{
  "mode": "suggested create",
  "name": "KFCrazy4 每日自动发文",
  "prompt": "运行 kfcrazy4-auto-post skill，完成每日疯狂星期四小红书发布的完整流程",
  "scheduleType": "recurring",
  "rrule": "FREQ=DAILY;BYDAY=MO,TU,WE,TH,FR,SA,SU",
  "cwds": "/Users/haku/WorkBuddy/2026-05-10-task-3",
  "status": "ACTIVE"
}
```

## 异常处理

| 问题 | 处理方式 |
|------|----------|
| MCP 服务未启动 | 用 `start-mcp.sh` 启动，等待 3 秒后重试 |
| MCP 连接失败 | 重试 1 次，仍失败则提示用户检查端口 18060 |
| 小红书未登录 | 调用 `get_login_qrcode` 获取二维码，引导用户扫码 |
| 图片生成失败 | `images` 是必填参数字段，**必须重试生成图片**，不能跳过。最多重试 2 次 |
| 发布返回 error（内容违规） | 重新生成段子，换一个选题再发 |
| 发布返回 error（网络/限流） | 等待 30 秒后重试 1 次 |
| 图片路径含中文 | MCP 会报错，必须使用无中文的英文路径 |
| 标题超过 20 字 | 截断到 20 字再提交 |
