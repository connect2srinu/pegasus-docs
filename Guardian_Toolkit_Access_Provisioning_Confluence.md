# Guardian Control Plane – Toolkit / AgentCore Gateway Access Provisioning

## Purpose

This document describes the current Guardian identity and access model and how the same enterprise entitlement model can be used to protect **AgentCore gateways, toolkits, gateway targets, and agents** managed through the Guardian Control Plane.

The proposed model separates three responsibilities:

- **Enterprise entitlement provisioning** — SailPoint and Active Directory
- **Authentication and identity claims** — Microsoft Entra
- **Resource-level authorization** — Guardian Control Plane

The companion draw.io file contains four swimlane-based visual flows aligned with this document.

---

# 1. Current System Connectivity

The current enterprise access chain is:

**SailPoint Role / Entitlement → Active Directory Group → Microsoft Entra Group → Entra Enterprise Application**

A SailPoint role is associated with an Active Directory security group. The AD group is synchronized to the corresponding Microsoft Entra group. That Entra group participates in the authentication and authorization model used by the Guardian AI / Control Plane Enterprise Application.

## Multiple groups can use one shared Entra Enterprise App

The scalable model is to avoid creating a separate Enterprise Application for every toolkit or agent.

For example:

- SailPoint Role A → AD Group A → Entra Group A
- SailPoint Role B → AD Group B → Entra Group B
- SailPoint Role C → AD Group C → Entra Group C
- SailPoint Role D → AD Group D → Entra Group D

All of these Entra groups can be associated with the **same shared Entra Enterprise Application**.

The Guardian Control Plane then performs resource-level authorization:

- Toolkit / Gateway A requires Entra Group A
- Toolkit / Gateway B requires Entra Group B
- Toolkit / Gateway C requires Entra Group C
- Agent D requires Entra Group D

At runtime, the Control Plane evaluates the user's Entra group membership and allows or denies access to the requested resource.

## Key principle

The Enterprise App provides the common authentication boundary.

The individual Entra groups provide the **resource-level authorization boundary**.

This allows potentially hundreds of toolkits, gateways, and agents to reuse the same Enterprise App while still having independent access control.

---

# 2. Toolkit / Gateway Creation – Existing AD Group

This flow applies when the toolkit owner already has an appropriate Active Directory group that can be used to protect the resource.

## Flow

1. The user starts **Create Toolkit / Gateway** from the Guardian Control Plane.
2. The user chooses the security model **Protect using Existing AD Group**.
3. The user searches for and selects the required AD security group.
4. The Control Plane resolves the selected AD group to the corresponding Microsoft Entra group.
5. The Control Plane captures the immutable **Entra Group Object ID**.
6. If required by the selected Entra configuration, the group is associated with the shared Guardian AI / Control Plane Enterprise App.
7. The Control Plane stores the authorization mapping for the resource.
8. The toolkit / gateway is created or registered.

## Recommended authorization metadata

For each protected toolkit or gateway, the Control Plane should store:

- Toolkit / Gateway ID
- Toolkit / Gateway Name
- Security model
- AD Group Name
- AD Group Identifier, if available
- Entra Group Name
- Entra Group Object ID
- Enterprise App Identifier
- Created By
- Created Date
- Updated Date

The **Entra Group Object ID** should be the primary runtime authorization identifier rather than relying only on group display names.

---

# 3. Toolkit / Gateway Creation – New AD Group Required

This flow applies when no existing AD group is suitable for the new toolkit or gateway.

The process is split into two stages.

## Step 0 – Establish the enterprise entitlement

1. The toolkit owner starts the existing **SailPoint intake process**.
2. A new SailPoint role / entitlement is created or configured.
3. A corresponding Active Directory group is created or associated.
4. The AD group is synchronized to Microsoft Entra.
5. The corresponding Entra group becomes available.

This keeps new entitlement creation within the existing Guardian governance process rather than bypassing SailPoint.

## Step 1 – Create the toolkit in Control Plane

After the enterprise entitlement is available:

1. The user returns to the Guardian Control Plane.
2. The user starts toolkit / gateway creation.
3. The newly created AD group is selected.
4. The Control Plane resolves it to the matching Entra group.
5. The Entra Group Object ID is captured.
6. The Control Plane stores the resource-to-group authorization mapping.
7. The toolkit / gateway is created and protected by that group.

---

# 4. End-User Access Request Flow

