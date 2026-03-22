# OpenClaw 微博助手部署排查指南

> 本文档基于 2026-03-22 在两台服务器（175.24.228.165 / 111.229.157.247）的实战排查经验编写。  
> 目标：新环境部署后，按此清单逐项验证，避免重复踩坑。

## 快速验证流程（部署后必做）

部署完成后，**按顺序**执行以下 5 项验证。每一项都必须通过，否则企微用户触发"登录微博"时必然失败。

### 第 1 步：确认 tools.deny 未禁用浏览器

```bash
openclaw config get tools
```

**正确输出**应该只有 `profile: full`，**不能**出现 `deny` 字段：
```json
{ "profile": "full" }
```

**如果看到 `deny` 列表包含 `browser`、`web_search`、`web_fetch`**：
```bash
# 用 python3 直接删除（openclaw config set 无法删除字段）
python3 -c "
import json, os
path = os.path.expanduser('~/.openclaw/openclaw.json')
cfg = json.load(open(path))
if 'deny' in cfg.get('tools', {}):
    del cfg['tools']['deny']
    json.dump(cfg, open(path, 'w'), indent=2)
    print('已删除 tools.deny')
else:
    print('tools.deny 不存在，无需操作')
"
openclaw gateway restart
```

**原理**：`tools.deny` 优先级高于 `tools.profile`。即使 profile 是 `full`，deny 列表中的工具仍然不可用。接手别人部署的机器时，这是最常见的隐藏坑。

---

### 第 2 步：确认浏览器正常运行

```bash
openclaw browser status
```

**正确输出**应包含 `running: true`。如果 `running: false`：
```bash
openclaw browser start
# 再次检查
openclaw browser status
```

进一步验证浏览器可操作：
```bash
openclaw browser navigate https://passport.weibo.com/sso/signin
openclaw browser snapshot | head -20
```

应看到微博登录页的 DOM 内容。如果报错"无法连接到浏览器"，检查 Chromium 进程和 CDP 端口。

---

### 第 3 步：确认 weibo-assistant Skill 已被 OpenClaw 加载

```bash
# 检查 Skill 文件是否存在
ls ~/.openclaw/workspace/skills/weibo-assistant/SKILL.md

# 通过 CLI 创建临时 session，检查 skill 是否在 snapshot 中
openclaw agent --agent main -m "你好" --json 2>/dev/null | python3 -c "
import sys, json
data = json.load(sys.stdin)
report = data.get('systemPromptReport', {})
skills = report.get('skills', {}).get('entries', [])
names = [s.get('name') for s in skills]
print(f'总共 {len(names)} 个 skill')
if 'weibo-assistant' in names:
    weibo = [s for s in skills if s.get('name') == 'weibo-assistant'][0]
    print(f'✅ weibo-assistant 已加载，blockChars={weibo.get(\"blockChars\")}')
    if weibo.get('blockChars', 0) < 500:
        print('⚠️  警告：blockChars 过小，description 可能不完整')
else:
    print('❌ weibo-assistant 未找到！')
    print(f'  已有 skill: {names}')
"
```

**正确输出**：`✅ weibo-assistant 已加载，blockChars=813`（或类似的 >500 的值）。

**如果 blockChars < 300**：说明 description 还是旧版短摘要，Agent 看不到完整操作步骤。需要更新 SKILL.md。

**如果 weibo-assistant 未找到**：
1. 检查文件路径是否正确
2. `openclaw gateway restart` 后重试
3. 如果仍未出现，检查 SKILL.md 的 YAML frontmatter 格式是否正确

---

### 第 4 步：清理企微旧 Session

**这一步至关重要**。OpenClaw 的企微 session 会缓存 `skillsSnapshot`，创建后不再刷新。如果 session 创建于 Skill 部署之前，Agent 将永远看不到 weibo-assistant。

```bash
# 查看所有企微 session 及其 skill 数量
python3 -c "
import json, os
path = os.path.expanduser('~/.openclaw/agents/main/sessions/sessions.json')
d = json.load(open(path))
for k in d:
    if 'wecom' in k and 'cron' not in k:
        snap = d[k].get('skillsSnapshot', {})
        skills = snap.get('resolvedSkills', snap.get('skills', []))
        if isinstance(skills, list):
            names = [s.get('name') if isinstance(s, dict) else s for s in skills]
            has_weibo = 'weibo-assistant' in names
        else:
            names = []
            has_weibo = False
        print(f'{k}')
        print(f'  skills: {len(names)}, has_weibo: {has_weibo}')
        if not has_weibo:
            print(f'  ⚠️  该 session 缺少 weibo-assistant，需要删除重建！')
"
```

**如果任何企微 session 缺少 weibo-assistant**：
```bash
# 备份
cp ~/.openclaw/agents/main/sessions/sessions.json ~/.openclaw/agents/main/sessions/sessions.json.bak

# 删除缺少 weibo-assistant 的企微 session
python3 -c "
import json, os
path = os.path.expanduser('~/.openclaw/agents/main/sessions/sessions.json')
d = json.load(open(path))
to_delete = []
for k in d:
    if 'wecom' in k and 'cron' not in k:
        snap = d[k].get('skillsSnapshot', {})
        skills = snap.get('resolvedSkills', snap.get('skills', []))
        names = [s.get('name') if isinstance(s, dict) else s for s in skills] if isinstance(skills, list) else []
        if 'weibo-assistant' not in names:
            to_delete.append(k)
for k in to_delete:
    # 删除关联的 .jsonl 历史文件
    session_id = d[k].get('id', '')
    if session_id:
        jsonl_path = os.path.expanduser(f'~/.openclaw/agents/main/sessions/{session_id}.jsonl')
        if os.path.exists(jsonl_path):
            os.remove(jsonl_path)
            print(f'已删除历史: {session_id}.jsonl')
    del d[k]
    print(f'已删除 session: {k}')
json.dump(d, open(path, 'w'), indent=2)
print(f'共删除 {len(to_delete)} 个 session')
"
```

