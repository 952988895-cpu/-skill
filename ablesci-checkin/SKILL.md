---
name: ablesci-checkin
description: |
  AbleSci科研通自动签到Skill。自动化完成多个邮箱的签到任务，包含浏览器登录、签到、状态检测、结果评估。

  **首次使用 / 配置**：上传 Skill 文件后，对 AI 说「设置 AbleSci」，AI 自动检测 Skill 位置和配置文件状态，引导完成所有配置。

  **主动检测**：AI 读取本 Skill 后，无论以何种方式触发，均会主动检测配置文件状态并提示用户进行配置。

  **执行签到**：对 AI 说「执行 AbleSci 签到」，AI 自动读取配置并执行签到流程。

  **核心流程**：
  1. 读取邮箱配置（`account/ablesci/accounts.json`，相对于工作区根目录），获取邮箱和密码
  2. 打开浏览器，导航到 https://www.ablesci.com/
  3. 退出当前登录（确保处于未登录状态）
  4. 循环执行：账号1登录→签到→退出→账号2登录→签到→退出（自动遍历所有账号）
  5. 验证结果并更新Memory状态

  状态记忆：使用 MEMORY.md 或 memory/YYYY-MM-DD.md 记录签到状态
metadata:
  openclaw:
    emoji: "📚"
---

# AbleSci 自动签到 Skill

## 概述

本Skill用于自动完成AbleSci（科研通）的签到任务，支持多邮箱循环签到、状态检测、结果评估。

## 首次配置（Setup Wizard）

> **通用原则**：AI 读取本 Skill 后，无论以何种方式触发（主动读取 / 上传 / 触发词），**必须立即检测配置文件状态并主动提示用户进行配置**。不要等用户主动说触发词，AI 读 skill = 主动检测 + 主动提示。

### 首次读取：AI 自动检测并提示

AI 读取本 SKILL.md 后，立即执行两项检测（并行）：

#### 检测 A：Skill 文件位置
判断是否在标准路径 `skills/ablesci-checkin/`（相对于工作区根目录）。

#### 检测 B：账号配置文件状态
读取 `account/ablesci/accounts.json`（相对于工作区根目录）。

---

#### 检测完成后，AI 必须主动输出以下提示

**若 Skill 不在标准路径**（检测 A 发现）：
> 检测到 Skill 文件位置：`[实际路径]`
> 标准路径：`skills/ablesci-checkin/`
> 是否需要移动 Skill 到标准路径？（输入新路径以自定义位置，直接回车移动到标准路径，输入 skip 跳过）

**若配置文件不存在或无效**（检测 B 发现）：
> ⚠️ 未检测到 AbleSci 账号配置，需要先设置才能使用签到功能。
> 请提供你的科研通账号信息，我将自动完成配置。
> **输入方式**：直接回复你的 AbleSci 邮箱，我会引导你完成剩余步骤。
> 示例：`952988895@qq.com`

**若 Skill + 配置均正常**：
> ✅ AbleSci 签到功能已就绪！
> - 账号：X 个
> - 定时任务：已设置 / 未设置
> 如需修改配置，说「设置 AbleSci」即可。

### 步骤 0：AI 读取 Skill 并检测状态

AI 执行以下两项检测（并行进行）：

#### 检测 A：Skill 文件位置

读取本 SKILL.md，获取 Skill 所在目录。

判断标准路径：`skills/ablesci-checkin/`（相对于工作区根目录）

| 检测结果 | AI 行为 |
|---------|---------|
| Skill 在标准路径下 | 无需操作，继续 |
| Skill 不在标准路径下 | 询问用户是否移动到 `skills/ablesci-checkin/`，确认后执行移动 |

#### 检测 B：账号配置文件状态

读取 `account/ablesci/accounts.json`（相对于工作区根目录）。

| 检测结果 | 分支 |
|---------|------|
| 文件不存在 | 进入**情况 A（无配置）** |
| 文件存在但格式错误/无 ablesci key | 进入**情况 A（无有效配置）**，告知用户并清空重建 |
| 文件存在且有 ablesci key | 进入**情况 B（有配置）** |

---

### 情况 A：无配置（新建）

