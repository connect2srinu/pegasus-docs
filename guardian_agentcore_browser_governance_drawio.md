# Guardian AI Control Plane — AgentCore Browser Integration

**Status:** Draft for Technical Governance  
**Runtime:** AWS Bedrock AgentCore Runtime, ARM64 / Graviton-compatible container  
**Browser:** AWS-managed built-in `aws.browser.v1`  
**Owner:** Guardian AI Platform / Control Plane

**Architecture diagrams:** [Editable draw.io multi-page file](guardian_agentcore_browser_flows.drawio)

## 1. Purpose and decision

Enable approved agents deployed through the Guardian Control Plane to use the AWS-managed AgentCore Browser for web navigation and automation. **Recommended Phase 1:** invoke Browser directly from the AgentCore Harness using the **Control Plane Execution Agent Role**. **Alternative:** expose a Browser adapter through AgentCore Gateway; in that case, permissions to Browser belong to the adapter's actual execution identity, **not automatically** the Control Plane Execution Gateway Role.

The Browser is a separate managed service, not a browser bundled into the ARM64 agent container. Deploying an agent does not automatically enable Browser access.

## 2. Direct invocation — recommended

**Editable draw.io diagram:** See [`guardian_agentcore_browser_flows.drawio`](guardian_agentcore_browser_flows.drawio) (pages: Direct, Gateway, AgentCore vs EKS, Browser Action Loop).

**Execution:** The agent's Harness decides to call a configured Browser tool (for example, an SDK/Playwright adapter). AgentCore Browser hosts the actual browser session. The agent's execution role signs Browser API requests. Close sessions on success, error, and timeout.

## 3. Gateway-mediated invocation — alternative

**Editable draw.io diagram:** See [`guardian_agentcore_browser_flows.drawio`](guardian_agentcore_browser_flows.drawio) (pages: Direct, Gateway, AgentCore vs EKS, Browser Action Loop).

**Role distinction:** For a **Lambda target**, the Control Plane Execution Gateway Role needs `lambda:InvokeFunction` on that target; the **Lambda execution role** needs Browser session permissions. For an MCP/HTTP adapter, its workload identity needs Browser permissions. Only assign Browser permissions to the Gateway role if that role is genuinely the principal calling Browser APIs. Gateway is not itself a built-in Browser automation executor.

## 4. IAM permission matrix

| Identity | Required permissions | Purpose |
|---|---|---|
| **Control Plane Execution Agent Role** (direct) | `bedrock-agentcore:StartBrowserSession`, `GetBrowserSession`, `ConnectBrowserAutomationStream`, `StopBrowserSession` | Run Browser sessions directly |
| **Agent caller** (Gateway option) | `bedrock-agentcore:InvokeGateway` scoped to approved Gateway | Invoke the Browser tool exposed by Gateway |
| **Control Plane Execution Gateway Role** (Lambda target) | `lambda:InvokeFunction` scoped to Browser adapter | Invoke adapter; no Browser permissions by default |
| **Browser adapter execution role** | Same Browser session permissions as direct agent role | Perform Browser operations |
| **Control Plane management role** | Browser creation/configuration actions only if custom Browser resources are used | Provision resources, not ordinary built-in Browser sessions |

### 4.1 Browser session policy — direct agent role OR adapter role

Replace `<REGION>` and `<ACCOUNT_ID>`. The resource scope below is a **starting template**, not a validated least-privilege production policy. Confirm the built-in system-browser ARN and per-action resource-level support in the target account and AWS Service Authorization Reference; tighten the resource to `aws.browser.v1` where supported.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AgentCoreBrowserSessionAccess",
      "Effect": "Allow",
      "Action": [
        "bedrock-agentcore:StartBrowserSession",
        "bedrock-agentcore:GetBrowserSession",
        "bedrock-agentcore:ConnectBrowserAutomationStream",
        "bedrock-agentcore:StopBrowserSession"
      ],
      "Resource": "arn:aws:bedrock-agentcore:<REGION>:<ACCOUNT_ID>:browser/*"
    }
  ]
}
```

`GetBrowser`, `ListBrowserSessions`, `UpdateBrowserStream`, and `ConnectBrowserLiveViewStream` are **optional**, only if implementation/operations require them. Model invocation, CloudWatch, KMS, S3, or Secrets Manager permissions are separate and must be scoped to actual dependencies. Avoid `BedrockAgentCoreFullAccess` for agent execution.

### 4.2 Gateway execution role — Lambda target only

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InvokeBrowserAdapter",
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:<REGION>:<ACCOUNT_ID>:function:<BROWSER_ADAPTER_FUNCTION>"
    }
  ]
}
```

