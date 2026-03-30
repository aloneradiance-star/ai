# 小雅「雅衣曼体」半自动化内容生产与发布工作流需求分析（小红书 / 抖音 / 视频号）

## 1. 目标重述与范围边界

### 1.1 业务目标拆解
基于你给出的方案，项目的核心不是“全自动发内容”，而是“**审核驱动的人机协同生产线**”：

- AI 负责高并发产出（选题、脚本、素材建议、剪辑版本、发布时间建议）
- 人工只在关键闸门做决策（通过 / 驳回 / 修改 / 重做 / 紧急处理）
- 最终达成“**日均人工 55 分钟内完成关键审核**”的运营模式

### 1.2 范围（MVP）建议
第一阶段建议只覆盖你点名的平台与主链路：

1. 平台：小红书、抖音、视频号
2. 流程：选题 → 脚本 → 素材 → 剪辑 → 发布 → 监控回流
3. 角色：运营审核员、发布负责人、管理员
4. 目标节奏：每日 3–6 条内容，24 小时内闭环

> 说明：评论自动互动、追投建议可先做半自动（建议+待确认），避免前期合规与误触发风险。

---

## 2. 需求结构化分析

## 2.1 功能模块分析与落地建议

### 模块A：审核中枢机器人（核心控制台）
**关键价值**：把所有“待决策事项”收敛到一个界面，降低切换成本。

**建议最小能力集**：
- 任务看板（按优先级/截止时间/平台/流程节点过滤）
- 一键决策（通过、驳回、重做、指派）
- 批量审核（同一选题下镜头批量通过）
- 审核日志（谁在何时做了什么决定）
- 紧急命令（暂停工作流、紧急下线、回滚排期）

**数据对象**：
- `ReviewTask`：审核任务
- `ReviewDecision`：审核决策
- `EscalationEvent`：紧急事件

---

### 模块B：选题与脚本生成工作流（22:00 定时触发）
**输入**：
- 热点数据（平台榜单、关键词、话题热度）
- 历史表现（近 180 天：完播率、互动率、转粉率）
- 品牌约束（禁用词、风格词典、品类优先级）

**输出（每条选题）**：
1. 选题标题与核心观点
2. 目标受众与痛点映射
3. 完整脚本（口播 / 字幕 / 互动钩子）
4. 分镜草案（镜头时长、画面建议、转场建议）
5. 风险提示（平台规则、敏感词）

**审核界面关键点**：
- 5 个方案并排比较
- 支持“选 1 主推 + 2 备选”
- 支持“局部改写而非整条重做”

---

### 模块C：智能素材生产工作流
**并行策略建议**：
- 以“选题”为并行单元（最多并发 3）
- 以“镜头”为子任务（每镜头产出 3 个版本）

**素材包建议结构**：
- 视频片段（按分镜编号）
- 静态图 / 封面候选
- 配图文案 / 字幕建议
- BGM 建议（风格 + BPM + 情绪）
- 可复用模板标记（便于下次检索）

**审核优化点**：
- 支持镜头级批量通过
- 支持“指定第2版本为默认”
- 支持“仅重生成第3镜头”

---

### 模块D：自动化剪辑工作流
**平台适配策略**：
- 抖音版：节奏快、开头 3 秒强钩子、时长更短
- 视频号版：信息更完整、节奏稍稳、可加观点层次
- 小红书版：封面与标题策略更关键，强调“可收藏价值”

**输出规范**：
- 每平台至少 1 条主版本 + N 条 AB 测试版本
- 多版封面、多版标题/文案
- 自动打包发布素材（视频+文案+标签建议）

**审核能力**：
- 预览 + 预计完播率/互动率评分
- 版本对比（A/B）
- 一键退回到素材层重剪

---

### 模块E：智能发布与监控
**发布前**：
- 排期确认（时间、平台、账号）
- 文案最终审校（敏感词、违禁词、广告法风险）
- 模板化参数保存（下次复用）

