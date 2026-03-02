# signals/m365_thresholds.md

```markdown
---
signal: m365_account_activity_thresholds
category: business-rules
domain: M365-identity-governance
calibration_cycle: quarterly
last_calibrated: 2026-07-27
owner: M365-Admin-Team
---

# M365 账号活跃度判定阈值规则

> 🎯 **核心原则**：以「真实使用行为」为第一优先级，配置/状态指标仅作补充验证；阈值可校准，但核心行为阈值变更需记录决策日志

---

## 📊 阈值总览（按优先级排序）

```yaml
priority_order:
  - core_usage_behavior      # 🔴 核心使用行为（权重 60%）
  - traffic_resource_feature # 🟡 流量与资源特征（权重 20%）
  - config_status            # 🔵 配置与状态（权重 15%）
  - anomaly_risk_flag        # ⚫ 异常/风险标识（权重 5%，但具一票否决权）
```

---

## 🔴 核心使用行为阈值（权重 60%｜判定主依据）

### 1️⃣ 最后登录时间
| 阈值项 | 默认值 | 校准范围 | 判定逻辑 | 数据来源 |
|--------|--------|----------|----------|----------|
| `login_inactive_days` | `30` | `14~60` | `lastLogonTime < NOW - {value}d` → 高闲置风险 | Graph: `/auditLogs/signIns`<br>EXO: `Get-MailboxStatistics` |
| `login_critical_days` | `90` | `60~180` | `lastLogonTime < NOW - {value}d` → 僵尸账号候选 | 同上 |

> ⚠️ **校准提示**：销售/外勤岗位可放宽至 45 天；财务/研发等敏感岗位建议收紧至 21 天

### 2️⃣ 邮件发送/接收行为
| 阈值项 | 默认值 | 校准范围 | 判定逻辑 | 数据来源 |
|--------|--------|----------|----------|----------|
| `send_zero_consecutive_days` | `14` | `7~30` | 连续 `{value}` 天发送量=0 → 闲置信号 | EXO: `Get-MessageTraceV2` |
| `receive_daily_avg_threshold` | `5` | `3~20` | 日均接收 `< {value}` 封 + 发送=0 → 低活跃 | Graph: `/insights/mailboxUsage` |
| `interaction_inactive_days` | `30` | `14~60` | 连续 `{value}` 天无回复/转发/附件操作 → 僵尸候选 | Graph: `/me/messages` + 操作日志 |

> ✅ **主动交互定义**：`isRead=true` + `hasAttachments=true` + `lastModifiedDateTime` 由用户触发（非系统自动）

### 3️⃣ 服务账号排除规则（避免误判）
```yaml
service_account_patterns:
  name_keywords: ["svc_", "automation", "bot", "noreply", "system"]
  login_behavior:
    clientAppUsed: ["Exchange ActiveSync", "Other", "Managed Identity"]
    interactive: false  # 非交互式登录
  sending_pattern:
    regular_volume: true    # 稳定发送量（如每日 10±2 封）
    recipient_fixed: true   # 收件人固定（如仅发往监控组）
  exclusion_logic: |
    IF (name_matches_pattern OR login_behavior.non_interactive) 
    AND sending_pattern.regular_volume 
    THEN mark_as "service_account" EXCLUDE from zombie detection
