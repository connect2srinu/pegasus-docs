# Guardian AI Control Plane — Architecture Overview

Companion notes for the four-page draw.io architecture (draft v0.3). Items marked **[P]** in the diagram are proposed or require confirmation.

## Page 1 — AWS system architecture

Shows the platform, execution accounts, model access and enterprise integrations.

- **Account A — Control Plane:** Guardian users receive access through Entra groups and sign in to the Control Plane. Separate UI and API repositories deploy Kubernetes containers that assume `ControlPlane-App`. The database stores configuration and resource records. The proposed production VPC spans two availability zones, with public ingress/egress and private workload/data subnets.
- **Account B — Execution plane:** Each organization has separate Dev, UAT and Production AWS accounts. The Control Plane provisions Harness agents and AgentCore Gateways (Toolkits) in the selected account. Future capabilities include Browser, Code Interpreter, Memory and Evaluations.
- **Model access:** Harness calls Bedrock within its execution account. Guardian apps in Account C and Claude Code in Account D call the Inference Gateway endpoint in Account A using API keys.
- **Integrations:** Harness telemetry flows through ADOT to Arize. Control Plane logs go to Splunk, and application performance telemetry goes to AppDynamics. Guardian AWS managed services are shown as supporting references.

## Page 2 — Environment onboarding and identity

Explains how an organization connects its execution environments and receives isolated application identities.

- Each environment has its own AWS account, **Agent Entra app** and **Gateway Tools Entra app**. Dev, UAT and Production do not share these app pairs.
- The security team creates the apps, secrets and corresponding AWS credential providers, then supplies the provider references.
- During onboarding, the Control Plane registers the organization/environment, connects its AWS execution account and stores the supplied provider references and metadata.
- Control Plane user SSO is separate from the environment-specific Agent and Gateway Tools identities.

## Page 3 — Role flows and observability

Explains how the Control Plane provisions and invokes resources across accounts.

| Role | Purpose |
| --- | --- |
| `ControlPlane-App` | Assumed by Control Plane Kubernetes containers in Account A. |
| `ControlPlane-Management` | Assumed in the execution account to create or update resources. |
| `ControlPlane-Execution-Agent` | Assumed in the execution account to invoke Harness agents. |
| `ControlPlane-Execution-Gateway` | Assumed in the execution account to invoke Gateways/Toolkits. |

The three execution-account roles authorize Control Plane calls. Runtime and Gateway service execution permissions are separate and remain to be defined.

Harness calls Bedrock locally for model requests and uses Toolkits for approved downstream tools. Its configured OpenTelemetry signals follow **Harness → AWS ADOT Collector → Arize**. ADOT applies configured PII/confidential-data redaction before export; S3 is not part of this path.

Governance reviews the complete reusable pattern and versioned infrastructure code. Deployments must stay within approved template and configuration boundaries; changes or deviations require review.

## Page 4 — Inference Gateway and Bedrock

Details the two current model-access paths:

- **Shared model access:** Guardian apps in Account C and Claude Code running in Account D authenticate with API keys to the Inference Gateway. This is an endpoint within the Control Plane API repository and containers in Account A. It invokes Account A Bedrock endpoints, including Claude models.
- **Harness model access:** Harness agents in Account B invoke Bedrock directly within Account B. They do not use Account A's Inference Gateway in this version.

## Open design decisions

Confirm Kubernetes/runtime hosting, ingress and private connectivity, database engine and high availability, service execution permissions, and ADOT placement, redaction policy and exporter support. The diagram is a design draft, not evidence of governance approval.
