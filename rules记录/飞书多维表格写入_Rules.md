# 顶层 Rules：飞书多维表格写入 / GPT Action / Codex 项目规则

> 适用对象：以后所有涉及 Codex、GPT、Vercel、GitHub、飞书多维表格写入的项目。

---

## 1. 总体目标规则

当用户要实现“GPT / Codex / OpenClaw / Vercel 写入飞书多维表格”时，默认架构为：

```text
自然语言输入
→ GPT 结构化字段
→ GPT Action 或后端接口
→ Vercel 公网 HTTPS 服务
→ 飞书开放平台 API
→ 飞书多维表格新增 / 更新记录
```

不要一开始就做复杂功能，第一版只跑通：

```text
新增一条记录
```

---

## 2. 安全规则

1. 不允许把任何真实密钥写入 GitHub。
2. 不允许把 FEISHU_APP_SECRET、用户 token、Vercel token、GitHub token 写进代码。
3. 所有密钥必须通过 Vercel Environment Variables 注入。
4. README 只能写变量名，不能写变量值。
5. 报错时可以返回飞书错误码、msg、log_id，但不得返回真实 secret。
6. 如果用户截图里出现 App Secret 或 token，应提醒其打码或重置。

---

## 3. 飞书权限三层规则

飞书写入多维表格时，必须同时检查三层权限：

### 第一层：开放平台 API 权限

确认应用已开通：

```text
多维表格读取 / 编辑 / 新增记录 / 更新记录
Wiki 或知识库节点读取
云文档相关读取权限
```

### 第二层：应用版本发布

开通权限后必须：

```text
创建版本 → 发布
```

否则权限不会生效。

### 第三层：目标文档授权

目标多维表格必须添加文档应用：

```text
多维表格右上角 ...
→ 更多
→ 添加文档应用
→ 添加当前自建应用
→ 权限设为可编辑
```

如果出现：

```text
HTTP 403
code 91403
forbidden
```

优先检查第三层，不要只反复开 API 权限。

---

## 4. Wiki 链接规则

如果飞书链接是：

```text
/wiki/{token}?table=tblxxx
```

不能直接把 wiki token 当 app_token。

必须走：

```text
WIKI_TOKEN
→ wiki get_node
→ node.obj_token
→ bitable app_token
→ bitable records API
```

Vercel 环境变量应使用：

```text
FEISHU_APP_ID
FEISHU_APP_SECRET
WIKI_TOKEN
BITABLE_TABLE_ID
```

如果链接是：

```text
/base/{token}?table=tblxxx
```

才可以直接使用：

```text
BITABLE_APP_TOKEN
BITABLE_TABLE_ID
```

---

## 5. 字段一致性规则

飞书 bitable records API 写入 fields 时，字段名必须和飞书表头完全一致。

字段名、中文字符、空格、标点都必须一致。

当前任务表标准字段为：

```text
任务名称
任务类型
主线
优先级
状态
任务日
提醒时间
任务目标
AI介入方式
复盘
```

API 字段映射为：

```text
title → 任务名称
task_type → 任务类型
mainline → 主线
priority → 优先级
status → 状态
task_date → 任务日
remind_at → 提醒时间
task_goal → 任务目标
ai_method → AI介入方式
review → 复盘
```

不得写入不存在的字段：

```text
最小完成标准
下一步动作
```

如果出现：

```text
FieldNameNotFound
```

优先检查字段名，而不是权限。

---

## 6. 单选字段规则

如果字段是单选，写入值必须是已有选项。

当前 AI介入方式可选：

```text
不使用
辅助思考
检索资料
生成草稿
写代码
复盘总结
```

如果出现：

```text
invalid field value
```

先检查单选选项是否存在。

---

## 7. Vercel 规则

1. 所有后端接口放在 `/api` 目录。
2. 环境变量放在 Vercel Project Environment Variables。
3. 添加或修改环境变量后必须 Redeploy。
4. `vercel.json` 保持简单：

```json
{
  "version": 2
}
```

5. 不要写不确定 runtime，例如：

```json
"runtime": "nodejs20.x"
```

6. 如果 GET 返回 405 且 allow: POST，说明接口存在，是好信号。

---

## 8. GPT Action 规则

1. 同一个 Vercel 域名只保留一个 Action。
2. 不要为 `/api/task-test` 和 `/api/create-task` 分别新建 Action。
3. 所有接口放进同一个 OpenAPI schema。
4. `servers.url` 必须是真实地址：

```yaml
servers:
  - url: https://feishu-ai-task-agent.vercel.app
```

5. 不得保留占位符：

```yaml
https://YOUR_VERCEL_PROJECT.vercel.app
```

6. schema 更新后必须在 GPT Preview 测试。
7. 不要声称“已写入飞书”，除非 Action 返回 `ok: true`。

---

## 9. Codex / WSL / GitHub 规则

1. Codex 主要负责改代码，不强依赖 Codex 本地 shell。
2. 如果 Codex 报本地 shell 不可用，不要卡住。
3. 使用 WSL 执行 git 和 curl。
4. 标准命令：

```bash
git status
git add .
git commit -m "..."
git push
```

5. GitHub push 后检查 Vercel 是否部署最新 commit。
6. 如果 Vercel 没自动部署，手动 Redeploy。

---

## 10. 标准排查顺序

遇到任何问题，按层排查，不要跳层：

```text
1. GPT Action schema 是否正确
2. server URL 是否真实
3. Vercel endpoint 是否存在
4. Vercel 是否部署最新 commit
5. 环境变量是否配置并重新部署
6. tenant_access_token 是否成功
7. wiki token 是否能解析 app_token
8. 开放平台 API 权限是否开通
9. 应用版本是否发布
10. 目标多维表格是否添加文档应用且可编辑
11. 字段名是否完全一致
12. 单选字段选项是否存在
13. 日期格式是否正确
```

---

## 11. 常见错误判断规则

```text
NOT_FOUND
→ Vercel 没部署到该接口，检查 GitHub 最新 commit 和 Vercel Deployment。

HTTP 405 allow: POST
→ 接口存在，GET 不允许，继续用 POST 测试。

Missing required environment variable
→ Vercel 环境变量缺失或没 redeploy。

wiki:node:read required
→ 飞书应用缺 Wiki / 知识库节点读取权限，开权限并发布版本。

HTTP 403 / 91403 forbidden
→ 目标多维表格没有添加文档应用或没有可编辑权限。

FieldNameNotFound
→ 代码字段名和飞书表头字段名不一致。

invalid field value
→ 单选选项不存在或日期格式不对。
```

---

## 12. 默认测试请求

```bash
curl -X POST "https://feishu-ai-task-agent.vercel.app/api/create-task" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "复盘第一份实习",
    "task_type": "主线任务",
    "mainline": "求职",
    "priority": "P0",
    "status": "未开始",
    "task_date": "2026-05-21",
    "remind_at": "2026-05-21 09:30",
    "task_goal": "写出职责、流程、痛点、成果，并整理成可放进简历的表达",
    "ai_method": "辅助思考",
    "review": ""
  }'
```