#### 情况 A-1：Skill 位置待确认

若检测 A 发现 Skill 不在标准路径：
- 告知用户 Skill 当前路径和标准路径
- 询问："是否将 Skill 移动到 `skills/ablesci-checkin/`？直接回车移动，输入其他路径则不动"
- 用户确认 → 执行移动
- 用户自定义 → 记录新路径，跳过移动

#### 情况 A-2：引导收集账号

告知用户"未检测到账号配置，开始引导配置"。

逐个账号收集，每账号依次：
1. **邮箱**：提示"请输入第 X 个账号的邮箱（科研通注册邮箱）"，检查是否包含 `@`，格式错误时提示重新输入
2. **密码**：提示"请输入对应密码"，确认非空
3. **名称（可选）**：提示"是否给这个账号起个名字？不输入则默认用邮箱前缀命名"

每完成一个账号后：
- 询问"还有更多账号吗？输入下一个邮箱继续，直接回车结束收集"
- 继续收集下一个账号
- 回车结束 → 进入情况 A-3

#### 情况 A-3：确认并写入

1. 汇总所有待写入账号（表格形式：名称 | 邮箱 | 密码掩码 `***`）
2. 提示"以上配置是否正确？确认后写入（回复 确认/Y/是，不正确请回复修改内容）"
3. **用户确认** → 写入 `account/ablesci/accounts.json`
   - 若 `account/ablesci/` 目录不存在 → **自动创建**
   - 若 `accounts.json` 文件不存在 → **自动创建**
4. **用户否定** → 根据用户回复定位问题，针对性修改后重新汇总确认

#### 情况 A-4：完成

输出配置摘要和使用说明（详见步骤 4）。

---

### 情况 B：有配置（查看/修改）

#### 情况 B-1：Skill 位置待确认

同上——若检测 A 发现 Skill 不在标准路径，询问是否移动。

#### 情况 B-2：展示现有配置

读取并列出当前所有账号（表格形式：序号 | 名称 | 邮箱，密码用 `***` 掩码）。

然后询问用户意图：

| 用户回复 | AI 行为 |
|---------|---------|
| `新增` / `+` | 继续情况 B-3 新增流程 |
| `修改` | 询问序号，然后只替换该账号的邮箱/密码/名称 |
| `删除` | 询问序号，删除后重新写入完整列表 |
| `重新配置` / `清空` | 清空所有账号，进入情况 A-2 重新收集 |
| `跳过` / `取消` | 结束配置，输出使用说明（步骤 4） |

#### 情况 B-3：新增账号

引导收集新账号（同情况 A-2，逐个收集），完成后：
1. 将新账号**追加**到现有账号列表（不是覆盖）
2. 汇总展示（含原有账号 + 新增账号）
3. 确认写入（同情况 A-3）

---

### 步骤 4：完成配置（情况 A 和 B 通用）

写入成功后，输出配置摘要和使用说明：

```
✅ 配置完成！

📋 账号列表：
  1. 账号名称 | email@example.com

📁 配置文件：account/ablesci/accounts.json

📖 使用方法：

【手动触发】
  对 AI 说：执行 AbleSci 签到

【定时自动签到】
  推荐设置时段 08:00-09:00，太早网站可能未开放
  设置方法：对 AI 说「每天 08:10 执行 AbleSci 签到」即可自动创建定时任务

⚠️ 注意：
  - 定时任务需要电脑保持开机、QClaw 运行；可以通过电源配置保证电脑处于唤醒状态
  - 首次使用建议手动触发一次，确认账号配置正确后再设置定时任务
```

---

### 旧配置迁移说明

如果 `account/ablesci/accounts.json` 配置文件已存在，但 Skill 文件在其他位置：
1. 告知用户配置文件已存在，显示现有账号（走情况 B-2 流程）
2. 询问是否需要移动 Skill 文件到标准路径 `skills/ablesci-checkin/`
3. 确认后执行移动（或告知用户手动移动）
4. 后续 Skill 更新只需替换 `skills/ablesci-checkin/SKILL.md`，不影响 `account/ablesci/accounts.json`
4. 后续 Skill 更新只需替换 `skills/ablesci-checkin/SKILL.md`，不影响 `account/ablesci/accounts.json`

