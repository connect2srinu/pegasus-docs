I want to extend our existing design-folder-driven SDLC so that it works consistently
for both the Control Plane API repo and Control Plane UI repo.

Workspace structure:

<master-folder>/
  <api-repo>/
  <ui-repo>/

Both repositories already have:
- AGENTS.md or equivalent repo instructions
- .claude/skills/
- our six-stage SDLC skills:
  - design-start-feature
  - design-session
  - design-check-controls
  - design-plan-implementation
  - design-record-decision
  - design-open-pr
  - design-close-feature
  - design-ingest-control

Important:
Do not replace or redesign the existing SDLC.
Extend it so that features can be correctly handled as:

1. API-only
2. UI-only
3. Cross-repository

Please first inspect BOTH repositories, their AGENTS.md files, .claude/skills,
design folder conventions, hooks, templates, and any generated-file rules.

Before modifying anything, show me:
1. what exists today,
2. what files you propose changing,
3. what files you propose adding,
4. how the new behavior will integrate with the existing SDLC,
5. whether any existing skill behavior could conflict with this approach.

Do not make changes until I approve the proposal.

Target behavior
===============

A. Feature scope classification

At the beginning of design-start-feature, every feature must be classified as:

- API-only
- UI-only
- Cross-repository

The classification must be based on the actual behavior required by the feature,
not just the repository from which the command was started.

Examples:

UI-only:
- presentation/layout/client-side behavior changes
- no backend contract, persistence, authorization, or server behavior changes

API-only:
- backend-only behavior
- no UI behavior or UI-consumed contract change

Cross-repository:
- both UI and API change
- API request/response contract changes that require UI changes
- backend behavior plus corresponding user-facing workflow
- shared lifecycle/state behavior spanning both repos

B. Intent ownership

For API-only features:
- intent.md lives in the API repository design folder.
- API repo follows the normal six-stage SDLC.

For UI-only features:
- intent.md lives in the UI repository design folder.
- UI repo follows the normal six-stage SDLC.

For Cross-repository features:
- there must be ONE authoritative feature intent for the overall capability.
- both repositories use the same stable Feature-ID.
- each repository owns its own repo-specific design and implementation artifacts.
- do not create two independent copies of the feature intent that can drift.

If the current master workspace provides an appropriate location for shared feature
intent, propose a clean structure for it.

If storing the authoritative intent outside each git repo creates operational
problems, explain the tradeoffs and propose an alternative that still avoids
independently editable duplicate intents.

C. Cross-repository design

For a cross-repository feature, structure the work so that:

Shared feature level:
- intent.md
- optional contract.md when UI/API interaction is involved

API repo:
- repo-specific design.md
- controls-check.md
- implementation plan
- API branch and PR

UI repo:
- repo-specific design.md
- controls-check.md
- implementation plan
- UI branch and PR

Both repositories must use the same Feature-ID.

Example:

Feature-ID: create-agent
Feature-Scope: cross-repository

API branch:
feature/create-agent

UI branch:
feature/create-agent

Each repo still follows:
Intent -> Design -> Controls -> Build -> Review -> Close

The cross-repository feature should not create a second SDLC.
It should coordinate the existing repo SDLCs.

D. Standalone repository behavior

This is critical.

Developers may clone ONLY the API repo or ONLY the UI repo.

Therefore each repository must independently know that the peer repository exists
and that some features may be cross-repository.

Do not make the master workspace the only place where these rules exist.

Add concise mandatory repository guidance so that:

If Claude is running only inside the API repo and determines a feature is cross-repo:
- identify the feature as cross-repository
- state that the UI repo is also affected
- use/require the same Feature-ID
- do not invent or modify UI implementation artifacts
- do not claim the end-to-end feature is complete
- proceed with API-owned work only after the shared intent has been established or
  explicitly approved according to the process

Mirror the same behavior in the UI repo.

The API repo should know its peer repo is the Control Plane UI repo.
The UI repo should know its peer repo is the Control Plane API repo.

E. Scope-change guardrail

If a feature begins as UI-only or API-only and implementation/design later reveals
that the other repository is required:

DO NOT silently start changing the other repo.