**发布后**：
- 监控指标：播放、完播、点赞、评论、转发、涨粉
- 异常预警：限流、审核失败、评论舆情突增
- 追投建议：是否加热、是否二次分发、是否改标题复投

---

## 2.2 非功能需求评估

### 性能目标可行性
- 选题 < 5 分钟：可行（需缓存热点数据 + 并行调用模型）
- 素材并发 3 选题：可行（需任务队列 + GPU/渲染配额控制）
- 单视频剪辑 < 15 分钟：可行（模板化剪辑、预设参数）
- 可用性 > 99%：需要高可用调度器、重试机制、告警值班

### 数据治理建议
- 表现数据保留 180 天：用于策略学习足够
- 审核决策永久存储：用于偏好学习、审计合规
- 素材库存储建议冷热分层：近 30 天热存，历史归档冷存

### 安全与权限
- RBAC：审核员/发布员/管理员三级
- 所有动作写审计日志（不可篡改）
- 涉及发布与删除动作必须二次确认

---

## 3. 关键流程（审核驱动状态机）

建议把系统实现成“状态机 + 人工闸门”，避免流程失控：

1. `TopicGenerated`（选题已生成）
2. `TopicApproved` / `TopicRejected`
3. `AssetsGenerated`（素材已生成）
4. `AssetsApproved` / `AssetsNeedRegeneration`
5. `EditRendered`（剪辑完成）
6. `EditApproved` / `EditBackToAssets`
7. `ReadyToPublish`（待发布）
8. `Published`
9. `Monitoring`
10. `ClosedLoopLearned`（反馈回流完成）

每次状态变化都绑定：
- 操作人
- 时间戳
- 变更原因
- 上下文快照（当时模型版本、提示词版本、素材版本）

---

## 4. 技术架构建议（可实施版本）

### 4.1 分层架构（与你方案对齐）
- 前端层：Web 审核台 + 移动端 H5
- 应用层：审核中枢 API
- 工作流层：任务编排（定时、回调、重试、补偿）
- AI 服务层：选题生成、脚本生成、素材建议、评分模型
- 数据层：内容库、素材库、审核决策库、指标仓
- 集成层：抖音/小红书/视频号 API、通知渠道（钉钉/企微/邮件）

### 4.2 必要技术机制
- 工作流引擎：支持 DAG、重试、人工节点、超时回滚
- 队列系统：削峰填谷（生成任务、渲染任务、发布任务分离）
- 特征回流：发布后指标回写，用于下一轮选题排序
- 策略中心：平台规则、敏感词、账号白名单统一管理

---

## 5. 关键难点与应对

1. **AI 质量波动**
   - 应对：多候选 + 自动评分 + 人审闸门
2. **平台规则变化快**
   - 应对：规则配置化，热更新，不写死在代码
3. **审核疲劳导致误判**
   - 应对：任务优先级、批量操作、高风险项高亮
4. **多平台内容适配成本高**
   - 应对：一稿多版模板系统（平台差异参数化）

---

## 6. 里程碑实施建议（12 周样板）

### 第 1-3 周（基础设施）
- 审核中枢最小可用版
- 工作流编排与任务状态机
- 通知与审计日志

### 第 4-6 周（生产链路）
- 选题/脚本生成上线
- 素材并行生产上线
- 基础审核界面完善

### 第 7-9 周（剪辑与发布）
- 自动剪辑模板接入
- 三平台发布适配
- 发布前合规校验

### 第 10-12 周（优化与试运行）
- 数据看板与异常告警
- 偏好学习（基于审核决策）
- 小规模灰度试运行 + 调参

---

## 7. 成功指标落地（可度量）

### 运营效率
- 人工审核总时长（日均）
- 从选题到发布的 Lead Time
- 每日产能（3-6 条达成率）

### 内容质量
- 完播率、互动率、转粉率
- AI 初稿一次通过率
- 违规率（目标 <1%）

### 系统质量
- 流程成功率
- 平均重试次数
- 任务积压时长

