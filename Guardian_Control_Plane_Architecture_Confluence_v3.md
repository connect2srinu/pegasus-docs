# Guardian AI Control Plane — Architecture Overview

Companion notes for the four-page editable draw.io architecture, draft v0.5, 26 October 2026. Solid connections show the current agreed design; dashed connections show planned integrations or items requiring confirmation. Design status does not establish production deployment or governance approval.

## Page 1 — AWS system architecture

The **AWS AI Account** hosts the Control Plane UI, API and Bedrock model access. Separate UI and API Git repositories feed the reviewed build/deployment pipeline and Amazon ECR. The platform uses EKS, corporate-only ALB ingress, private subnets and two availability zones. Each availability zone explicitly contains a Control Plane UI replica and a Control Plane API replica. The shared UI and API load balancers span both zones.

The Console is the **Control Plane UI**, containing a React UI and Express BFF. Each zone shows UI/BFF → Control Plane API. Calls use the shared API endpoint and the user's Entra token; the illustration does not impose same-zone routing affinity. The API includes the Inference Gateway endpoint, tenancy, governance and provisioning orchestration. Aurora PostgreSQL stores platform metadata, audit records and usage; Redis supports device-flow state and admission control.

Supporting connections include Route 53/ACM, Secrets Manager, KMS, IAM/STS and ServiceNow GSC. Application logs flow through CloudWatch and Firehose to Splunk; ALB access logs go to S3. AppDynamics is shown with implementation status to confirm.

BU execution accounts contain Harness agents, Gateways/Toolkits and Memory. Browser, Code Interpreter, Evaluations and API Catalog integration are connected as planned capabilities. API Catalog hosting remains to be defined.

## Page 2 — Environment onboarding and identity

Each BU has a **shared Dev/UAT AWS account** and a **separate Production account**. Dev and UAT remain distinct logical execution planes with separate resource bindings, names, tags and identities. The Control Plane itself uses separate AI Dev, UAT and Production accounts.

The security team creates a separate **Agent app and Gateway Tools app pair for each environment**, together with the corresponding AgentCore Identity credential providers. These apps reside in the Entra tenant; the diagram associates them with their execution environments. The Control Plane registers provider references. SailPoint governs group membership.

| IAM role | Purpose |
| --- | --- |
| EKS pod role | Supplies workload credentials through IRSA. |
| `ControlPlane-App` | Environment-specific hub role for cross-account access. |
| `ControlPlane-Management` | Creates or updates execution-account resources. |
| `ControlPlane-Execution-Agent` | Used by the Control Plane to invoke agents. |
| `ControlPlane-Execution-Gateway` | Used by the Control Plane to invoke Toolkits. |

Agent invocation includes both the execution-agent IAM role and **inbound Entra bearer-token validation**. The Control Plane obtains an OBO token for the Agent app; Harness validates the token, required access and Control Plane client binding. IAM access and inbound token validation are separate layers.

## Page 3 — Agent pattern and observability

The page is organized into three levels: **setup and access**, **runtime and tool access**, and **telemetry and quality**. Provisioning, identity and policy sit above the runtime flow; optional tools and Memory sit below Harness, with observability in a separate bottom section.

The runtime pattern connects **Harness → Gateway/Toolkit → API Catalog**, with catalog integration marked planned. AgentCore Identity provides tool credentials, and the Gateway is associated with PolicyEngine in its documented `LOG_ONLY` mode.

Harness stores and retrieves context through AgentCore Memory. Browser and Code Interpreter connect directly to Harness as planned tools. AWS documents `agentcore_browser` and `agentcore_code_interpreter` as Harness tool types, separate from `agentcore_gateway`; their native integration does not require a Gateway hop. See [AWS Harness tools documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-tools.html). The management role provisions and configures the supported pattern.

Planned tracing follows **Harness → ADOT → Arize**, with configured PII/confidential-data redaction before export. A proposed sanitized CloudWatch trace branch feeds AgentCore Evaluations, with evaluation results returned to CloudWatch. Collector placement, exporter compatibility, evaluation inputs and redaction effects require validation.

Governance reviews the complete pattern and versioned infrastructure/provisioning code. Repeated deployments must remain within approved templates and configuration boundaries.

## Page 4 — Inference Gateway and Bedrock

All callers use the same governed model-access path:

**Claude clients / Harness agents / Guardian applications → Inference Gateway → Bedrock in the AWS AI Account.**

Claude Code/Desktop runs on developer machines and uses Entra device sign-in followed by a gateway session JWT. Harness and Guardian applications use workload API keys.

Inference Gateway remains part of the Control Plane API repository and deployment. It resolves the caller to a workload, applies model allowlists and Bedrock Guardrails, performs admission control, and records usage and audit metadata. The Bedrock adapter calls Claude or other approved models **only within the AWS AI Account**. Responses return along the request path.

## Diagram icons

User symbols and the official Microsoft Entra ID icon are embedded in the relevant boxes. The Entra asset comes from the [Microsoft Entra architecture icon collection](https://learn.microsoft.com/en-us/entra/architecture/architecture-icons). AWS service icons, text and connectors remain editable in draw.io.
