# Spring Boot API Exposure Patterns for Amazon Bedrock AgentCore Gateway

**Status:** Architecture Proposal  
**Audience:** AI Platform, Application, Security, Cloud Platform, and API Engineering teams  
**Last updated:** August 11, 2026  
**Scope:** Exposing existing Spring Boot REST APIs running on Amazon EKS as agent tools through Amazon Bedrock AgentCore Gateway.

---

## 1. Purpose

This document evaluates three patterns for exposing existing Spring Boot APIs to AI agents through Amazon Bedrock AgentCore Gateway. The Spring Boot application is assumed to run on Amazon EKS in an application/business-unit AWS account, while the enterprise Control Plane runs in a separate AWS account.

The three patterns are:

1. **Spring Boot as an MCP Server** — add MCP capabilities to the Spring Boot application and register it as an MCP server target.
2. **Existing Spring REST API as an OpenAPI Target** — keep the application as REST and let AgentCore Gateway translate MCP tool calls to REST. This includes an **Okta Access Gateway protected variant** for customer-entitlement use cases.
3. **Spring REST API behind Amazon API Gateway** — expose an API Gateway REST API stage as the AgentCore Gateway target and privately integrate API Gateway with EKS.

The objective is to select a pattern that preserves existing APIs, minimizes unnecessary protocol coupling, supports enterprise authorization, and works cleanly with the cross-account Control Plane model.

---

## 2. Proposed Cross-Account Deployment Model

### Control Plane AWS Account

The Control Plane remains the centralized management plane. Typical responsibilities include:

- Organization/project configuration and approvals.
- Tool registration and lifecycle management.
- Storage and validation of OpenAPI specifications and deployment metadata.
- Deployment orchestration into application/BU AWS accounts.
- Assuming a pre-provisioned cross-account deployment/execution role in the target AWS account.
- Recording the AgentCore Gateway, target, and environment identifiers returned by AWS.

The Control Plane should not be in the runtime request path for normal agent tool execution.

### Application / BU AWS Account

The application account contains the runtime resources:

- Amazon Bedrock AgentCore Runtime or other approved agent runtime.
- Amazon Bedrock AgentCore Gateway.
- AgentCore Gateway targets.
- AgentCore Identity credential providers as required.
- Gateway service/execution IAM role.
- Amazon VPC Lattice private-endpoint resources for private MCP/OpenAPI targets.
- Amazon API Gateway and VPC Link where Pattern 3 is selected.
- Internal ALB/NLB as applicable.
- Amazon EKS cluster and Spring Boot workloads.
- Okta Access Gateway route where existing customer entitlement enforcement is required.

**Recommendation:** Keep AgentCore Gateway and its targets in the application/BU account when practical. This keeps runtime network access and target dependencies close to the protected application. The Control Plane manages these resources cross-account through an IAM role rather than routing data-plane traffic through the Control Plane account.

---

## 3. Common Runtime Model

Regardless of the target type, agents connect to a single AgentCore Gateway endpoint and discover/invoke permitted tools. AgentCore Gateway provides the agent-facing MCP boundary and maps the request to the configured target.

At a high level:

```text
User / Application
       |
       v
Agent Runtime
       |
       | MCP tools/list + tools/call
       v
AgentCore Gateway
       |
       +--> Target-specific outbound call
       |
       v
Spring Boot capability
```

Inbound authorization determines whether the caller may access the AgentCore Gateway. Target/policy authorization determines which tools the caller or agent may invoke. Outbound authorization determines how AgentCore Gateway authenticates to the selected downstream target.

---

# 4. Pattern 1 — Spring Boot Application as an MCP Server

## 4.1 Architecture

The existing Spring Boot application is enhanced with an MCP server interface. The application publishes agent-oriented tools directly and is registered in AgentCore Gateway as an **MCP server target**.

```text
Agent Runtime
     |
     | MCP
     v
AgentCore Gateway
     |
     | MCP
     v
Spring Boot MCP Server on EKS
     |
     v
Existing application services
```

If the EKS endpoint is private, AgentCore Gateway can use a private endpoint backed by Amazon VPC Lattice to reach the MCP server without exposing it publicly.

## 4.2 Advantages

- Full control over MCP tool definitions and agent-oriented contracts.
- MCP server can expose higher-level business tools rather than one tool per REST endpoint.
- Suitable when the application needs MCP-native behavior, capability discovery, prompts/resources, or stateful MCP interactions.
- Tool definitions can intentionally differ from the existing REST interface.

## 4.3 Disadvantages

- Adds MCP protocol responsibilities to an existing business service.
- The application team now owns REST and MCP lifecycle/testing/compatibility.
- Additional implementation and operational complexity for a service that already exposes usable REST endpoints.
- Potentially tighter coupling between business application releases and agent-tool contract changes.

