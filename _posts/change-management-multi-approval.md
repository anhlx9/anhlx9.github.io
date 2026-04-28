# Lab: Change Management với Multi-Department Approval

## 1. Tổng quan

Bài lab này mở rộng hệ thống ITIL Workflow hiện tại bằng cách thêm **Change Management** với khả năng multi-approval theo phòng ban.

### Điểm khác biệt: Incident vs Change

| Khía cạnh | Incident Management | Change Management |
|-----------|-------------------|-------------------|
| **Mục đích** | Khôi phục dịch vụ nhanh nhất | Thay đổi có kiểm soát, giảm risk |
| **Trigger** | Sự cố xảy ra (reactive) | Lên kế hoạch trước (proactive) |
| **Approval** | Không cần (gán round-robin) | **Bắt buộc** theo level risk |
| **Timeline** | Càng nhanh càng tốt | Theo schedule window |
| **Stakeholder** | IT Operations | **Multi-department** |

### Kịch bản thực tế

```
CR-001: Standard Change (Low Risk)
├─ Cập nhật Zabbix Agent 7.4.0 → 7.4.1
├─ Impact: Chỉ monitoring, không ảnh hưởng service
└─ Approval: 1 người (IT Manager)

CR-002: Major Change (High Risk)  
├─ Nâng cấp AD DC: Windows Server 2022 → 2025
├─ Impact: Authentication, DNS, GPO, LDAP apps
└─ Approval: 2 người (IT Infra + Security) - parallel
    ├─ phamvand (IT Infrastructure Lead)
    └─ hoangvane (Security & Compliance Lead)
    Logic: CẢ 2 phải approve thì mới tiếp tục
```

---

## 2. Kiến trúc Change Management Workflow

### 2.1. ITIL Change Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    Change Lifecycle                             │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐
│ 1. Submit CR │ ← Requester tạo Change Request
└──────┬───────┘
       │
┌──────▼────────────┐
│ 2. Risk Assessment│ ← Auto hoặc Change Manager đánh giá
└──────┬────────────┘
       │
       ├─ Low Risk ───────► Standard Change (1 approver)
       │
       ├─ Medium Risk ────► Normal Change (2+ approvers)
       │
       └─ High Risk ──────► CAB Meeting (3+ approvers + board)
       
┌──────▼───────────────────────────────────────────────────┐
│ 3. Multi-Approval Stage                                  │
│                                                           │
│  ┌─────────────────┐      ┌──────────────────┐           │
│  │ IT Infra Lead   │      │  Security Lead   │           │
│  │  (phamvand)     │◄────►│   (hoangvane)    │           │
│  └────────┬────────┘      └────────┬─────────┘           │
│           │                        │                     │
│           └────────┬───────────────┘                     │
│                    │ Cả 2 approve                        │
└────────────────────┼─────────────────────────────────────┘
                     │
       ┌─────────────▼──────────────┐
       │ 4. Schedule Implementation │
       └─────────────┬──────────────┘
                     │
       ┌─────────────▼────────────┐
       │ 5. Execute Change        │
       └─────────────┬────────────┘
                     │
       ┌─────────────▼────────────┐
       │ 6. Post-Implementation   │
       │    Review (PIR)          │
       └──────────────────────────┘
```

### 2.2. Approval Matrix

| Change Type | Risk Level | Approvers | Logic | SLA |
|-------------|------------|-----------|-------|-----|
| **Standard** | Low | IT Manager (1 người) | Single | 4h |
| **Normal** | Medium | Dept Leads (2+ người) | **Parallel (AND)** | 24h |
| **Major** | High | CAB Board (3+ người) | **Sequential** | 72h |
| **Emergency** | Critical | CIO/CTO | Single (post-approval) | Immediate |

---

## 3. Setup ServiceDesk Plus - Change Management

### ⚠️ Quan trọng: Approval KHÔNG hardcode!

**Lỗi phổ biến khi mới học:**
```
❌ SAI: Hardcode approver email vào template
   → Template: approver = "phamvand@anhlx.lab"
   → Vấn đề: phamvand nghỉ phép / resign → phải sửa template

✅ ĐÚNG: Dùng Role-based approval rules
   → Rule: IF risk=High THEN assign role="IT Manager"  
   → SDP auto-pick người có role đó từ AD
   → Linh hoạt, không cần sửa code
```

### Workflow thực tế trong SDP GUI:

```
User tạo CR trong GUI:
─────────────────────────

Step 1: Click "New Change"
Step 2: Fill form:
        - Subject: "Nâng cấp AD DC..."
        - Category: [Infrastructure] ← chọn từ dropdown
        - Risk: [High] ← chọn từ dropdown
        - Impact: [High]
        - Scheduled time: ...
        
Step 3: Chọn Template (nếu có):
        - Template: "Major Change - Infrastructure"
        
Step 4: Click "Submit"

─────────────────────────────────────────────────
SDP Backend tự động xử lý (KHÔNG CẦN user chọn approver):
─────────────────────────────────────────────────

1. Check Approval Rules:
   IF category = "Infrastructure" AND risk = "High"
   THEN apply rule "Infrastructure-HighRisk-Approval"

2. Rule config:
   Approver Group 1: Role = "IT Infrastructure Lead"
     → Query AD: Ai có role này?
     → Result: phamvand@anhlx.lab
     → Auto-assign
   
   Approver Group 2: Role = "Security Lead"  
     → Query AD: Ai có role này?
     → Result: hoangvane@anhlx.lab
     → Auto-assign
   
   Logic: ALL must approve (parallel)

3. Send notification email tới cả 2

4. CR Status = "Pending Approval (0/2)"
```

### 3.1. Tạo Approval Rules (KHÔNG hardcode approver!)

**❌ SAI:** Hardcode approver vào template/script  
**✅ ĐÚNG:** Dùng Approval Rules để auto-assign dựa trên điều kiện

**Bước 1: Setup Approval Rules**

Vào **Admin → Change Management → Approval Rules**

```yaml
Rule 1: Standard Change Approval
─────────────────────────────────
Trigger Conditions:
  - Change Type = "Standard"
  - Risk = "Low"
  
Auto-assign Approver:
  Method: Role-based
  Approver Role: "IT Manager"
  How to pick: "Currently logged-in manager" hoặc "Manager of requester"
  
  KHÔNG HARDCODE: phamvand@anhlx.lab
  MÀ DÙNG: Role = "IT Manager" (ai có role này thì auto-assign)
```

```yaml
Rule 2: Infrastructure Change - Multi-Approval
───────────────────────────────────────────────
Trigger Conditions:
  - Category = "Infrastructure" 
  - Risk = "High" OR Impact = "High"
  
Auto-assign Approvers (Parallel):
  Stage 1 - Approver Group 1:
    - Department: "IT Infrastructure"
    - Role: "Department Lead"
    - Pick: All members in role (hoặc Round-robin)
  
  Stage 1 - Approver Group 2:
    - Department: "Security & Compliance"  
    - Role: "Security Lead"
    - Pick: All members in role
  
  Approval Logic: ALL groups must approve (parallel)