---

## 8. 对你当前方案的补充建议（优先级最高）

1. 增加“**审核 SLA**”定义：不同任务必须在多久内处理（例如素材审核 1 小时内）。
2. 增加“**回滚机制**”：发布后异常可一键下线并回退到上一个安全版本。
3. 增加“**品牌语料库**”：固定品牌语气、禁用表达、常见结构模板，保证一致性。
4. 增加“**实验框架**”：封面/标题/前3秒钩子的系统化 AB 测试。
5. 增加“**跨平台策略映射**”：同一主题在小红书/抖音/视频号分别输出推荐结构。

---

## 9. 下一步可执行清单

如果你确认这个分析方向，下一步可以直接进入：

1. 输出《MVP 功能清单（按周排期）》
2. 输出《审核中枢页面原型（Web+移动）》
3. 输出《工作流状态机与数据库表结构草案》
4. 输出《三平台发布字段映射清单（小红书/抖音/视频号）》
5. 输出《上线前测试用例（功能/性能/合规）》

这样你就能从“方案说明”快速进入“可开发执行”的阶段。

---

## 10. MVP 功能清单（按周排期，可直接进入研发）

### 10.1 P0（必须上线）

| 周次 | 能力 | 交付项 | 验收标准 |
|---|---|---|---|
| W1 | 审核中枢基础 | 登录、任务列表、任务详情、通过/驳回 | 可完成单任务审核闭环 |
| W2 | 状态机与调度 | Topic→Assets→Edit→Publish 全链路状态流转 | 任务状态可追踪、可回放 |
| W3 | 通知与告警 | 钉钉/企微通知，紧急告警 | 关键节点 1 分钟内通知到人 |
| W4 | 选题生成 | 每日22:00产出5个选题 | 生成时延 < 5 分钟 |
| W5 | 脚本与分镜 | 每选题输出脚本+分镜 | 支持“局部重写” |
| W6 | 素材并行生成 | 并发3选题，镜头3版本 | 失败任务可自动重试 |
| W7 | 审核批处理 | 镜头级批量通过/重生成 | 批量操作成功率 > 99% |
| W8 | 自动剪辑 | 抖音/视频号/小红书版本输出 | 单条渲染 < 15 分钟 |
| W9 | 发布排期 | 排期确认、人工最终发布 | 支持发布前最后审校 |
| W10 | 发布回传 | 播放、完播、互动、涨粉回流 | 回流延迟 < 10 分钟 |
| W11 | 数据看板 | 每日产能、审核耗时、通过率 | 指标口径一致 |
| W12 | 灰度试运行 | 1个账号先跑，问题复盘 | 连续7天稳定运行 |

### 10.2 P1（增强项）
- 评论自动回复建议（人工确认后发送）
- 追投策略推荐（含预算区间）
- 偏好学习（按审核员偏好排序候选）
- AB实验自动归因（封面/标题/钩子分层）

---

## 11. 数据库表结构草案（核心最小集）

### 11.1 工作流与审核

1. `content_project`
- `id`（PK）
- `topic_id`（选题ID）
- `status`（枚举：TopicGenerated...Published）
- `priority`（P0/P1/P2）
- `owner_user_id`
- `created_at`, `updated_at`

2. `review_task`
- `id`（PK）
- `project_id`（FK）
- `task_type`（topic/assets/edit/publish）
- `task_payload`（JSON）
- `sla_deadline`
- `status`（pending/approved/rejected/rework）
- `assignee_user_id`
- `created_at`, `updated_at`

3. `review_decision`
- `id`（PK）
- `review_task_id`（FK）
- `decision`（approve/reject/rework/escalate）
- `decision_note`
- `operator_user_id`
- `snapshot`（JSON：模型版本/提示词/素材版本）
- `created_at`

### 11.2 内容生产

4. `topic_candidate`
- `id`（PK）
- `project_id`（FK）
- `platform`（xiaohongshu/douyin/wechat_channels）
- `title`
- `audience`
- `pain_point`
- `score_predict`
- `risk_flags`（JSON）