```

---

## 🟡 流量与资源特征阈值（权重 20%｜补充验证）

### 1️⃣ 存储容量变化
| 阈值项 | 默认值 | 校准范围 | 判定逻辑 | 数据来源 |
|--------|--------|----------|----------|----------|
| `storage_no_change_days` | `30` | `14~60` | 连续 `{value}` 天 `UsedStorage` 增量=0 → 无操作信号 | Graph: `/insights/mailboxUsage`<br>EXO: `Get-MailboxStatistics` |

> 🔍 **校准建议**：归档策略开启的邮箱需单独标记，避免误判

### 2️⃣ 内/外部交互占比
| 阈值项 | 默认值 | 校准范围 | 判定逻辑 | 数据来源 |
|--------|--------|----------|----------|----------|
| `external_interaction_min` | `1` | `1~10` | 连续 30 天外部交互 `< {value}` 封 → 无效账号候选 | EXO: `Get-MessageTraceV2` + 域名分析 |
| `internal_interaction_max` | `10` | `5~50` | 外部=0 + 内部 `< {value}` 封 → 高闲置风险 | 同上 |

> 🌐 **域名白名单**：`@company.com`, `@partner-approved.com` 视为内部；其余为外部

---

## 🔵 配置与状态阈值（权重 15%｜辅助判定）

### 1️⃣ 自动回复配置
| 阈值项 | 默认值 | 校准范围 | 判定逻辑 | 数据来源 |
|--------|--------|----------|----------|----------|
| `auto_reply_enabled_days` | `15` | `7~30` | `automaticRepliesSetting.status=enabled` 持续 `>{value}d` → 需人工核实 | Graph: `/users/{id}/mailboxSettings` |

> 📋 **核实动作**：确认是否为休假/离职状态，更新 HR 系统同步状态

### 2️⃣ 邮件转发配置
| 阈值项 | 默认值 | 校准范围 | 判定逻辑 | 数据来源 |
|--------|--------|----------|----------|----------|
| `forwarding_external_risk` | `true` | 固定 | `ForwardingSmtpAddress` 含外部域名 + 原账号 `lastLogonTime > 30d` → 🔴 高风险 | EXO: `Get-Mailbox` |

> ⚠️ **一票否决**：外部转发 + 无登录 = 立即告警，无论其他指标

### 3️⃣ 设备关联
| 阈值项 | 默认值 | 校准范围 | 判定逻辑 | 数据来源 |
|--------|--------|----------|----------|----------|
| `device_inactive_days` | `60` | `30~90` | 连续 `{value}` 天 `registeredDevices.count=0` → 无实际使用 | Graph: `/users/{id}/registeredDevices` |

---

## ⚫ 异常/风险标识阈值（权重 5%｜一票否决权）

### 1️⃣ 密码安全
| 阈值项 | 默认值 | 校准范围 | 判定逻辑 | 数据来源 |
|--------|--------|----------|----------|----------|
| `password_unchanged_days` | `90` | `60~180` | `lastPasswordChangeDateTime < NOW - {value}d` + `lastLogonTime > 30d` → 安全风险 | Graph: `/users/{id}` |

### 2️⃣ 登录失败与异常
| 阈值项 | 默认值 | 校准范围 | 判定逻辑 | 数据来源 |
|--------|--------|----------|----------|----------|
| `login_failure_threshold` | `5` | `3~10` | 7 天内失败次数 `>= {value}` → 触发安全复核 | Graph: `/auditLogs/signIns` |
| `abnormal_location_risk` | `medium` | `low/medium/high` | `riskLevel >= {value}` + 无常用设备 → 异常风险 | Graph: `/auditLogs/signIns.riskLevel` |

> 🚨 **一票否决规则**：满足任一即标记「异常风险」，优先于活跃度分级

---

## 🎯 僵尸账号判定规则（AND/OR 逻辑）

```yaml
zombie_account_rule:
  logic: "OR"  # 满足以下任一组合即判定
  combinations:
    - name: "核心行为组合"
      conditions:
        - lastLogonTime > 30d
        - send_zero_consecutive_days >= 14 AND receive_daily_avg < 5
        - interaction_inactive_days >= 30
      min_match: 3  # 3 项中满足≥3 项

    - name: "资源+配置组合"
      conditions:
        - storage_no_change_days >= 30
        - device_inactive_days >= 60
        - auto_reply_enabled_days >= 15
      min_match: 2  # 3 项中满足≥2 项

    - name: "高风险配置组合"
      conditions:
        - forwarding_external_risk = true AND lastLogonTime > 30d
        - password_unchanged_days >= 90 AND lastLogonTime > 30d
      min_match: 1  # 满足任一即判定
