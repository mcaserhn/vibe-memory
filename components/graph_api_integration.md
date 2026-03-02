# components/graph_api_integration.md

```markdown
---
component: graph_api_integration
category: data-collection
tech_stack: [Python3.14, PowerShell7+, Microsoft-Graph-API, Azure-AD]
last_updated: 2026-07-27
status: production-ready
---

# Microsoft Graph API + Exchange Online 集成规范

> 🎯 **核心目标**：稳定、高效、安全地采集 M365 账号活跃度核心指标，支持 v1.0/beta 版本混合调用策略

---

## 🔐 认证方式（Client Credentials Flow）

### Azure AD 应用注册要求
```yaml
azure_ad_app:
  type: ConfidentialClient
  permissions:
    application:
      - User.Read.All
      - Mail.Read
      - AuditLog.Read.All
      - Directory.Read.All
  # 避免使用 delegated 权限，确保后台任务无需用户交互
```

### Python 认证实现
```python
from azure.identity import ClientSecretCredential
from msgraph import GraphServiceClient

credential = ClientSecretCredential(
    tenant_id=os.getenv("AZURE_TENANT_ID"),
    client_id=os.getenv("GRAPH_APP_ID"),
    client_secret=os.getenv("GRAPH_APP_SECRET")
)

graph_client = GraphServiceClient(credential, scopes=["https://graph.microsoft.com/.default"])
```

### PowerShell 认证实现
```powershell
# Graph API 令牌获取
$Body = @{
    client_id     = $env:GRAPH_APP_ID
    client_secret = $env:GRAPH_APP_SECRET
    tenant_id     = $env:AZURE_TENANT_ID
    grant_type    = "client_credentials"
    scope         = "https://graph.microsoft.com/.default"
}
$Response = Invoke-RestMethod -Uri "https://login.microsoftonline.com/$env:AZURE_TENANT_ID/oauth2/v2.0/token" -Method POST -Body $Body
$AccessToken = $Response.access_token

# Exchange Online 连接（保留密码特殊字符）
$SecurePwd = ConvertTo-SecureString $env:EXO_PASSWORD -AsPlainText -Force
$Cred = New-Object PSCredential($env:EXO_USERNAME, $SecurePwd)
Connect-ExchangeOnline -Credential $Cred -ShowBanner:$false -ExchangeEnvironmentName "O365Default"
```

> ⚠️ **安全提醒**：凭证必须通过环境变量注入，禁止硬编码；密码字段在 YAML 中不加引号，保留所有特殊字符

---

## 📦 API 版本选择策略

### 混合调用原则
| 功能场景 | 推荐版本 | 端点示例 | 选择理由 |
|---------|---------|---------|---------|
| 用户基础信息 | `v1.0` | `/users` | 稳定、生产就绪、限流宽松 |
| 许可详情 | `v1.0` | `/users/{id}/licenseDetails` | 许可字段在 v1.0 已完整 |
| 登录审计日志 | `beta` | `/auditLogs/signIns` | 含 riskLevel、deviceDetail 等高级字段 |
| 邮箱设置 | `beta` | `/users/{id}/mailboxSettings` | 自动回复/转发配置仅 beta 完整 |
| 设备注册信息 | `beta` | `/users/{id}/registeredDevices` | beta 返回设备类型/OS 等扩展属性 |
| 存储使用统计 | `beta` | `/insights/mailboxUsage` | v1.0 无此聚合端点 |

### 版本回退机制
```python
def call_graph_api(endpoint: str, api_version: str = "v1.0", fallback: bool = True):
    """带 fallback 的 Graph API 调用"""
    try:
        return _request(f"https://graph.microsoft.com/{api_version}/{endpoint}")
    except ApiVersionNotSupportedError:
        if fallback and api_version == "beta":
            logger.warning(f"beta 端点 {endpoint} 不可用，尝试回退至 v1.0")
            return _request(f"https://graph.microsoft.com/v1.0/{endpoint}")
        raise
