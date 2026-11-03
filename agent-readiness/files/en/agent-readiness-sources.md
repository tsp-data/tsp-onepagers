# Sources: AI Agent Readiness Advisor

All sources verified on: 3 November 2026.

This document collects verified findings from primary sources that the AI Agent Readiness Advisor relies on: Grafana Labs documentation and announcements, official GitHub repositories, OpenTelemetry specifications, original research on agent autonomy levels, the OWASP and NIST security frameworks, Anthropic guidance and the EU AI Act. It is neither an instrumentation guide nor a product comparison. It lists claims with links, so anyone can check where a claim comes from and when it was last checked.

Writing rules:

- Quotes are in the original language of the source, in quotation marks. Paraphrases have no quotation marks.
- Where a claim could not be verified from a primary source, the text says so and marks it `EVIDENCE-GAP`.
- Where two primary sources disagree, both are given and the disagreement is marked `CONFLICT`. A summary of conflicts is at the end of the document.
- Product status (GA, public preview) is valid as of the verification date. Grafana Labs ships new features every few weeks, so every entry has its own date.
- Numbers from vendor surveys are given only with the specific source and a note that they come from a vendor survey.

---

# A. Grafana Labs: products and their status

## Z-01 – Agent Observability is GA (formerly AI Observability, internally Sigil, SDK agento11y)

**What the source covers:** Two entries in the official Grafana Labs changelog ("What's new"): the product rename and the general availability announcement.

**What it says:**
- Entry released on 22 July 2026 (label "Generally Available"): "Grafana’s AI Observability (alias: Sigil) is being renamed to Grafana Agent Observability." The rename requires changes to SDK packages, custom dashboards, alerts and RBAC permissions. The old names keep working during a transition period, planned to end in late October 2026. The linked migration guide has `migrate-sigil-sdk-to-agento11y` in its URL.
- Entry released on 30 July 2026 (labels "Generally Available", "No self-managed release"): "Agent Observability is now generally available in Grafana Cloud." The product adds "a new, fifth telemetry signal: agent sessions" and describes the areas Observe, Evaluate ("score response quality continuously and alert when pass rates drop"), Guard ("enforce policies in real time"), Experiment and Optimize costs ("see exactly what each agent spends, per agent, model, and session").
- In primary sources `agento11y` appears as a technical identifier: plugin ID `grafana-agento11y-app`, tool prefix in the MCP server, SDK name in the migration guide. The product name is "Agent Observability".

**Use in the tool:** Main reference for gates G1 (traceability of agent steps) and G3 (cost per task): where Grafana shows agent conversations, tool calls, response quality and cost. Grafana Cloud only, not available in self-managed Grafana.

**Link:** https://grafana.com/whats-new/2026-07-30-say-goodbye-to-black-box-agents-with-agent-observability/ and https://grafana.com/whats-new/2026-07-22-ai-observability-is-now-agent-observability--updated-branding-and-migration-guide/

**Verified on:** 3 November 2026

---

## Z-02 – Agent Observability: what it captures and how it relates to OpenTelemetry

**What the source covers:** The documentation page "Introduction to Agent Observability" and the product landing page.

**What it says:**
- The basic unit is a generation: every call to an LLM provider with the model, input messages (system prompt, user messages, tool results), outputs (responses, tool calls), token usage and timing.
- Generations are grouped into conversations by `conversation_id`. A conversation shows a timeline with traces, token usage, cost breakdown and quality scores.
- Agent catalog: the product tracks agents by `agent_name` and effective version. The default version is derived from the system prompt, so a prompt change means a new version.
- Multi-agent: `parent_generation_ids` build a dependency graph, and "if an upstream generation fails evaluation, downstream dependents are flagged."
- Workflow steps capture non-LLM steps (routing, planning, retrieval, tool orchestration).
- "Agent Observability is built on OpenTelemetry. The SDKs emit standard `gen_ai.*` semantic convention spans and metrics alongside generation data."
- SDKs for Go, Python, TypeScript, Java and .NET; integrations for LangChain, LangGraph, OpenAI Agents, LlamaIndex, Google ADK and Vercel AI SDK. Other paths: plain OpenTelemetry ingestion, traces from Claude Platform and forwarding of coding agent sessions without an SDK.
- "OpenTelemetry ingestion is in public preview."

**CONFLICT:** The press release of 20 October 2026 (Z-17) says "Agent Observability with OpenTelemetry support (GA)". On the same verification date the documentation (both the introduction and "Get started") says that OpenTelemetry ingestion without the SDK is in public preview. For the tool: Agent Observability through its own SDK is GA; for plain OpenTelemetry ingestion the primary sources disagree.

**Use in the tool:** The map "what to measure and where to see it in Grafana": traceability (conversations, generations, tool calls), agent version (a prompt change means a new version, so the gates need another pass), cost per agent and model. Do not describe instrumentation, only link to it.

**Link:** https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/introduction/ and https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/

**Verified on:** 3 November 2026

---

## Z-03 – Agent Observability: guards (synchronous check before the model call)

**What the source covers:** The documentation pages "Protect agents" and "Set up guards".

**What it says:**
- "Guards are synchronous checks that run on the request path before an LLM call. Use them to block harmful prompts, redact sensitive data, or filter dangerous tool calls."
- Guards use the same evaluator types as online evaluation (LLM judge, regex, JSON schema, heuristic) and return an SDK decision `allow` or `deny`, or a sanitized copy of the input.
- Action when an evaluator fails: `deny` (blocks the request) or `warn` (logs it and lets it through).
- The tool filter guard blocks tool calls by name using glob patterns (examples in the documentation: `shell_exec`, `file_delete`, `Bash(*rm*)`); patterns with parentheses are matched against the call arguments as well.
- The guard overview shows the counts of matches, denials, warnings and redactions over the last 24 hours, plus a list of denied or warned conversations.
- The guards documentation does not mention human approval. Guards are an automated check with no human in the loop.

**Use in the tool:** Example of limits enforced at the level of a single tool call. For U3 (agent acts on its own within limits) it shows that limits can be enforced and measured (denial rate). Guards do not replace human approval.

**Link:** https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/protect-agents/ and https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/protect-agents/guards/

**Verified on:** 3 November 2026

---

## Z-04 – Agent Observability: quality evaluation (online, offline, human review)

**What the source covers:** The documentation page "Evaluate and improve" and the product introduction.