```

> ✅ **排除项**：`userType=ServicePrincipal` OR 匹配 `service_account_patterns` → 自动排除

---

## 📈 活跃度分级标准（交叉验证，非单一指标）

| 级别 | 判定条件 | 处置建议 |
|------|----------|----------|
| **✅ 正常活跃** | 核心行为 3 项全达标 + 无高风险配置 | 无需操作 |
| **🟡 低活跃** | 核心行为 1-2 项不达标 + 无高风险配置 | 邮件提醒用户确认使用状态 |
| **🟠 高闲置风险** | 核心行为≥3 项不达标 + 符合僵尸规则 1-2 项 | 部门核实 + 7 日内反馈 |
| **🔴 异常风险** | 满足任一「一票否决」规则 | 立即告警 + 24 小时内处置 |

> 🔁 **冲突处理原则**：当指标冲突时，按 `core_usage_behavior > traffic_resource > config_status` 优先级判定

---

## 🔄 校准记录（关联 decisions/ 目录）

| 日期 | 阈值项 | 原值 → 新值 | 校准原因 | 关联决策文件 |
|------|--------|-------------|----------|-------------|
| 2026-07-20 | `login_inactive_days` | 45 → 30 | 销售团队反馈 45 天误判率高，收紧至 30 天平衡准确率 | `decisions/threshold_calibration_2026Q3.md` |
| 2026-07-22 | `service_account_patterns.name_keywords` | +["deploy_", "ci_"] | 新增 CI/CD 服务账号误判案例 | 同上 |
| 2026-07-25 | `forwarding_external_risk` | 新增一票否决 | 安全审计发现 3 起外部转发泄露事件 | `decisions/security_policy_update_202607.md` |

> 📌 **校准流程**：阈值变更 → 小范围 A/B 测试 → 记录决策日志 → 全量生效

---

## 🧪 配置示例（YAML 片段，供 config.yaml 引用）

```yaml
# config/default/config.yaml
activity:
  thresholds:
    # 核心使用行为
    login_inactive_days: 30
    login_critical_days: 90
    send_zero_consecutive_days: 14
    receive_daily_avg_threshold: 5
    interaction_inactive_days: 30
    
    # 流量与资源
    storage_no_change_days: 30
    external_interaction_min: 1
    internal_interaction_max: 10
    
    # 配置与状态
    auto_reply_enabled_days: 15
    device_inactive_days: 60
    
    # 异常/风险
    password_unchanged_days: 90
    login_failure_threshold: 5
    abnormal_location_risk: "medium"
  
  zombie_account:
    min_core_conditions: 3
    exclude_service_patterns: ["svc_", "automation", "bot", "noreply", "system", "deploy_", "ci_"]
  
  priority_weights:
    core_usage_behavior: 0.6
    traffic_resource_feature: 0.2
    config_status: 0.15
    anomaly_risk_flag: 0.05  # 但具一票否决权
```

---

## ✅ 快速复用 Checklist

- [ ] 阈值已通过 `config.yaml` 注入，支持环境变量覆盖
- [ ] 服务账号排除规则已同步至 `components/service_account_detector.py`
- [ ] 「一票否决」规则在分析引擎中优先执行
- [ ] 校准记录已关联至 `decisions/` 目录，支持追溯
- [ ] 阈值变更时自动触发 `weekly_review` 模板更新

> 💡 **Vibe-Memory 提示**：本信号文件需与 `components/graph_api_integration.md`（数据采集）和 `decisions/ai_prompt_calibration.md`（AI 分析校准）联动使用。每次阈值校准后，务必更新 AI Prompt 中的判定规则描述，保持「采集→分析→决策」一致性。
```
