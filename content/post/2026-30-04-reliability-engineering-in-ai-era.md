---
layout:     post

title:      "If an AI Agent Can Delete Your Production Database, That's on You"
subtitle:   "Reliability Engineering in the AI Era"
author: Jamie Craane
date: 2026-04-30
description: "Defense in Depth practices for running autonomous AI agents safely — from least-privilege credentials and environment segregation to soft deletes, PITR, and RPO/RTO targets."
image: ""
published: true
showtoc: false
tags:
- AI
- Reliability
- DevOps
- Security
- AWS

categories: [ Reliability ]
URL: "/2026/30/04/reliability-engineering-in-ai-era/"
---

We've all seen the viral stories circulating on X recently: an AI coding agent runs amok and deletes a production database. Immediately, the blame game begins: users point fingers at the agent framework, at coding assistants like Cursor, or at cloud platforms like Railway.

My take? The operator carries the responsibility. Tool vendors absolutely should ship safer defaults — scoped tokens, confirmation prompts on destructive operations, dry-run modes — and I'd happily criticize them for not doing so. But even with perfect vendor defaults, the buck stops with whoever wired the agent into production. Cursor and Railway are stand-ins here for *any* autonomous agent with tool access; the same arguments apply to whatever framework you're running.

Blaming an AI agent for dropping a production database is like blaming your mechanical keyboard when you accidentally type `rm -rf /`. If an AI agent *can* delete your production database, your operational guardrails have already failed. As AI agents become more autonomous, the "blast radius" of their mistakes increases exponentially.

Here is a non-exhaustive list of the engineering practices required to prevent these disasters, organized by a **Defense in Depth** strategy.

---

## Layer 1: The Perimeter (Prevention)

### 1. Stop Storing Prod Secrets in Local `.env` Files
AI coding assistants ingest your local workspace to provide context. If your production `.env` file is sitting locally, the LLM is reading it — and, worse, it may pass those credentials into a tool call. The risk isn't just exfiltration; it's that the agent now has a working production token to *act* with.
*   **Never store production secrets locally**.
*   Use secret managers (like AWS Secrets Manager or HashiCorp Vault) to inject secrets only at runtime within the secure production environment.
*   Local environments should only ever contain local or staging mock secrets.

### 2. Enforce Strict Environment Segregation
AI agents should operate entirely in sandbox or ephemeral development environments.
*   Production and Development should be completely segregated, ideally in different cloud accounts.
*   IAM-only separation within a single account isn't enough. A leaked admin token can reach anything routable from that account, whereas a token issued in a separate account can't reach what it can't route to. Make the *network and account boundary* the primary control, not just IAM policy.
*   An agent running locally should physically lack the network routing or credential access required to reach production infrastructure.

### 3. Treat Every Agent Input as Untrusted
Most failure modes discussed in posts like this assume the agent is *mistaken*. The more dangerous case is when it's *adversarially controlled*. An attacker plants instructions in a GitHub issue, a Jira ticket, a vendor doc, or a file the agent crawls — and the agent dutifully executes them with whatever credentials it has. This is prompt injection, and it is the highest-leverage AI-specific risk in this entire list.
*   Treat any text the agent ingests from an external source as potential instructions, not just data.
*   Scope tool access *per task*, not per session. A code-review agent shouldn't have shell or deploy access just because it might be useful later.
*   Assume your guardrails will be probed by something that actively wants them to fail, not just by an agent that occasionally hallucinates.

---

## Layer 2: The Process (Authorization & Visibility)

### 4. Apply the Principle of Least Privilege (WAF Security Pillar)
Following the **Security Pillar** of the **AWS Well-Architected Framework**, access should be granted only for the specific task at hand.
*   Application-level tokens used by an agent should have **DML rights** (e.g., `SELECT`, `UPDATE`) but **never DDL rights** (e.g., `DROP`, `ALTER`).
*   Administrative credentials must be vaulted and used exclusively by CI/CD pipelines or DBAs.
*   The **[OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)** explicitly calls out over-privileged agents (Identity and Privilege Abuse) as a leading cause of catastrophic failures — treat agent identities as first-class principals in your IAM model.

### 5. Require a "Human-in-the-Loop" for Infrastructure
Agents should not execute infrastructure or schema changes directly via API or CLI.
*   Adopt a **GitOps** approach where all changes are managed via Infrastructure as Code (IaC).
*   If an agent proposes a change, it should open a Pull Request.
*   A human must explicitly approve the "plan" before any destructive action is applied to production.

### 6. Implement Audit Logging and Concrete Anomaly Alerting
You must have visibility into what your agents are doing — and the alerting needs to be specific enough to act on. "Set up alerts for suspicious activity" is the kind of advice that's true and useless.
*   Enable comprehensive audit logs (like AWS CloudTrail) to record exactly which token or user executed a command.
*   Define concrete alerting rules. Examples: any `Delete*`, `Drop*`, or `Terminate*` API call issued by a non-CI principal; any destructive call from an agent identity outside a defined maintenance window; any IAM policy change touching production roles.
*   The goal is to react before the damage is total — which means alerts have to fire on *the first suspicious call*, not the hundredth.

---

## Layer 3: The Safety Net (Resilience)

### 7. Implement Termination Protection and Soft Deletes
You need layers of defense that prevent permanent deletion even when a destructive command is authorized.
*   Enable **termination protection** on all critical production resources.
*   Ensure your cloud provider utilizes "soft deletes" (a recycle bin), where deleted resources sit in a suspended state for a mandatory retention period before permanent purging.

### 8. Establish RPO and RTO Targets (WAF Reliability Pillar)
Meeting the **Reliability Pillar** of the Well-Architected Framework means knowing exactly how you will recover when things go wrong.
*   **Recovery Point Objective (RPO):** how much data are you willing to lose?
*   **Recovery Time Objective (RTO):** how long can the system be down during a restore?
*   RPO and RTO are *targets*; PITR is one *mechanism*. **Point-in-Time Recovery (PITR)** lets you restore to a specific second and is how most teams meet their RPO. But be honest about RTO: restoring a multi-terabyte database from a snapshot can take hours. If your RTO is measured in minutes, you'll need warm standbys, read replicas promotable to primary, or multi-region failover — PITR alone won't get you there.

---

## Conclusion
This is not an exhaustive list, and treat it as a maturity ladder rather than a checkbox. A two-person team operating from a single staging environment isn't going to set up multi-region failover and probably shouldn't, but they should still get the secrets, least-privilege, and human-in-the-loop layers right. Depending on your scale, you may also need drift detection, chaos engineering, or multi-region failover.

AI tools like Cursor are massive multipliers for productivity. But with great power comes a massive blast radius. Don't blame the AI when it does exactly what your system allowed it to do. Build fail-safe systems, enforce zero-trust policies, and treat your autonomous agents with the same strict security posture you would apply to any human engineer — knowing that, unlike a human, the agent will move faster than you can react.

## References & Further Reading

- [AWS Well-Architected Framework: Reliability Pillar — Plan for Disaster Recovery](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/plan-for-disaster-recovery-dr.html)
- [AWS Well-Architected Framework: Security Pillar — Grant Least Privilege (SEC03-BP02)](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/sec_permissions_least_privileges.html)
- [OWASP Top 10 for Agentic Applications (2026)](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
- [Establishing RPO and RTO Targets for Cloud Applications — AWS Cloud Operations Blog](https://aws.amazon.com/blogs/mt/establishing-rpo-and-rto-targets-for-cloud-applications/)