5. `script_version`
- `id`（PK）
- `topic_candidate_id`（FK）
- `version_no`
- `script_text`
- `storyboard`（JSON）
- `is_selected`

6. `asset_item`
- `id`（PK）
- `project_id`（FK）
- `scene_no`
- `variant_no`
- `asset_type`（video/image/audio/subtitle）
- `storage_url`
- `meta`（JSON：分辨率/时长/BPM等）
- `status`

7. `edit_version`
- `id`（PK）
- `project_id`（FK）
- `platform`
- `version_no`
- `video_url`
- `cover_url`
- `caption_text`
- `predict_metrics`（JSON）
- `is_selected`

### 11.3 发布与监控

8. `publish_job`
- `id`（PK）
- `project_id`（FK）
- `platform`
- `account_id`
- `schedule_time`
- `publish_status`（scheduled/success/failed/canceled）
- `platform_post_id`
- `error_message`

9. `performance_daily`
- `id`（PK）
- `publish_job_id`（FK）
- `date`
- `impressions`
- `plays`
- `completion_rate`
- `likes`
- `comments`
- `shares`
- `followers_gain`

10. `audit_log`
- `id`（PK）
- `actor_user_id`
- `action`
- `resource_type`
- `resource_id`
- `before_data`（JSON）
- `after_data`（JSON）
- `created_at`

---

## 12. 三平台发布字段映射清单（首版）

| 统一字段 | 小红书 | 抖音 | 视频号 | 备注 |
|---|---|---|---|---|
| `title` | 标题 | 标题/文案首句 | 标题 | 小红书标题权重最高 |
| `caption` | 正文文案 | 视频文案 | 描述文案 | 统一存储，平台侧再裁剪 |
| `cover_image` | 封面图 | 封面图 | 封面图 | 建议统一 9:16 主图 |
| `video_file` | 视频文件 | 视频文件 | 视频文件 | 保留原始母版+平台转码版 |
| `hashtags` | 话题标签 | 话题标签 | 话题标签 | 平台数量限制不同 |
| `publish_time` | 发布时间 | 发布时间 | 发布时间 | 排期由 `publish_job` 维护 |
| `location` | 可选定位 | 可选定位 | 可选定位 | 涉及门店可启用 |
| `comment_policy` | 评论设置 | 评论设置 | 评论设置 | 默认开启，异常时关闭 |
| `music_id` | 背景音乐 | 背景音乐 | 背景音乐 | 版权合规检查必过 |
| `product_link` | 商品/店铺链接 | 商品锚点 | 小商店/链接 | 电商场景必填 |

---

## 13. 上线前测试用例（功能 / 性能 / 合规）

### 13.1 功能测试
- 用例 F-01：22:00 定时触发后，5 个选题在 5 分钟内生成。
- 用例 F-02：审核员可对选题执行“通过/驳回/局部重写”。
- 用例 F-03：素材镜头支持批量通过、单镜头重生成。
- 用例 F-04：三平台版本均可生成并进入待发布。
- 用例 F-05：发布失败后自动重试并告警。

### 13.2 性能测试
- 用例 P-01：并发 3 个选题生成素材，队列无明显积压。
- 用例 P-02：单视频渲染时长稳定 < 15 分钟。
- 用例 P-03：任务看板在 2000 条任务下查询 < 2 秒。

### 13.3 合规测试
- 用例 C-01：敏感词/违禁词命中后禁止进入发布。
- 用例 C-02：未授权角色无法执行“最终发布”。
- 用例 C-03：发布、下架、删除操作均有审计日志。
- 用例 C-04：素材版权风险命中后触发阻断与替换建议。

---

## 14. 建议的下一步（你确认后我可继续输出）

1. 《审核中枢 Web 页面低保真原型（含移动端交互）》
2. 《状态机 API 草案（接口+状态码+错误码）》
3. 《任务队列与重试补偿策略（含死信队列）》
4. 《数据看板 SQL 指标口径与计算公式》
5. 《小红书/抖音/视频号 运营策略模板（按品类）》