Instead:
- flag "Feature Scope Change"
- explain what dependency was discovered
- recommend updating the approved intent
- change Feature-Scope to cross-repository only after the appropriate approval
- then coordinate the peer repo using the same Feature-ID

Example:

Original scope: UI-only

Discovered:
The UI requires a new deploymentFailureReason field that the API does not expose.

Expected behavior:
Stop and report that the feature should become cross-repository before changing
the API.

F. Repository instructions

Add a concise rule to each repository AGENTS.md (or the authoritative equivalent)
that explains:

- features may be UI-only, API-only, or cross-repository
- feature scope must be classified before design begins
- cross-repository work uses a common Feature-ID
- the peer repository may be absent from the local checkout
- do not silently implement changes in another repository
- do not mark a cross-repo feature complete based on one repository alone
- refer to the detailed cross-repo development instructions

Do not overload AGENTS.md with the entire process.
Keep detailed documentation in an appropriate docs/development location.

G. Shared skill behavior

Update the appropriate existing Claude skill, preferably design-start-feature,
rather than creating unnecessary parallel skills.

The skill should:

1. Read repo instructions.
2. Inspect the requested feature.
3. Classify the feature:
   - API-only
   - UI-only
   - Cross-repository
4. Show the classification and rationale.
5. Identify affected repositories.
6. Determine where the authoritative intent belongs.
7. Generate/use a stable Feature-ID.
8. Verify whether the peer repo is available.
9. For cross-repo features, locate or require the shared intent.
10. Continue with the existing Intent-stage behavior:
    - one-page intent
    - approval
    - feature branch
    - draft PR

Do not remove any existing hard gates.

H. Intent template

Update the feature intent template so it includes a small section such as:

## Feature Scope

Feature-ID: <stable-id>
Feature-Scope: API-only | UI-only | Cross-repository

Affected repositories:
- ...

Unaffected repositories:
- ...

Reason:
...

For cross-repo features also include:

Peer repositories:
- ...

Shared contract required:
Yes | No

Do not put detailed implementation design in intent.md.

I. Contract artifact

For cross-repository UI/API features, support an optional shared contract.md.

The contract may contain:
- endpoint/request/response expectations
- lifecycle states
- validation behavior
- error semantics
- fields required by the UI
- ownership boundaries

contract.md must not become a replacement for API design.md or UI design.md.

J. PR traceability

For cross-repository work, ensure both repo PRs can be correlated using:

Feature-ID: <id>
Feature-Scope: Cross-repository

Where practical, add a place for:
Peer PR: <reference>

Do not hard-code GitHub-specific behavior unless the current process already assumes GitHub.

K. Compatibility

Preserve all current rules, including:
- one feature branch / one PR per affected repository
- draft PR starts during Intent
- merge, not rebase, while the PR is open
- generated files are not hand-edited
- required feature/ADR commit footer behavior
- design.md is not produced while design-shaping questions remain unanswered
- ADRs are not created unless explicitly elected
- controls remain part of the existing controls process
- closeout remains the final commit before squash merge

For cross-repository features, interpret:
"one feature = one branch = one PR"

as:

"one feature uses one common Feature-ID, with one branch and one PR in each affected repository."

L. Copilot compatibility

Although these workflows are implemented as Claude skills, the durable development
rules must live in repo-controlled documentation/instructions so developers using
Copilot can follow the same process.

Do not make correctness depend exclusively on Claude.

Deliverables
============

After inspection, propose the exact changes.

I expect something similar to:

API repo:
- AGENTS.md update
- docs/development/cross-repo-features.md
- update to .claude/skills/design-start-feature/
- any necessary template changes

UI repo:
- equivalent changes

Master workspace:
- only add shared workspace guidance if it adds value
- do not make the parent workspace mandatory for standalone repo development

Also tell me which files should be identical across both repos and which should
be repo-specific.

Important execution rule
========================

Do this in two phases.

PHASE 1 — ANALYZE AND PROPOSE ONLY
Do not modify files.

Show:
- current-state assessment
- proposed target structure
- proposed file changes
- proposed skill behavior
- migration impact
- risks/questions

Wait for my approval.

PHASE 2 — IMPLEMENT
After I approve, make the changes in both repos and show me the diff/summary.
