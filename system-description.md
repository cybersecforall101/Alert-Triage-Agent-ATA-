# System description: AI Alert-Triage Agent (fictional)

> **Disclaimer.** This is a fictional system built for a learning and threat-modeling exercise. It is not based on any real product, employer, customer, incident or internal material. Everything here is generalized.

---

## 1. Purpose and scope

The **Alert-Triage Agent (ATA)** is an LLM-based assistant that helps a SOC handle the first stage of alert triage in a fictional mid-sized organization ("Example Corp").

**What it does**
1. Receives alerts from the SIEM and other detection sources.
2. Gathers context by querying logs, asset data and threat intelligence.
3. Writes a short investigation summary and a verdict with a confidence level.
4. Recommends next actions and, depending on its autonomy tier, creates tickets or executes limited response actions.

**What it does not do**
- It does not write or change detection rules.
- It does not communicate with end users or external parties.
- It does not have access to production systems other than through the tools listed in section 4.

**Why this system?** Alert triage is a common first use of AI in SOCs. It combines untrusted input, tool access and decisions with real consequences, which makes it a good subject for threat modeling.

## 2. Operating context (fictional)

- Example Corp: ~3,000 employees, hybrid cloud, 24/7 SOC with L1, L2 and an on-call incident lead.
- Detection sources: SIEM (Splunk or ELK style), EDR, email security gateway, cloud audit logs, identity provider logs.
- Typical volume: ~1,500 alerts per day, a large share of them low-value or false positives.
- Goal of the deployment: reduce analyst time spent on repetitive triage without lowering detection quality.

## 3. Components

| # | Component | Role | Trust notes |
|---|-----------|------|-------------|
| C1 | Detection sources and SIEM | Produce alerts and hold the raw logs | Alert fields and logs contain attacker-influenced content |
| C2 | Alert queue / orchestrator | Passes each alert to the agent and records the result | Trusted infrastructure |
| C3 | Agent core (LLM + system prompt + policy layer) | Reasons over the alert and decides which tools to call | The LLM cannot reliably separate instructions from data |
| C4 | Tool layer | Executes tool calls on the agent's behalf | Enforces permissions (should not depend on the LLM) |
| C5 | Threat-intel services | Return reputation and context for indicators | External, partially untrusted |
| C6 | Asset and identity data (CMDB, directory) | Return owner, criticality, user role | Internal, may be stale or wrong |
| C7 | Ticketing / SOAR | Stores cases, runs approved response actions | Trusted, but actions have real impact |
| C8 | Analyst console | Shows summaries and verdicts, collects approvals | Human decision point |
| C9 | Audit log | Records inputs, tool calls, outputs and approvals | Must be tamper-resistant |
| C10 | Model hosting | Where the LLM runs | External hosted LLM API: prompts and alert data leave the organization's network |

## 4. Tools and permissions

| Tool | Description | Access type |
|------|-------------|-------------|
| `siem_search` | Run bounded read-only queries (time window and result limits enforced) | Read |
| `get_alert_details` | Fetch alert metadata and raw event fields | Read |
| `ti_lookup` | Look up IPs, domains and file hashes in threat-intel sources | Read (external) |
| `asset_lookup` | Get host owner, criticality, OS, environment | Read |
| `user_lookup` | Get user role, department, recent sign-in context | Read |
| `ticket_create` / `ticket_update` | Create a case and add notes | Write (low impact) |
| `alert_close` | Close an alert with a reason | Write (medium impact) |
| `notify_oncall` | Page the on-call analyst | Write (low impact) |
| `soar_isolate_host` | Isolate an endpoint from the network | Write (high impact) |
| `soar_disable_account` | Disable a user account | Write (high impact) |

## 5. Autonomy tiers

The agent operates at one tier at a time. Moving up a tier requires evidence of safe and reliable behavior at the current tier.

| Tier | Name | What the agent may do | Human role |
|------|------|-----------------------|------------|
| 0 | Observe | Read tools only. Produces a summary and verdict, shown to analysts but not stored as the official triage result. | Analyst triages as usual and compares |
| 1 | Recommend | Read tools plus `ticket_create`, `ticket_update`, `notify_oncall`. Recommends closure or response actions. | Analyst makes every decision |
| 2 | Act with approval | Tier 1, plus `alert_close` and high-impact SOAR actions, **only after explicit analyst approval** in the console | Analyst approves each action |
| 3 | Act within an allow-list | Tier 2, plus autonomous `alert_close` for a pre-approved list of benign alert types, with sampling-based review | Analyst reviews a sample and exceptions |

**Assumed deployment:** Tier 1, with Tier 2 as the target.

High-impact actions (`soar_isolate_host`, `soar_disable_account`) are never available at Tier 0 or 1, and never autonomous at any tier.

## 6. Inputs and data

**Inputs the agent processes**
- Alert name, severity, timestamps, source and destination fields.
- Raw event fields: process command lines, file names and paths, URLs, user agents, email subjects and sender names, DNS names, usernames.
- Free text from tickets and analyst comments.
- Responses from tools, including threat-intel text.

**Key assumption:** many of these fields are controlled or influenced by an attacker (for example an email subject, a filename, a command line or a web page title). The agent must treat all of them as **untrusted data**, never as instructions.

**Sensitive data the agent can see**
- Usernames, hostnames, internal IP ranges, email metadata.
- Possibly snippets of email or document content inside alerts.
- No credentials, no customer payment data, no raw mailbox access (by design).

## 7. State and memory

- The agent keeps no long-term memory between alerts.
- It sees the case history for the alert it is working on.
- It also has read-only access to similar past cases (closed cases reviewed by analysts) for context. This improves accuracy but creates a path for poisoned or outdated history.

## 8. Design assumptions

- **Model hosting:** an external hosted LLM API is used, so alert data and prompts leave the organization's network. Data exposure and third-party dependency are therefore in scope.
- **Tool set:** the full tool set in section 4 is available, including high-impact SOAR actions, so that excessive agency and approval gates can be examined.
- **Memory:** similar past cases are readable (section 7), so poisoned or stale history is in scope.

## 9. Out of scope

- Attacks on the underlying SIEM, EDR or cloud platforms themselves.
- Physical security.
- Model-provider internal security and model training attacks (treated as an accepted dependency).
