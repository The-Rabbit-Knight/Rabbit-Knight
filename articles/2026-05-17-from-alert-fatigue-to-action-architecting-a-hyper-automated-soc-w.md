---
title: "From Alert Fatigue to Action: Architecting a Hyper-Automated SOC Workflow"
date: 2026-05-17
medium_url: https://medium.com/@rabbit_knight/from-alert-fatigue-to-action-architecting-a-hyper-automated-soc-workflow-093f91ec49df
---

> **Original Published Article on Medium:** [From Alert Fatigue to Action: Architecting a Hyper-Automated SOC Workflow](https://medium.com/@rabbit_knight/from-alert-fatigue-to-action-architecting-a-hyper-automated-soc-workflow-093f91ec49df)

### From Alert Fatigue to Action: Architecting a Hyper-Automated SOC Workflow

![Article Figure](https://cdn-images-1.medium.com/max/800/1*-ZQGPIkh3mznurTEhxif5w.jpeg)

I built three automation workflows for SOC over the past few months. One reports on client license usage across all managed sites. One hunts malicious scheduled tasks on endpoints and deletes them. One runs fifteen custom-built Active Directory attack detections in parallel on a custom set schedule and writes the triage narrative before the analyst even opens the alert.

This post explains how they work, why they’re designed the way they are, and what each one taught me about building automation that actually holds up in a production environment.

A quick note on scope: I’m keeping the detection logic and query internals high-level. The goal here isn’t a copy-paste tutorial - it’s to walk through the architecture and the thinking behind it.

#### The Platform

The three workflows in this post run on the SentinelOne HyperAutomation engine, built directly into the same console analysts use daily, with no external SOAR required. Direct access to the telemetry data lake, endpoint management APIs, and remote script execution comes out of the box.

The building blocks that matter for this post:

- Scheduled and manual triggers — workflows fire on a clock or on demand
- PowerQuery (PQ) — async queries against the telemetry data lake
- RemoteOps — execute scripts on endpoints directly from the workflow, with execution status feedback
- JQ — a functional transformation language used to reshape API responses and render HTML output
- Parallel loops — iterate over a dataset and process every item simultaneously, not sequentially

One pattern shows up in all three workflows and is worth understanding upfront: the async PowerQuery loop. You fire the query, get back a query ID, then poll ‘stepsCompleted’ against ‘stepsTotal’ in a while loop with a short delay between checks. When they match, the query is done. It adds a few nodes to every workflow but it’s the correct way to handle queries that might take anywhere from two seconds to thirty, so you’re not guessing at a fixed wait time and hoping for the best.

#### Workflow 1: Operational Visibility: Automated Asset Reporting

The Problem: In a managed security environment, keeping track of deployed agents and license utilization across multiple client sites is one of those tasks that sounds simple until you’re actually doing it. Someone pulls the data, formats it, and sends it. Every week. Manually. It’s not complex work, which is exactly why it shouldn’t require a human.

The Design

```
[Scheduled Trigger: Day Time]
        |
        ├──> [Fetch all active client sites via API]
        └──> [Prepare HTML rendering template]
                        |
                        v
              [Render & Send Email Report]
```

The workflow fires on a scheduled time. Two branches run in parallel from the trigger: one fetches the live site data from the platform API, the other prepares the JQ rendering template. They converge at the send step.

The interesting part is the JQ transformation. It doesn’t just pull raw numbers; it calculates live license utilization per client as a percentage, differentiates between license bundle types, and handles edge cases like zero-license sites without breaking the output. All of that happens in a single transformation pass before the email is sent. The output is a formatted HTML table delivered to stakeholders on a scheduled time without anyone touching it.

Design note: Even on a workflow this simple, the parallel branch structure is worth doing. Fetching data and preparing the template are independent operations; there’s no reason to do them sequentially. It’s a small thing, but thinking about step dependencies is a habit that pays off when workflows get more complex.

#### Workflow 2: Active Threat Remediation: Ghost Task Hunter

The Problem: Malicious scheduled tasks are one of the more “persistent” persistence mechanisms in the attacker playbook. They survive reboots, they blend in with legitimate system tasks, and they resurface after remediation if the root cause isn’t fully addressed. The detection side, relatively easy part; the annoying part is the gap between finding a malicious task and removing it. In a manual workflow, that gap looks like this: analyst reviews query results, opens a ticket, assigns it, someone remotes into the endpoint, runs the unregister command, and closes the ticket. In a busy SOC, that sequence stretches from minutes to hours. Attackers know this.

The Detection Logic: The query runs against the telemetry data lake and does two things simultaneously: it performs a lookup join against a curated threat intelligence table of known malicious task names, and it applies regex pattern matching to catch naming variants used by specific malware families, the kind of tasks that look almost legitimate until you notice the pattern. The output is minimal by design: site information, endpoint identity, and task name. Exactly what’s needed to act, nothing more.

The Remediation Architecture:

This is where the workflow earns its complexity. The structure has three nested layers:

```
[Manual Trigger]
        |
        v
[Run PowerQuery] --> [Async Poll Loop: wait for query completion]
                                |
                                v
                    [For-Each Loop: iterate over every detected endpoint]
                                |
                                v
                    [RemoteOps: execute custom unregister script]
                    [Pass task name as parameter: -Name "<taskname>"]
                                |
                                v
                    [While Loop: poll script status every 20s]
                    [Exit on: completed / failed / canceled / expired]
                                |
                                v
                    [Accumulate result: site, endpoint, task, status]
                                |
                                v
                    [Send single HTML email to SOC with full results table]
```

For each detected endpoint, the workflow fires the custom PowerShell unregistration script via RemoteOps, passing the exact task name as a parameter. It then enters a polling loop, checking execution status every 20 seconds until the script reaches a terminal state. The result (site name, endpoint, task name, execution status) gets appended to an accumulator array. Once every endpoint has been processed, a single email goes to the SOC with the complete remediation table.

The StoryLine Email: The SOC confirmation email doesn’t just say “tasks removed.” It shows a table with every endpoint that was processed, the exact task name that was targeted, and the execution status returned by the script. If something failed, it shows up in the table. The analyst doesn’t need to go check, the workflow reports its own outcome.

Here’s what a typical confirmation row looks like:

```
| Site Name  | Endpoint | Task Name  | Status                     |
|------------|----------|------------|----------------------------|
| GreenHouse | PC-042   | \PDFast
_R5 | Script Execution Completed |
```

Design Decisions Worth Noting:

Why a manual trigger on the production version? Full automated remediation, detect and delete without any human in the loop, is technically possible. It’s also a risk. A regex pattern match that catches a legitimate task with a similar name would silently delete it. The manual trigger gives the SOC a confirmation gate before mass remediation fires. Automation with a human checkpoint is often the right call.

Why accumulate results instead of sending one email per endpoint? Because alert noise is its own problem. If the query returns fifteen endpoints, fifteen separate emails are not a useful outcome. Batch the results, send once, give the analyst one thing to review.

The polling loop design: The script status check doesn’t assume success. It waits for one of four terminal states: ‘completed’, ‘failed’, ‘canceled’, or ‘expired’. If the script hangs, the workflow doesn’t hang with it, the loop exits on timeout and records the state. This kind of defensive design is easy to skip and painful to add later.

Workflow 3: Proactive Identity Detection

The Problem: Active Directory is the backbone of most enterprise environments and one of the most targeted attack surfaces in existence. The attack techniques are well-documented: Kerberoasting, AS-REP Roasting, DCSync, Overpass-the-Hash, RBCD abuse, and they all generate specific Windows Event IDs when they occur. The detection logic is one of the parts of the workflow. The problem is what happens after detection. An analyst gets an alert: “Event 4768 fired 7 times.” Now they need to answer: which user? Which domain controller? What encryption type? What source IP? What does this pattern mean in context? That reconstruction takes time, and in a SOC handling multiple clients, it’s time that gets spent on every single alert, every single time.

The goal with this project was not only to find the anomaly but to make the alert answer those questions before the analyst asks them.

Architecture: Detection as Configuration

The entire detection library is a single configuration array loaded at the start of the workflow. Each entry has four fields: a name og the detection, a target scope, a query, and a sliding time window. That’s it.

```
{
  
"name"
:
 
"AS-REP Roasting"
,
  
"clientIds"
:
 
[
"..."
]
,
  
"query"
:
 
"..."
,
  
"window_size_seconds"
:
 
300
}
```

The workflow fires every 5 minutes. It loads the config, then enters a parallel fan-out loop - all 15 queries fire simultaneously against the data lake. The workflow doesn’t wait for query one to finish before starting query two. They all run at the same time, each with its own async poll loop, each processing results independently ( costly but effective, note the cost if you have constarince on number of runs).

Adding a new detection technique means adding one object to the config array. The workflow engine doesn’t change. The deployment process doesn’t change. You write the detection logic, add it to the list, done.

The StoryLine: Context Built Into the Query

This is the part of the project I’m most satisfied with. Each query doesn’t just detect, it also narrates. Using aggregation functions, the query builds a human-readable sentence for each event, then aggregates them into a ‘StoryLine’ field that gets embedded directly in the alert description.

Here’s what that looks like in practice. Instead of an alert that says: AS-REP Roasting on DC1.

The analyst sees: Story lines: [“Alert: Event 4738. AS-REP Roasting Setup. Actor ‘Administrator’ made account ‘jim_admin’ vulnerable by modifying the UAC flag to ‘DONT_REQ_PREAUTH’ on DC ‘DC1’.”]. The best part is that this framework is fully customizable and can hold as much contextual information as required.

Alert Creation

When a query returns results, the workflow resolves the machine ID to a full machine identity via the XDR assets API, then creates a structured alert through the platform’s ingest endpoint. The alert carries the full StoryLine. It lands in the SOC queue like any other alert, plus it already contains the context that would normally take manual investigation to assemble.

#### The Bigger Picture

Three workflows. Three different problems. One consistent thread.

Workflow 1 answers: what do we have? Workflow 2 answers: what is threatening us right now, and can we remove it? Workflow 3 answers: are we being attacked, and what exactly is happening?

Each one was built to eliminate a specific category of work that didn’t require human judgment. That’s the only metric that matters when evaluating SOC automation: not how sophisticated the workflow is, but how much better the analyst’s day looks after it’s running.

A few things that generalize beyond this specific platform:

- Async polling is a pattern, not a one-off. Any workflow that calls an API and waits for a result needs this. Build it once, understand it well.
- Config-driven detection scales. Hardcoded rules don’t. When your detection library lives in a data structure rather than workflow nodes, adding coverage is a data problem, not an engineering problem.
- The alert is the product. An alert that fires without context creates work. An alert that arrives with a narrative reduces it. The difference is in how you write the query, not in the alerting infrastructure.
- Automation with a human checkpoint is often better than full automation. Especially when the action is irreversible. The Ghost Task Hunter’s manual trigger isn’t a limitation; it’s a design choice.

The patterns here — config-driven detection, async query polling, result accumulation, parallel fan-out — aren’t specific to any one platform. They apply anywhere you have an API and a workflow engine. The platform just determines how much plumbing you have to write yourself.
