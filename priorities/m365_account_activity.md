# priorities/m365_account_activity.md

```markdown
---
project: M365 账号活跃度核心检查维度分析系统
status: production-ready
tech_stack: [Python3.14, PowerShell7+, Microsoft-Graph-API, Exchange-Online, Azure-OpenAI]
owner: M365-Admin-Team
created: 2026-07-15
last_updated: 2026-07-27
version: v1.3
linked_signals:
  - signals/m365_thresholds.md
  - signals/m365_fields.md
linked_components:
  - components/graph_api_integration.md
  - components/ai_analysis_prompt.md
  - components/service_account_detector.md
linked_decisions:
  - decisions/ai_prompt_calibration.md
  - decisions/threshold_calibration_2026Q3.md
---

# M365 账号活跃度核心检查维度分析系统

> 🎯 **核心价值**：自动化识别闲置/异常/风险 M365 账号，排除「配置存在但无实际使用」的僵尸账号，输出 AI 增强型结构化分析报告

---

## 📦 项目概览

| 维度 | 内容 |
|------|------|
| **项目目标** | 基于真实使用行为量化判定账号活跃度，区分正常使用/闲置/异常风险 |
| **核心能力** | 数据采集 → 智能分析 → 报告生成 → 告警通知 → Web 仪表盘 |
| **技术栈** | Python 3.14 + PowerShell 7+ + Azure OpenAI + Graph API + Exchange Online |
| **数据源** | Microsoft Graph API (v1.0/beta) + Exchange Online PowerShell |
| **输出物** | 日报/周报/月报/季报 + Web 仪表盘 + 异常告警邮件 |
| **文档版本** | v1.3 |
| **最后更新** | 2026-07-27 |

---

## 🎯 核心检查维度（四大类｜按优先级排序）

```yaml
priority_order:
  - core_usage_behavior      # 🔴 核心使用行为（权重 60%）
  - traffic_resource_feature # 🟡 流量与资源特征（权重 20%）
  - config_status            # 🔵 配置与状态（权重 15%）
  - anomaly_risk_flag        # ⚫ 异常/风险标识（权重 5%，但具一票否决权）
```

### 🔴 核心使用行为（权重 60%｜判定主依据）
| 指标 | 阈值 | 数据来源 |
|------|------|----------|
| 最后成功登录时间 | >30 天无登录 = 高闲置风险 | Graph: `/auditLogs/signIns` |
| 邮件发送量 | 连续 14 天=0 | EXO: `Get-MessageTraceV2` |
| 邮件接收量 | 日均<5 封 + 发送=0 = 闲置 | Graph: `/insights/mailboxUsage` |
| 主动交互行为 | 连续 30 天无回复/转发/附件操作 | Graph: `/me/messages` |

### 🟡 流量与资源特征（权重 20%｜补充验证）
| 指标 | 阈值 | 数据来源 |
|------|------|----------|
| 存储容量变化 | 连续 30 天增量=0 | EXO: `Get-MailboxStatistics` |
| 内/外部交互占比 | 30 天无外部交互 + 内部<10 封 | EXO: `Get-MessageTraceV2` |

### 🔵 配置与状态（权重 15%｜辅助判定）
| 指标 | 阈值 | 数据来源 |
|------|------|----------|
| 自动回复开启状态 | >15 天需人工核实 | Graph: `/mailboxSettings` |
| 邮件转发配置 | 外部转发 + 无登录 = 高风险 | EXO: `Get-Mailbox` |
| 关联设备数量 | 连续 60 天=0 | Graph: `/registeredDevices` |

### ⚫ 异常/风险标识（权重 5%｜一票否决权）
| 指标 | 阈值 | 数据来源 |
|------|------|----------|
| 密码最后修改时间 | >90 天 + 无登录 = 安全风险 | Graph: `/users/{id}` |
| 异地/异常 IP 登录 | riskLevel≥medium + 无常用设备 | Graph: `/auditLogs/signIns` |
| 登录失败次数 | 7 天内≥5 次 | Graph: `/auditLogs/signIns` |

---

## 🎯 僵尸账号判定规则（≥3 项满足）

```yaml
zombie_account_rule:
  logic: "OR"
  combinations:
    - name: "核心行为组合"
      conditions:
        - lastLogonTime > 30d
        - send_zero_consecutive_days >= 14 AND receive_daily_avg < 5
        - interaction_inactive_days >= 30
      min_match: 3

    - name: "资源 + 配置组合"
      conditions:
        - storage_no_change_days >= 30
        - device_inactive_days >= 60
        - auto_reply_enabled_days >= 15
      min_match: 2

    - name: "高风险配置组合"
      conditions:
        - forwarding_external_risk = true AND lastLogonTime > 30d
        - password_unchanged_days >= 90 AND lastLogonTime > 30d
      min_match: 1

  exclusion:
    - userType = ServicePrincipal
    - userPrincipalName matches ["svc_", "automation", "bot", "noreply", "system", "deploy_", "ci_"]