## 邮箱配置

> **配置文件路径**：`account/ablesci/accounts.json`（相对于 OpenClaw 工作区根目录）
>
> **首次使用前**：使用「配置 AbleSci 签到」触发引导流程，AI 自动创建目录和文件。详见上方「首次配置（Setup Wizard）」章节。
>
> 工作区根目录通常位于：
> - Windows: `C:\Users\<用户名>\.qclaw\workspace\`
> - macOS/Linux: `~/.qclaw/workspace/`
>
> 因此完整路径示例（Windows）：`C:\Users\<用户名>\.qclaw\workspace\account\ablesci\accounts.json`

```json
{
  "ablesci": [
    {
      "name": "账号名称（可选，用于日志标识）",
      "email": "your-email@example.com",
      "password": "your-password"
    }
  ]
}
```
> 支持任意数量的账号，签到时会自动循环处理所有账号。

## 核心签到流程

### 步骤1：读取邮箱配置
- **必须读取** `account/ablesci/accounts.json`（相对于工作区根目录）
- 解析 JSON，分别提取**每个账号的邮箱和对应密码**
- 存为变量供后续使用（以下为示例，实际值从 JSON 中读取）：
  - `账号[i].email` = 第 i+1 个账号的邮箱
  - `账号[i].password` = 第 i+1 个账号的密码
- 账号总数 = `ablesci` 数组长度（动态检测，不固定）

### 步骤2：打开浏览器并初始化
- 使用 OpenClaw browser 工具
- 导航到 https://www.ablesci.com/
- 获取页面快照，检查当前登录状态

### 步骤3：确保未登录状态
- 如果页面显示"已连续签到X天"（已登录），先退出：
  - 访问 `https://www.ablesci.com/site/logout` 退出登录
  - 等待页面跳转到登录页
  - 确认页面显示"登录"链接（而非用户信息）

### 步骤4：循环签到（自动遍历所有账号）

> **循环规则**：遍历步骤1读取的 `ablesci` 数组，有多少个账号就循环多少次（0-indexed: i = 0, 1, 2, ...）
> 每个账号执行步骤 4.1 - 4.4，全部完成后进入步骤5。

#### 4.1 访问登录页并获取页面快照
- 导航到 `https://www.ablesci.com/site/login`
- 获取页面快照，找到：
  - 邮箱输入框的 ref（如 `textbox "邮箱"`）
  - 密码输入框的 ref（如 `textbox "密码"`）
  - 登录按钮的 ref（如 `button "登 录"`）

#### 4.2 输入账号并登录

**数据来源**：使用步骤1读取的变量（邮箱和密码都从 accounts.json 读取，不要凭记忆输入）
- 邮箱 = `当前账号.email`
- 密码 = `当前账号.password`

**ref 来源**：从步骤4.1的快照中获取（每次刷新都会变，不能硬编码）
- 邮箱输入框 ref = 快照中 `textbox "邮箱"` 的 ref
- 密码输入框 ref = 快照中 `textbox "密码"` 的 ref
- 登录按钮 ref = 快照中 `button "登 录"` 的 ref

**执行步骤（共3步）**：

1. 输入邮箱：
   ```
   act: { kind: "type", ref: 邮箱输入框ref, text: 当前账号.email }
   ```

2. 输入密码：
   ```
   act: { kind: "type", ref: 密码输入框ref, text: 当前账号.password }
   ```

3. 点击登录：
   ```
   act: { kind: "click", ref: 登录按钮ref }
   ```

4. 等待 2 秒，检查登录是否成功：
   - 使用 evaluate 检查页面状态：
     ```javascript
     document.querySelector('.fly-signin') ? 'LOGGED_IN' : document.querySelector('.user-login-box') ? 'LOGIN_PAGE' : 'UNKNOWN'
     ```
   - ✅ 返回 `LOGGED_IN`（跳转到首页，出现积分区域）→ 登录成功，继续步骤 4.2b
   - ❌ 返回 `LOGIN_PAGE`（仍在登录页）→ **登录失败**，记录失败原因，跳到步骤 4.4 处理下一个账号
   - ❌ 页面出现错误提示文字（如"密码错误"、"邮箱不存在"等）→ **登录失败**，记录错误信息，跳到步骤 4.4 处理下一个账号