## 4.4 Best Fit

Use this pattern when the team intentionally wants the Spring Boot application to become a first-class MCP product, or when agent tools need to perform higher-level orchestration that does not map naturally to individual REST operations.

For a small set of existing REST endpoints, this is normally not the first choice.

---

# 5. Pattern 2 — Existing Spring REST API as an OpenAPI Target

## 5.1 Architecture

The Spring Boot application remains unchanged as a REST service. An OpenAPI 3.x document defines only the approved operations that should be visible to agents. AgentCore Gateway registers the OpenAPI schema as a target and translates MCP tool calls into HTTP requests.

```text
Agent Runtime
     |
     | MCP
     v
AgentCore Gateway
     |
     | OpenAPI target
     | MCP -> HTTP translation
     v
Private Endpoint / VPC Lattice
     |
     v
Internal ALB
     |
     v
Spring Boot on EKS
```

The OpenAPI `operationId` is used as the MCP tool name, so operation names and descriptions should be deliberately agent-friendly.

## 5.2 Role of VPC Lattice

VPC Lattice solves private connectivity between the AWS-managed AgentCore Gateway service and an internal REST endpoint in the application VPC. For same-account private endpoints, AgentCore can manage the resource gateway and related resources when the target is configured with `privateEndpoint.managedVpcResource`.

VPC Lattice is a **network connectivity mechanism**, not an API-contract or authorization mechanism:

- **OpenAPI** describes what operations/tools exist.
- **AgentCore policy/authorization** determines who may invoke them.
- **VPC Lattice/private endpoint** provides the private network path to the application.
- **Spring Boot** implements the business operation.

For direct cross-account VPC Lattice connectivity, AWS documents a self-managed Lattice model shared through AWS RAM. In the proposed deployment model, this complexity can generally be avoided by placing AgentCore Gateway and the private Spring Boot target in the same application/BU account and letting the Control Plane manage them cross-account.

## 5.3 Advantages

- Minimal changes to the Spring Boot application.
- REST remains the reusable contract for agent and non-agent consumers.
- AgentCore Gateway owns MCP-to-HTTP translation.
- Easy to expose only a selected subset of the application's endpoints.
- Supports private EKS access without forcing the REST API onto the public internet.
- OpenAPI targets support several outbound authorization options, including OAuth client credentials, authorization code, OAuth token exchange/OBO, IAM where the target can validate SigV4, API key, or no-auth where explicitly allowed.

## 5.4 Disadvantages / Constraints

- OpenAPI must conform to the subset AgentCore Gateway supports.
- Approved operations should have clear `operationId` values.
- Complex OpenAPI constructs may require simplification.
- A low-level REST API may produce tools that are technically correct but not ideal for an LLM; descriptions and schemas must be curated.

## 5.5 Recommended Default

**This is the recommended default pattern for existing Spring Boot REST APIs.** It minimizes code changes and keeps the agent protocol boundary in AgentCore Gateway.

---

## 5.6 Pattern 2B — OpenAPI Target Through Okta Access Gateway

Some customer-facing operations already rely on Okta Access Gateway for entitlement checks. Those operations can retain that existing security boundary.

```text
External Customer
      |
      | Okta authenticated session/token
      v
Application / Agent Runtime
      |
      v
AgentCore Gateway
      |
      | OpenAPI target
      | OAuth OBO / token exchange where required
      v
Okta Access Gateway
      |
      | Customer entitlement / URL authorization
      v
Spring Boot API on EKS
```

### Authorization Responsibilities

A useful separation of concerns is:

- **AgentCore Gateway:** Is this caller/agent allowed to invoke this tool?
- **Okta Access Gateway:** Is this external customer entitled to perform the protected business operation?
- **Spring Boot:** Is the request valid under the application's business rules?

### Token Handling

For OpenAPI schema targets, AgentCore Gateway currently supports OAuth token exchange/on-behalf-of but does **not** support raw inbound token passthrough. Therefore, if the downstream Okta-protected route requires user-context authorization, design the flow around an OAuth token exchange/OBO model rather than assuming that the exact browser access token will simply be forwarded unchanged.

### Bypass Prevention

For operations that require Okta entitlement enforcement, the direct backend path should not become an authorization bypass. Network policy, routing, and application controls should ensure the protected operation is reached only through the approved Okta Access Gateway route, or otherwise performs equivalent authorization at the backend.

### Target Separation

Prefer separate AgentCore targets for different trust/security models, for example:

- `customer-profile-direct` → OpenAPI → private Spring Boot endpoint.
- `customer-entitled-operations` → OpenAPI → Okta Access Gateway → Spring Boot.