```

---

## 🎯 核心端点清单（按活跃度检查维度分类）

### 1️⃣ 核心使用行为
| 端点 | 版本 | 关键字段 | 用途 |
|-----|------|---------|------|
| `/auditLogs/signIns` | beta | `createdDateTime`, `status.success`, `clientAppUsed`, `deviceDetail.deviceType`, `location` | 最后登录时间、设备类型、地理位置 |
| `/users/{id}/mailboxSettings` | beta | `automaticRepliesSetting`, `forwardingSmtpAddress` | 自动回复/转发配置 |
| `Get-MailboxStatistics` (EXO PS) | - | `LastLogonTime`, `ItemCount`, `TotalItemSize` | 邮箱级登录时间、邮件数、存储量 |

### 2️⃣ 流量与资源特征
| 端点 | 版本 | 关键字段 | 用途 |
|-----|------|---------|------|
| `/insights/mailboxUsage` | beta | `usedStorage`, `sendCount`, `receiveCount` | 存储使用、收发邮件统计 |
| `Get-MessageTraceV2` (EXO PS) | - | `SenderAddress`, `RecipientAddress`, `Status` | 邮件交互行为、内外部分类 |
| `/me/messages` | v1.0/beta | `isRead`, `hasAttachments`, `lastModifiedDateTime` | 邮件主动交互行为识别 |

### 3️⃣ 配置与状态
| 端点 | 版本 | 关键字段 | 用途 |
|-----|------|---------|------|
| `/users/{id}/registeredDevices` | beta | `deviceType`, `operatingSystem`, `registeredDateTime` | 关联设备数量与类型 |
| `/users/{id}` | v1.0 | `passwordPolicies`, `lastPasswordChangeDateTime` | 密码策略与修改时间 |
| `Get-Mailbox` (EXO PS) | - | `ForwardingSmtpAddress`, `DeliverToMailboxAndForward` | 转发配置详情 |

### 4️⃣ 异常/风险标识
| 端点 | 版本 | 关键字段 | 用途 |
|-----|------|---------|------|
| `/auditLogs/signIns` | beta | `riskLevel`, `riskDetail`, `ipAddress`, `location.countryOrRegion` | 异常登录/IP/地理位置检测 |
| `/auditLogs/signIns` | beta | `status.failureReason` + `status.errorCode` | 登录失败原因聚合分析 |

---

## ⚡ 速率限制与重试策略

### Graph API 限流规则
```yaml
rate_limit:
  per_app:
    requests_per_second: 10
    burst: 20
  per_user:
    requests_per_second: 3
  headers:
    retry_after: "Retry-After"  # 遵循服务端返回的等待秒数