---

## 15. 审核中枢 Web 页面低保真原型（含移动端交互）

### 15.1 信息架构（IA）
- 一级导航：仪表盘 / 待审核 / 生产中 / 待发布 / 监控告警 / 资产库 / 系统设置
- 二级导航（待审核）：选题审核、素材审核、剪辑审核、发布审核
- 全局区域：
  - 顶部：全局搜索、账号切换、通知中心、紧急按钮
  - 左侧：导航树
  - 中部：任务列表与预览
  - 右侧：操作面板（通过/驳回/重做/指派）

### 15.2 Web 端页面原型（文本线框）

#### 页面 A：待审核任务总览
```
┌──────────────────────────────────────────────────────────────┐
│ Logo | 项目: 雅衣曼体 | 搜索框 | 通知(3) | 紧急控制 | 用户头像 │
├───────────────┬──────────────────────────────────────────────┤
│ 左侧导航       │ 中部：任务队列                               │
│ - 仪表盘       │ [筛选] 平台/优先级/SLA/状态                  │
│ - 待审核       │ ┌────┬──────┬──────┬────────┬────────────┐   │
│   - 选题审核   │ │ID  │平台  │类型  │SLA剩余 │状态        │   │
│   - 素材审核   │ ├────┼──────┼──────┼────────┼────────────┤   │
│   - 剪辑审核   │ │T01 │抖音  │选题  │00:35   │待处理      │   │
│   - 发布审核   │ │T02 │小红书│素材  │00:12   │待处理(高危)│   │
│ - 待发布       │ └────┴──────┴──────┴────────┴────────────┘   │
│ - 监控告警     │ 批量操作：[通过] [驳回] [重生成] [指派]        │
├───────────────┼──────────────────────────────────────────────┤
│ 右侧操作栏     │ 选中任务详情：预览/变更历史/决策建议/风险提示  │
│ [通过]         │                                              │
│ [驳回+原因]    │                                              │
│ [重做]         │                                              │
│ [保存模板]     │                                              │
└───────────────┴──────────────────────────────────────────────┘
```

#### 页面 B：选题审核对比页（5选题并排）
- 上方：筛选（目标人群/话题类型/风险等级）
- 中间：5列卡片并排，字段包括：标题、痛点、脚本摘要、预测分、风险标识
- 底部操作：`设为主推`、`设为备选`、`局部改写`、`整条驳回`
- 右上角快捷键提示：
  - `A` 通过
  - `R` 驳回
  - `E` 编辑
  - `B` 设备选

#### 页面 C：素材审核页（镜头级批处理）
- 时间轴视图：Scene01~SceneN
- 每个镜头 3 个版本（V1/V2/V3）支持并排预览
- 批量操作：
  - 全选当前选题镜头
  - 指定默认版本（如全设 V2）
  - 单镜头重生成（仅回流该节点）

#### 页面 D：发布审核页
- 左侧：平台版本列表（抖音/小红书/视频号）
- 中间：视频预览 + 文案预览 + 标签建议
- 右侧：排期时间、账号、评论策略、商品链接、发布前风控检查结果
- 发布按钮策略：
  - 仅 `发布负责人` 可见“最终发布”按钮
  - 按下后二次确认（不可逆提示）

### 15.3 移动端交互原型（H5）

#### 核心原则
- 大按钮（44px+）
- 单手可达（关键操作在底部）
- 关键动作二次确认

#### 关键流程
1. 推送到达：`素材审核即将超时（剩余15分钟）`
2. 打开任务卡片：展示风险提示 + 快速预览
3. 底部滑动决策：
   - 右滑通过
   - 左滑驳回
   - 上滑查看详情
4. 离线场景：
   - 本地缓存“已处理决策”
   - 恢复网络后自动同步

