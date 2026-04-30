---
layout:     post

title:      "Reliability Engineering in the AI Era"
subtitle:   ""
author: Jamie Craane
date: 2025-01-15
description: "This post, we will look at how to integrate your Firebase users in your own application database."
image: "/img/part1-firebase-ktor.png"
published: true
showtoc: false
tags:
- Firebase
- Kotlin
- Ktor

categories: [ Kotlin ]
URL: "/2026/30/04/reliability-engineering-in-ai-era/"
---

This revised draft adopts the **Defense in Depth** structure, incorporates the **AWS Well-Architected Framework**, and maintains a firm but fair tone. It emphasizes that while tools are powerful, the responsibility for safety rests solely with the engineer.

---

# If an AI Agent Can Delete Your Production Database, That’s on You

We’ve all seen the viral stories circulating on X recently: an AI coding agent runs amok and deletes a production database[cite: 1]. Immediately, the blame game begins: users point fingers at the agent framework, at coding assistants like Cursor, or at cloud platforms like Railway[cite: 1].

My take? Neither the agent, Cursor, nor Railway are to blame[cite: 1].

Blaming an AI agent for dropping a production database is like blaming your mechanical keyboard when you accidentally type `rm -rf /`[cite: 1]. If an AI agent *can* delete your production database, your operational guardrails have already failed[cite: 1]. As AI agents become more autonomous, the "blast radius" of their mistakes increases exponentially[cite: 1]. An AI agent is essentially a highly capable, incredibly fast junior developer—and you wouldn't hand a junior developer direct CLI access to production on their first day[cite: 1].

Here is a non-exhaustive list of the rigid engineering practices required to prevent these disasters, organized by a **Defense in Depth** strategy.

---

## Layer 1: The Perimeter (Prevention)

### 1. Stop Storing Prod Secrets in Local `.env` Files
AI coding assistants ingest your local workspace to provide context[cite: 1]. If your production `.env` file is sitting locally, the LLM is reading it[cite: 1].
*   **Never store production secrets locally**[cite: 1].
*   Use secret managers (like AWS Secrets Manager or HashiCorp Vault) to inject secrets only at runtime within the secure production environment[cite: 1].
*   Local environments should only ever contain local or staging mock secrets[cite: 1].

### 2. Enforce Strict Environment Segregation
AI agents should operate entirely in sandbox or ephemeral development environments[cite: 1].
*   Production and Development should be completely segregated, ideally in different cloud accounts or VPCs[cite: 1].
*   An agent running locally should physically lack the network routing or credential access required to reach production infrastructure[cite: 1].

---

## Layer 2: The Process (Authorization & Visibility)

### 3. Apply the Principle of Least Privilege (WAF Security Pillar)
Following the **Security Pillar** of the **AWS Well-Architected Framework**, access should be granted only for the specific task at hand[cite: 1].
*   Application-level tokens used by an agent should have **DML rights** (e.g., `SELECT`, `UPDATE`) but **never DDL rights** (e.g., `DROP`, `ALTER`)[cite: 1].
*   Administrative credentials must be vaulted and used exclusively by CI/CD pipelines or DBAs[cite: 1].

### 4. Require a "Human-in-the-Loop" for Infrastructure
Agents should not execute infrastructure or schema changes directly via API or CLI[cite: 1].
*   Adopt a **GitOps** approach where all changes are managed via Infrastructure as Code (IaC)[cite: 1].
*   If an agent proposes a change, it should open a Pull Request[cite: 1].
*   A human must explicitly approve the "plan" before any destructive action is applied to production[cite: 1].

### 5. Implement Audit Logging and Anomaly Alerting
You must have visibility into what your agents are doing[cite: 1].
*   Enable comprehensive audit logs (like AWS CloudTrail) to record exactly which token or user executed a command[cite: 1].
*   Set up high-priority alerts for any API calls that attempt to modify or delete critical infrastructure so you can react before the damage is total[cite: 1].

---

## Layer 3: The Safety Net (Resilience)

### 6. Implement Termination Protection and Soft Deletes
You need layers of defense that prevent permanent deletion even when a destructive command is authorized[cite: 1].
*   Enable **termination protection** on all critical production resources[cite: 1].
*   Ensure your cloud provider utilizes "soft deletes" (a recycle bin), where deleted resources sit in a suspended state for a mandatory retention period before permanent purging[cite: 1].

### 7. Establish RPO and RTO Targets (WAF Reliability Pillar)
Meeting the **Reliability Pillar** of the Well-Architected Framework means knowing exactly how you will recover when things go wrong[cite: 1].
*   **Recovery Point Objective (RPO):** How much data are you willing to lose?[cite: 1].
*   **Recovery Time Objective (RTO):** How long can the system be down during a restore?[cite: 1].
*   Use **Point-in-Time Recovery (PITR)** to allow restores to a specific second, ensuring you can "rewind" the mistake[cite: 1].

---

## Conclusion
This is not an exhaustive list—depending on your scale, you may also need drift detection or multi-region failover—but it represents the baseline of a professional production environment[cite: 1].

AI tools like Cursor are massive multipliers for productivity[cite: 1]. But with great power comes a massive blast radius. Don't blame the AI when it does exactly what your system allowed it to do[cite: 1]. Build fail-safe systems, enforce zero-trust policies, and treat your autonomous agents with the same strict security posture you would apply to any human engineer[cite: 1].

## References & Further Reading

- AWS Well-Architected Framework: Reliability Pillar - Plan for Disaster Recovery
- AWS Well-Architected Framework: Security Pillar - Grant Least Privilege
- OWASP Agentic Skills Top 10: Addressing Over-Privileged AI Agents
- RPO vs. RTO: What They Mean and How To Set Targets