The calling agent additionally needs Gateway invocation authorization (IAM `bedrock-agentcore:InvokeGateway` for IAM-authenticated invocation, or the configured supported OAuth/JWT flow). If Gateway Policy/Cedar is enabled, include its documented policy-engine permissions in the Gateway role. Cross-account Lambda targets also require appropriate Lambda resource-based permissions.

## 5. Browser execution from AgentCore Runtime vs. Kubernetes agents

### 5.1 AWS reference flow and responsibilities

The AWS Browser Tool diagram depicts **LLM tool selection → agent-side command translation (for example Playwright/browser-use) → remote managed browser execution → observation/screenshot returned to agent**. The LLM itself has no IAM identity and does not execute Browser API requests. The **process running the Browser SDK/client** is the AWS principal requiring Browser permissions. The browser is not embedded in the agent's container or EKS pod.

**Editable draw.io diagram:** See [`guardian_agentcore_browser_flows.drawio`](guardian_agentcore_browser_flows.drawio) (pages: Direct, Gateway, AgentCore vs EKS, Browser Action Loop).

The start/connect steps normally occur once per browser session, not for every click. If the implementation uses the newer **InvokeBrowser** OS-action API instead of CDP/Playwright, grant `bedrock-agentcore:InvokeBrowser` and adjust the adapter; do not automatically grant it for a CDP-only implementation.

### 5.2 Deployment and IAM ownership matrix

| Agent deployment | AWS identity used to call Browser | Where Browser permission is attached | Other required configuration |
|---|---|---|---|
| **AgentCore Runtime (Graviton/ARM64 container)** | AgentCore Runtime **execution role** credentials | Guardian **Control Plane Execution Agent Role**, if this is the role configured as the Runtime execution role | AgentCore Runtime trust policy; SDK/automation adapter in ARM64 image; connectivity to AgentCore APIs |
| **Amazon EKS / Kubernetes agent** | Pod IAM identity (EKS Pod Identity or IRSA) | Dedicated IAM role bound to the **agent Kubernetes service account** | EKS Pod Identity association or IRSA service-account annotation/trust; outbound connectivity to AWS API and browser automation endpoints |
| **Kubernetes outside EKS / outside AWS** | Federated workload role or brokered short-lived AWS credentials | IAM role assumed by workload through approved federation | OIDC/STS or other approved credential exchange; no static long-lived AWS keys |
| **AgentCore Gateway → Lambda/MCP Browser adapter** | The adapter's execution identity | Lambda/MCP **execution role**, not automatically the Gateway role | Agent allowed to call Gateway; Gateway allowed to invoke adapter; adapter IAM allowed to call Browser |

**Important:** Guardian's `Control Plane Execution Agent Role` is a project naming convention, not an AWS built-in role. An EKS pod does not automatically inherit this role because the Control Plane deployed or registered the agent. The actual AWS principal must be identified for each deployment. A Gateway execution role typically authorizes invocation of the target; it does not run the remote browser client.

### 5.3 Minimum Browser API policy (attach to the actual calling principal)