### 15.4 交互与可用性规范
- 状态色：
  - 待审核（灰）
  - 已通过（绿）
  - 需修改（橙）
  - 高风险（红）
- SLA 高亮：
  - 剩余 < 30 分钟变橙
  - 已超时变红并置顶
- 可追溯性：每个操作可展开看到“谁在何时改了什么”

---

## 16. 状态机 API 草案（接口 / 状态码 / 错误码）

### 16.1 统一约定
- Base URL：`/api/v1`
- 鉴权：`Authorization: Bearer <token>`
- 幂等：写接口支持 `Idempotency-Key`
- 响应格式：

```json
{
  "code": 0,
  "message": "ok",
  "data": {}
}
```

### 16.2 核心接口

#### 1) 创建项目与触发选题
- `POST /projects`
- 请求体：
```json
{
  "brand": "雅衣曼体",
  "platforms": ["xiaohongshu", "douyin", "wechat_channels"],
  "trigger_type": "scheduled",
  "trigger_time": "22:00"
}
```
- 成功返回：`project_id`, `initial_status=TopicGenerated`

#### 2) 获取待审核任务列表
- `GET /review/tasks?status=pending&type=topic&platform=douyin&page=1&page_size=20`
- 返回：任务基本信息 + SLA + 风险等级

#### 3) 提交审核决策
- `POST /review/tasks/{task_id}/decide`
- 请求体：
```json
{
  "decision": "approve",
  "note": "选题2主推，选题4备选",
  "patch": {
    "primary_topic_id": "topic_2",
    "backup_topic_ids": ["topic_4"]
  }
}
```
- 业务规则：
  - `approve`：推进到下一节点
  - `reject`：回到前一节点并记录原因
  - `rework`：仅重跑当前节点
  - `escalate`：升级到管理员队列

#### 4) 触发素材重生成（镜头级）
- `POST /projects/{project_id}/assets/regenerate`
- 请求体：
```json
{
  "scene_nos": [3, 5],
  "reason": "镜头表现力不足",
  "variants": 3
}
```

#### 5) 获取剪辑版本与预测指标
- `GET /projects/{project_id}/edits?platform=douyin`
- 返回：`video_url`, `cover_url`, `predict_metrics`, `version_no`

#### 6) 提交发布排期
- `POST /publish/jobs`
- 请求体：
```json
{
  "project_id": "p_123",
  "platform": "xiaohongshu",
  "account_id": "acc_001",
  "schedule_time": "2026-03-12T19:30:00+08:00"
}
```

#### 7) 最终发布（人工闸门）
- `POST /publish/jobs/{job_id}/confirm`
- 约束：仅发布负责人角色可调用

#### 8) 紧急控制
- `POST /ops/emergency/pause`
- `POST /ops/emergency/resume`
- `POST /ops/emergency/takedown`

### 16.3 状态迁移规则（服务端校验）
- 合法迁移：
  - `TopicGenerated -> TopicApproved | TopicRejected`
  - `TopicApproved -> AssetsGenerated`
  - `AssetsGenerated -> AssetsApproved | AssetsNeedRegeneration`
  - `AssetsApproved -> EditRendered`
  - `EditRendered -> EditApproved | EditBackToAssets`
  - `EditApproved -> ReadyToPublish`
  - `ReadyToPublish -> Published`
  - `Published -> Monitoring -> ClosedLoopLearned`
- 非法迁移：返回 `409 Conflict`

### 16.4 错误码草案

| 错误码 | HTTP | 含义 | 建议处理 |
|---|---:|---|---|
| `1001` | 400 | 参数缺失或格式错误 | 前端校验并重试 |
| `1002` | 401 | 未登录或 token 失效 | 重新登录 |
| `1003` | 403 | 无权限执行该操作 | 提示联系管理员 |
| `1004` | 404 | 任务或项目不存在 | 刷新列表 |
| `1005` | 409 | 状态迁移冲突 | 拉取最新状态后重试 |
| `1006` | 422 | 风控校验未通过 | 修改文案/素材后重提 |
| `1007` | 429 | 调用频率超限 | 指数退避重试 |
| `1008` | 500 | 系统内部错误 | 记录 trace_id，稍后重试 |
| `1009` | 503 | 下游平台不可用 | 进入重试队列并告警 |