```

---

## 📈 活跃度分级标准

| 级别 | 判定条件 | 处置建议 |
|------|----------|----------|
| **✅ 正常活跃** | 核心行为 3 项全达标 + 无高风险配置 | 无需操作 |
| **🟡 低活跃** | 核心行为 1-2 项不达标 + 无高风险配置 | 邮件提醒用户确认 |
| **🟠 高闲置风险** | 核心行为≥3 项不达标 + 符合僵尸规则 1-2 项 | 部门核实 +7 日内反馈 |
| **🔴 异常风险** | 满足任一「一票否决」规则 | 立即告警 +24 小时内处置 |

> 🔁 **冲突处理原则**：当指标冲突时，按 `core_usage_behavior > traffic_resource > config_status` 优先级判定

---

## 🗂️ 文件索引（快速导航）

### 📁 信号规则（signals/）
| 文件 | 内容 | 用途 |
|------|------|------|
| `signals/m365_thresholds.md` | 活跃度判定阈值规则 | 阈值配置+ 校准记录 |
| `signals/m365_fields.md` | 属性字段列表 | Graph API+PowerShell 字段映射 |

### 🧩 可复用组件（components/）
| 文件 | 内容 | 用途 |
|------|------|------|
| `components/graph_api_integration.md` | Graph API 集成规范 | 认证 + 端点 + 速率限制 |
| `components/ai_analysis_prompt.md` | AI 分析报告 Prompt 模板 | 完整 Prompt+ 变量替换 |
| `components/service_account_detector.md` | 服务账号识别组件 | 排除规则 + 识别逻辑 |

### 📝 决策记录（decisions/）
| 文件 | 内容 | 用途 |
|------|------|------|
| `decisions/ai_prompt_calibration.md` | AI Prompt 校准记录 | 问题 + 校准动作 + 效果验证 |
| `decisions/threshold_calibration_2026Q3.md` | 阈值校准记录 | 阈值变更原因 + 测试结果 |

### 📊 报告模板（reports/）
| 文件 | 内容 | 用途 |
|------|------|------|
| `reports/daily_template.md` | 日报模板 | 简要汇总 + 异常告警 |
| `reports/weekly_template.md` | 周报模板 | 完整 6 部分 + 趋势分析 |
| `reports/monthly_template.md` | 月报模板 | 完整报告 + 部门对比 |

---

## 🔧 技术配置摘要

### API 端点选择策略
| 端点 | 推荐版本 | 理由 |
|------|---------|------|
| `/users` | v1.0 | 稳定 + 生产就绪 |
| `/auditLogs/signIns` | beta | 登录详情更完整 |
| `/mailboxSettings` | beta | 高级配置仅 beta 支持 |
| `/registeredDevices` | beta | 设备类型/OS 等扩展属性 |

### PowerShell 命令
```powershell
# Exchange Online 连接
Connect-ExchangeOnline -Credential $Cred -ShowBanner:$false -ExchangeEnvironmentName "O365Default"

# 获取邮箱统计
Get-MailboxStatistics -Identity $userUPN

# 获取邮件跟踪
Get-MessageTraceV2 -StartDate $start -EndDate $end -RecipientAddress $userUPN
```

### 核心阈值配置（config.yaml 片段）
```yaml
activity:
  thresholds:
    login_inactive_days: 30
    send_zero_consecutive_days: 14
    receive_daily_avg_threshold: 5
    interaction_inactive_days: 30
    storage_no_change_days: 30
    auto_reply_enabled_days: 15
    device_inactive_days: 60
    password_unchanged_days: 90
```

---

## 📊 系统架构

```
数据采集层 → 数据处理层 → 智能分析层 → 报告展示层
    ↑            ↑            ↑            ↑
数据源层    业务逻辑层    AI 集成层    告警通知层
    ↑            ↑            ↑            ↑