This improves policy isolation, auditing, credential configuration, and troubleshooting.

---

# 6. Pattern 3 — Spring REST API Behind Amazon API Gateway

## 6.1 Architecture

Amazon API Gateway becomes the API management layer in front of the Spring Boot service. AgentCore Gateway registers an API Gateway REST API stage as a target.

```text
Agent Runtime
     |
     | MCP
     v
AgentCore Gateway
     |
     | API Gateway REST API target
     v
Amazon API Gateway
     |
     | Private Integration / VPC Link
     v
Internal ALB/NLB
     |
     v
Spring Boot on EKS
```

## 6.2 Advantages

- Useful where API Gateway is already the enterprise-standard API ingress layer.
- Central API lifecycle, throttling, monitoring, routing, and API governance.
- Tool filters can explicitly allow only selected resource-path and method combinations.
- API Gateway can privately integrate with resources inside the application VPC.
- Strong fit when the same APIs are also intended for broader non-agent API consumption.

## 6.3 Disadvantages / Constraints

- Additional managed service, configuration, cost, and network hop.
- The native AgentCore **API Gateway stage target** currently requires:
  - API Gateway REST API, not HTTP API or WebSocket API.
  - Same AWS account as the AgentCore Gateway.
  - Same AWS Region as the AgentCore Gateway.
  - A public Regional API Gateway endpoint; the API Gateway can then use private integration/VPC Link to reach EKS.
- Native API Gateway stage targets have fewer outbound authorization options than OpenAPI targets; AWS currently documents IAM and API-key approaches for that target type.
- If the API Gateway deployment cannot meet native target constraints (for example, cross-account), export/use its OpenAPI schema and register it as an **OpenAPI target** instead.

## 6.4 Best Fit

Use this pattern when API Gateway provides required enterprise API-management controls, when organization standards mandate it, or when the Spring Boot API is intentionally published as a managed enterprise API beyond the agent platform.

Do not introduce API Gateway solely because the agent needs MCP tools; AgentCore Gateway already converts OpenAPI operations into MCP tools.

---

# 7. Comparison

| Area | Pattern 1: Spring MCP Server | Pattern 2: OpenAPI Target | Pattern 3: API Gateway Target |
|---|---|---|---|
| Spring Boot changes | Medium/High | Low | Low/Medium |
| MCP implementation in app | Yes | No | No |
| Reuse existing REST API | Indirect | **Yes** | **Yes** |
| AgentCore performs MCP translation | Aggregates MCP | **Yes, MCP → HTTP** | **Yes, MCP → API Gateway HTTP** |
| Private EKS connectivity | VPC Lattice private endpoint | **VPC Lattice private endpoint** | API Gateway private integration/VPC Link |
| API-management features | Limited to app implementation | Limited to existing API stack | **Strong** |
| User-context OAuth/OBO | Supported for MCP target | **Supported** | Not supported by native API Gateway stage target |
| Best for existing ~10 REST APIs | Usually no | **Yes — recommended** | When API Gateway governance is required |
| Application/agent coupling | Higher | **Lower** | Medium |
| Operational components | Medium | **Lowest** | Highest |

---

# 8. Recommended Architecture

For the existing Spring Boot application, use **Pattern 2: OpenAPI Target** as the standard approach.

### Standard internal/private route

```text
AgentCore Gateway
      |
      | OpenAPI target
      v
Managed private endpoint / VPC Lattice
      |
      v
Internal load balancer
      |
      v
Spring Boot / EKS
```

### Existing customer-entitlement route

```text
AgentCore Gateway
      |
      | OpenAPI target + OAuth OBO/token exchange
      v
Okta Access Gateway
      |
      | entitlement check
      v
Spring Boot / EKS
```

### API-management route

Use Pattern 3 only when API Gateway provides a required enterprise control or is already the canonical ingress path for that API.

### MCP-native route

Use Pattern 1 only where the business/application team intentionally wants to own an MCP server or needs higher-level agent tools that do not map well to REST endpoints.

---

# 9. Request Flows

## Flow A — Direct OpenAPI Target

1. User invokes an agent capability.
2. Agent calls the AgentCore Gateway MCP endpoint.
3. AgentCore Gateway validates inbound identity and policy.
4. Gateway identifies the OpenAPI target/tool.
5. Gateway translates the MCP tool invocation into the corresponding HTTP request.
6. The private endpoint/VPC Lattice path carries the request into the application VPC.
7. Internal ALB routes the request to the EKS service.
8. Spring Boot executes the operation and returns JSON.
9. AgentCore Gateway converts the target response into an MCP tool response.
10. Agent consumes the result.

## Flow B — Okta-Protected OpenAPI Target