For the **built-in** system browser `aws.browser.v1`, use this as the starting policy for a CDP/Playwright integration:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UseBuiltinAgentCoreBrowser",
      "Effect": "Allow",
      "Action": [
        "bedrock-agentcore:StartBrowserSession",
        "bedrock-agentcore:GetBrowserSession",
        "bedrock-agentcore:ConnectBrowserAutomationStream",
        "bedrock-agentcore:StopBrowserSession"
      ],
      "Resource": "arn:aws:bedrock-agentcore:<REGION>:aws:browser/aws.browser.v1"
    }
  ]
}
```

This is a **proposed least-privilege policy**, not a guarantee that every SDK feature works without additional permissions. Verify resource-level support and effective permissions in the target region. Add `bedrock-agentcore:InvokeBrowser` **only** for OS-level browser actions; add live-view, profile, recording, proxy-secret, or custom-browser permissions only when explicitly used. `CreateBrowser`, `DeleteBrowser`, and `ListBrowsers` are **not** needed merely to use the built-in browser.

The agent may also need **separate model invocation permissions** (for example `bedrock:InvokeModel`) depending on its LLM, and standard log/trace permissions. Those are not Browser-specific.

### 5.4 EKS workload identity and cross-account access

For an EKS-hosted agent, prefer a dedicated Kubernetes service account and an IAM role mapped through **EKS Pod Identity** or **IRSA**. The role receives the Browser policy above. Its trust policy must be restricted to the approved EKS cluster/service account and the chosen identity mechanism; the trust policy differs between Pod Identity and IRSA. Do **not** copy the AgentCore Runtime service trust policy to EKS.

If EKS and the Browser are in **different AWS accounts**, determine whether the Browser resource and its APIs support the proposed cross-account access pattern. Where supported, use a controlled role-assumption or resource authorization design, including `sts:AssumeRole` and target-role trust as appropriate. Otherwise deploy the adapter in the Browser-owning account and expose a secured endpoint. Cross-account support must be validated rather than assumed.

### 5.5 Example execution paths

**AgentCore Runtime / Graviton:** `User → AgentCore Runtime (ARM64 Harness) → LLM selects browser tool → Browser SDK (Runtime execution role) → aws.browser.v1 session → website → observation → LLM → response`.

**EKS agent:** `User → EKS pod → LLM selects browser tool → Browser SDK (Pod Identity/IRSA role) → aws.browser.v1 session → website → observation → response`.

**Gateway adapter:** `User → agent → AgentCore Gateway → approved Lambda/MCP adapter → Browser API (adapter execution role) → aws.browser.v1 → adapter → agent`. The Gateway role is scoped to target invocation, while the adapter role receives Browser access.

### 5.6 Control Plane enablement / governance flow

**Editable draw.io diagram:** See [`guardian_agentcore_browser_flows.drawio`](guardian_agentcore_browser_flows.drawio) (pages: Direct, Gateway, AgentCore vs EKS, Browser Action Loop).

Control Plane metadata should capture: agent/project, hosting type (`AGENTCORE_RUNTIME` / `EKS` / `OTHER`), execution principal ARN, browser identifier, approval, permitted actions/destinations where enforceable, session limits, and trace correlation ID. These are Guardian-defined governance fields, not necessarily native AWS Browser settings.

## 6. Runtime and security requirements

- **ARM64 / Graviton:** Build the AgentCore Runtime container and its Python/native dependencies for `linux/arm64`. Use compatible Browser SDK/automation libraries; the actual managed browser does not run in this container.
- **IAM isolation:** If multiple agents share one Execution Agent Role, every agent using that role can inherit its Browser API permissions. Prefer per-agent roles or a separately enforced authorization boundary.
- **Browser authorization:** Require approved capability and restrict destinations via verified network/proxy controls or a trusted adapter. A Control Plane UI flag alone is not an IAM boundary.
- **Action safety:** Require explicit approval for purchases, submissions, data changes, or sensitive actions; guard against prompt injection from web pages.
- **Audit:** Record agent identity, session ID, target domain (without secrets), start/stop timestamps, outcome, and trace correlation; redact credentials and sensitive page content.
- **Operations:** Set session timeouts, concurrency quotas, retry/backoff, and `finally` cleanup. Monitor cost, failures, and session duration.

## 7. Validation checklist / open decisions

- [ ] Verify Browser availability in Guardian target region and built-in resource ARN/IAM resource matching.
- [ ] Validate session start/connect/stop with the actual Execution Agent Role.
- [ ] Confirm ARM64 Harness image and Browser automation dependencies work end-to-end.
- [ ] Decide whether direct Browser access meets destination and audit requirements or a Gateway adapter is required.
- [ ] Confirm per-agent IAM isolation and provisioning/revocation behavior.
- [ ] Test unauthorized agent, denied destination, browser timeout, failed cleanup, and rate-limit cases.
- [ ] Decide whether built-in Browser security controls suffice or a custom Browser resource is necessary.

## 8. AWS references

- [AgentCore Browser quickstart](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/browser-quickstart.html)
- [Managing Browser sessions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/browser-managing-sessions.html)
- [StartBrowserSession API](https://docs.aws.amazon.com/bedrock-agentcore/latest/APIReference/API_StartBrowserSession.html)
- [Gateway service role permissions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-prerequisites-permissions.html)
- [Gateway and Cedar policy permissions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-permissions.html)
- [AgentCore Runtime ARM64 container requirements](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-agui-protocol-contract.html)

**Governance decision requested:** Approve **direct Harness-to-Browser invocation** as Phase 1, subject to IAM isolation, destination controls, auditability, and development-account validation. Retain Gateway-mediated adapter as a governed alternative.

- [AgentCore Browser fundamentals and IAM](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/browser-resource-session-management.html)
- [AgentCore Browser OS actions (InvokeBrowser)](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/browser-invoke.html)
- [AgentCore IAM service authorization reference](https://docs.aws.amazon.com/service-authorization/latest/reference/list_bedrock-agentcore.html)
- [AgentCore Runtime execution role permissions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-permissions.html)