```

**Lợi ích:**
- ✅ Không cần sửa code khi đổi người
- ✅ Scale được (thêm/bớt approver chỉ cần update AD group)
- ✅ Dynamic: người đang oncall, manager của requester, v.v.

### 3.3. GUI Workflow Demo

**Màn hình tạo CR trong SDP (không có field "Approver"):**

```
┌─────────────────────────────────────────────────────────────┐
│  ServiceDesk Plus - New Change Request                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Subject: [Nâng cấp AD DC: Windows Server 2022 → 2025]      │
│                                                              │
│  Template: [Major Change - Infrastructure ▼]                │
│                                                              │
│  Change Type: [Major ▼]                                     │
│  Category:    [Infrastructure ▼]  ← Trigger approval rule   │
│  Risk:        [High ▼]            ← Trigger approval rule   │
│  Impact:      [High ▼]                                      │
│  Priority:    [High ▼]                                      │
│                                                              │
│  Requester: nguyenvana@anhlx.lab (auto-fill from login)     │
│                                                              │
│  Scheduled Start: [2026-05-01 00:00] 📅                     │
│  Scheduled End:   [2026-05-01 04:00] 📅                     │
│                                                              │
│  Description:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Nâng cấp Domain Controller...                         │ │
│  │ ...                                                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ⚠️ KHÔNG CÓ field "Approver" để user điền!                 │
│     SDP sẽ tự động assign theo rule!                        │
│                                                              │
│              [Cancel]    [Save as Draft]    [Submit] ✅      │
└─────────────────────────────────────────────────────────────┘
```

**Sau khi click Submit, SDP backend xử lý:**

```
┌──────────────────────────────────────────────────────────────┐
│  Auto-assignment Process (Backend)                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Match Approval Rule:                                     │
│     ✓ Category = Infrastructure                              │
│     ✓ Risk = High                                            │
│     → Apply rule: "Infra-HighRisk-MultiApproval"             │
│                                                               │
│  2. Query AD for approvers:                                  │
│     Group 1: IT Infrastructure                               │
│       LDAP query: (&(objectClass=user)                       │
│                     (memberOf=CN=IT-Managers,...))           │
│       Result: phamvand@anhlx.lab ✓                           │
│                                                               │
│     Group 2: Security                                        │
│       LDAP query: (&(objectClass=user)                       │
│                     (memberOf=CN=Security-Leads,...))        │
│       Result: hoangvane@anhlx.lab ✓                          │
│                                                               │
│  3. Assign approvers:                                        │
│     CR-002.approvers = [                                     │
│       {email: "phamvand@anhlx.lab", level: 1, group: 1},     │
│       {email: "hoangvane@anhlx.lab", level: 1, group: 2}     │
│     ]                                                         │
│     Logic: ALL (parallel)                                    │
│                                                               │
│  4. Send notifications:                                      │
│     ✉️  → phamvand@anhlx.lab                                 │
│     ✉️  → hoangvane@anhlx.lab                                │
│                                                               │
│  5. Update CR status:                                        │
│     Status: "Pending Approval (0/2)"                         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**Approver nhận email và click link:**

```
┌─────────────────────────────────────────────────────────────┐
│  CR-002: Pending Your Approval                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Subject: Nâng cấp AD DC: Windows Server 2022 → 2025        │
│  Type: Major Change                                         │
│  Risk: ⚠️  High                                              │
│  Impact: Critical                                           │
│                                                              │
│  Requester: Nguyen Van A (nguyenvana@anhlx.lab)             │
│  Created: 2026-04-24 09:00 AM                               │
│                                                              │
│  Approval Status:                                           │
│  ├─ IT Infrastructure Lead: ⏳ Pending (You)                │
│  └─ Security Lead: ⏳ Pending (hoangvane)                   │
│                                                              │
│  Logic: Both approvers must approve (parallel)              │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ [Description, Implementation Plan, Rollback Plan...]   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Comments: [Optional feedback...]                           │
│                                                              │
│              [Reject] ❌           [Approve] ✅              │
└─────────────────────────────────────────────────────────────┘
```

### 3.4. AD Group → SDP Role Mapping (Setup một lần)

Vào **Admin → AD Settings → Group Mapping**

```yaml
AD Group Mapping:
─────────────────

AD Group: CN=IT-Managers,OU=IT-Department,DC=anhlx,DC=lab
  → SDP Role: "IT Manager"
  → Members: phamvand, levanc (auto-sync từ AD)

AD Group: CN=Security-Leads,OU=Security,DC=anhlx,DC=lab  
  → SDP Role: "Security Lead"
  → Members: hoangvane (auto-sync từ AD)

AD Group: CN=Change-Advisory-Board,OU=ITIL-Lab,DC=anhlx,DC=lab
  → SDP Role: "CAB Member"
  → Members: (board members)
```

**Lúc này:**
- User tạo CR → chọn template
- SDP check approval rules
- Auto-assign approver dựa trên AD group membership
- KHÔNG CẦN hardcode email!

### 3.2. Configure Approval Workflow

**File:** `sdp-change-approval-workflow.json`

```json
{
  "workflow_name": "Multi-Department Change Approval",
  "stages": [
    {
      "stage_id": 1,
      "name": "Risk Assessment",
      "owner": "Change Manager",
      "auto_transition": true,
      "transition_rule": {
        "risk_low": "stage_2_single",
        "risk_medium_high": "stage_2_multi"
      }
    },
    {
      "stage_id": "2_single",
      "name": "Single Approval (Standard)",
      "type": "approval",
      "approvers": [
        {
          "email": "phamvand@anhlx.lab",
          "role": "IT Manager"
        }
      ],
      "logic": "ANY",
      "sla_hours": 4
    },
    {
      "stage_id": "2_multi",
      "name": "Multi-Approval (Major)",
      "type": "approval",
      "approvers": [
        {
          "email": "phamvand@anhlx.lab",
          "role": "IT Infrastructure Lead",
          "department": "IT Operations"
        },
        {
          "email": "hoangvane@anhlx.lab", 
          "role": "Security Lead",
          "department": "InfoSec"
        }
      ],
      "logic": "ALL",
      "parallel": true,
      "sla_hours": 24,
      "escalation": {
        "if_not_approved_in_hours": 12,
        "escalate_to": "cio@anhlx.lab"
      }
    },
    {
      "stage_id": 3,
      "name": "Implementation Scheduled",
      "auto_transition": false
    },
    {
      "stage_id": 4,
      "name": "Implementation In Progress",
      "notifications": [
        "telegram_bot",
        "email_stakeholders"
      ]
    },
    {
      "stage_id": 5,
      "name": "Post-Implementation Review",
      "owner": "Change Manager",
      "require_closure_notes": true
    }
  ]
}
```

---

## 4. Demo CR-001: Standard Change (1 Approver)

### Scenario: Cập nhật Zabbix Agent

```yaml
Change Request: CR-001
Title: "Cập nhật Zabbix Agent 7.4.0 → 7.4.1 trên DC server"
Type: Standard Change
Category: Software - Monitoring
Risk: Low
Impact: Low (không ảnh hưởng service, chỉ cải thiện monitoring)

Requester: nguyenvana@anhlx.lab
Affected CI: dc.anhlx.lab (10.10.200.12)
Planned Start: 2026-04-25 02:00 AM
Planned End: 2026-04-25 02:30 AM
Duration: 30 minutes

Approval Required:
  ├─ Level 1: phamvand@anhlx.lab (IT Manager)
  └─ Logic: Single approval

Implementation Plan:
1. Backup current Zabbix Agent config
2. Download Zabbix Agent 7.4.1 MSI
3. Uninstall old version
4. Install new version
5. Verify agent connectivity
6. Monitor for 15 minutes

Rollback Plan:
- Reinstall Zabbix Agent 7.4.0 from backup
- Restore config file
- Estimated rollback time: 10 minutes

Success Criteria:
- Agent version = 7.4.1
- Zabbix Server nhận được data từ agent
- No alert trong 30 phút sau upgrade
```