**What it says:**
- Online evaluation scores production traffic continuously. Four evaluator types: LLM judge, JSON schema, regex, heuristic.
- "You can also create Grafana alert rules directly from evaluation rules to be notified when pass rates drop below a threshold."
- Offline experiments compare models, prompts and agent versions against a versioned test set; production conversations can be used as test cases.
- Human review: people answer questions about a sample of conversations, and their answers are compared with the LLM judge scores.

**Use in the tool:** Gate G2 (quality, error rate and evaluation): a concrete measurable value (evaluation pass rate, alert on a drop) and calibration of automated scoring against human review. For the decision record: "which pass rate on which test set was the condition for raising autonomy".

**Link:** https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/evaluate-and-improve/

**Verified on:** 3 November 2026

---

## Z-05 – Grafana Assistant: status and capabilities

**What the source covers:** The GA announcement, the documentation pages "Get started" and "Explore Assistant capabilities", and related "What's new" entries.

**What it says:**
- "We are excited to announce the General Availability of Grafana Assistant in Grafana Cloud." Released on 8 October 2025, no self-managed release.
- "Grafana Assistant is an AI-powered observability agent." Five main workflows: Assistant Investigations, dashboard management, querying (PromQL, LogQL, TraceQL, SQL, k6), navigating Grafana, knowledge and best practices.
- Other capabilities: Workspace, Pages, Skills, Automations, Assistant Watchers, Investigations, Infrastructure memory, Knowledge Graph mode, CLI, Managed Sandboxes.
- Status of individual features according to separate "What's new" entries: Skills in Assistant GA (17 March 2026), Assistant in Slack GA (17 March 2026), Assistant CLI and Assistant Tunnel public preview (13 March 2026). On the verification date the CLI documentation page still says public preview.
- Assistant uses Anthropic's Claude models (directly, through Google Vertex AI or through Amazon Bedrock); customers do not choose the provider. For EU stacks, conversation storage, inference and Agent Observability processing stay in the EU/EEA.

**Use in the tool:** Shows that Grafana itself runs both an AI assistant and an agent in IT operations. The tool links to it as the place where these things are done and does not describe how to operate them.

**Link:** https://grafana.com/whats-new/2025-10-08-grafana-assistant-is-now-generally-available/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/get-started/ and https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/

**Verified on:** 3 November 2026

---

## Z-06 – Grafana Assistant: identity, audit trail and conversation retention

**What the source covers:** The documentation pages "Your data and security" and "Your data and privacy".

**What it says:**
- "Assistant actions run under your user identity. When the Assistant creates or updates resources, Grafana Cloud records the changes in user-level audit logs the same way as manual actions."
- "Grafana Cloud doesn’t provide a separate Assistant-only audit log. Audit logs don’t include full chat transcripts."
- RBAC allows fine-grained Assistant permissions (read, write, run investigations, manage MCP servers); Assistant can only query data sources the user has access to.
- Conversations (prompts, responses, metadata) are stored in the tenant in the stack's region. Model providers process data with zero retention.
- Chat retention applies to all Assistant chats "including chats created by investigations, automations, watchers, and other agents". From 1 September 2026 Grafana planned a default 90-day retention period for organizations that had not chosen one. Once enabled, retention cannot be turned off, only the period can be changed.
- The RBAC documentation lists, among others, the roles Assistant Admin, Assistant User, Assistant Investigation User, Assistant MCP User, Assistant Cloud MCP User, Assistant Coding Admin and Assistant Durable Agents User/Admin (agents called "Grots").

**EVIDENCE-GAP:** The product status of "Durable Agents" (Grots) could not be established from a primary source; the name appears only in the RBAC documentation. Secondary reports mention "Grafana Cloud Agents" in public preview; no official announcement under that name was found.

**Use in the tool:** Gate G6 (audit trail): a change made by the agent shows up in the audit log as a change by the user, and the conversation transcript is not part of the audit log. Recommendation for the decision record: check whether conversation retention is long enough to trace the agent's decisions, and set it deliberately.

**Link:** https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/privacy-and-security/security/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/privacy-and-security/privacy/ and https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/privacy-and-security/rbac/

**Verified on:** 3 November 2026

---

## Z-07 – Grafana Assistant: approving actions (MCP tools, Skills, Pages, CLI)

**What the source covers:** The documentation pages "MCP servers", "Create Skills", "Configure Grafana Pages" and "Use Grafana Assistant CLI".

**What it says:**
- External MCP servers: each tool can be enabled or disabled and is marked **Read** or **Write**. Approval policy per tool: **Default**, **Auto-approve** ("Allow Assistant to run the enabled tool without asking first"), **Always ask** ("Require explicit approval every time").
- "Use auto-approval carefully. Auto-approving enabled tools lets Assistant run them without prompting you, including tools that may write, modify, or delete data in the connected service."
- Change review of pull requests: "Nobody approves tool calls during a review, so a tool that would ask for approval doesn’t run."
- Skills can have a list of auto-approved MCP tools; "If an admin sets a tool to **Always ask**, allowed-tools auto-approval can’t override it."
- Grafana Pages (public preview): before an action the user sees the target and the values; the option "Always allow for this page" allows all supported actions of the page without further approval, "including requests that change external systems". "Revoking approval can’t undo an action already sent."
- Assistant CLI (public preview): access to local files is read-only by default; running shell commands is optional and requires approval.

**EVIDENCE-GAP:** The documentation does not specify what the **Default** policy means for Grafana's built-in tools (dashboards, alerts).

**Use in the tool:** A direct example of moving between U2 (agent acts with approval) and U3 (agent acts on its own within limits) within one product: switching from Always ask to Auto-approve is exactly the decision the tool should lead to a decision record for. The documentation's warning about auto-approve supports raising autonomy one tool at a time.

**Link:** https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/mcp-servers/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/skills/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/pages/ and https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/cli/

**Verified on:** 3 November 2026

---

## Z-08 – Assistant Investigations: GA, automatic triggering, limits

**What the source covers:** The "What's new" entry of 29 July 2026 and the documentation pages "Grafana Assistant Investigations" and "Configure investigation settings".

**What it says:**
- "Assistant Investigations is now generally available." An investigation goes through metrics, logs, traces and profiles, forms hypotheses and produces a structured report "with findings, supporting evidence, and recommended next steps".
- Triggering: manually, through alert enrichment or IRM webhooks ("an investigation starts automatically"), from Slack, from the Grafana Cloud MCP server or from gcx.
- A person can mark a hypothesis **Disprove** or **Reopen** and redirect the investigation; after the report they rate the verdict.
- Protection against overload: limits on new investigations by severity; "By default, **All** limits alert enrichment to 50 new investigations per 20 minutes." IRM webhooks have a separate limit with the same default.
- According to the documentation, Sift Investigation is a separate feature for automatic triage in Kubernetes without a prompt.