Creating and protecting a toolkit is separate from giving an individual user access to that toolkit.

A user who needs access must receive the corresponding **SailPoint role assignment**.

## Access request sequence

1. The user identifies the toolkit / gateway they need to access.
2. The user submits a SailPoint request for the SailPoint role associated with that resource.
3. The request passes through **Approval Step 1**.
4. The request passes through **Approval Step 2**.
5. After both approvals complete, the SailPoint role is assigned.
6. The user is provisioned into the corresponding **Active Directory security group**.
7. As part of the existing enterprise synchronization cycle, the AD group membership is reflected in the corresponding **Microsoft Entra group**.
8. The current expected synchronization window is approximately **10 minutes**.
9. The user signs in again or refreshes authentication.
10. Entra provides the user's relevant group membership through the OIDC identity flow.
11. The Guardian Control Plane performs the runtime authorization check.
12. If the required Entra Group Object ID is present, access is allowed.

## Runtime behavior when the SailPoint role is not assigned

The SailPoint role assignment is the governing entitlement.

Without that role assignment:

- the user is not provisioned into the required AD group;
- the required membership does not propagate to the corresponding Entra group;
- the Control Plane does not see the required group during runtime authorization;
- access to the protected toolkit / gateway is denied; and
- the user receives a **security / access exception**.

Conceptually:

**SailPoint Role Assigned → AD Group Membership → Entra Group Membership → Runtime Access Allowed**

versus:

**No SailPoint Role Assignment → Required Entra Group Missing → Runtime Authorization Failure → Security / Access Exception**

---

# 5. Separation of Responsibilities

## SailPoint

SailPoint remains responsible for:

- entitlement / role requests;
- approval workflow;
- role assignment;
- enterprise access governance.

## Active Directory

Active Directory remains responsible for:

- enterprise security groups;
- user-to-group membership resulting from approved entitlement assignments.

## Microsoft Entra

Microsoft Entra remains responsible for:

- authentication;
- synchronized enterprise group membership;
- Enterprise App integration;
- OIDC identity / group claims.

## Guardian Control Plane

The Control Plane is responsible for:

- letting toolkit owners select the security group protecting a resource;
- resolving AD-to-Entra mappings;
- storing resource-to-Entra-group authorization metadata;
- evaluating the user's runtime group membership;
- allowing or denying access to the toolkit, gateway, or agent.

---

# 6. Design Principles

1. **Do not bypass SailPoint for user entitlement provisioning.**  
   SailPoint remains the enterprise governance path for role assignment.

2. **Reuse one shared Entra Enterprise App where practical.**  
   Multiple Entra groups can be associated with the same application.

3. **Authorize at the resource level using Entra group membership.**  
   Each toolkit, gateway, or agent can require a different group.

4. **Store immutable identifiers.**  
   The Control Plane should store the Entra Group Object ID rather than relying only on display names.

5. **Separate resource creation from user access assignment.**  
   Toolkit creation establishes which group protects the resource. SailPoint determines which users receive that group.

6. **Fail closed at runtime.**  
   If the required group is not present in the user's identity, the Control Plane should deny access.

---

# 7. Open Design Questions

The following points should be validated during detailed design:

1. How will the Control Plane resolve the AD group to its corresponding Entra group?
2. Is explicit association of each Entra group with the shared Enterprise App required for the selected Entra group-claim configuration?
3. What happens when the AD group exists but the Entra synchronization has not completed yet?
4. Should the Control Plane expose the estimated 10-minute synchronization delay to the user?
5. Should the runtime error distinguish between:
   - no entitlement;
   - entitlement approved but synchronization pending;
   - invalid / deleted group;
   - authorization configuration error?
6. Should toolkit creation validate that the selected group is an approved enterprise-managed group?
7. Should agents and AgentCore gateways use the same authorization metadata model, or separate resource types backed by a common authorization policy?

---

# 8. Summary

The proposed architecture uses the existing Guardian enterprise identity governance model rather than creating a parallel access-management process.

- **SailPoint** governs who is entitled to access.
- **Active Directory** holds the enterprise group membership.
- **Microsoft Entra** exposes the synchronized identity and group membership.
- **Guardian Control Plane** maps individual toolkits, gateways, and agents to required Entra groups and performs the runtime authorization check.

This design supports a scalable model in which many protected AI resources can reuse one shared Entra Enterprise App while maintaining independent, enterprise-governed access control.