### PowerShell Script: Auto-create CR-001 (Dùng Template)

```powershell
# File: create-standard-change.ps1
# Tạo Standard Change Request - SDP tự động assign approver theo rule

param(
    [string]$SDPBaseURL = "http://10.10.200.11:8400",
    [string]$APIKey = "YOUR_SDP_API_KEY"
)

$headers = @{
    "Content-Type" = "application/json"
    "TECHNICIAN_KEY" = $APIKey
}

$changeRequest = @{
    "change" = @{
        "subject" = "Cập nhật Zabbix Agent 7.4.0 → 7.4.1 trên dc.anhlx.lab"
        "description" = @"
**Mô tả:**
Nâng cấp Zabbix Agent lên version mới nhất để fix security vulnerabilities và cải thiện performance.

**Affected Systems:**
- dc.anhlx.lab (10.10.200.12) - Windows Server 2022

**Change Window:**
- Start: 2026-04-25 02:00 AM
- End: 2026-04-25 02:30 AM
- Timezone: GMT+7

**Downtime:** Không có (agent chạy local)
"@
        # ✅ CHỈ CẦN chọn template - SDP tự assign approver
        "template" = @{
            "name" = "Standard Change - Software Update"
        }
        
        # Template này đã config sẵn:
        # - change_type = "Standard"
        # - risk = "Low"  
        # - impact = "Low"
        # → Trigger approval rule → auto-assign "IT Manager" role
        # → KHÔNG CẦN hardcode approver email!
        
        "change_type" = @{
            "name" = "Standard"
        }
        "risk" = @{
            "name" = "Low"
        }
        "impact" = @{
            "name" = "Low"  
        }
        "priority" = @{
            "name" = "Medium"
        }
        "category" = @{
            "name" = "Software"
        }
        "requester" = @{
            "email_id" = "nguyenvana@anhlx.lab"
        }
        "scheduled_start_time" = (Get-Date "2026-04-25 02:00:00").ToUniversalTime().ToString("o")
        "scheduled_end_time" = (Get-Date "2026-04-25 02:30:00").ToUniversalTime().ToString("o")
    }
} | ConvertTo-Json -Depth 10

try {
    $response = Invoke-RestMethod -Uri "$SDPBaseURL/api/v3/changes" `
                                   -Method POST `
                                   -Headers $headers `
                                   -Body $changeRequest
    
    Write-Host "✅ Change Request created successfully!" -ForegroundColor Green
    Write-Host "CR ID: $($response.change.id)" -ForegroundColor Cyan
    Write-Host "Subject: $($response.change.subject)" -ForegroundColor Yellow
    Write-Host "Status: $($response.change.status.name)" -ForegroundColor Magenta
    
    # ✅ Approver được SDP auto-assign theo rule
    Write-Host "`n📋 Auto-assigned Approvers:" -ForegroundColor Blue
    foreach ($approver in $response.change.approvers) {
        Write-Host "  - $($approver.email_id) [$($approver.role)]" -ForegroundColor Cyan
    }
    
    # Gửi notification qua Telegram
    Send-TelegramNotification -Message @"
🔄 **New Change Request Created**

**CR-ID:** $($response.change.id)
**Type:** Standard Change
**Subject:** $($response.change.subject)

**Auto-assigned Approver:** $($response.change.approvers[0].name)
**Status:** ⏳ Pending Approval

**Scheduled:** 
📅 2026-04-25 02:00 - 02:30 AM

[View in SDP](http://10.10.200.11:8400/WorkOrder.do?woMode=viewWO&woID=$($response.change.id))
"@
    
} catch {
    Write-Host "❌ Error creating change request: $($_.Exception.Message)" -ForegroundColor Red
}

