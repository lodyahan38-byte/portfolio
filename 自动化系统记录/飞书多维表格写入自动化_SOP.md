# 飞书多维表格写入自动化 SOP

> 适用场景：ChatGPT / 自定义 GPT / Codex / Vercel / OpenClaw 等外部工具，需要通过飞书开放平台 API 向飞书多维表格新增、读取或更新记录。

---

## 0. 这类操作是什么？

这类操作本质是：

```text
自然语言输入
→ GPT 结构化任务字段
→ GPT Action / 后端接口
→ Vercel 公网服务
→ 飞书开放平台 API
→ 飞书多维表格新增或更新记录
```

当前项目最终链路：

```text
手机 ChatGPT
→ 自定义 GPT
→ GPT Action
→ Vercel /api/create-task
→ 飞书开放平台 API
→ 飞书多维表格实时新增任务
```

---

## 1. 标准架构

### 1.1 组件分工

| 组件 | 负责什么 | 注意事项 |
|---|---|---|
| ChatGPT / 自定义 GPT | 理解自然语言，整理结构化字段 | 不直接操作飞书，只调用 Action |
| GPT Action | 根据 OpenAPI schema 调用后端接口 | 同一个域名只保留一个 Action |
| Vercel | 提供公网 HTTPS 后端接口 | 真实密钥放 Environment Variables |
| GitHub | 保存代码并触发 Vercel 部署 | 不提交任何真实密钥 |
| Codex | 修改代码、生成接口、生成 schema | 不强依赖 Codex 本地 shell |
| WSL 终端 | git commit / push / curl 测试 | 最稳定的执行命令入口 |
| 飞书开放平台 | 认证、权限、API 调用 | 权限需要开通并发布版本 |
| 飞书多维表格 | 存储最终任务数据 | 字段名必须和代码完全一致 |

---

## 2. 飞书多维表格设计 SOP

### 2.1 先定字段，不要边写代码边改字段

当前推荐任务字段：

| 字段名 | 建议类型 | 用途 |
|---|---|---|
| 任务名称 | 文本 | 任务标题 |
| 任务类型 | 单选 | 主线任务 / 支线任务 |
| 主线 | 单选 | 求职 / 论文 / AI实践 |
| 优先级 | 单选 | P0 / P1 / P2 / P3 |
| 状态 | 单选 | 未开始 / 进行中 / 已完成 / 暂停 / 取消 |
| 任务日 | 日期 | 任务安排在哪天 |
| 提醒时间 | 日期时间 | 具体提醒时间 |
| 任务目标 | 文本 / 多行文本 | 本次任务做到什么程度 |
| AI介入方式 | 单选 | AI 以什么方式参与 |
| 复盘 | 文本 / 多行文本 | 完成后的总结 |

### 2.2 AI介入方式建议选项

```text
不使用
辅助思考
检索资料
生成草稿
写代码
复盘总结
```

这个字段建议长期保留，因为它能帮助沉淀“做事前判断 AI 能不能做、能不能做好”的能力。

---

## 3. 飞书开放平台应用 SOP

### 3.1 创建应用

```text
飞书开放平台
→ 开发者后台
→ 创建应用
→ 企业自建应用
```

应用名示例：

```text
ChatGPT 任务同步助手
```

### 3.2 保存应用凭证

在应用后台的「应用凭证」中保存：

```text
FEISHU_APP_ID
FEISHU_APP_SECRET
```

注意：

```text
App Secret 绝对不要写进 GitHub，不要发给任何人。
```

### 3.3 开通 API 权限

至少开通以下类型权限：

```text
多维表格读取
多维表格编辑
多维表格记录新增
多维表格记录编辑
Wiki / 知识库节点读取
云文档相关读取权限
```

如果有「用户身份」和「应用身份」，优先开「应用身份」。测试阶段可以两种都开。

### 3.4 发布版本

每次新增权限后都必须：

```text
创建版本
→ 发布
```

否则权限不会生效。

---

## 4. 飞书权限三层模型

飞书写入多维表格要同时满足三层权限。

### 第一层：开放平台 API 权限

确认应用已开通：

```text
多维表格 API 权限
Wiki / 知识库节点读取权限
云文档相关权限
```

如果缺这一层，常见报错：

```text
Access denied
wiki:node:read required
scope required
```

### 第二层：应用版本已发布

开了权限但没发布，等于没生效。

如果刚开完权限仍然报权限错误，先检查：

```text
应用是否已经创建版本并发布
```

### 第三层：目标文档授权

这是本次最关键卡点。