```

### 指数退避重试实现（Python）
```python
import time
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def create_retry_session():
    session = requests.Session()
    retry = Retry(
        total=3,
        backoff_factor=2,  # 1s → 2s → 4s
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["GET", "POST"]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount("https://", adapter)
    return session
```

### Exchange Online PowerShell 分段查询
```powershell
# MessageTraceV2 最大支持 10 天，按 7 天分段避免截断
$StartDate = Get-Date "2026-01-01"
$EndDate = Get-Date "2026-01-31"
$StepDays = 7

for ($start = $StartDate; $start -lt $EndDate; $start = $start.AddDays($StepDays)) {
    $end = [Math]::Min($start.AddDays($StepDays), $EndDate)
    Get-MessageTraceV2 -StartDate $start -EndDate $end -RecipientAddress $userUPN
}
```

---

## 🗃️ 缓存策略

### 缓存层级设计
```yaml
cache:
  api_response:
    enabled: true
    ttl_seconds: 300  # 5 分钟，平衡实时性与 API 调用成本
    key_pattern: "{endpoint}?{query_params}"
  computed_metrics:
    enabled: true
    ttl_seconds: 3600  # 1 小时，如活跃度评分、僵尸账号判定
  user_list:
    enabled: true
    ttl_seconds: 86400  # 24 小时，用户列表变化频率低
```

### Python 缓存装饰器示例
```python
from functools import lru_cache
import hashlib, json

def cache_api_call(ttl: int = 300):
    def decorator(func):
        cache = {}
        def wrapper(*args, **kwargs):
            key = hashlib.md5(json.dumps({"args": args, "kwargs": kwargs}, sort_keys=True).encode()).hexdigest()
            if key in cache:
                result, timestamp = cache[key]
                if time.time() - timestamp < ttl:
                    return result
            result = func(*args, **kwargs)
            cache[key] = (result, time.time())
            return result
        return wrapper
    return decorator

@cache_api_call(ttl=300)
def get_signins(user_id: str, start_date: str):
    return graph_client.audit_logs.sign_ins.request().filter(f"userId eq '{user_id}' and createdDateTime ge {start_date}").get()
```

---

## 🛡️ 错误处理与降级方案

### 分级异常处理
```python
class GraphAPIError(Exception): pass
class RateLimitError(GraphAPIError): pass
class AuthError(GraphAPIError): pass
class DataNotFoundError(GraphAPIError): pass

def safe_graph_call(endpoint, **kwargs):
    try:
        return call_graph_api(endpoint, **kwargs)
    except RateLimitError:
        logger.warning(f"速率限制，等待后重试: {endpoint}")
        time.sleep(int(response.headers.get("Retry-After", 5)))
        return call_graph_api(endpoint, **kwargs)
    except AuthError:
        logger.error("认证失败，检查凭证或权限")
        raise
    except DataNotFoundError:
        logger.info(f"数据不存在（预期内）: {endpoint}")
        return None  # 业务层处理缺失值
    except Exception as e:
        logger.error(f"未预期错误: {e}", exc_info=True)
        return None  # 降级：返回空，避免中断全流程
```

### Exchange Online 连接失败降级
```powershell
# 优先使用 Exchange Online PowerShell，失败时回退到 Graph API
try {
    $Stats = Get-MailboxStatistics -Identity $userUPN
} catch {
    Write-Warning "EXO 获取失败，尝试 Graph API 备用方案"
    $Stats = Get-GraphMailboxStats -UserId $userUPN  # 自定义备用函数
}
```

---

## 🔌 PowerShell ↔ Python 集成

### 数据交换格式：JSON
```powershell
# PowerShell 输出标准化 JSON
$Result = @{
    userPrincipalName = $user.UserPrincipalName
    lastLogonTime = $stats.LastLogonTime?.ToString("o")
    itemCount = $stats.ItemCount
    totalItemSizeBytes = $stats.TotalItemSize.Value
    forwardingAddress = $mailbox.ForwardingSmtpAddress
} | ConvertTo-Json -Depth 10 -Compress

# Python 通过 subprocess 调用并解析
import subprocess, json
result = subprocess.run(
    ["pwsh", "-File", "get-m365-account-activity-v0.1.ps1", "-UserUPN", upn],
    capture_output=True, text=True, check=True
)
data = json.loads(result.stdout)
```

### 异步批量调用优化
```python
import asyncio
from aiohttp import ClientSession

async def fetch_user_activity(session, user_id):
    async with session.get(f"{GRAPH_BASE}/users/{user_id}") as resp:
        return await resp.json()

async def batch_fetch(user_ids):
    async with ClientSession(headers={"Authorization": f"Bearer {token}"}) as session:
        tasks = [fetch_user_activity(session, uid) for uid in user_ids]
        return await asyncio.gather(*tasks, return_exceptions=True)
```

---

## 📋 权限清单（最小权限原则）

```yaml
required_permissions:
  application:
    - name: User.Read.All
      description: 读取所有用户基础信息
      justification: 获取 userPrincipalName, accountEnabled, lastPasswordChangeDateTime
    - name: Mail.Read
      description: 读取所有邮箱内容（仅元数据）
      justification: 分析邮件交互行为、附件使用情况
    - name: AuditLog.Read.All
      description: 读取审计日志
      justification: 获取 signInLogs, mailbox audit logs
    - name: Directory.Read.All
      description: 读取目录数据
      justification: 获取设备注册信息、组织单元
  # ❌ 避免使用 Mail.ReadWrite, User.ReadWrite.All 等写权限
```

> ✅ **最佳实践**：在 Azure Portal → App registrations → API permissions 中逐项添加，并执行 **Admin consent**

---

## 🧪 配置示例（config/default/config.yaml）

```yaml
graph_api:
  base_url: "https://graph.microsoft.com"
  versions:
    stable: "v1.0"
    preview: "beta"
  auth:
    tenant_id: "${AZURE_TENANT_ID}"
    client_id: "${GRAPH_APP_ID}"
    client_secret: "${GRAPH_APP_SECRET}"
  rate_limit:
    requests_per_second: 10
    retry_after_header: "Retry-After"
    max_retries: 3
  cache:
    enabled: true
    ttl_seconds: 300
    backend: "memory"  # 或 "redis"

exchange_online:
  module_version: "3.7.0"  # ExchangeOnlineManagement
  connection:
    username: "${EXO_USERNAME}"
    password: "${EXO_PASSWORD}"  # 保留所有特殊字符，YAML 中不加引号
    environment: "O365Default"
  query:
    message_trace_days: 7  # 分段查询步长
    batch_size: 100

monitoring:
  log_api_calls: true
  metrics_endpoint: "/metrics"  # Prometheus 兼容
```

---

## 🔄 校准记录（decisions/graph_api_calibration.md 关联）

| 日期 | 问题 | 校准动作 | 效果 |
|-----|------|---------|------|
| 2026-07-20 | beta 端点 `/auditLogs/signIns` 偶发 404 | 添加 fallback 至 v1.0 + 记录缺失字段 | 采集成功率 92% → 99.6% |
| 2026-07-22 | 密码字段 `lastPasswordChangeDateTime` 部分用户为空 | 增加空值兜底：用 `passwordPolicies` + 登录时间推断风险 | 误报率下降 40% |
| 2026-07-25 | Exchange Online 连接因密码含 `"` 失败 | YAML 配置中密码字段不加引号 + Python 侧 `ConvertTo-SecureString -AsPlainText` | 连接稳定性 100% |

> 📌 **校准原则**：优先保证核心指标（登录时间、发送量、交互行为）的准确性，高级字段（如 riskLevel）作为补充验证

---

## 🚀 快速复用 Checklist

- [ ] Azure AD 应用已注册并授予 4 项 application 权限
- [ ] 环境变量 `AZURE_TENANT_ID`/`GRAPH_APP_ID`/`GRAPH_APP_SECRET` 已配置
- [ ] Exchange Online Management 模块 ≥ 3.7.0 已安装
- [ ] `config.yaml` 中密码字段未加引号，保留特殊字符
- [ ] 速率限制重试逻辑已集成（指数退避 + Retry-After 头）
- [ ] 缓存 TTL 按业务场景差异化配置（API 响应 5min / 指标 1h / 用户列表 24h）
- [ ] 关键端点调用已添加 fallback 降级方案

> 💡 **Vibe-Memory 提示**：本组件需配合 `signals/m365_thresholds.md`（判定规则）和 `decisions/ai_prompt_calibration.md`（AI 分析校准）联动使用，实现「采集→分析→决策」闭环。
```