#### 4.2b 检测今日签到状态（登录成功后执行）
**目的**：避免重复签到，判断今天是否已经签过。

**使用 evaluate 获取页面信息**：
```javascript
document.querySelector('.fly-signin').innerHTML
```

返回内容示例（已签到时）：
```html
<div>
  <a class="signin-points" ...>当前拥有<cite id="user-point-now">104</cite>积分</a>
  <span class="signin-days">已连续签到<cite id="sign-count">4</cite>天 ...</span>
</div>
```

**分两步判断**：

**步骤 4.2b-1（优先）**：检测"已连续签到X天"文本
- 解析 innerHTML，检查是否包含"已连续签到"
- ✅ **包含"已连续签到"** → 今日**已签到**，跳过步骤 4.3，直接进入步骤 4.4
- ❌ **不包含"已连续签到"** → 继续步骤 4.2b-2

**步骤 4.2b-2（仅 4.2b-1 未发现时执行）**：检测签到按钮
- 检查 innerHTML 中是否存在签到按钮元素（如 `<a>` 或 `<button>`）
- ✅ **有签到按钮** → 今日**未签到**，继续执行步骤 4.3
- ❌ **无签到按钮** → 无法判断，提示用户检查

#### 4.3 执行签到
- 前置条件：步骤 4.2b 确认今日未签到
- 在首页快照中找到签到按钮（包含"签到"文字的按钮/link）
- 点击签到按钮
- 等待页面更新

#### 4.3a 验证签到结果
获取页面快照，检查以下内容确认签到成功：
- 快照中出现 **"已连续签到X天"**（X为天数，签到后+1）
- **积分数值**（签到成功通常 +10）
- 如果快照中显示"已签到"或积分未变 → 可能今日已签过

#### 4.4 退出当前登录并处理下一个账号
- 访问 `https://www.ablesci.com/site/logout` 退出登录
- 等待页面跳转到登录页
- **循环处理下一个账号**：重复步骤4.1-4.4（使用下一个账号的邮箱和密码），直到所有账号处理完毕
- 如果上一个账号登录失败（步骤4.2 检测到），跳过退出，直接进入下一个账号的步骤4.1

### 步骤5：输出签到报告
- 汇总所有账号的签到结果（成功/已签到/登录失败），按账号顺序列出
- **登录失败的账号**：告知用户邮箱和失败原因（如"密码错误"、"仍停留在登录页"），无需记忆到 Memory
- 签到成功的账号：记录积分和连续天数到 memory/YYYY-MM-DD.md

## 关键操作细节

### 登录操作（3步）
```
错误方式：使用 fill（在此网站不兼容，登录会失败）
act: { kind: "fill", fields: [...] }

正确方式：直接 type 输入（实测可用，无需 click 聚焦）
1. act: { kind: "type", ref: "从快照获取的邮箱输入框ref", text: "从账号配置读取的邮箱" }
2. act: { kind: "type", ref: "从快照获取的密码输入框ref", text: "从账号配置读取的密码" }
3. act: { kind: "click", ref: "从快照获取的登录按钮ref" }
```

### 验证签到成功
使用 evaluate 获取 `.fly-signin` 的 innerHTML，从中提取：
- ✅ **"已连续签到X天"**（`<cite id="sign-count">X</cite>`）→ 确认连续天数更新
- ✅ **积分数值**（`<cite id="user-point-now">XXX</cite>`）→ 签到成功通常 +10
- ⚠️ 如果签到后积分未变 → 今日已签过

**推荐用 evaluate 而非 snapshot**：evaluate 直接返回 HTML 结构，比 snapshot 更稳定可靠，不受 selector 范围影响。

### ref 是动态的
- **必须从当前页面快照中获取**，不能硬编码
- 快照中定位方式：`textbox "邮箱"`、`textbox "密码"`、`button "登 录"`