配置管理层   异常检测层   监控系统层   调度管理层
```

### 核心组件
| 层级 | 组件 | 文件 |
|------|------|------|
| 数据采集 | PowerShell 脚本 | `get-m365-account-activity-v0.1.ps1` |
| 数据采集 | Graph API 客户端 | `graph_api.py` |
| 数据处理 | 数据处理器 | `processor.py` |
| 智能分析 | OpenAI 分析器 | `openai.py` |
| 报告展示 | 报告生成器 | `generator.py` |
| 报告展示 | Web 仪表盘 | `dashboard.py` / `modern_dashboard.py` |
| 监控系统 | 监控管理器 | `monitoring/manager.py` |
| 调度管理 | 任务调度器 | `scheduler/manager.py` |

---

## 🔄 校准记录（最近 3 次）

| 日期 | 校准项 | 原值 → 新值 | 原因 | 效果 |
|------|--------|-------------|------|------|
| 2026-07-20 | `login_inactive_days` | 45 → 30 | 销售团队反馈误判率高 | 准确率 78% → 94% |
| 2026-07-22 | 服务账号排除规则 | +["deploy_", "ci_"] | 新增 CI/CD 账号误判 | 误判率 12% → 0% |
| 2026-07-25 | `forwarding_external_risk` | 新增一票否决 | 安全审计发现泄露事件 | 安全漏报 15% → 2% |

> 📌 **校准流程**：阈值变更 → 小范围 A/B 测试 → 记录决策日志 → 全量生效

---

## ✅ 快速复用 Checklist

### 首次部署
- [ ] Azure AD 应用已注册并授予 4 项 application 权限
- [ ] 环境变量 `AZURE_TENANT_ID`/`GRAPH_APP_ID`/`GRAPH_APP_SECRET` 已配置
- [ ] Exchange Online Management 模块 ≥ 3.7.0 已安装
- [ ] `config.yaml` 中密码字段未加引号，保留特殊字符
- [ ] Azure OpenAI 服务已配置（可选，用于智能分析）

### 日常运维
- [ ] 每日检查 `.ai/` 中 AI 草稿，更新 `priorities/m365_*.md`
- [ ] 每周运行 `weekly_review` 模板，记录校准动作
- [ ] 每月回顾 `signals/` 阈值，根据业务反馈调整
- [ ] 每季度生成季度报告，分析趋势变化

### 阈值校准
- [ ] 阈值变更已记录至 `decisions/` 目录
- [ ] 已同步更新 `components/ai_analysis_prompt.md` 中的判定规则
- [ ] 已完成小范围 A/B 测试验证
- [ ] 已更新 `weekly_review` 模板

---

## 📋 报告分发配置

| 报告类型 | 标题格式 | 发送对象 | 周期 |
|---------|---------|---------|------|
| 日报 | `YYYYMMDD-M365 邮箱账号活跃度分析 - 分析报告（AI 化）` | 小组负责人（主）、总负责人（抄送） | 每日 |
| 周报 | `YYYYW02-M365 邮箱账号活跃度分析 - 分析报告（AI 化）` | 同上 | 每周一 9:00 |
| 月报 | `YYYYM01-M365 邮箱账号活跃度分析 - 分析报告（AI 化）` | 同上 + 部门负责人 | 每月 1 号 |
| 季报 | `YYYYQ02-M365 邮箱账号活跃度分析 - 分析报告（AI 化）` | 同上 + CTO | 每季度首月 5 号 |

---

## 🛡️ 安全与合规

```yaml
security:
  authentication:
    method: OAuth2_Client_Credentials
    token_storage: environment_variables
    permission_principle: least_privilege
  data_handling:
    encryption: at_rest_and_in_transit
    retention_policy: 90_days
    pii_protection: enabled
  audit:
    log_all_api_calls: true
    log_all_data_access: true
    log_all_report_generation: true
  compliance:
    gdpr_compliant: true
    data_minimization: true
    privacy_by_design: true
```

---

## 📈 性能指标（v2.3 实测）

| 指标 | 目标值 | 实测值 |
|------|--------|--------|
| 报告格式一致性 | 100% | 100% |
| 僵尸账号识别准确率 | ≥90% | 94% |
| 服务账号误判率 | 0% | 0% |
| 安全漏报率 | ≤5% | 2% |
| 平均响应时间 | ≤30 秒 | 18 秒 |
| Token 消耗 | ≤8000/报告 | 6500/报告 |

---

## 🔄 后续演进计划

| 日期 | 功能 | 优先级 | 负责人 |
|------|------|--------|--------|
| 2026-08-15 | 多租户支持 | 🟡 中 | M365-Admin-Team |
| 2026-09-01 | 季度/年度报告格式优化 | 🟢 高 | M365-Admin-Team |
| 2026-10-01 | 机器学习模型辅助判定 | 🔴 高 | AI-Engineering-Team |
| 2026-12-01 | 多语言报告支持 | 🟡 中 | M365-Admin-Team |
| 2027-01-01 | 自动化处置（禁用/归档） | 🔴 高 | M365-Admin-Team |

---

## 📞 技术支持

| 角色 | 联系方式 | 职责 |
|------|---------|------|
| 项目负责人 | m365-admin@company.com | 整体协调 + 决策审批 |
| 技术支持 | support@company.com | 故障排查 + 系统维护 |
| 安全合规 | security@company.com | 安全审计 + 合规检查 |
| 文档地址 | https://docs.company.com/m365-account-review | 技术文档 + 用户指南 |
| GitHub 仓库 | https://github.com/company/m365-account-review | 代码 +Issue 跟踪 |

---

> 💡 **Vibe-Memory 提示**：本优先级文件是 M365 账号活跃度分析系统的核心入口，需配合 `signals/m365_thresholds.md`（阈值规则）、`components/graph_api_integration.md`（数据采集）、`components/ai_analysis_prompt.md`（AI 分析）、`decisions/ai_prompt_calibration.md`（校准记录）联动使用。每次阈值校准或 Prompt 更新后，务必同步更新本文件中的相关配置和链接。
```

✅ 文件已按 vibe-memory 规范生成，可直接保存至 `priorities/m365_account_activity.md`

---