**CONFLICT:** On the verification date, a reference file in the official `grafana/skills` repository (skill `ml-ai`, `references/llm-and-graph.md`) still says "Assistant Investigations (public preview)". The newer primary sources (What's new of 29 July 2026, press release of 27 July 2026) say GA.

**Use in the tool:** Pattern for the alert triage scenario: the agent starts the analysis on its own (read-only access to data) and the result is a report for a person. The investigation limit is an example of a gate on cost.

**Link:** https://grafana.com/whats-new/2026-07-29-assistant-investigations-is-now-generally-available-in-grafana-cloud/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/investigation/ and https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/investigation-alerts/

**Verified on:** 3 November 2026

---

## Z-09 – Assistant Watchers: GA, runs under a service account, self-recalibration

**What the source covers:** The "What's new" entry of 19 October 2026 and the documentation page "Use Assistant Watchers".

**What it says:**
- "Assistant Watchers are now generally available." Watchers are "always-on agents for the problems that are hard to write an alert rule for". Critical findings can automatically start an Assistant Investigation.
- "Runs use the stack’s Assistant service account, not your identity."
- Automatic recalibration: Assistant periodically (weekly or monthly) reviews and adjusts checks, thresholds and baselines. The reviews "run in the background without pausing scanning or asking questions" and run under the identity of the watcher's creator. Changes are versioned and can be reverted.
- From 1 December 2026 Watchers are billed by Assistant token usage; the watcher page shows a token estimate.

**Use in the tool:** Example of an agent that changes its own configuration (thresholds) without asking and runs under a service account instead of a person. For gates G5 (who approves) and G6 (audit trail) it raises the model question: under which identity does the agent act, and where are its changes visible (here, in watcher versions).

**Link:** https://grafana.com/whats-new/2026-10-19-assistant-watchers-are-now-generally-available-in-grafana-cloud/ and https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/watchers/

**Verified on:** 3 November 2026

---

## Z-10 – Assistant Automations and Grafana Workflows

**What the source covers:** The documentation page "Use Automations", the "What's new" entry on Grafana Workflows (released on 20 October 2026) and the press release of 27 July 2026.

**What it says:**
- Automations are saved Assistant prompts triggered manually, on a schedule or by events (for example a GitHub pull request). Run history shows the trigger and a link to the run's conversation; a failed run is recorded with status `failed`. According to the press release of 27 July 2026, Automations are GA.
- Bounded schedules: before starting, Assistant shows a single confirmation with the cadence, the end, the expected number of runs and a warning about token usage.
- "Grafana Workflows is now in public preview for all Grafana Cloud stacks." Workflow steps can declare and update an incident, send a page through IRM, call any HTTP API and run Python in a sandbox. The "Ask Assistant" step hands a task to Assistant. Workflows have RBAC, token storage as secrets and "inspect every run step by step".

**Use in the tool:** Automatic service restart scenario: in the Grafana ecosystem this would combine a deterministic workflow with an AI step. The tool should help tell which step the AI decides and which is predefined, and set the gate accordingly. Workflows are in preview, and the assessment says so.

**Link:** https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/automations/ and https://grafana.com/whats-new/2026-10-13-automate-incident-response-and-operations-with-grafana-workflows/

**Verified on:** 3 November 2026

---

## Z-11 – Grafana Managed Sandboxes and Grafana Pages (public preview)

**What the source covers:** The documentation pages "Use Grafana Managed Sandboxes" and "Configure Grafana Pages".

**What it says:**
- Both features carry a banner in the documentation: "currently in public preview. Grafana Labs offers limited support, and breaking changes might occur prior to the feature being made generally available."
- Managed Sandboxes: the agent edits code in a sandbox and prepares a pull request. Repository publishing policy: **Ask before publishing** ("Assistant pushes nothing to GitHub until you approve") or **Auto-publish PRs** (opens a draft PR without asking, only for verified private repositories).
- Handing a task to an external provider (Cursor, Claude): "Grafana can’t review changes before they reach GitHub."
- Change review checks a pull request against live telemetry for five kinds of risk: deploy hazards, production impact, observability breakage, sensitive telemetry, instrumentation.

**Use in the tool:** The fix proposal scenario: the line between "the agent proposes" and "the agent changes" is the merge of the pull request, which stays with a person. Useful as an example of U2 (agent acts with approval).

**Link:** https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/managed-sandboxes/ and https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/pages/

**Verified on:** 3 November 2026

---

## Z-12 – Grafana Cloud MCP server (hosted): GA and access levels

**What the source covers:** The "What's new" entry of 28 July 2026 and the documentation pages "Use Grafana Cloud MCP tools" and "Choose Grafana Cloud MCP tool areas".

**What it says:**
- The entry released on 28 July 2026 carries the label "Generally Available": "Give your AI agents instant, zero-infrastructure access to your entire observability ecosystem."
- Access levels when authorizing an external agent: **Read** (view resources and run read-only queries), **Query**, **Write** ("Also create, change, and delete resources"). Write requires the Assistant Admin role.
- Write tools include, among others, `alerting_rules_write`, `alerting_silences_write`, `alerting_routing_write`. `ask_assistant` requires write access "because Assistant can perform actions that change resources".
- "Grafana Cloud MCP tool areas is currently in public preview." (choosing which tool groups an external agent sees)

**CONFLICT (name):** The subtitle of the press release of 27 July 2026 says "the Grafana MCP server", the body of the same release says "the Grafana Cloud MCP server". This is the hosted Grafana Cloud server, separate from the open-source `grafana/mcp-grafana` (Z-13).

**Use in the tool:** Check question for every deployment of an external agent (Claude, Copilot, Cursor) on top of Grafana: which access level is it connected with. Read without Write is a simple, measurable limit for U1 (assistant advises).

**Link:** https://grafana.com/whats-new/2026-07-28-cloud-mcp-server--connect-your-agents-to-grafana-cloud-with-a-single-url/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/cloud-mcp/ and https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/cloud-mcp-tool-areas/

**Verified on:** 3 November 2026

---

## Z-13 – Open-source MCP server `grafana/mcp-grafana`: read-only mode

**What the source covers:** The `grafana/mcp-grafana` repository on GitHub, README. Latest release v2.2.0 of 2 November 2026, Apache-2.0 license.

**What it says:**
- "The `--disable-write` flag provides a way to run the MCP server in read-only mode, preventing any write operations to your Grafana instance."
- Tool categories can be switched off with `--disable-<category>` flags; some categories are off by default and are enabled through `--enabled-tools`.
- Agent Observability tools (`agento11y_*`) are "disabled by default and work only in Grafana Cloud". They can read conversations, generations, evaluation scores, the agent catalog and agent versions, evaluators, eval rules and guards ("guards (hook rules) that run inline and can warn or deny") and offline experiments.
- Assistant tools are off by default and are "write tools (the assistant may mutate stack state), so they are skipped when `--disable-write` is set."
- Write tools include, among others, incident management in Grafana Incident and `update_alert_group` (acknowledge, resolve).

**Use in the tool:** A concrete technical limit for U1 (assistant advises) when working with Grafana from your own agent. It shows that reading data can be separated from writing even for an agent.

**Link:** https://github.com/grafana/mcp-grafana

**Verified on:** 3 November 2026

---

## Z-14 – Sift

**What the source covers:** The documentation page "Sift investigations".

**What it says:**
- Sift is "a powerful, free diagnostic assistant included in Grafana Cloud" that runs a set of checks over telemetry and compiles a list of "interesting results".
- "By default, users with the **Viewer** basic role can start investigations because running an investigation only queries existing data." Editing checks requires the Sift Editor role.
- The Assistant Investigations documentation describes Sift Investigation as a separate feature "for automatic Kubernetes triage without a prompt".

**EVIDENCE-GAP:** No formal GA announcement for Sift was found. On the verification date the documentation has no preview banner and describes Sift as a free part of Grafana Cloud.

**Use in the tool:** Example of automation that only reads data, which is why Grafana allows it for the Viewer role. For the tool this supports the point that the gate depends on write access and the impact of the action. Using AI or ML by itself does not raise the level.

**Link:** https://grafana.com/docs/grafana-cloud/machine-learning/sift/

**Verified on:** 3 November 2026

---

## Z-15 – Knowledge Graph and infrastructure memory

**What the source covers:** The documentation page "Knowledge graph".

**What it says:**
- "The knowledge graph is included in every Grafana Cloud stack." Observability solutions add the entities they monitor and insights about their health.
- According to the documentation, entities and insights "typically add 5% to 10% more active series. There is no separate charge."
- Grafana Assistant uses entities and relationships from the graph; "Infrastructure memory is a capability of the knowledge graph."
- It includes the entity catalog, the entity graph (dependencies) and the RCA workbench (timeline of insights).

**Use in the tool:** Context for the agent: without a dependency model the agent's remediation proposals do not account for the impact on neighbouring services. In the assessment this becomes the question "does the agent have the topology?", with a link here.

**Link:** https://grafana.com/docs/grafana-cloud/knowledge-graph/

**Verified on:** 3 November 2026

---

## Z-16 – Grafana IRM

**What the source covers:** The documentation pages "Grafana IRM" and "Configure IRM webhooks for automated investigations".

**What it says:**
- "Grafana IRM is the on-call management and incident response product in Grafana Cloud." It covers on-call schedules, escalations, declaring and managing incidents through to post-incident review. It is a paid add-on billed by monthly active IRM users.
- IRM webhooks can automatically start an Assistant Investigation when an alert group is created or an incident is declared, "providing context and insights before human responders arrive".

**Use in the tool:** The place in Grafana where the handover between agent and person happens (who is on call, who approves). For gate G5 (who approves), link to the escalation chains in IRM.

**Link:** https://grafana.com/docs/grafana-cloud/alerting-and-irm/irm/ and https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/irm-webhooks/

**Verified on:** 3 November 2026

---

## Z-17 – Grafana Labs press releases of 27 July 2026 and 20 October 2026: status summary and survey numbers

**What the source covers:** Two Grafana Labs press releases (AI Week, ObservabilityCON 2026).

**What it says:**
- 27 July 2026: GA of six features: Grafana Assistant Investigations, Grafana Assistant Workspace, Grafana Assistant Automations, Grafana Cloud MCP server, gcx and Grafana Agent Observability. Quote from Mat Ryer (Grafana Labs): "It’s the same assistant end-to-end, and it doesn’t just advise; it acts." Vendor survey number: according to "Grafana Labs’ 2026 Observability Survey", 92% of practitioners see value in AI catching anomalies, but only 57% say they do anything about observability of their own AI systems.
- 20 October 2026 (headline "Grafana Labs Charts the Path to Agentic Operations with Watchers, Pages, and Agent Observability at ObservabilityCON 2026"): Assistant Watchers (GA), Agent Observability with OpenTelemetry support (GA), Grafana Pages (public preview), Grafana Managed Sandboxes (public preview). Assistant Investigations "generally available since late July". Managed Sandboxes prepare a fix as a pull request "for human review and approval". Grafana Labs says it builds agents that act and leaves it to each organization to decide how much human oversight to keep "based on its own level of trust and risk profile".
- Vendor survey number in the release of 20 October 2026 (source named in the release: Grafana Labs and ETR report "2026 State of Observability", 300 IT and engineering leaders): "85% report comfort with read-only investigation, compared to 52% for full autonomous remediation."

**Note:** Both numbers come from vendor surveys, each from a different survey. If the tool uses them, it always gives this source and labels them "Grafana Labs survey". The full survey reports were not verified in this research (`EVIDENCE-GAP` for survey methodology).

**Use in the tool:** The context of the event on 25 November 2026: Grafana itself says that each organization chooses its level of oversight. That is the decision our tool helps with. The gap between 85% and 52% shows that trust drops as autonomy rises, with the same technology underneath.

**Link:** https://grafana.com/press/2026/07/27/grafana-labs-ships-six-tools-that-power-agentic-operations-from-planning-to-production/ and https://grafana.com/press/2026/10/20/observabilitycon-2026-ai-updates/

**Verified on:** 3 November 2026

---

# B. Official agent skills from Grafana

## Z-18 – The `grafana/skills` repository: what it contains and what the tool must not duplicate

**What the source covers:** The `grafana/skills` repository on GitHub (README, file tree). Apache-2.0 license, latest commit on branch `main` on 27 October 2026 (8fb72a5).

**What it says:**
- "Public skills for working with Grafana, Prometheus, Loki, Tempo, Pyroscope, k6, and the broader LGTM observability stack. Compatible with Claude Code, Cursor, Codex, and any tool supporting the Agent Skills open standard."
- Installation through `npx skills add grafana/skills` or as a plugin marketplace in Claude Code (`claude plugin marketplace add grafana/skills`).
- The repository contains 53 SKILL.md files in seven groups (52 are listed in the README tables, plus `grafana-core/skill-authoring`):
  - **grafana-core (8):** grafana-oss, dashboarding, promql, alerting-irm, alloy, beyla, opentelemetry, skill-authoring.
  - **grafana-cloud (20):** admin, send-data, fleet-management, cloud-integrations, infrastructure, app-observability (APM, Frontend Observability, AI Observability), frontend-session-rca, database-observability, adaptive-metrics, cost-management, dpm-finder, loki-label-analyzer, prometheus-label-strategy, prometheus-cardinality-troubleshooter, oncall-irm, ml-ai (Assistant, Dynamic Alerting, Sift, Knowledge Graph, LLM Plugin), assistant-mcp, private-connectivity, testing, synthetic-monitoring-checks.
  - **grafana-lgtm (7):** loki, tempo, prometheus, mimir, pyroscope, profilecli-insights, performance-rca.
  - **grafana-plugins (5):** plugin-bundle-size, react-19-plugin-migration, grafana-scenes, check-npm, audit-and-reduce-dependencies.
  - **grafana-app-sdk (4):** app-sdk-concepts, cue-kind-definition, reconciler-logic, admission-control.
  - **grafana-k6 (8):** k6, k6-docs, k6-perf-test-website, k6-manage, k6-cloud-investigate-test, k6-trend-analysis, k6-test-maintenance, k6-browser-test.
  - **grafana-datasources (1):** datasources-provisioning.
- Full-text search across all skills: no skill deals with agent autonomy levels, human in the loop, readiness or deciding where to let an agent in. AI topics are covered only by `app-observability` (instrumenting LLM applications, token and cost metrics, evals and guards through OpenLIT), `ml-ai` (enabling Grafana's AI and ML features) and `assistant-mcp` (connecting coding agents to Grafana Cloud through MCP).

**CONFLICT:** The `app-observability` skill describes AI Observability through OpenLIT (`openlit.init()`, dashboard "GenAI Observability"). Current Grafana documentation (Z-02) describes Agent Observability through its own SDK, OpenTelemetry, Claude Platform and forwarding of coding agent sessions, and does not mention OpenLIT. The `ml-ai` skill lists Investigations as public preview (see Z-08). In its AI part the repository lags behind the documentation.

**Use in the tool:** Scope boundary. The tool must not do: instrumentation (OpenTelemetry, Alloy, OpenLIT, SDK), dashboards, PromQL, LogQL, TraceQL, alerts, IRM and on-call configuration, setup of Grafana's AI features, connecting an agent through MCP. Our space: deciding where to let AI into operations, how much autonomy to give and which gates must be met. Link to Grafana's skills as "how to measure it afterwards".

**Link:** https://github.com/grafana/skills

**Verified on:** 3 November 2026

---

# C. OpenTelemetry GenAI semantic conventions

## Z-19 – GenAI semantic conventions: move to a separate repository and Development status

**What the source covers:** The `open-telemetry/semantic-conventions-genai` repository, the GenAI page on opentelemetry.io and the original location in `open-telemetry/semantic-conventions`.

**What it says:**
- The page on opentelemetry.io (under Semantic conventions version 1.44.0) and the file in the main repository: "GenAI semantic conventions have moved to the OpenTelemetry GenAI semantic conventions repository. This page has moved and is no longer maintained in this repository." The main repository released v1.45.0 on 2 November 2026, and it no longer contains the GenAI conventions.
- The new `semantic-conventions-genai` repository (created on 5 May 2026) has no released version and no schema URL yet (README: "Schema URL: TODO").
- "Semantic conventions for generative AI systems. **Status**: Development". The conventions cover these signals: Client Inference, Client Embeddings, Events, Exceptions, Metrics, Inference Token Metrics, Model spans, Agent spans. Further conventions cover Anthropic, Azure AI Inference, AWS Bedrock, OpenAI, Agent2Agent Protocol (A2A) and Model Context Protocol (MCP).
- Attribute shared across operations: `gen_ai.conversation.id`.

**Use in the tool:** Reference for gate G1 (traceability) as an open, vendor-neutral standard. The assessment always says that the standard has Development status, so attribute names may change.

**Link:** https://github.com/open-telemetry/semantic-conventions-genai and https://opentelemetry.io/docs/specs/semconv/gen-ai/

**Verified on:** 3 November 2026

---

## Z-20 – GenAI spans and metrics for agents and tools

**What the source covers:** The documents `gen-ai-agent-spans.md`, `gen-ai-spans.md`, `gen-ai-metrics.md`, `gen-ai-token-metrics.md` and `mcp.md` in the `semantic-conventions-genai` repository.

**What it says:**
- Agent spans (all with Development status), value of `gen_ai.operation.name`: `create_agent`, `invoke_agent` (client and internal span), `invoke_workflow`, `plan`. Plus spans for agent skills (load skill, read skill resource) and command execution.
- Tool span: `gen_ai.operation.name` "SHOULD be `execute_tool`". "GenAI instrumentations that can instrument tool execution calls SHOULD do so." Tool calls over MCP can also be traced with MCP instrumentation.
- Metrics (Development): `gen_ai.client.operation.duration`, `gen_ai.server.request.duration`, `gen_ai.server.time_per_output_token`, `gen_ai.server.time_to_first_token`, `gen_ai.invoke_workflow.duration`, `gen_ai.invoke_agent.duration`, `gen_ai.invoke_agent.inference_calls`, `gen_ai.invoke_agent.tool_calls`, `gen_ai.execute_tool.duration`.
- Token metrics: `gen_ai.client.inference.usage.input_tokens`, `.output_tokens`, `.cache_read.input_tokens`, `.cache_write.input_tokens`, `.reasoning.output_tokens` and `gen_ai.client.inference.operation.input_tokens` / `.output_tokens`.
- MCP metrics: `mcp.client.operation.duration`, `mcp.server.operation.duration`, `mcp.client.session.duration`, `mcp.server.session.duration`.

**Use in the tool:** Vocabulary for the question "what must be measured": number of tool calls per task, agent run time, tokens per task (cost). The tool only names what should be visible and links here; how to set it up is covered by Grafana and its skills.

**Link:** https://github.com/open-telemetry/semantic-conventions-genai/tree/main/docs/gen-ai

**Verified on:** 3 November 2026

---

## Z-21 – GenAI conventions: prompt and response content is not captured by default

**What the source covers:** The section "Capturing instructions, inputs, and outputs" in `gen-ai-spans.md`.

**What it says:**
- "Model instructions, user messages, and model outputs are considered sensitive and are often large in size."
- "OpenTelemetry instrumentations SHOULD NOT capture them by default, but SHOULD provide an option for users to opt in."
- Default pattern: do not capture instructions, inputs or outputs. Recording them in the attributes `gen_ai.system_instructions`, `gen_ai.input.messages`, `gen_ai.output.messages` fits where the volume is manageable and either no privacy regulations apply or the telemetry storage complies with them; the alternative is uploading content to external storage.

**Use in the tool:** A real tension between traceability and data protection. Gate G1 (traceability of agent steps) must include a deliberate decision on whether and where conversation content is stored. Without it, traces contain only metadata. Legal assessment (GDPR) is out of scope; the tool only points it out.

**Link:** https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md

**Verified on:** 3 November 2026

---

# D. Frameworks for autonomy levels and human oversight

## Z-22 – Feng, McDonald, Zhang: Levels of Autonomy for AI Agents (Knight First Amendment Institute)

**What the source covers:** Original paper by K. J. Kevin Feng, David W. McDonald, Amy X. Zhang (University of Washington), arXiv:2506.12469, v1 of 14 June 2025, v2 of 28 July 2025. The PDF is an "accompaniment to the web publication by the Knight 1st Amendment Institute", license CC BY 4.0.

**What it says:**
- "We define an AI agent’s autonomy as the extent to which it is designed to act without user involvement." Autonomy is "a deliberate design decision", separate from capability: "a capable agent ... can still act with a low level of autonomy if it is required to consult its user before taking each action."
- Five levels by the user's role, with required controls (Table 1):
  - **L1 Operator:** the user plans, the agent acts on request. Controls: "User-managed planning", "Invocation or approval before actions".
  - **L2 Collaborator:** joint planning and execution, the user can take over at any time. Controls: handover of control in both directions, shared view of progress.
  - **L3 Consultant:** the agent plans and executes most of the work and consults the user for expertise and preferences. Control: an interface for rich feedback, beyond plain approval.
  - **L4 Approver:** the agent involves the user only on failure, high risk or when approval is needed. Controls: "Approval elicitation for consequential actions", "Customizable conditions for seeking approval". Open question in the paper: "How to prevent user disengagement and meaningless approvals?"
  - **L5 Observer:** the agent plans and executes everything, the user can "monitor and audit agent via activity logs". Control: "Emergency off switch".
- Proposes "autonomy certificates": a certificate is valid for a specific technical specification and operating environment, and a change to them (for example a new tool that can modify a database) can invalidate it.
- The example systems in the table reflect the state as of June 2025.

**Use in the tool:** Main theoretical basis for our four working levels U1 to U4 (assistant advises, agent acts with approval, agent acts on its own within limits, agent acts on its own). The tool derives the mapping itself and labels it as its own interpretation. The idea of a certificate valid only for a given set of tools directly supports the decision record and the rule "new tool means another pass through the gates". The question about "meaningless approvals" supports the gate against approval fatigue.

**Link:** https://arxiv.org/abs/2506.12469 and https://knightcolumbia.org/content/levels-of-autonomy-for-ai-agents-1

**Verified on:** 3 November 2026

---

## Z-23 – AWS Agentic AI Security Scoping Matrix

**What the source covers:** The AWS page "Agentic AI Security Scoping Matrix".

**What it says:**
- Four scopes by degree of agency and human oversight:
  - **Scope 1, No Agency:** "Human-initiated systems with no autonomous change capabilities." The agent cannot change external systems "without explicit human action".
  - **Scope 2, Prescribed Agency:** "Human-initiated systems that can recommend actions, including the ability to make changes to the environment." Changes require "mandatory human approval before execution".
  - **Scope 3, Supervised Agency:** "Human-initiated systems with autonomous execution capabilities." Once activated it acts "without requiring further HITL approvals".
  - **Scope 4, Full Agency:** "Self-initiating systems that operate continuously with minimal human oversight."
- Six security dimensions: Identity Context; Data, Memory & State Protection; Audit & Logging; Agent & LLM Controls; Agency Perimeters & Policies; Orchestration.
- The page has no publication or update date (`EVIDENCE-GAP` for the creation date).

**Use in the tool:** Second basis for the levels. The split between Scope 3 (acts on its own within limits, started by a person) and Scope 4 (acts on its own, starts itself) matches our two top levels U3 and U4 more closely than Feng et al. The six dimensions can serve as a checklist for the gates.

**Link:** https://aws.amazon.com/ai/security/agentic-ai-scoping-matrix/

**Verified on:** 3 November 2026

---

## Z-24 – Ethics Guidelines for Trustworthy AI (HLEG): human-in-the-loop, on-the-loop, in-command

**What the source covers:** Ethics Guidelines for Trustworthy AI by the High-Level Expert Group on Artificial Intelligence (HLEG) of 8 April 2019, official English version of the document on the European Commission portal.

**What it says (official English text, requirement "Human agency and oversight", part "Human oversight"):**
- "Oversight may be achieved through governance mechanisms such as a human-in-the-loop (HITL), human-on-the-loop (HOTL), or human-in-command (HIC) approach."
- "HITL refers to the capability for human intervention in every decision cycle of the system, which in many cases is neither possible nor desirable."
- "HOTL refers to the capability for human intervention during the design cycle of the system and monitoring the system’s operation."
- "HIC refers to the capability to oversee the overall activity of the AI system ... and the ability to decide when and how to use the system in any particular situation."

**Use in the tool:** Definitions of human in the loop and human on the loop in the glossary, using the official wording. This is not a legally binding text (that is the AI Act, Z-33).

**Link:** https://digital-strategy.ec.europa.eu/en/library/ethics-guidelines-trustworthy-ai (English PDF: https://ec.europa.eu/newsroom/dae/document.cfm?doc_id=60419, Czech PDF: https://ec.europa.eu/newsroom/dae/document.cfm?doc_id=60421)

**Verified on:** 3 November 2026

---

# E. Security frameworks and guidance on agents

## Z-25 – OWASP Top 10 for LLM Applications 2025: LLM06 Excessive Agency

**What the source covers:** Entry LLM06:2025 of the OWASP GenAI Security Project.

**What it says:**
- Excessive Agency is a vulnerability that enables damaging actions "in response to unexpected, ambiguous or manipulated outputs from an LLM".
- Three root causes: "excessive functionality", "excessive permissions", "excessive autonomy".
- Mitigations: Minimize extensions; Minimize extension functionality; Avoid open-ended extensions; Minimize extension permissions; Execute extensions in user's context; Require user approval; Complete mediation; Sanitise LLM inputs and outputs.

**Use in the tool:** Three questioning axes (which tools the agent may call, with which permissions, how much it acts on its own). The mitigation "Require user approval" matches U2 (agent acts with approval); "Complete mediation" (authorization in the target system, outside the model) matches the gate for U3 (agent acts on its own within limits).

**Link:** https://genai.owasp.org/llmrisk/llm062025-excessive-agency/

**Verified on:** 3 November 2026

---

## Z-26 – OWASP Top 10 for Agentic Applications for 2026 and the Least-Agency principle

**What the source covers:** OWASP GenAI Security Project document published on 9 December 2025 (PDF, 57 pages).

**What it says:**
- Ten risks: ASI01 Agent Goal Hijack; ASI02 Tool Misuse and Exploitation; ASI03 Identity and Privilege Abuse; ASI04 Agentic Supply Chain Vulnerabilities; ASI05 Unexpected Code Execution (RCE); ASI06 Memory & Context Poisoning; ASI07 Insecure Inter-Agent Communication; ASI08 Cascading Failures; ASI09 Human-Agent Trust Exploitation; ASI10 Rogue Agents.
- "We expand on the concepts of Least-Privilege and Excessive Agency by citing Least-Agency. This captures our advice to organizations to avoid unnecessary autonomy; deploying agentic behavior where it is not needed expands the attack surface without adding value."
- "Similarly, strong observability becomes non-negotiable: without clear visibility into what agents are doing, why they are doing it, and which tools they are invoking, unnecessary autonomy can quietly expand the attack surface."
- Among the mitigations: human approval for privilege escalation (ASI03), a kill switch, immutable and signed audit logs.

**Use in the tool:** The Least-Agency principle states the tool's default position exactly: where an assistant is enough, do not deploy an agent. The quote on observability links security to the topic of the event (Grafana AI Observability Breakfast).

**Link:** https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/

**Verified on:** 3 November 2026

---

## Z-27 – OWASP Agentic AI: Threats and Mitigations

**What the source covers:** Guide by the OWASP Agentic Security Initiative.

**What it says:**
- T8 Repudiation & Untraceability: occurs when agent actions "cannot be traced back or accounted for due to insufficient logging or transparency in decision-making processes". Mitigations: comprehensive logging, cryptographic verification, enriched metadata, real-time monitoring.
- T10 Overwhelming Human in the Loop: the threat targets systems with human oversight and exploits "human cognitive limitations". Recommendations: adaptive trust mechanisms with dynamic intervention thresholds based on risk, confidence and context, and hierarchical collaboration "where low-risk decisions are automated, and human intervention is prioritized for high-risk anomalies".

**CONFLICT (version):** The source page gives a publication date of 17 February 2025 (version 1.0, threats T1 to T15). The file offered for download on the verification date is "Version 1.1, December 2025" and contains threats T1 to T17 (new: T16 Insecure Inter-Agent Protocol Abuse and T17 Supply Chain Compromise). The quotes above are from version 1.1.

**Use in the tool:** T8 is the security rationale for gates G1 (traceability) and G6 (audit trail). T10 explains why U2 (agent acts with approval) is not safe by default: at high volume, approval turns into a formality.

**Link:** https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/

**Verified on:** 3 November 2026

---

## Z-28 – NIST AI Risk Management Framework 1.0 (NIST AI 100-1): relevant subcategories

**What the source covers:** NIST AI 100-1, Artificial Intelligence Risk Management Framework 1.0 (January 2023), only the subcategories relevant to overseeing and operating agents.

**What it says:**
- MAP 3.5: "Processes for human oversight are defined, assessed, and documented in accordance with organizational policies from the GOVERN function."
- MEASURE 2.4: "The functionality and behavior of the AI system and its components ... are monitored when in production."
- MEASURE 2.6: the system is evaluated regularly for safety, residual risk does not exceed the risk tolerance, and "it can fail safely".
- MANAGE 2.4: "Mechanisms are in place and applied, and responsibilities are assigned and understood, to supersede, disengage, or deactivate AI systems that demonstrate performance or outcomes inconsistent with intended use."
- MANAGE 4.1: post-deployment monitoring plans include, among other things, "appeal and override, decommissioning, incident response, recovery, and change management".
- GOVERN 1.7: processes for safely decommissioning an AI system.

**Use in the tool:** Gate G4 (rollback and kill switch): the kill switch from MANAGE 2.4, including clarity on who may use it, and rollback from MANAGE 4.1 (recovery and change management). Monitoring in production from MEASURE 2.4. The framework is voluntary.

**Link:** https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf

**Verified on:** 3 November 2026

---

## Z-29 – NIST CAISI: AI Agent Standards Initiative (status)

**What the source covers:** NIST announcement of 17 February 2026.

**What it says:**
- The initiative was announced by the Center for AI Standards and Innovation (CAISI) at NIST. Three pillars: supporting industry-led standards for agents, community-led open protocols for agents, and research on agent security and identity.
- "In the months ahead, NIST will announce research, guidelines, and further deliverables." Input collection: an RFI on agent security (until 9 March) and a concept paper on agent identity and authorization (until 2 April).

**EVIDENCE-GAP:** This research could not verify whether NIST published a final document on agents between February and November 2026. The tool does not refer to NIST on agents as a finished standard.

**Use in the tool:** Only as a note that standards for agents are still being written. Do not derive requirements from it.

**Link:** https://www.nist.gov/news-events/news/2026/02/announcing-ai-agent-standards-initiative-interoperable-and-secure

**Verified on:** 3 November 2026

---

## Z-30 – Anthropic: Building effective agents

**What the source covers:** Anthropic Engineering article published on 19 December 2024, authors Erik S. and Barry Zhang. The page now carries a note that the tooling landscape has changed considerably since December 2024.

**What it says:**
- The distinction: "Workflows are systems where LLMs and tools are orchestrated through predefined code paths. Agents, on the other hand, are systems where LLMs dynamically direct their own processes and tool usage."
- "We recommend finding the simplest solution possible, and only increasing complexity when needed. This might mean not building agentic systems at all."
- "The autonomous nature of agents means higher costs, and the potential for compounding errors. We recommend extensive testing in sandboxed environments, along with the appropriate guardrails."
- Agents can "pause for human feedback at checkpoints or when encountering blockers"; stopping conditions such as a maximum number of iterations are also common.
- Three principles: simplicity of design, transparency ("explicitly showing the agent’s planning steps"), a carefully designed agent-computer interface for tools.

**Use in the tool:** The workflow vs. agent distinction is the first question for every place in operations: is it a predefined procedure with an AI step, or does the AI direct the procedure? The "simplest solution" recommendation supports Least-Agency (Z-26).

**Link:** https://www.anthropic.com/engineering/building-effective-agents

**Verified on:** 3 November 2026

---

## Z-31 – Anthropic: Demystifying evals for AI agents

**What the source covers:** Anthropic Engineering article published on 9 January 2026.

**What it says:**
- "An evaluation (“eval”) is a test for an AI system: give an AI an input, then apply grading logic to its output to measure success."
- Capability evals start at a low pass rate, regression evals "should have a nearly 100% pass rate". Capability evals that pass can "graduate" into the regression suite, which runs continuously.
- Metrics: pass@k (at least one success in k attempts) and pass^k (all k attempts successful). Example from the article: with a 75% success rate per attempt and three attempts, the probability that all three pass is about 42%. pass^k "especially matters for customer-facing agents where users expect reliable behavior every time."
- Grader types: code-based, model-based and human (humans also calibrate model-based graders).
- "In reality, 20-50 simple tasks drawn from real failures is a great start."
- Reading run transcripts is necessary, otherwise you cannot tell whether the agent failed or the grader did.

**Use in the tool:** Gate G2 (quality, error rate and evaluation): for repeated operational actions (service restart) the right metric is reliability across attempts (pass^k), because one successful run says little. Starting with 20 to 50 tasks from real incidents is a realistic first item in "Steps now" (at most five actions).

**Link:** https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

**Verified on:** 3 November 2026

---

## Z-32 – Anthropic: Measuring AI agent autonomy in practice

**What the source covers:** Anthropic research (Societal Impacts team) published on 18 February 2026, an analysis of interactions in Claude Code and the public API.

**What it says:**
- "Newer users (<50 sessions) employ full auto-approve roughly 20% of the time; by 750 sessions, this increases to over 40% of sessions." The authors note that Claude Code's default setting requires manual approval of every action, so part of the shift is a configuration change.
- Experienced users auto-approve more often, but they also interrupt runs more often.
- "On the most complex tasks, Claude Code stops to ask for clarification more than twice as often as humans interrupt it." A stop initiated by the agent is a separate form of oversight.
- In the public API: 80% of tool calls come from agents with at least one safeguard (restricted permissions or human approval), 73% have a human in the loop in some form, and only 0.8% of actions appear irreversible. Risk and autonomy were classified by the Claude model, without human raters.
- "Our central conclusion is that effective oversight of agents will require new forms of post-deployment monitoring infrastructure and new human-AI interaction paradigms."

**Use in the tool:** Empirical support for two points: in practice autonomy grows gradually with trust (so the gates are a path, raised step by step), and post-deployment monitoring is a condition of oversight. The numbers come from Anthropic's products; do not present them as general statistics.

**Link:** https://www.anthropic.com/research/measuring-agent-autonomy

**Verified on:** 3 November 2026

---

# F. AI Act (link only, no interpretation)

## Z-33 – AI Act, Article 14: Human oversight

**What the source covers:** Text of Article 14 of Regulation (EU) 2024/1689 on the AI Act Service Desk portal (European Commission).

**What it says:** The page is titled "Article 14: Human oversight". The tool does not interpret the content of the article.

**Use in the tool:** Link only. For obligations and when they apply, refer to the AI Act advisor (profile P3 AI Operator), which describes the state after amendment (EU) 2026/1744 in its sources Z-07 and Z-08 (`ai-act-zdroje-eu.md`).

**Link:** https://ai-act-service-desk.ec.europa.eu/en/ai-act/article-14

**Verified on:** 3 November 2026

---

## Z-34 – AI Act, Article 12: Record-keeping

**What the source covers:** Text of Article 12 of Regulation (EU) 2024/1689 on the AI Act Service Desk portal.

**What it says:** The page is titled "Article 12: Record-keeping". The tool does not interpret the content of the article.

**Use in the tool:** Link only, same as Z-33. The full text of the regulation is on EUR-Lex (https://eur-lex.europa.eu/eli/reg/2024/1689/oj); in this research EUR-Lex returned no content to automated access (HTTP 202 with an empty body), so the link is taken from the official ELI identifier.

**Link:** https://ai-act-service-desk.ec.europa.eu/en/ai-act/article-12

**Verified on:** 3 November 2026

---

# Summary of conflicts between sources

| Topic | Source 1 | Source 2 | Entry |
|---|---|---|---|
| Agent Observability and OpenTelemetry | Press release of 20 Oct 2026: "with OpenTelemetry support (GA)" | Documentation: "OpenTelemetry ingestion is in public preview" | Z-02, Z-17 |
| Status of Assistant Investigations | What's new of 29 Jul 2026 and press release of 27 Jul 2026: GA | `grafana/skills`, skill `ml-ai`: public preview | Z-08, Z-18 |
| Path to AI/Agent Observability | Grafana documentation: own SDK, OpenTelemetry, Claude Platform, coding agents | `grafana/skills`, skill `app-observability`: OpenLIT | Z-02, Z-18 |
| MCP server name | Subtitle of the press release of 27 Jul 2026: "Grafana MCP server" | Body of the same release and What's new: "Grafana Cloud MCP server" | Z-12 |
| Version of OWASP Threats and Mitigations | Source page: 17 Feb 2025, T1 to T15 | Downloaded file: Version 1.1, December 2025, T1 to T17 | Z-27 |
| Grafana Labs survey numbers | "Grafana Labs’ 2026 Observability Survey" (92%, 57%) | Grafana Labs and ETR report "2026 State of Observability" (85%, 52%) | Z-17 |

# What could not be verified from a primary source

- Product status of "Durable Agents" (Grots) in Grafana Assistant and whether a product named "Grafana Cloud Agents" exists (Z-06).
- Formal GA date of Sift (Z-14).
- Behaviour of the **Default** policy for Grafana Assistant's built-in tools (Z-07).
- Methodology of the Grafana Labs surveys behind the numbers in the press releases (Z-17).
- Creation and update date of the AWS Agentic AI Security Scoping Matrix (Z-23).
- Whether NIST CAISI published a final document on agents by 3 November 2026 (Z-29).
- EUR-Lex content (automated access failed); the AI Act articles were verified through the AI Act Service Desk (Z-34).
