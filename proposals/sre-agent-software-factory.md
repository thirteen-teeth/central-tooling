# Proposal: Internal SRE Agent and Software Factory

**Goal:** Build an internal system that investigates observability alerts and implements Jira requests, while keeping production changes and live-system tests under human control.

## Proposed architecture

**Alerts / Jira → workflow orchestrator → investigation or coding agent → evidence and pull request → human approval → controlled deployment and verification**

| Capability | Recommended tools |
|---|---|
| Alert intake | [Prometheus Alertmanager](https://github.com/prometheus/alertmanager) and Datadog webhooks feed a small internal service that authenticates, deduplicates and routes events. |
| Orchestration | [LangGraph](https://github.com/langchain-ai/langgraph) coordinates investigation, approvals and resumable workflows. |
| Investigation | Build scoped adapters for the Datadog and Prometheus APIs, Kubernetes and SSH. Give the agent access to approved runbooks and service ownership data. |
| Coding | Run [OpenHands](https://github.com/OpenHands/OpenHands) in isolated workspaces to implement fixes or Jira acceptance criteria, add tests and open pull requests. |
| Policy and audit | Use [Open Policy Agent](https://github.com/open-policy-agent/opa) for authorization decisions; retain tool calls, approvals, evidence and results in an audit store. |
| Deployment | Use existing CI and GitOps; for Kubernetes, consider [Argo Rollouts](https://github.com/argoproj/argo-rollouts) for canary deployments and metric-based rollback. |

Datadog and Jira remain external services; their integrations and the organization-specific adapters would need to be built. An approved hosted model or a self-hosted model can power the agents.

## Two workflows

**Incident response:** An alert triggers read-only queries and bounded diagnostic sessions against affected Kafka, OpenSearch or logging-agent systems. The agent produces a triage summary with evidence, likely causes and uncertainty; when appropriate, it proposes a tested fix as a pull request. Humans review and merge it.

**Feature delivery:** A Jira ticket triggers a coding workspace. The agent implements the acceptance criteria, runs automated tests and opens a pull request. After review and deployment approval, it runs an approved live-system verification plan and reports results back to Jira. Kubernetes canaries use Argo Rollouts; VM deployments use the existing deployment and rollback mechanism.

## Operating boundaries

Start with **read-only production access**. Enforce restrictions outside the LLM: short-lived, least-privilege credentials; server-side restricted SSH commands; scoped Kubernetes permissions; network isolation; and redaction. Do not grant unrestricted shell or Kubernetes exec as a supposedly read-only capability. Treat logs, tickets and command output as untrusted input.

Require explicit human approval for production deployments, state-changing commands and live tests. Bound tests by target, duration and impact; define abort thresholds and rollback before execution. Never auto-merge or let the agent bypass approval gates.

## Rollout and success

1. **Pilot:** One alert type and one service; measure triage usefulness against human investigations.
2. **Expand:** Add Jira-to-PR workflows and isolated CI tests; measure accepted PRs and escaped defects.
3. **Live verification:** Add approved canary tests, monitoring and rollback; track time to triage, false conclusions and operational impact.