1. External customer authenticates with Okta through the existing application flow.
2. The application invokes the agent with the customer identity context.
3. Agent calls the approved AgentCore Gateway tool.
4. AgentCore Gateway authenticates/authorizes the tool invocation.
5. When user-context downstream authorization is required, AgentCore uses OAuth token exchange/OBO to obtain a downstream token appropriate for the protected resource.
6. Gateway invokes the Okta Access Gateway-facing endpoint.
7. Okta Access Gateway performs the configured customer entitlement/authorization check.
8. Authorized requests are forwarded to Spring Boot.
9. Response returns through AgentCore Gateway to the agent.

## Flow C — API Gateway Target

1. Agent calls AgentCore Gateway.
2. AgentCore validates inbound identity/policy.
3. Gateway maps the tool to the configured API Gateway REST API stage operation.
4. AgentCore invokes API Gateway using the configured target authorization.
5. API Gateway applies API policy/throttling/routing.
6. API Gateway private integration/VPC Link reaches the application load balancer.
7. The request reaches Spring Boot on EKS.
8. Response returns through API Gateway and AgentCore Gateway to the agent.

---

# 10. Security and Governance Recommendations

1. **Keep the runtime data plane out of the Control Plane account.** Use the Control Plane for provisioning, policy, metadata, and lifecycle orchestration.
2. **Place AgentCore Gateway close to the target trust boundary.** In the proposed model, Gateway and targets live in the application/BU account.
3. **Expose only approved REST operations.** Do not automatically convert the entire Spring Boot API surface into tools.
4. **Curate tool contracts.** Use stable `operationId` values, concise descriptions, bounded parameter schemas, and agent-safe error responses.
5. **Use separate targets for separate trust models.** Direct internal APIs and Okta-entitled customer APIs should not share credentials/policies merely for convenience.
6. **Apply least privilege to the AgentCore Gateway execution role.** Scope target access to only required resources.
7. **Keep protected APIs private where practical.** Use AgentCore private endpoints/VPC Lattice for MCP/OpenAPI targets rather than exposing EKS publicly.
8. **Avoid an entitlement bypass.** If Okta Access Gateway is the required authorization point, the backend must not expose an equivalent unprotected route.
9. **Centralize observability.** Correlate agent invocation ID, gateway/tool name, target request ID, application trace ID, and user/security principal where permitted.
10. **Version the OpenAPI contract.** Treat the agent-facing OpenAPI schema as a controlled artifact promoted through environments alongside target configuration.

---

# 11. Implementation Guidance

For the first Spring Boot integration with approximately ten REST endpoints:

1. Generate or curate an OpenAPI 3.x schema from Spring Boot.
2. Select only the operations that are safe and meaningful as agent tools.
3. Assign stable, agent-friendly `operationId` values.
4. Validate request/response schemas against AgentCore Gateway OpenAPI limitations.
5. Create the AgentCore Gateway and target in the application/BU account.
6. For a private Spring endpoint, configure the target `privateEndpoint` using the application VPC, subnets, and security group.
7. Configure outbound authorization appropriate to the endpoint.
8. Create a separate OpenAPI target for the Okta Access Gateway protected route where customer entitlement is required.
9. Test `tools/list`, individual tool invocation, denied-user/denied-agent cases, and backend authorization failures.
10. Promote the OpenAPI schema and target configuration through non-production and production using the Control Plane's cross-account deployment workflow.

---

# 12. Decision

**Primary pattern:** Existing Spring REST → AgentCore Gateway OpenAPI Target → private EKS service.  
**Customer entitlement variant:** Existing Spring REST → AgentCore OpenAPI Target → Okta Access Gateway → Spring Boot.  
**API governance variant:** AgentCore Gateway → API Gateway REST API target → VPC Link → EKS.  
**MCP-native exception:** Spring Boot as MCP Server only when MCP-native behavior or higher-level tool composition is intentionally required.

---

# 13. AWS / Okta References

- Amazon Bedrock AgentCore — OpenAPI schema targets: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-schema-openapi.html
- Amazon Bedrock AgentCore — MCP server targets: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-MCPservers.html
- Amazon Bedrock AgentCore — API Gateway REST API stages as targets: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-api-gateway.html
- Amazon Bedrock AgentCore — Gateway VPC egress: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-vpc-egress.html
- Amazon Bedrock AgentCore — Private resources with VPC Lattice: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/vpc-egress-private-endpoints.html
- Amazon Bedrock AgentCore — Gateway outbound authorization: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-outbound-auth.html
- Okta Access Gateway — Application integration overview: https://help.okta.com/oag/en-us/content/topics/access-gateway/about-application-integration.htm
- Okta Access Gateway — Header application integration / reverse proxy model: https://help.okta.com/oag/en-us/content/topics/access-gateway/integrate-app-header.htm