### 16.5 审计日志事件（接口级）
- 每次调用写入：
  - `trace_id`
  - `actor_user_id`
  - `api_path`
  - `request_snapshot`
  - `response_code`
  - `resource_before` / `resource_after`


---

## 17. 任务队列与重试补偿策略（含死信队列）

### 17.1 队列分层设计
建议按业务阶段拆分队列，避免“一个队列堵死全链路”：

1. `q.topic.generate`：选题与脚本生成
2. `q.assets.generate`：素材生成（GPU/渲染密集）
3. `q.edit.render`：自动剪辑与导出
4. `q.publish.submit`：平台发布提交
5. `q.metrics.pull`：发布后数据回流
6. `q.notify.dispatch`：通知分发

### 17.2 消息结构（统一 envelope）
```json
{
  "message_id": "msg_xxx",
  "trace_id": "tr_xxx",
  "project_id": "p_123",
  "task_type": "assets.generate",
  "payload": {},
  "retry_count": 0,
  "max_retry": 5,
  "next_retry_at": "2026-03-12T10:00:00+08:00",
  "created_at": "2026-03-12T09:59:00+08:00"
}
```

### 17.3 重试策略（分错误类型）
- 可重试错误：网络抖动、下游超时、平台 5xx、限流 429
- 不可重试错误：参数非法、权限不足、内容风控拒绝

建议指数退避 + 抖动：
- 第1次：30秒
- 第2次：2分钟
- 第3次：5分钟
- 第4次：15分钟
- 第5次：30分钟（最后一次）

超出 `max_retry` 后进入 `DLQ`（死信队列）。

### 17.4 补偿策略（Saga 思路）

| 场景 | 失败点 | 补偿动作 |
|---|---|---|
| 发布中断 | 平台返回失败 | 回滚为 `ReadyToPublish`，通知人工重试 |
| 剪辑失败 | 渲染超时 | 回退到 `AssetsApproved`，重建渲染任务 |
| 素材缺失 | 单镜头生成失败 | 仅回滚失败镜头，保持其余镜头结果 |
| 数据回流失败 | 平台指标接口不可用 | 标记待回补，定时补拉 |

### 17.5 死信队列（DLQ）处理机制
- DLQ 队列：`q.dlq.workflow`
- 触发告警：
  - 同一项目 10 分钟内 DLQ > 3
  - 同一平台 30 分钟内 DLQ > 20
- 人工处理动作：
  - `retry_now`（立即重投）
  - `discard`（丢弃并记录原因）
  - `manual_takeover`（转人工流程）

### 17.6 幂等与去重
- 生产侧：`message_id` 全局唯一
- 消费侧：以 `idempotency_key = task_type + project_id + scene_no + version_no` 去重
- 发布侧：同一 `publish_job_id` 多次提交只允许一次成功

---

## 18. 数据看板 SQL 指标口径与计算公式

> 说明：以下以 `performance_daily`、`publish_job`、`review_task`、`review_decision` 为基础表，SQL 为示意（可按 MySQL/PostgreSQL 微调）。

### 18.1 核心指标定义
1. `daily_output_count`：当日成功发布内容数
2. `avg_review_time_min`：审核任务平均处理时长（分钟）
3. `review_pass_rate`：审核通过率
4. `avg_completion_rate`：平均完播率
5. `engagement_rate`：互动率 = (点赞+评论+分享)/播放
6. `followers_gain_total`：当日涨粉总量
7. `publish_success_rate`：发布成功率
8. `sla_on_time_rate`：SLA按时完成率

### 18.2 SQL 示例

#### A. 当日成功发布数
```sql
SELECT COUNT(*) AS daily_output_count
FROM publish_job
WHERE publish_status = 'success'
  AND DATE(schedule_time) = CURRENT_DATE;
```