### 页面状态判断
| 状态 | 页面特征 |
|------|----------|
| 已登录 | 显示"已连续签到X天"、积分数量、用户消息通知 |
| 未登录 | 显示"登录"链接、没有积分信息 |

### 登录页面URL
- **正确**：`https://www.ablesci.com/site/login`
- **错误**：`https://www.ablesci.com/user/login`（返回404）

### 退出登录URL
- `https://www.ablesci.com/site/logout`（会自动重定向到登录页）

### 签到按钮位置
- 首页右侧积分信息旁
- 文字通常为"今日打卡签到"或包含"签到"
- ref 是动态的，需从快照获取

## 使用方法

### 首次配置 / 查看 / 修改账号
```
设置 AbleSci
```
AI 自动读取本 Skill，检测 Skill 位置和配置文件状态，自动引导完成配置（新建/查看/新增/修改/删除）。详见上方「首次配置（Setup Wizard）」章节。

### 手动触发
```
执行 AbleSci 签到
```
AI 自动读取本 Skill 并执行签到流程。

### 定时自动触发（cron）

#### 如何对 AI 助手说
和手动触发一样简单，只需在前面加时间描述：

| 场景 | 你这样说 |
|------|---------|
| 每日签到 | `每天 08:10 执行 AbleSci 签到` |
| 临时测试 | `今天 12:00 执行 AbleSci 签到` |
| X分钟后 | `30分钟后执行 AbleSci 签到` |

助手会自动帮你创建对应的一次性或周期性 cron 任务。

#### Cron 配置参考

> ⚠️ **payload.message 规则**：
> - 必须包含"读取 skill 文件"指令（隔离会话无上下文，不读 skill 会凭记忆操作）
> - 禁止调用 message 工具（由 announce 机制自动投递结果）
> - **不要重复 skill 里的步骤细节**，skill 文件本身就是完整流程

**每日定时签到**：
```json
{
  "name": "AbleSci_Daily_Checkin",
  "schedule": {"kind":"cron","expr":"10 8 * * *","tz":"Asia/Shanghai"},
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "每天 08:10 自动签到。执行 AbleSci 签到，必须先读取 skill 文件 skills/ablesci-checkin/SKILL.md。禁止调用message工具。"
  }
}
```

**一次性签到测试**：
```json
{
  "name": "AbleSci_Test",
  "schedule": {"kind":"at","at":"2026-04-08T12:00:00+08:00"},
  "sessionTarget": "isolated",
  "deleteAfterRun": true,
  "payload": {
    "kind": "agentTurn",
    "message": "今天 12:00 签到测试。执行 AbleSci 签到，必须先读取 skill 文件 skills/ablesci-checkin/SKILL.md。禁止调用message工具。"
  }
}
```

**周期间隔签到**（每30分钟）：
```json
{
  "name": "AbleSci_Interval",
  "schedule": {"kind":"every","everyMs":1800000},
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "每 30 分钟自动签到。执行 AbleSci 签到，必须先读取 skill 文件 skills/ablesci-checkin/SKILL.md。禁止调用message工具。"
  }
}
```

> **Skill 文件路径说明**：`skills/ablesci-checkin/SKILL.md` 相对于 OpenClaw 工作区根目录（如 `~/.qclaw/workspace/skills/ablesci-checkin/SKILL.md`）。隔离会话与主会话在同一工作区，可直接用相对路径。

## 依赖工具

- **OpenClaw Browser** - 浏览器自动化操作
- **Memory 工具** - 状态记录和读取
- **cron 工具** - 定时任务触发

## 注意事项

1. **输入方式**：直接 type 输入，无需 click 聚焦，禁止使用 fill
2. **数据来源**：从 accounts.json 配置文件读取，不要凭记忆输入
3. **ref 来源**：从当前页面快照获取，每次刷新都会变，不能硬编码
4. **验证方式**：签到后从快照确认"已连续签到X天"和积分变化
5. **保险步骤**：保留登录状态检测和退出步骤，确保多账号切换不出错
6. 网站登录可能需要验证码，需用户协助
7. 签到按钮位置可能随网站更新变化
8. 签到后记录日志到 memory/YYYY-MM-DD.md