下次用户通过企微发消息时，OpenClaw 会自动创建新 session，新 session 的 skillsSnapshot 将包含最新的全部 skill。

---

### 第 5 步：端到端测试

```bash
# 用 CLI 模拟企微用户发送"登录微博"
openclaw agent --agent main -m "登录微博" --timeout 180
```

**正确行为**（按顺序）：
1. Agent 运行 `python3 .../weibo_cookies.py check`
2. Agent 执行 `browser navigate https://passport.weibo.com/sso/signin`
3. Agent 等待 3 秒
4. Agent 执行 `browser screenshot --element ".flex-col.items-center.justify-center"`
5. Agent 回复包含截图和扫码提示

**异常排查**：

| 异常表现 | 可能原因 | 解决方案 |
|---------|---------|---------|
| Agent 说"没有浏览器工具" | `tools.deny` 禁用了 browser | 回到第 1 步 |
| Agent 打开 `weibo.com` 首页 | Skill 未加载或 description 太短 | 回到第 3/4 步 |
| Agent 打开 `passport.weibo.cn` | SKILL.md 是旧版 | 更新 SKILL.md（从 GitHub 拉取最新版） |
| 截图是微博 logo（很小的图） | 用了 `--element img` | 确认 description 中选择器是 `.flex-col.items-center.justify-center` |
| 截图空白或报错 | 二维码未加载 | 确认网络可达 passport.weibo.com |
| Agent 完全不提微博 | session 没有 weibo-assistant skill | 回到第 4 步清理 session |

---

## 常见陷阱速查

### 陷阱 1：`tools.deny` 优先级高于 `tools.profile`

`openclaw.json` 中如果同时存在：
```json
{
  "tools": {
    "profile": "full",
    "deny": ["browser", "web_search", "web_fetch"]
  }
}
```
Agent **仍然无法使用浏览器**。`deny` 列表是硬性禁止，profile 无法覆盖。接手别人的机器时必须检查。

### 陷阱 2：OpenClaw Skill 只注入 description，不注入 body

OpenClaw 的 Skill 机制中，**只有 YAML frontmatter 中的 `description` 字段会被注入到 Agent 的 system prompt**。SKILL.md 的 body 部分（`---` 下方的 Markdown 内容）受懒加载机制控制，通常不会被注入。

**判断方法**：查看 `systemPromptReport.skills.entries` 中 `blockChars` 的值。如果只有几百个字符（如 229），说明 Agent 只看到了 description，没看到 body。

**解决方案**：将所有关键操作步骤直接写在 description 中（使用 YAML `|` 多行字符串语法）。当前 SKILL.md 的 description 为 813 字符，包含完整的登录步骤、选择器和 URL。

### 陷阱 3：Session skillsSnapshot 创建后不刷新

企微 session 的 `skillsSnapshot` 在 session 创建时一次性生成。如果 session 创建于以下场景之一：
- Gateway restart 后的短暂窗口期（Skill 尚未全部加载）
- Skill 文件部署之前
- Skill 文件有格式错误

那么该 session 将**永远缺少**对应的 Skill，后续发多少消息都不会修复。

**唯一修复方式**：删除旧 session，让 OpenClaw 创建新 session。

### 陷阱 4：`passport.weibo.cn` 已废弃

截至 2026 年 3 月，`passport.weibo.cn/signin/login` 已经 302 重定向到 `passport.weibo.com/sso/signin`。直接使用后者可以避免不必要的重定向。

`passport.weibo.com/sso/signin` 在云服务器（腾讯云/阿里云等）上不会被"异常网络环境"拦截，二维码正常显示。但 `weibo.com` 首页会被拦截。

### 陷阱 5：截图选择器 `--element img` 会截到 logo

`passport.weibo.com/sso/signin` 页面的第一个 `<img>` 元素是微博 logo（112x36 像素），第二个才是二维码。使用 `--element img` 会截到 logo。

正确选择器：`--element ".flex-col.items-center.justify-center"`，截取整个二维码面板（约 330x475 像素），包含二维码图片和"扫描二维码登录"提示文字。

---

## 新环境部署完整流程

```bash
# 1. 克隆仓库
git clone https://github.com/lewisjlzeng/weibo-assistant.git
cd weibo-assistant

# 2. 复制 Skill 到 OpenClaw
cp -r weibo-assistant/ ~/.openclaw/workspace/skills/

# 3. 运行一键安装（系统依赖 + Playwright + Chromium）
bash ~/.openclaw/workspace/skills/weibo-assistant/scripts/setup.sh

# 4. 按上方"快速验证流程"逐项检查
# 第 1 步：检查 tools.deny
# 第 2 步：检查浏览器状态
# 第 3 步：确认 Skill 已加载（blockChars > 500）
# 第 4 步：清理旧企微 session
# 第 5 步：CLI 端到端测试

# 5. 在企微中发"登录微博"做最终验证
```