#### B. 发布成功率
```sql
SELECT
  SUM(CASE WHEN publish_status = 'success' THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS publish_success_rate
FROM publish_job
WHERE DATE(schedule_time) = CURRENT_DATE;
```

#### C. 审核通过率
```sql
SELECT
  SUM(CASE WHEN decision = 'approve' THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS review_pass_rate
FROM review_decision
WHERE DATE(created_at) = CURRENT_DATE;
```

#### D. 平均审核耗时（分钟）
```sql
SELECT AVG(TIMESTAMPDIFF(MINUTE, rt.created_at, rd.created_at)) AS avg_review_time_min
FROM review_task rt
JOIN review_decision rd ON rd.review_task_id = rt.id
WHERE DATE(rd.created_at) = CURRENT_DATE;
```

#### E. 平均完播率与互动率
```sql
SELECT
  AVG(completion_rate) AS avg_completion_rate,
  SUM(likes + comments + shares) * 1.0 / NULLIF(SUM(plays), 0) AS engagement_rate
FROM performance_daily
WHERE date = CURRENT_DATE;
```

#### F. SLA 按时率
```sql
SELECT
  SUM(CASE WHEN rd.created_at <= rt.sla_deadline THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS sla_on_time_rate
FROM review_task rt
JOIN review_decision rd ON rd.review_task_id = rt.id
WHERE DATE(rd.created_at) = CURRENT_DATE;
```

### 18.3 看板分层建议
- 管理层：产能、成功率、违规率、成本
- 运营层：选题通过率、平台表现、最佳发布时间
- 执行层：待处理任务、超时任务、DLQ任务、异常告警

---

## 19. 小红书 / 抖音 / 视频号运营策略模板（按品类）

### 19.1 通用模板字段
- `content_goal`：涨粉 / 转化 / 品牌认知
- `audience_segment`：目标人群
- `hook_3s`：前3秒钩子
- `value_structure`：内容结构（问题-方案-证明-行动）
- `cta`：引导动作（评论/私信/收藏/下单）
- `risk_notes`：平台与合规风险提示

### 19.2 品类模板A：塑形知识科普

#### 小红书
- 标题：问题式 + 结果承诺（避免绝对化）
- 封面：前后对比 + 关键词贴纸
- 正文：
  1) 常见误区
  2) 正确动作要点
  3) 7天执行清单
- CTA：收藏 + 打卡

#### 抖音
- 前3秒：反常识结论 + 动作演示
- 节奏：15-35秒，强字幕节拍
- 结尾：评论区领取动作表

#### 视频号
- 时长：40-90秒
- 结构：原理解释 + 动作示范 + 注意事项
- CTA：关注系列内容

### 19.3 品类模板B：用户见证与案例

#### 小红书
- 标题：`真实用户第X天变化` 类型
- 内容：背景 -> 过程 -> 结果 -> 注意事项
- 风险：避免“保证效果”表达

#### 抖音
- 开头：结果先行（前后变化）
- 中段：关键动作片段
- 结尾：引导评论关键词获取方案

#### 视频号
- 强调可信度：时间线、方法论、边界说明
- 支持挂合集形成连续观看

### 19.4 品类模板C：课程/服务转化

#### 小红书
- 重点：可收藏清单 + 场景化痛点
- 建议：图文+短视频双形态联动

#### 抖音
- 重点：单一强卖点 + 限时动作引导
- 建议：A/B 测试不同开场承诺句

#### 视频号
- 重点：直播/社群承接
- 建议：视频结尾加入下一步行动路径

### 19.5 发布节奏建议（首版）
- 每日 3-6 条：
  - 1 条科普（拉新）
  - 1-2 条案例（信任）
  - 1 条转化（成交）
  - 余量做热点快速响应
- 周维度复盘：
  - 淘汰后 30% 素材模板
  - 放大前 20% 表现主题

