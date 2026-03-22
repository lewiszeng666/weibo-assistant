---
name: weibo-assistant
description: |
  微博自动化助手。当用户说"登录微博""微博消息""微博热搜""谁给我点赞""微博动态"时激活。
  ⛔ 禁止访问 weibo.com 首页（会被拦截）。
  【登录微博】步骤：
  1. python3 {baseDir}/scripts/weibo_cookies.py check — 检查Cookie，valid:true则跳到恢复登录
  2. browser navigate https://passport.weibo.com/sso/signin — 打开登录页
  3. 等待3秒让二维码加载
  4. browser screenshot --element ".flex-col.items-center.justify-center" — 截取二维码面板（⚠️禁止用 --element img，那会截到logo！）。失败则用 browser screenshot 全屏截图
  5. 发送截图+提示"📱 请用微博APP扫描二维码登录（我的页面→扫一扫）。扫码后回复已扫码"
  6. 用户回复已扫码后：browser wait --url m.weibo.cn --timeout 15000 → browser navigate https://m.weibo.cn/ → 验证Cookie含SUB= → 保存Cookie
  【恢复登录】python3 {baseDir}/scripts/weibo_cookies.py load → 逐个 browser cookies set → browser navigate https://m.weibo.cn/ 验证
  【信息抓取】全部用 m.weibo.cn：消息=/message，评论=/message/cmt，点赞=/message/attitude，热搜=API containerid=106003，首页=/feed/friends
metadata: { "openclaw": { "os": ["linux"], "emoji": "📱", "requires": { "bins": ["python3", "chromium"], "config": ["browser.enabled"] } } }
---

# 微博自动化助手

**⛔ 禁止使用 weibo.com 首页（会被拦截）**
**✅ 登录统一使用 passport.weibo.com/sso/signin**
**✅ 浏览/API 使用 m.weibo.cn（移动端）**

## 登录微博

当用户说"登录微博"时，按以下步骤操作：

**第1步**：检查 Cookie
```bash
python3 {baseDir}/scripts/weibo_cookies.py check
```
如果 `valid: true`，跳到"恢复登录"。否则继续第2步。

**第2步**：打开登录页
```
browser navigate https://passport.weibo.com/sso/signin
```
等待 3 秒让二维码加载。

**第3步**：截取二维码面板（⚠️ 不要用 `--element img`，那会截到 logo）
```
browser screenshot --element ".flex-col.items-center.justify-center"
```
如果失败，备选：`browser screenshot`（全屏截图）

**第4步**：发送截图给用户，提示：
> 📱 请用微博 APP 扫描二维码登录（我的页面→扫一扫）。扫码后回复"已扫码"。

**第5步**（用户回复"已扫码"后）：等待跳转并保存 Cookie
```
browser wait --url m.weibo.cn --timeout 15000
browser navigate https://m.weibo.cn/
```
验证登录：
```
browser evaluate --fn '() => { return document.cookie.includes("SUB=") ? "logged_in" : "not_logged_in"; }'
```
保存 Cookie：
```bash
openclaw browser cookies 2>/dev/null | grep -v '^\[plugins\]' | python3 {baseDir}/scripts/weibo_cookies.py save
```
告诉用户"✅ 登录成功！Cookie 已保存。"

### 恢复登录（已有 Cookie）

```bash
python3 {baseDir}/scripts/weibo_cookies.py load
```
对每个 cookie 执行：`browser cookies set --name <name> --value <value> --domain <domain> --path <path>`
然后 `browser navigate https://m.weibo.cn/` 验证。

## 查看消息

```
browser navigate https://m.weibo.cn/message
browser snapshot --labels
```
API备选：`browser evaluate --fn '() => fetch("https://m.weibo.cn/api/msg/list?page=1").then(r=>r.json())'`

## 查看评论

```
browser navigate https://m.weibo.cn/message/cmt
browser snapshot --labels
```
API备选：`browser evaluate --fn '() => fetch("https://m.weibo.cn/api/msg/cmt?page=1").then(r=>r.json())'`

## 查看点赞

```
browser navigate https://m.weibo.cn/message/attitude
browser snapshot --labels
```
API备选：`browser evaluate --fn '() => fetch("https://m.weibo.cn/api/msg/attitude?page=1").then(r=>r.json())'`

## 查看热搜

```
browser evaluate --fn '() => fetch("https://m.weibo.cn/api/container/getIndex?containerid=106003type%3D25%26t%3D3%26disable_hot%3D1%26filter_type%3Drealtimehot").then(r=>r.json())'
```
提取前 15 条热搜。

## 查看首页动态

```
browser evaluate --fn '() => fetch("https://m.weibo.cn/feed/friends?max_id=0").then(r=>r.json())'
```

## 推送格式

```
📊 微博动态摘要（YYYY-MM-DD HH:MM）

🔥 热搜 TOP5：
1. [标题]（热度）
...

📩 新消息（X条）：
· [发送者]: [内容摘要]

💬 新评论（X条）：
· [评论者] 评论了"[微博摘要]": [评论内容]

👍 新点赞（X个）：
· [点赞者] 赞了"[微博摘要]"
```

## Cookie 监控

每次任务前先 `python3 {baseDir}/scripts/weibo_cookies.py check`。
- `hours_remaining > 24`：正常执行
- `hours_remaining < 24`：执行 + 提醒用户续期
- `valid: false`：通知用户重新扫码