普通分享协作者里可能搜不到机器人 / 应用。正确路径是：

```text
打开目标多维表格
→ 右上角「...」
→ 更多
→ 添加文档应用
→ 添加当前自建应用
→ 权限设为「可编辑」
```

如果缺这一层，常见报错：

```text
Create bitable record failed with HTTP 403
code: 91403
msg: forbidden
```

优先处理方式：

```text
不是继续盲目开 API 权限，而是到目标多维表格里添加文档应用并给可编辑权限。
```

---

## 5. Wiki 链接处理 SOP

### 5.1 识别链接类型

如果链接长这样：

```text
https://xxx.feishu.cn/base/xxxxxx?table=tblxxxxxx
```

那么：

```text
base/ 后面的 xxxxxx = BITABLE_APP_TOKEN
table= 后面的 tblxxxxxx = BITABLE_TABLE_ID
```

如果链接长这样：

```text
https://xxx.feishu.cn/wiki/CFuZwssomiNTcpkgPY3cFvVXnnG?table=tbl2BohwZEV7c5i7
```

那么：

```text
/wiki/ 后面的 CFuZwssomiNTcpkgPY3cFvVXnnG = WIKI_TOKEN
table= 后面的 tbl2BohwZEV7c5i7 = BITABLE_TABLE_ID
```

### 5.2 Wiki 链接不能直接当 app_token 用

Wiki 链接需要走转换：

```text
WIKI_TOKEN
→ 调用 wiki get_node
→ 获取 node.obj_token
→ 把 obj_token 当真实多维表格 app_token
→ 调用 bitable records API
```

### 5.3 环境变量配置

如果使用 Wiki 链接，Vercel 环境变量至少需要：

```text
FEISHU_APP_ID
FEISHU_APP_SECRET
WIKI_TOKEN
BITABLE_TABLE_ID
```

---

## 6. Vercel 配置 SOP

### 6.1 创建项目

```text
Vercel
→ New Project
→ Import Git Repository
→ 选择 GitHub 仓库
→ Deploy
```

### 6.2 配置环境变量

路径：

```text
Project Settings
→ Environments / Environment Variables
→ Add Environment Variable
```

添加：

```text
FEISHU_APP_ID
FEISHU_APP_SECRET
WIKI_TOKEN
BITABLE_TABLE_ID
```

变量名必须一字不差，大写、下划线都不能错。

### 6.3 修改环境变量后必须重新部署

每次新增或修改环境变量后：

```text
Deployments
→ 最新部署
→ Redeploy
```

否则线上函数读不到新变量。

### 6.4 vercel.json 保持简单

推荐：

```json
{
  "version": 2
}
```

不要写：

```json
"runtime": "nodejs20.x"
```

否则可能报：

```text
Function Runtimes must have a valid version
```

---

## 7. 后端接口 SOP

### 7.1 最小接口结构

保留测试接口：

```text
POST /api/task-test
```

正式写入接口：

```text
POST /api/create-task
```

### 7.2 create-task 标准逻辑

```text
1. 检查请求方法必须是 POST
2. 读取环境变量
3. 用 FEISHU_APP_ID + FEISHU_APP_SECRET 获取 tenant_access_token
4. 用 WIKI_TOKEN 调用 wiki get_node
5. 得到真实 app_token
6. 根据请求体构造 fields
7. 调用 bitable create record
8. 返回 ok / record_id / 飞书原始错误
```

### 7.3 字段映射

API 请求字段：

```json
{
  "title": "复盘第一份实习",
  "task_type": "主线任务",
  "mainline": "求职",
  "priority": "P0",
  "status": "未开始",
  "task_date": "2026-05-21",
  "remind_at": "2026-05-21 09:30",
  "task_goal": "写出职责、流程、痛点、成果",
  "ai_method": "辅助思考",
  "review": ""
}
```

映射关系：

| API 字段 | 飞书字段 |
|---|---|
| title | 任务名称 |
| task_type | 任务类型 |
| mainline | 主线 |
| priority | 优先级 |
| status | 状态 |
| task_date | 任务日 |
| remind_at | 提醒时间 |
| task_goal | 任务目标 |
| ai_method | AI介入方式 |
| review | 复盘 |

不要再写入不存在字段：

```text
最小完成标准
下一步动作
```

---

## 8. 字段名一致性规则

飞书新增记录时，fields 里的字段名必须和表格字段名完全一致。

错误示例：

```text
代码写：最小完成标准
表格实际：任务目标
```

会报：

```text
FieldNameNotFound
```

排查方法：

