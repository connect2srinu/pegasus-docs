# Guardian Control Plane — Use Case Requirements

Status: Proposed requirements aligned with the three use-case diagrams.

## 1. Notification Center

### High-level use case
Provide a notification center in the Control Plane UI where authorized users can view platform issues, agent failures and guardrail activity for their permitted organizations, projects and environments.

### What we are trying to achieve
- Show counts and trends for HTTP errors, including 500, 429 and 404, along with agent and tool failures.
- Show guardrail intervention counts and categories, including contextual grounding and configured custom policies.
- Separate platform reliability issues from safety interventions; a guardrail intervention does not automatically indicate a platform failure.
- Group related alerts, avoid duplicate notifications, and support unread/read state, acknowledgement and recovery updates.
- Present current counts and trends separately from threshold-triggered notifications.

### Request flow
1. The Control Plane API, UI BFF, Inference Gateway and BU execution resources emit structured telemetry to CloudWatch. Application instrumentation captures exact HTTP status codes and sanitized guardrail assessments.
2. CloudWatch evaluates configured thresholds. Alarm state changes flow through EventBridge to SQS.
3. A notification worker groups, deduplicates and classifies events, then stores notification history and user state in Aurora PostgreSQL.
4. The user opens the notification center. The UI BFF calls the Control Plane API, which checks access and retrieves notifications and current metrics.
5. The user filters by scope, opens details and acknowledges notifications. Recovery events update incident status.

## 2. Guardian Help Assistant

### High-level use case
Provide an assistant within the Control Plane UI that explains platform capabilities, guides users through workflows and helps them navigate features such as notifications, agents, toolkits and workloads. Resource creation is a proposed extension.

### What we are trying to achieve
- Answer questions using reviewed documentation matching the deployed release, with source references and valid UI links.
- Publish Markdown guides from the UI and API repositories to S3 and synchronize them into a managed knowledge base after releases or documentation updates.
- Help users prepare resource configurations and, after explicit review and confirmation, submit them through existing Control Plane APIs.
- Apply the same permissions, validation and deployment controls as the normal UI.
- Retrieve live resource status through authorized APIs; do not treat documentation as live operational data.

### Request flow
1. The release pipeline publishes versioned Markdown to S3, starts knowledge base ingestion, validates retrieval and activates the documentation version for the deployed release.
2. The user opens Help and asks a question or requests an action. The UI BFF forwards the request to the Control Plane API.
3. The API validates the Entra identity and resolves the authorized organization, project and environment before invoking the Help Assistant Harness.
4. For guidance, the assistant retrieves relevant documentation through AgentCore Gateway and the managed knowledge base. It uses Inference Gateway to call Bedrock in the AWS AI Account and returns an answer with sources and navigation links.
5. For resource creation, the assistant collects required inputs and presents the exact proposed configuration for user confirmation.
6. After confirmation, the backend rechecks permissions and submits through existing Control Plane APIs and provisioning workflows. The UI receives the resource link and actual deployment status.

## 3. Guardian Control Plane Operations Assistant

### High-level use case
Provide a platform-admin assistant that investigates incidents, explains platform health issues and recommends evidence-backed resolutions. Initial scope is administrator-initiated, read-only investigation; automated remediation is a future extension.

### What we are trying to achieve
- Investigate notifications involving internal errors, throttling, agent or tool failures and unusual guardrail activity.
- Correlate current telemetry, deployment changes, approved runbooks and similar historical incidents.
- Separate observed facts from hypotheses and provide recommended actions with supporting evidence.
- Use AgentCore episodic memory to retain verified incident experiences, including successful and unsuccessful remedies.
- Keep the authoritative incident history in Aurora or the incident-management system, with access scoped by environment and organization.

### Request flow
1. A platform administrator opens a notification and selects “Investigate with Operations Assistant,” or submits an investigation request.
2. The Control Plane API validates access, resolves the incident, service, environment and time window, and creates or resumes an investigation session.
3. The Operations Harness retrieves evidence through authorized Gateway tools: CloudWatch telemetry, operational API results, runbooks from the managed knowledge base and incident history. It also retrieves relevant AgentCore episodic memories.
4. The assistant calls Bedrock through Inference Gateway in the AWS AI Account and returns observed facts, possible causes, evidence links and suggested next steps.
5. The administrator reviews the recommendations and follows the approved operational process. Any future automated remediation must use scoped tools and existing approval controls.
6. The assistant rechecks live telemetry after the action and updates the incident record with the observed outcome.
7. After resolution is verified, a sanitized incident summary is published to shared episodic memory for future investigations. Unconfirmed hypotheses remain distinguished from verified findings.
