# Decision record for an AI agent in operations: EXAMPLE

**This is an EXAMPLE.** The company, services, names, numbers and deadlines are made up and serve only as an illustration. Take the form of the record and set the values yourself to fit your operations.

A decision record is a one-page record stating that an agent may act at a specific autonomy level in a specific place. It says who decided, which tools and permissions the agent works with, what is measured and until when the permission is valid. When someone asks in six months why the agent restarted a service at night, the answer is this page with a name, a date and a reason.

Create one for every place where the agent acts with approval or on its own (U2 to U4), every time the level goes up and every time the agent gets a new tool or new permissions.

## Filled-in record: automatic service restart

In this example the team ran the agent at U2 for a month. On a memory alert the agent proposed a restart and the on-call engineer confirmed or rejected it. After that month the team decided the agent may restart two specific services on its own (U3).

```
EXAMPLE, values are illustrative

Record ID: DR-2026-007
Date: 4 Nov 2026

Place of deployment:
On a memory saturation alert for the orders-api and billing-api services in
production, the agent assesses whether a restart will help and, if so,
restarts the service.

Decision:
U3, agent acts on its own within limits. Only for restarting the two services
listed below. Anything else the agent only proposes in the on-call channel (U2).

Decided by:
Jana Horáková, Head of Operations

Agent owner:
Lukáš Beneš, SRE

Agent and version:
Model and its version as set in the agent configuration (stored in the
ops-agents repository), prompt version 3 (tag restart-agent-v3). Triggered by
the MemoryHigh alert through a webhook.

Tools and permissions:
Reading metrics and logs of both services: read only.
Restarting the orders-api and billing-api deployments in namespace shop-prod: write.
The agent acts under the service account sa-restart-agent, which nobody else uses.

Closed list of actions without approval:
1. Restart the orders-api deployment.
2. Restart the billing-api deployment.
At most one restart per service per 30 minutes. A second alert within an hour
is not handled by a restart; the agent only escalates.

How the limits are enforced:
The service account's role in the cluster allows only restarting these two
deployments. A guard in front of tool calls blocks all other tools.
The limit of one restart per 30 minutes is enforced by the webhook gateway,
outside the prompt.

Who approves and how many approvals per day:
Does not apply to actions on the list, they run without approval. Proposals
outside the list are approved by the on-call SRE, estimated 2 to 3 per week.

Escalation goes to:
The on-call SRE according to the on-call schedule, as a message in #ops-noc
with a link to the agent run.

Kill switch:
The on-call SRE or the agent owner disables the webhook for the MemoryHigh
alert. The procedure is in runbook RB-112. Dry run done on 28 Oct 2026.

Rollback:
A restart changes neither configuration nor data. If the service does not
come back up within 5 minutes, the agent stops and escalates. The on-call SRE
follows runbook RB-112.

What is measured and the threshold:
One month at U2 (1 to 31 Oct 2026): 46 restart proposals, on-call rejected 3.
The rejected cases are in the test set together with another 37 alerts from
past incidents. The set runs five times after every prompt change.
Threshold set by the team: rejected or wrong decisions at most 10% per month.
If exceeded, the agent goes back to U2.

Cost per task and cap:
One alert assessment costs about USD 0.13 on average. Cap of 20 runs per hour
in the webhook gateway.

What is stored from conversations, where and for how long:
The agent's prompts and responses are kept for 30 days in the agent
observability tool. Decided by Jana Horáková on 1 Oct 2026.

Where you can see what the agent did:
Each run (model and tool calls) in the agent observability tool. The restart
in the cluster audit log under the sa-restart-agent account. The message in
#ops-noc with a link to the run.

Valid:
Until 31 Jan 2027. It stops being valid earlier if the agent gets a new tool
or new permissions, if the model or prompt changes, if it is to be extended
to another service or raised to a higher level, and after any incident caused
by the agent.

Confirmed by:
Jana Horáková, 4 Nov 2026
```

## What to look at in the example

The Valid field matters most. Without a review date and a list of changes that invalidate the record, nobody withdraws the permission, and the agent gets a new tool without a new assessment.

The field How the limits are enforced lists the cluster permissions, the guard and the gateway limit. An instruction in the prompt ("restart only these two services") does not belong there, because the model can break it.

The field What is measured and the threshold shows the path from U2 to U3. The team first measured the share of rejected proposals for a month and only then decided. In the example the team set the 10% threshold itself; no standard prescribes it.

## Blank form

```
Record ID:
Date:

Place of deployment:
(what the agent does, in one or two sentences)

Decision:
(level U1 to U4 the agent may have in this place)

Decided by:

Agent owner:

Agent and version:
(model, prompt or agent version, where it is triggered from)

Tools and permissions:
(list of tools, read or write for each, the account the agent acts under)

Closed list of actions without approval:
(U3 and U4 only)

How the limits are enforced:
(permissions in the target system, guard, list of allowed tools)

Who approves and how many approvals per day:
(U2 only)

Escalation goes to:

Kill switch:
(who may use it and how)

Rollback:
(how each action is reverted)

What is measured and the threshold:
(evaluation, test set, threshold set by the team)

Cost per task and cap:

What is stored from conversations, where and for how long:

Where you can see what the agent did:

Valid:
(until when, and that it stops being valid with a new tool, new permissions, a change of model or prompt, a higher level and after an incident caused by the agent)

Confirmed by:
```

## Where to keep the records

Somewhere you can show that a record was not edited after the fact: in a Git repository next to the agent configuration, in shared storage with version history or in a document management system. If the agent works through pull requests, the decision record can live in the same repository.

The record captures the team's decision and its conditions. It makes no claim about the agent's safety and does not replace an audit.