```text
1. 看飞书表头真实字段名
2. 看 api/create-task.js 里的 fields 映射
3. 一字不差对齐
4. 单选字段还要确认选项值也存在
```

建议后续新增调试接口：

```text
GET /api/list-fields
```

用于读取飞书真实字段名，避免肉眼对字段。

---

## 9. GPT Action Schema SOP

### 9.1 核心原则

同一个 Vercel 域名只保留一个 Action。

不要这样：

```text
Action 1: /api/task-test
Action 2: /api/create-task
```

要这样：

```text
同一个 Action schema 中包含：
- POST /api/task-test
- POST /api/create-task
```

否则会报：

```text
Cannot have multiple custom actions for the same domain
```

### 9.2 server URL 必须是真实地址

不要保留占位符：

```yaml
servers:
  - url: https://YOUR_VERCEL_PROJECT.vercel.app
```

必须改为：

```yaml
servers:
  - url: https://feishu-ai-task-agent.vercel.app
```

### 9.3 替换 schema 步骤

```text
自定义 GPT
→ 配置
→ Actions
→ 打开原来的 Action
→ Ctrl + A 删除旧 schema
→ 粘贴 gpt-action-openapi.yaml 全部内容
→ 检查是否识别出 /api/task-test 和 /api/create-task
→ 保存 / 更新
```

---

## 10. Codex / GitHub / WSL 工作流 SOP

### 10.1 固定分工

```text
Codex = 改代码 / 生成文件
WSL = 运行命令 / git push / curl 测试
GitHub = 代码仓库
Vercel = 自动部署
```

不要强依赖 Codex 本地 shell。它可能提示：

```text
/bin/bash: No such file or directory
本地 shell 不可用
```

这不一定是 WSL 坏了，而是 Codex 没接到你本机 shell。

### 10.2 标准提交命令

```bash
cd "/mnt/c/Users/MR/Documents/Codex/2026-05-20/chatgpt-gpt-action-step-1-vercel"

git status
git add .
git commit -m "你的提交说明"
git push
```

### 10.3 测试接口是否存在

```bash
curl -i "https://feishu-ai-task-agent.vercel.app/api/create-task"
```

成功信号：

```text
HTTP/2 405
allow: POST
```

这说明接口存在，只是不允许 GET。

### 10.4 测试正式写入

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

---

## 11. 标准排查顺序

遇到问题必须按层排查，不要跳层。

```text
1. GPT Action schema 是否正确
2. server URL 是否是真实 Vercel 地址
3. Vercel endpoint 是否存在
4. Vercel 是否部署最新 commit
5. 环境变量是否配置并重新部署
6. tenant_access_token 是否成功
7. Wiki token 是否能解析 app_token
8. 飞书开放平台权限是否开通
9. 应用版本是否发布
10. 目标多维表格是否添加文档应用且可编辑
11. 字段名是否完全一致
12. 单选字段选项是否存在
13. 日期 / 日期时间格式是否正确
```

---

## 12. 常见报错对照表

| 报错 | 所属层级 | 原因 | 处理 |
|---|---|---|---|
| Cannot have multiple custom actions for same domain | GPT Action | 同域名建了多个 Action | 合并到一个 schema |
| YOUR_VERCEL_PROJECT not under root origin | GPT Action | schema 里还是占位符 | 改成真实 Vercel URL |
| NOT_FOUND | Vercel | 接口没部署上 | 检查最新 commit 和部署状态 |
| Function Runtimes must have a valid version | Vercel build | vercel.json runtime 不合法 | 简化 vercel.json |
| Missing required environment variable | Vercel env | 环境变量缺失或没 redeploy | 补变量并重新部署 |
| wiki:node:read required | 飞书 API 权限 | 缺 Wiki 节点读取权限 | 开权限并发布版本 |
| 91403 forbidden | 目标文档权限 | 目标多维表格未授权应用 | 添加文档应用，可编辑 |
| FieldNameNotFound | 字段映射 | 字段名不一致 | 对齐表头字段 |
| invalid field value | 字段类型 | 单选选项或日期格式不对 | 补选项或改格式 |

---

## 13. 可复用结论

以后所有“外部工具写入飞书多维表格”的项目，都先问这 5 个问题：

```text
1. 这个表是 wiki 链接还是 base 链接？
2. 应用 API 权限是否已开通并发布？
3. 目标多维表格是否添加了文档应用并给可编辑？
4. 字段名是否和代码完全一致？
5. Vercel 环境变量是否配置并重新部署？
```

这 5 个问题比盲目改代码更重要。
