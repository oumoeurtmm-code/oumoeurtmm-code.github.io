# n8n + OpenCode Multi-Model IT Automation

> **Concept:** Use n8n as the outer workflow engine (triggers, approvals, branching) and call OpenCode as the multi-model "brain" that talks to Claude Code, Gemini, Grok, and Perplexity, while using native n8n nodes for Okta, Entra, and your clouds.

**References:**
- OpenCode n8n node: https://libraries.io/npm/n8n-nodes-opencode-ai
- n8n Okta + Entra integrations: https://n8n.io/integrations/microsoft-entra-id/and/okta/
- OpenCode: https://opencode.ai

---

## Overall Pattern

- **n8n** handles triggers (webhooks from HRIS/ITSM, schedules, chat commands), approvals, and all app integrations (Okta, Entra, Slack, Jira, cloud APIs).
- **OpenCode** runs as a separate service; called from n8n via the community OpenCode node or HTTP — internally fans out to Claude Code, Gemini, Grok, and Perplexity.
- For each flow (provisioning, access, ops):
  1. Normalize the event in n8n.
  2. Send a structured "task" payload to an OpenCode agent.
  3. Get back a plan or concrete commands.
  4. Execute those commands via n8n's Okta/Entra/cloud nodes, with safety checks.

---

## Core Building Blocks

### 1) n8n–OpenCode Integration

- Install the OpenCode community node (`n8n-nodes-opencode` or `n8n-nodes-opencode-ai`), which exposes an OpenCode Chat/Session node in n8n.
  - Ref: https://www.youtube.com/watch?v=0ffYAhPfpS0
- Point it at your OpenCode server URL and credentials; the node can auto-load models/agents from OpenCode and supports tool calling and multiple providers.
- In OpenCode, configure providers/models for Claude, Gemini, Grok, and Perplexity and define agents:
  - **CoordinatorAgent** — entitlement decisions across Okta/Entra
  - **CodeAgent** — script generation (Python, Bash, PowerShell)
  - **PolicyAgent** — access request reasoning
  - **RunbookAgent** — documentation and verification steps
  - **OpsAgent** — anomaly detection and cloud ops checks
- n8n just selects which agent to call per use case.

### 2) Okta + Entra Nodes in n8n

- Use dedicated n8n integrations:
  - **Okta**: create/update/delete user, get users, bulk queries, group membership
  - **Microsoft Entra ID**: create/update users and groups, manage app roles
  - Ref: https://hackceleration.com/okta-n8n/
- The AI layer outputs "desired state"; n8n applies those changes via native nodes.

---

## Workflow 1: New Hire → Mixed Okta/Entra Provisioning

```
[Trigger: Webhook/HRIS]
    → [Function: Normalize newHireEvent JSON]
    → [OpenCode: CoordinatorAgent]
        payload: newHireEvent + cloudProfile
        output:  { oktaActions[], entraActions[], cloudActions[] }
    → [IF: JSON Schema Validator]
        invalid → [Manual Review]
    → [IF: High-risk role?]
        yes → [Slack/Jira: Approval Request] → wait
    → [Loop: Okta nodes — apply oktaActions]
        Create User / Update User / Add to Group
    → [Loop: Entra nodes — apply entraActions]
        Create User / Assign Groups / App Roles
    → [Optional: Cloud nodes — apply cloudActions]
        AWS IAM / Azure RBAC / GCP IAM
    → [OpenCode: RunbookAgent]
        input:  summary of actions + errors
        output: runbook snippet with verification + rollback steps
    → [Confluence/Git node: Save runbook]
```

**CoordinatorAgent system prompt key points:**
- Decide entitlements across Okta (SaaS, VPN, SSO apps) and Entra (AAD groups, app roles)
- Output strict JSON plan with `oktaActions[]`, `entraActions[]`, optional `cloudActions[]`
- Uses Gemini/Perplexity for policy reasoning, Claude Code for script suggestions

---

## Workflow 2: Access Request Decision

```
[Trigger: Slack command or ITSM ticket]
    → [Function: Extract request context]
        fields: user, requested_system, reason
    → [OpenCode: PolicyAgent (Gemini/Perplexity)]
        output: { decision: auto_grant|needs_approval|deny, entitlements: {...} }
    → [Router]
        auto_grant    → [Okta/Entra nodes: apply entitlements]
        needs_approval → [Slack/ITSM: send approval] → wait → apply
        deny          → [Notify requester with explanation]
    → [OpenCode: RunbookAgent]
        Log decision logic + steps to access runbook
```

---

## Workflow 3: Multi-Cloud Ops Checks (Grok)

```
[Schedule Trigger: every 15 min]
    → [Cloud metric/log nodes]
        AWS CloudWatch + Azure Monitor + GCP Logging
    → [Function: Compact payload]
    → [OpenCode: OpsAgent (Grok)]
        output: anomalies[], severity, suggested_runbook_step_id
    → [IF: severity == high|critical]
        yes → [OpenCode: RunbookAgent] → generate steps → Slack alert
             → [Optional: cloud nodes run safe diagnostics]
```

---

## Multi-Model Routing Logic (Inside OpenCode)

| Agent | Models Used | Use Case |
|---|---|---|
| CoordinatorAgent | Gemini + Perplexity | Policy reasoning, entitlement decisions |
| CodeAgent | Claude Code | Script generation (PowerShell, Bash, Python) |
| PolicyAgent | Gemini + Perplexity | Access request evaluation |
| RunbookAgent | Gemini | Human-readable docs, verification steps |
| OpsAgent | Grok | Anomaly detection, real-time ops analysis |

**n8n payload flags to OpenCode:**

```json
{
  "agent": "CoordinatorAgent",
  "mode": "plan-only",
  "riskLevel": "high",
  "task": { ... }
}
```

- `plan-only` — JSON plan for n8n to apply, no direct execution
- `generate-script` — Claude Code writes PowerShell/Bash; n8n runs via SSH/Command node in sandbox
- `explain-and-document` — Gemini/Perplexity produce human-readable runbook content

---

## Next Steps

- [ ] Build concrete n8n JSON export for new-hire provisioning flow
- [ ] Build concrete n8n JSON export for access request flow
- [ ] Set up OpenCode server with all 4 model providers configured
- [ ] Define agent system prompts for each agent type
- [ ] Test Okta + Entra n8n nodes in sandbox environment