function Send-TelegramNotification {
    param([string]$Message)
    
    $telegramToken = "YOUR_TELEGRAM_BOT_TOKEN"
    $chatId = "YOUR_CHAT_ID"
    
    $body = @{
        chat_id = $chatId
        text = $Message
        parse_mode = "Markdown"
    } | ConvertTo-Json
    
    Invoke-RestMethod -Uri "https://api.telegram.org/bot$telegramToken/sendMessage" `
                      -Method POST `
                      -ContentType "application/json" `
                      -Body $body
}
```

### Approval Process (Single Approver)

```
Timeline CR-001:

09:00 AM - nguyenvana submit CR-001
           └─► SDP auto-assign to phamvand
           └─► Email: "You have 1 pending approval"
           └─► Telegram: "CR-001 chờ approval từ phamvand"

09:15 AM - phamvand login SDP
           └─► Review change details
           └─► Check implementation plan
           └─► Check rollback plan

09:20 AM - phamvand click "Approve"
           └─► Status: Approved → Scheduled
           └─► Auto-notify requester
           └─► Telegram: "✅ CR-001 đã được approve"

02:00 AM (next day) - Implementation window starts
           └─► Zabbix: Tạo planned maintenance window
           └─► Disable alerting cho dc.anhlx.lab
           └─► Execute change

02:30 AM - Implementation completed
           └─► PIR: Success
           └─► CR Status: Closed
```

---

## 5. Demo CR-002: Major Change (2 Approvers - Parallel)

### Scenario: Nâng cấp Active Directory Domain Controller

```yaml
Change Request: CR-002
Title: "Nâng cấp AD DC: Windows Server 2022 → 2025"
Type: Major Change
Category: Infrastructure - Active Directory
Risk: High
Impact: High (ảnh hưởng toàn bộ authentication, DNS, GPO)

Requester: phamvand@anhlx.lab
Affected CIs:
  - dc.anhlx.lab (10.10.200.12)
  - anhlx.lab domain
  - Tất cả AD-integrated services (Zabbix, SDP, Grafana)

Planned Start: 2026-05-01 00:00 AM (Maintenance Window)
Planned End: 2026-05-01 04:00 AM
Duration: 4 hours

Approval Required (PARALLEL - cả 2 phải approve):
  ├─ Level 1a: phamvand@anhlx.lab (IT Infrastructure Lead)
  │             ├─ Kiểm tra: Server capacity, backup strategy
  │             └─ Focus: Technical feasibility
  │
  └─ Level 1b: hoangvane@anhlx.lab (Security & Compliance Lead)
                ├─ Kiểm tra: Security policies, GPO compatibility
                └─ Focus: Security & compliance impact

Logic: ALL_MUST_APPROVE (parallel)
- Nếu 1 trong 2 reject → CR bị reject
- Chỉ khi CẢ 2 approve → CR proceed to implementation

Dependencies:
- AD replication health check
- Backup DC (optional) hoặc snapshot
- Application compatibility test (LDAP apps)

Implementation Plan:
1. Pre-check:
   - Verify AD replication
   - Backup System State
   - Snapshot VM (ESXi)
   
2. Upgrade:
   - Download Windows Server 2025 ISO
   - In-place upgrade DC
   - Monitor Event Logs
   
3. Post-upgrade:
   - Verify FSMO roles
   - Test authentication (Zabbix, SDP, Grafana)
   - Check GPO application
   - Monitor for 2 hours

Rollback Plan:
- Option 1: Restore from snapshot (15 minutes)
- Option 2: Demote DC, rebuild (2 hours)
- Option 3: Keep old backup DC promoted

Success Criteria:
- AD services running normal
- DNS resolution working
- LDAP auth working (Zabbix, SDP, Grafana)
- GPO applied correctly
- No replication errors
```

### PowerShell Script: Auto-create CR-002

```powershell
# File: create-major-change.ps1
# Tạo Major Change Request với multi-approval

param(
    [string]$SDPBaseURL = "http://10.10.200.11:8400",
    [string]$APIKey = "YOUR_SDP_API_KEY"
)

$headers = @{
    "Content-Type" = "application/json"
    "TECHNICIAN_KEY" = $APIKey
}

$changeRequest = @{
    "change" = @{
        "subject" = "Nâng cấp AD Domain Controller: Windows Server 2022 → 2025"
        "description" = @"
## Mục đích
Nâng cấp Active Directory Domain Controller lên Windows Server 2025 để:
- Hỗ trợ tính năng mới (Windows Hello for Business, Kerberos improvements)
- Extended security support lifecycle
- Performance improvements

## Impact Analysis
### Hệ thống bị ảnh hưởng:
- ✅ dc.anhlx.lab (Primary DC)
- ✅ Zabbix 7.4 (LDAP authentication)
- ✅ ServiceDesk Plus 14 (AD integration)
- ✅ Grafana (LDAP auth)
- ✅ Tất cả domain-joined computers (3 stations)

### Risk Assessment:
- **Risk Level:** HIGH
- **Impact:** CRITICAL (toàn bộ authentication)
- **Downtime:** 4 hours (planned maintenance window)

## Pre-requisites
✅ Full backup AD (System State)
✅ VM snapshot on ESXi
✅ Tested in lab environment
✅ Communication sent to all users
✅ Fallback DC available (optional)

## Approval Requirements
Cần approval từ 2 bên (parallel):
1. IT Infrastructure Lead - phamvand
2. Security & Compliance Lead - hoangvane

## Change Window
📅 **2026-05-01 00:00 - 04:00 AM (GMT+7)**
- Off-peak hours
- Minimal user impact
"@
        "change_type" = @{
            "name" = "Major"
        }
        "risk" = @{
            "name" = "High"
        }
        "impact" = @{
            "name" = "High"
        }
        "priority" = @{
            "name" = "High"
        }
        "category" = @{
            "name" = "Infrastructure"
        }
        "subcategory" = @{
            "name" = "Active Directory"
        }
        "requester" = @{
            "email_id" = "phamvand@anhlx.lab"
        }
        "template" = @{
            "name" = "Major Change - Infrastructure"
        }
        
        # Multi-approval configuration
        "approval_status" = "requested"
        "approval_settings" = @{
            "approval_mode" = "parallel"  # Cả 2 approve cùng lúc
            "approval_logic" = "all"      # ALL must approve
        }
        
        "approvers" = @(
            @{
                "email_id" = "phamvand@anhlx.lab"
                "level" = 1
                "role" = "IT Infrastructure Lead"
            },
            @{
                "email_id" = "hoangvane@anhlx.lab"
                "level" = 1
                "role" = "Security & Compliance Lead"
            }
        )
        
        "scheduled_start_time" = (Get-Date "2026-05-01 00:00:00").ToUniversalTime().ToString("o")
        "scheduled_end_time" = (Get-Date "2026-05-01 04:00:00").ToUniversalTime().ToString("o")
        
        # Custom fields
        "udf_fields" = @{
            "udf_char1" = "dc.anhlx.lab"          # Affected Server
            "udf_char2" = "anhlx.lab"             # Domain Name
            "udf_char3" = "Windows Server 2025"   # Target Version
            "udf_long1" = "ADBackup_20260430.zip" # Backup Reference
        }
    }
} | ConvertTo-Json -Depth 10

try {
    $response = Invoke-RestMethod -Uri "$SDPBaseURL/api/v3/changes" `
                                   -Method POST `
                                   -Headers $headers `
                                   -Body $changeRequest
    
    Write-Host "✅ Major Change Request created successfully!" -ForegroundColor Green
    Write-Host "CR ID: $($response.change.id)" -ForegroundColor Cyan
    Write-Host "Subject: $($response.change.subject)" -ForegroundColor Yellow
    Write-Host "Risk: $($response.change.risk.name)" -ForegroundColor Red
    Write-Host "Status: $($response.change.status.name)" -ForegroundColor Magenta
    
    Write-Host "`n📋 Approvers (Parallel - ALL must approve):" -ForegroundColor Blue
    foreach ($approver in $response.change.approvers) {
        Write-Host "  - $($approver.email_id) [$($approver.role)]" -ForegroundColor Cyan
    }
    
    # Gửi notification qua Telegram
    Send-TelegramNotification -Message @"
🔴 **MAJOR CHANGE REQUEST Created**

**CR-ID:** CR-$($response.change.id)
**Type:** Major Change (2 approvers - parallel)
**Risk:** ⚠️ HIGH
**Impact:** CRITICAL

**Subject:** 
Nâng cấp AD DC: Windows Server 2022 → 2025

**Approvers (CẢ 2 phải approve):**
├─ ⏳ phamvand (IT Infrastructure)
└─ ⏳ hoangvane (Security & Compliance)

**Scheduled Window:**
📅 2026-05-01 00:00 - 04:00 AM
⏱️ Duration: 4 hours

**Affected Systems:**
- dc.anhlx.lab
- All AD-integrated services

[View in SDP](http://10.10.200.11:8400/WorkOrder.do?woMode=viewWO&woID=$($response.change.id))

⚠️ **Action Required:** Approval needed from both leads
"@
    
} catch {
    Write-Host "❌ Error creating change request: $($_.Exception.Message)" -ForegroundColor Red
}

function Send-TelegramNotification {
    param([string]$Message)
    
    $telegramToken = "YOUR_TELEGRAM_BOT_TOKEN"
    $chatId = "YOUR_CHAT_ID"
    
    $body = @{
        chat_id = $chatId
        text = $Message
        parse_mode = "Markdown"
    } | ConvertTo-Json
    
    Invoke-RestMethod -Uri "https://api.telegram.org/bot$telegramToken/sendMessage" `
                      -Method POST `
                      -ContentType "application/json" `
                      -Body $body
}
```

### Approval Process (Multi-Approver - Parallel)

```
Timeline CR-002:

Monday 09:00 AM - phamvand submit CR-002
                  └─► SDP create ticket
                  └─► Auto-assign to 2 approvers (parallel)
                  └─► Email to both: "You have 1 pending high-risk approval"
                  └─► Telegram: "🔴 CR-002 cần approval từ 2 leads"

Monday 10:30 AM - phamvand (Approver 1) login SDP
                  ├─► Review technical details
                  ├─► Check backup strategy
                  ├─► Verify rollback plan
                  └─► ✅ Click "Approve"
                      └─► Status: 1/2 approved
                      └─► Telegram: "✅ phamvand approved CR-002 (1/2)"
                      └─► hoangvane nhận reminder email

Monday 02:00 PM - hoangvane (Approver 2) login SDP
                  ├─► Review security implications
                  ├─► Check GPO compatibility
                  ├─► Verify compliance requirements
                  └─► ❓ Add comment: "Cần thêm firewall rule documentation"

Monday 02:30 PM - phamvand update CR-002
                  └─► Add attachment: "firewall-rules-2025.pdf"
                  └─► Notify hoangvane

Monday 03:00 PM - hoangvane review lại
                  └─► ✅ Click "Approve"
                      └─► Status: 2/2 approved ✅
                      └─► CR Status: Approved → Scheduled for Implementation
                      └─► Telegram: "✅✅ CR-002 FULLY APPROVED (2/2)"
                      └─► Auto-create Implementation Task

Thursday 23:55 PM (Maintenance Window - 5 phút trước)
                  └─► Telegram: "⚠️ CR-002 sẽ start trong 5 phút"
                  └─► Zabbix: Enable maintenance mode cho dc.anhlx.lab

Friday 00:00 AM - Implementation starts
                  ├─► Snapshot VM
                  ├─► Backup System State
                  └─► Begin upgrade...

Friday 03:45 AM - Implementation completed
                  ├─► Test AD services
                  ├─► Test LDAP auth (Zabbix, SDP, Grafana)
                  └─► All tests passed ✅

Friday 04:00 AM - Post-Implementation Review
                  └─► CR Status: Successful → Closed
                  └─► PIR document uploaded
                  └─► Telegram: "✅ CR-002 completed successfully"
```

### Logic so sánh: 1 vs 2 Approvers

```
╔══════════════════════════════════════════════════════════════════╗
║            COMPARISON: Single vs Multi Approval                   ║
╚══════════════════════════════════════════════════════════════════╝

┌──────────────────────────────────────────────────────────────────┐
│ CR-001: Standard Change (1 Approver)                             │
├──────────────────────────────────────────────────────────────────┤
│ Submit → phamvand approve → Done                                 │
│                                                                   │
│ Timeline: ~20 phút                                                │
│ Logic: ANY (1 người approve là đủ)                               │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ CR-002: Major Change (2 Approvers - Parallel)                    │
├──────────────────────────────────────────────────────────────────┤
│ Submit → phamvand approve (1/2)                                   │
│       → hoangvane approve (2/2) → Done                            │
│                    ↑                                              │
│         Cả 2 phải approve mới proceed                             │
│                                                                   │
│ Timeline: ~6 giờ (tùy workload của approvers)                     │
│ Logic: ALL (CẢ 2 phải approve, thiếu 1 = reject)                 │
│                                                                   │
│ Nếu phamvand reject → CR auto-reject (không cần hoangvane)       │
│ Nếu hoangvane reject → CR auto-reject (không cần phamvand)       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Tích hợp với Stack hiện tại

### 6.1. Zabbix Integration - Planned Maintenance Window

Khi CR được approve và scheduled, tự động tạo maintenance window trong Zabbix:

```python
# File: zabbix_maintenance_integration.py
# Tạo Zabbix maintenance window khi CR approved

import requests
from datetime import datetime, timedelta

class ZabbixMaintenanceManager:
    def __init__(self, zabbix_url, api_token):
        self.url = f"{zabbix_url}/api_jsonrpc.php"
        self.token = api_token
        self.headers = {"Content-Type": "application/json"}
    
    def create_maintenance(self, change_request):
        """
        Tạo maintenance window từ CR schedule
        """
        # Get host ID
        host_id = self.get_host_id(change_request['affected_host'])
        
        # Parse CR schedule
        start_time = datetime.fromisoformat(change_request['scheduled_start'])
        end_time = datetime.fromisoformat(change_request['scheduled_end'])
        
        # Zabbix maintenance payload
        payload = {
            "jsonrpc": "2.0",
            "method": "maintenance.create",
            "params": {
                "name": f"CR-{change_request['id']}: {change_request['subject']}",
                "active_since": int(start_time.timestamp()),
                "active_till": int(end_time.timestamp()),
                "description": f"Maintenance window for Change Request CR-{change_request['id']}",
                "maintenance_type": 0,  # 0 = with data collection
                "hostids": [host_id],
                "timeperiods": [
                    {
                        "timeperiod_type": 0,  # One-time
                        "start_date": int(start_time.timestamp()),
                        "period": int((end_time - start_time).total_seconds())
                    }
                ]
            },
            "auth": self.token,
            "id": 1
        }
        
        response = requests.post(self.url, json=payload, headers=self.headers)
        result = response.json()
        
        if 'result' in result:
            maintenance_id = result['result']['maintenanceids'][0]
            print(f"✅ Maintenance window created: {maintenance_id}")
            return maintenance_id
        else:
            print(f"❌ Error: {result.get('error', {}).get('data')}")
            return None
    
    def get_host_id(self, hostname):
        """Lấy host ID từ hostname"""
        payload = {
            "jsonrpc": "2.0",
            "method": "host.get",
            "params": {
                "filter": {"host": [hostname]}
            },
            "auth": self.token,
            "id": 1
        }
        
        response = requests.post(self.url, json=payload, headers=self.headers)
        result = response.json()
        return result['result'][0]['hostid']

# Usage example
if __name__ == "__main__":
    # CR-002 example
    cr_data = {
        "id": "002",
        "subject": "Nâng cấp AD DC: Windows Server 2022 → 2025",
        "affected_host": "dc.anhlx.lab",
        "scheduled_start": "2026-05-01T00:00:00+07:00",
        "scheduled_end": "2026-05-01T04:00:00+07:00"
    }
    
    zbx = ZabbixMaintenanceManager(
        zabbix_url="http://10.10.200.11:8080",
        api_token="YOUR_ZABBIX_API_TOKEN"
    )
    
    zbx.create_maintenance(cr_data)
```

### 6.2. Grafana Dashboard - Change Request Metrics

Tạo dashboard để track CR:

```json
{
  "dashboard": {
    "title": "ITIL Change Management Dashboard",
    "panels": [
      {
        "id": 1,
        "title": "Change Requests by Status",
        "type": "piechart",
        "targets": [
          {
            "rawSql": "SELECT status, COUNT(*) as count FROM changes WHERE created_at >= NOW() - INTERVAL '30 days' GROUP BY status"
          }
        ]
      },
      {
        "id": 2,
        "title": "Approval Timeline (Multi-Approver Changes)",
        "type": "table",
        "targets": [
          {
            "rawSql": "SELECT change_id, subject, requester, approvers_count, approval_duration_hours, status FROM changes WHERE approvers_count > 1 ORDER BY created_at DESC LIMIT 20"
          }
        ]
      },
      {
        "id": 3,
        "title": "Success Rate by Change Type",
        "type": "bargauge",
        "targets": [
          {
            "rawSql": "SELECT change_type, (SUM(CASE WHEN status='Successful' THEN 1 ELSE 0 END)::float / COUNT(*)) * 100 as success_rate FROM changes GROUP BY change_type"
          }
        ]
      },
      {
        "id": 4,
        "title": "Average Approval Time (Hours)",
        "type": "stat",
        "targets": [
          {
            "rawSql": "SELECT AVG(EXTRACT(EPOCH FROM (approved_at - created_at))/3600) as avg_approval_hours FROM changes WHERE status='Approved'"
          }
        ]
      },
      {
        "id": 5,
        "title": "Change Calendar (Upcoming)",
        "type": "calendar",
        "targets": [
          {
            "rawSql": "SELECT scheduled_start, scheduled_end, subject, risk_level FROM changes WHERE scheduled_start >= NOW() AND status IN ('Approved', 'Scheduled') ORDER BY scheduled_start"
          }
        ]
      }
    ]
  }
}
```

### 6.3. PostgreSQL Schema - Change Management Tables

```sql
-- File: change_management_schema.sql
-- Tạo tables để lưu CR data trong PostgreSQL

CREATE SCHEMA IF NOT EXISTS itil;

-- Table: Changes
CREATE TABLE itil.changes (
    change_id SERIAL PRIMARY KEY,
    subject VARCHAR(500) NOT NULL,
    description TEXT,
    change_type VARCHAR(50) NOT NULL, -- Standard, Normal, Major, Emergency
    risk_level VARCHAR(20) NOT NULL,   -- Low, Medium, High, Critical
    impact_level VARCHAR(20) NOT NULL,
    priority VARCHAR(20),
    status VARCHAR(50) NOT NULL,       -- Draft, Pending Approval, Approved, Rejected, Scheduled, In Progress, Successful, Failed, Cancelled
    
    -- Requester info
    requester_email VARCHAR(255) NOT NULL,
    requester_name VARCHAR(255),
    
    -- Approval tracking
    approvers_required INT DEFAULT 1,
    approvers_count INT DEFAULT 0,
    approval_logic VARCHAR(20) DEFAULT 'ANY', -- ANY, ALL
    approval_mode VARCHAR(20) DEFAULT 'sequential', -- sequential, parallel
    
    -- Scheduling
    scheduled_start TIMESTAMP WITH TIME ZONE,
    scheduled_end TIMESTAMP WITH TIME ZONE,
    actual_start TIMESTAMP WITH TIME ZONE,
    actual_end TIMESTAMP WITH TIME ZONE,
    
    -- Affected CIs
    affected_cis TEXT[], -- Array of affected configuration items
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    approved_at TIMESTAMP WITH TIME ZONE,
    implemented_at TIMESTAMP WITH TIME ZONE,
    closed_at TIMESTAMP WITH TIME ZONE,
    
    -- Additional metadata
    rollback_plan TEXT,
    success_criteria TEXT,
    pir_notes TEXT, -- Post-Implementation Review
    
    CONSTRAINT chk_risk_level CHECK (risk_level IN ('Low', 'Medium', 'High', 'Critical')),
    CONSTRAINT chk_change_type CHECK (change_type IN ('Standard', 'Normal', 'Major', 'Emergency'))
);

-- Table: Change Approvers
CREATE TABLE itil.change_approvers (
    approval_id SERIAL PRIMARY KEY,
    change_id INT NOT NULL REFERENCES itil.changes(change_id) ON DELETE CASCADE,
    approver_email VARCHAR(255) NOT NULL,
    approver_name VARCHAR(255),
    approver_role VARCHAR(100),
    department VARCHAR(100),
    
    -- Approval tracking
    approval_level INT DEFAULT 1,
    status VARCHAR(20) DEFAULT 'pending', -- pending, approved, rejected
    approved_at TIMESTAMP WITH TIME ZONE,
    rejection_reason TEXT,
    comments TEXT,
    
    CONSTRAINT chk_approval_status CHECK (status IN ('pending', 'approved', 'rejected'))
);

-- Table: Change Tasks (Implementation steps)
CREATE TABLE itil.change_tasks (
    task_id SERIAL PRIMARY KEY,
    change_id INT NOT NULL REFERENCES itil.changes(change_id) ON DELETE CASCADE,
    task_sequence INT NOT NULL,
    task_description TEXT NOT NULL,
    assigned_to VARCHAR(255),
    status VARCHAR(20) DEFAULT 'pending',
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT chk_task_status CHECK (status IN ('pending', 'in_progress', 'completed', 'failed', 'skipped'))
);

-- Indexes
CREATE INDEX idx_changes_status ON itil.changes(status);
CREATE INDEX idx_changes_scheduled ON itil.changes(scheduled_start, scheduled_end);
CREATE INDEX idx_changes_created ON itil.changes(created_at DESC);
CREATE INDEX idx_approvers_change ON itil.change_approvers(change_id);
CREATE INDEX idx_approvers_status ON itil.change_approvers(status);

-- View: Pending Approvals
CREATE VIEW itil.v_pending_approvals AS
SELECT 
    c.change_id,
    c.subject,
    c.change_type,
    c.risk_level,
    ca.approver_email,
    ca.approver_name,
    ca.approver_role,
    ca.department,
    c.scheduled_start,
    EXTRACT(EPOCH FROM (NOW() - c.created_at))/3600 AS hours_pending
FROM itil.changes c
JOIN itil.change_approvers ca ON c.change_id = ca.change_id
WHERE c.status = 'Pending Approval'
  AND ca.status = 'pending'
ORDER BY c.risk_level DESC, c.created_at;

-- View: Approval Statistics
CREATE VIEW itil.v_approval_stats AS
SELECT 
    c.change_id,
    c.subject,
    c.approvers_required,
    COUNT(ca.approval_id) FILTER (WHERE ca.status = 'approved') AS approved_count,
    COUNT(ca.approval_id) FILTER (WHERE ca.status = 'rejected') AS rejected_count,
    COUNT(ca.approval_id) FILTER (WHERE ca.status = 'pending') AS pending_count,
    CASE 
        WHEN c.approval_logic = 'ALL' AND COUNT(ca.approval_id) FILTER (WHERE ca.status = 'approved') = c.approvers_required THEN 'Fully Approved'
        WHEN c.approval_logic = 'ANY' AND COUNT(ca.approval_id) FILTER (WHERE ca.status = 'approved') > 0 THEN 'Partially Approved'
        WHEN COUNT(ca.approval_id) FILTER (WHERE ca.status = 'rejected') > 0 THEN 'Rejected'
        ELSE 'Pending'
    END AS overall_status
FROM itil.changes c
LEFT JOIN itil.change_approvers ca ON c.change_id = ca.change_id
GROUP BY c.change_id, c.subject, c.approvers_required, c.approval_logic;
```

### 6.4. ServiceDesk Plus Webhook - Auto-update on Approval

```python
# File: sdp_webhook_handler.py
# Webhook receiver để update Zabbix/Grafana khi CR approved

from flask import Flask, request, jsonify
import requests
from datetime import datetime

app = Flask(__name__)

@app.route('/webhook/change-approved', methods=['POST'])
def handle_change_approval():
    """
    ServiceDesk Plus gọi webhook này khi CR được approve
    """
    data = request.json
    
    change_id = data.get('change_id')
    status = data.get('status')
    approver = data.get('approver_email')
    
    print(f"📨 Webhook received: CR-{change_id} - {status} by {approver}")
    
    if status == 'Approved':
        # 1. Update PostgreSQL
        update_database(data)
        
        # 2. Create Zabbix maintenance window
        create_zabbix_maintenance(data)
        
        # 3. Send Telegram notification
        send_telegram_notification(data)
        
        # 4. Update Grafana annotations
        create_grafana_annotation(data)
        
        return jsonify({
            "status": "success",
            "message": f"CR-{change_id} approved successfully"
        })
    
    elif status == 'Rejected':
        send_telegram_notification(data, rejected=True)
        return jsonify({
            "status": "acknowledged",
            "message": f"CR-{change_id} rejected"
        })

def update_database(data):
    """Update PostgreSQL với approval info"""
    import psycopg2
    
    conn = psycopg2.connect(
        host="10.10.200.11",
        database="itil",
        user="postgres",
        password="your_password"
    )
    cur = conn.cursor()
    
    cur.execute("""
        UPDATE itil.change_approvers
        SET status = 'approved',
            approved_at = NOW()
        WHERE change_id = %s AND approver_email = %s
    """, (data['change_id'], data['approver_email']))
    
    # Check if all approvers approved
    cur.execute("""
        SELECT COUNT(*) FROM itil.change_approvers
        WHERE change_id = %s AND status = 'pending'
    """, (data['change_id'],))
    
    pending_count = cur.fetchone()[0]
    
    if pending_count == 0:
        # All approved → update change status
        cur.execute("""
            UPDATE itil.changes
            SET status = 'Approved',
                approved_at = NOW()
            WHERE change_id = %s
        """, (data['change_id'],))
    
    conn.commit()
    cur.close()
    conn.close()

def create_zabbix_maintenance(data):
    """Tạo Zabbix maintenance window"""
    # (Sử dụng code từ section 6.1)
    pass

def send_telegram_notification(data, rejected=False):
    """Gửi notification qua Telegram"""
    token = "YOUR_TELEGRAM_BOT_TOKEN"
    chat_id = "YOUR_CHAT_ID"
    
    if rejected:
        message = f"""
🔴 **Change Request REJECTED**

CR-{data['change_id']}: {data['subject']}
Rejected by: {data['approver_email']}
Reason: {data.get('rejection_reason', 'N/A')}
"""
    else:
        message = f"""
✅ **Change Request APPROVED**

CR-{data['change_id']}: {data['subject']}
Approved by: {data['approver_email']}
Status: {data['approval_progress']} ({data['approved_count']}/{data['total_approvers']})

Scheduled: {data['scheduled_start']}
"""
    
    requests.post(
        f"https://api.telegram.org/bot{token}/sendMessage",
        json={"chat_id": chat_id, "text": message, "parse_mode": "Markdown"}
    )

def create_grafana_annotation(data):
    """Tạo annotation trong Grafana dashboard"""
    grafana_url = "http://10.10.200.11:3000"
    api_key = "YOUR_GRAFANA_API_KEY"
    
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }
    
    annotation = {
        "text": f"CR-{data['change_id']} Approved: {data['subject']}",
        "tags": ["change-management", f"risk-{data['risk_level']}"],
        "time": int(datetime.now().timestamp() * 1000)
    }
    
    requests.post(
        f"{grafana_url}/api/annotations",
        json=annotation,
        headers=headers
    )

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## 7. Testing & Validation

### Test Case 1: Single Approver (CR-001)

```bash
# Test CR-001: Standard Change với 1 approver

# Step 1: Tạo CR
./create-standard-change.ps1

# Expected output:
# ✅ Change Request created successfully!
# CR ID: 1234
# Approver: phamvand@anhlx.lab
# Status: ⏳ Pending Approval

# Step 2: Login as phamvand, approve CR
# Expected: CR status → Approved → Scheduled

# Step 3: Verify Zabbix maintenance window
curl -X POST http://10.10.200.11:8080/api_jsonrpc.php \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "maintenance.get",
    "params": {
      "output": "extend",
      "selectTimeperiods": "extend"
    },
    "auth": "YOUR_AUTH_TOKEN",
    "id": 1
  }'

# Expected: Maintenance window created cho dc.anhlx.lab
```

### Test Case 2: Multi-Approver (CR-002)

```bash
# Test CR-002: Major Change với 2 approvers (parallel)

# Step 1: Tạo CR
./create-major-change.ps1

# Expected output:
# ✅ Major Change Request created successfully!
# CR ID: 1235
# Approvers (Parallel - ALL must approve):
#   - phamvand@anhlx.lab [IT Infrastructure]
#   - hoangvane@anhlx.lab [Security & Compliance]

# Step 2: Login as phamvand, approve
# Expected: Status → 1/2 approved (still pending)

# Step 3: Login as hoangvane, approve
# Expected: Status → 2/2 approved → Approved → Scheduled

# Step 4: Verify trong PostgreSQL
psql -h 10.10.200.11 -U postgres -d itil -c "SELECT * FROM itil.v_approval_stats WHERE change_id = 1235;"

# Expected:
# change_id | approved_count | rejected_count | pending_count | overall_status
# ----------+----------------+----------------+---------------+----------------
#   1235    |       2        |       0        |       0       | Fully Approved
```

### Test Case 3: Rejection Scenario

```bash
# Test: 1 trong 2 approver reject → CR auto-reject

# Step 1: Tạo CR-003 (clone từ CR-002)

# Step 2: Login as phamvand, REJECT với lý do
# Rejection reason: "Backup strategy chưa đầy đủ"

# Expected:
# - CR status → Rejected
# - hoangvane không cần approve nữa
# - Telegram: "🔴 CR-003 REJECTED by phamvand"
```

---

## 8. Best Practices & Recommendations

### 8.1. Approval Matrix Design

```yaml
Organizational Structure:

ANHLX.LAB
├─ IT Department
│  ├─ Infrastructure Team
│  │  └─ Lead: phamvand (approves: server, network, AD changes)
│  ├─ Application Team  
│  │  └─ Lead: levanc (approves: app deployment, database changes)
│  └─ Operations Team (L1/L2/L3 support)
│     
├─ Security & Compliance
│  └─ Lead: hoangvane (approves: security policies, access control)
│
└─ Management
   └─ CIO (approves: emergency changes, budget > $10k)

Approval Rules:
┌────────────────────────────────────────────────────────────┐
│ Change Type        │ Risk  │ Approvers                     │
├────────────────────────────────────────────────────────────┤
│ Patch/Update       │ Low   │ Team Lead (1)                 │
│ Config Change      │ Low   │ Team Lead (1)                 │
│ New Service Deploy │ Med   │ Dept Lead (1-2) parallel      │
│ Infrastructure     │ High  │ IT Lead + Security (2+)       │
│ AD/Identity        │ High  │ IT Lead + Security (2+)       │
│ Emergency          │ Crit  │ CIO (post-approval OK)        │
└────────────────────────────────────────────────────────────┘
```

### 8.2. SLA Guidelines

| Change Type | Approval SLA | Implementation Window | Rollback Time |
|-------------|--------------|----------------------|---------------|
| **Standard** | 4 hours | Anytime (business hours) | < 30 min |
| **Normal** | 24 hours | Off-peak (evening/weekend) | < 2 hours |
| **Major** | 72 hours | Planned maintenance window | < 4 hours |
| **Emergency** | 1 hour (post-approval) | Immediate | N/A (forward-fix) |

### 8.3. Communication Plan

```
Before Change (T-48h):
├─ Email to all affected users
├─ Update company intranet
└─ Post in Telegram announcement channel

During Change (T=0):
├─ Telegram real-time updates
├─ Update Grafana dashboard
└─ Status page for external users

After Change (T+24h):
├─ PIR document
├─ Lessons learned meeting
└─ Update knowledge base
```

### 8.4. Metrics to Track

```sql
-- KPI Queries

-- 1. Change Success Rate
SELECT 
    change_type,
    COUNT(*) as total_changes,
    SUM(CASE WHEN status = 'Successful' THEN 1 ELSE 0 END) as successful,
    ROUND(SUM(CASE WHEN status = 'Successful' THEN 1 ELSE 0 END)::numeric / COUNT(*) * 100, 2) as success_rate
FROM itil.changes
WHERE created_at >= NOW() - INTERVAL '90 days'
GROUP BY change_type;

-- 2. Average Approval Time
SELECT 
    change_type,
    approval_logic,
    AVG(EXTRACT(EPOCH FROM (approved_at - created_at))/3600) as avg_approval_hours
FROM itil.changes
WHERE status = 'Approved'
GROUP BY change_type, approval_logic;

-- 3. Top Reasons for Rejection
SELECT 
    rejection_reason,
    COUNT(*) as count
FROM itil.change_approvers
WHERE status = 'rejected'
  AND approved_at >= NOW() - INTERVAL '90 days'
GROUP BY rejection_reason
ORDER BY count DESC
LIMIT 10;

-- 4. Changes by Department (Requester)
SELECT 
    SUBSTRING(requester_email FROM '@(.*)$') as department,
    COUNT(*) as total_changes,
    SUM(CASE WHEN status = 'Successful' THEN 1 ELSE 0 END) as successful
FROM itil.changes
GROUP BY department
ORDER BY total_changes DESC;
```

---

## 9. Troubleshooting

### Issue 1: Approver không nhận được email

**Symptom:** CR created nhưng approver không thấy notification

**Root Cause:**
- SMTP settings trong SDP chưa config
- Email bị spam filter
- AD sync chưa chạy

**Solution:**
```bash
# Check SMTP settings
curl http://10.10.200.11:8400/api/v3/admin/mail_server_settings \
  -H "TECHNICIAN_KEY: YOUR_API_KEY"

# Test email
echo "Test email" | mail -s "SDP Test" phamvand@anhlx.lab

# Force AD sync
# Admin → AD Settings → Sync Now
```

### Issue 2: Multi-approval stuck (1 người approve, 1 người không)

**Symptom:** CR-002 có 1/2 approved, nhưng người thứ 2 không thấy pending approval

**Solution:**
```sql
-- Check approval status
SELECT * FROM itil.change_approvers WHERE change_id = 1235;

-- Resend notification
UPDATE itil.change_approvers
SET status = 'pending'
WHERE change_id = 1235 AND approver_email = 'hoangvane@anhlx.lab';

-- Trigger notification webhook manually
```

### Issue 3: Zabbix maintenance window không tự tạo

**Symptom:** CR approved nhưng maintenance window không xuất hiện trong Zabbix

**Root Cause:** Webhook handler không chạy hoặc Zabbix API token expired

**Solution:**
```bash
# Check webhook handler status
systemctl status sdp-webhook-handler

# Test Zabbix API
curl -X POST http://10.10.200.11:8080/api_jsonrpc.php \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "user.login",
    "params": {
      "username": "Admin",
      "password": "your_password"
    },
    "id": 1
  }'

# Manual create maintenance
python3 zabbix_maintenance_integration.py
```

---

## 10. Tổng kết

### So sánh Incident vs Change Management

| Khía cạnh | Incident (hiện tại) | Change (bài lab này) |
|-----------|---------------------|----------------------|
| **Trigger** | Zabbix alert (auto) | Human submit (manual) |
| **Approval** | Không cần | **Bắt buộc** (1-3 người) |
| **Assignment** | Round-robin auto | Manual hoặc workflow-based |
| **SLA** | ASAP (minutes) | Theo plan (hours/days) |
| **Risk** | Reactive (fix issue) | Proactive (controlled) |
| **Stakeholder** | IT Ops only | **Multi-department** |

### Key Takeaways

✅ **CR-001 (1 approver):** Phù hợp cho standard changes, low risk  
✅ **CR-002 (2 approvers - parallel):** Bắt buộc cho major changes ảnh hưởng nhiều phòng ban  
✅ **Parallel approval logic:** Cả 2 phải approve thì mới proceed  
✅ **Integration:** Zabbix (maintenance) + Grafana (dashboard) + Telegram (notification)  

### Next Steps

1. **Extend approval levels:** Thêm Level 3 (CAB board) cho critical changes
2. **Automation:** Tự động detect risk level dựa trên affected CIs
3. **Reporting:** Monthly Change Management reports
4. **Compliance:** Audit trail cho SOC 2, ISO 27001

---

## 11. Appendix

### A. ServiceDesk Plus API Reference

```bash
# List all change requests
curl -X GET "http://10.10.200.11:8400/api/v3/changes" \
  -H "TECHNICIAN_KEY: YOUR_API_KEY"

# Get specific change
curl -X GET "http://10.10.200.11:8400/api/v3/changes/1234" \
  -H "TECHNICIAN_KEY: YOUR_API_KEY"

# Approve change
curl -X PUT "http://10.10.200.11:8400/api/v3/changes/1234/approve" \
  -H "TECHNICIAN_KEY: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "approval": {
      "approver_email": "phamvand@anhlx.lab",
      "comments": "Approved - backup verified"
    }
  }'

# Reject change
curl -X PUT "http://10.10.200.11:8400/api/v3/changes/1234/reject" \
  -H "TECHNICIAN_KEY: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "approval": {
      "approver_email": "hoangvane@anhlx.lab",
      "comments": "Rejected - need more security documentation"
    }
  }'
```

### B. AD User Setup for Approvers

```powershell
# File: setup-ad-approvers.ps1
# Tạo AD users cho lab

Import-Module ActiveDirectory

# Create OU structure
New-ADOrganizationalUnit -Name "ITIL-Lab" -Path "DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "IT-Department" -Path "OU=ITIL-Lab,DC=anhlx,DC=lab"
New-ADOrganizationalUnit -Name "Security" -Path "OU=ITIL-Lab,DC=anhlx,DC=lab"

# Create users
$users = @(
    @{Name="Nguyen Van A"; SamAccountName="nguyenvana"; Email="nguyenvana@anhlx.lab"; OU="IT-Department"; Title="L1 Support"},
    @{Name="Tran Thi B"; SamAccountName="tranthib"; Email="tranthib@anhlx.lab"; OU="IT-Department"; Title="L1 Support"},
    @{Name="Le Van C"; SamAccountName="levanc"; Email="levanc@anhlx.lab"; OU="IT-Department"; Title="Application Lead"},
    @{Name="Pham Van D"; SamAccountName="phamvand"; Email="phamvand@anhlx.lab"; OU="IT-Department"; Title="IT Infrastructure Lead"},
    @{Name="Hoang Van E"; SamAccountName="hoangvane"; Email="hoangvane@anhlx.lab"; OU="Security"; Title="Security & Compliance Lead"}
)

foreach ($user in $users) {
    New-ADUser `
        -Name $user.Name `
        -SamAccountName $user.SamAccountName `
        -UserPrincipalName "$($user.SamAccountName)@anhlx.lab" `
        -EmailAddress $user.Email `
        -Title $user.Title `
        -Path "OU=$($user.OU),OU=ITIL-Lab,DC=anhlx,DC=lab" `
        -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
        -Enabled $true `
        -PasswordNeverExpires $true
}

# Create security groups for change approvers
New-ADGroup -Name "Change-Approvers-L1" -GroupScope Global -Path "OU=IT-Department,OU=ITIL-Lab,DC=anhlx,DC=lab"
New-ADGroup -Name "Change-Approvers-L2" -GroupScope Global -Path "OU=IT-Department,OU=ITIL-Lab,DC=anhlx,DC=lab"

Add-ADGroupMember -Identity "Change-Approvers-L1" -Members "phamvand","levanc"
Add-ADGroupMember -Identity "Change-Approvers-L2" -Members "hoangvane"
```

### C. Useful Links

- [ITIL 4 Change Enablement](https://www.axelos.com/certifications/itil-service-management/itil-4-foundation)
- [ServiceDesk Plus API Documentation](https://www.manageengine.com/products/service-desk/sdpod-v3-api/)
- [Zabbix Maintenance API](https://www.zabbix.com/documentation/current/en/manual/api/reference/maintenance)
- [Grafana Annotations API](https://grafana.com/docs/grafana/latest/developers/http_api/annotations/)

---

**Author:** Anh Le (anhlx9)  
**Lab Environment:** Ubuntu 22.04 + Windows Server 2022  
**Stack:** Zabbix 7.4 + ServiceDesk Plus 14 + Grafana + PostgreSQL 16  
**Created:** April 2026  
**Version:** 1.0
