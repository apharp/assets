# ThousandEyes Enterprise Agent Baseline Alerting Framework

A baseline alerting design for an enterprise with 90 offices (10 critical, 80 standard) using ThousandEyes Enterprise Agents.

---

## How ThousandEyes alerting works (the parts that shape this design)

A few mechanics matter before picking thresholds:

**Tests vs. alert rules.** Enterprise Agents run *tests* (Agent-to-Server, Agent-to-Agent, HTTP Server, DNS, Voice, etc.). Network-layer tests collect loss, latency, jitter, MTU, and path trace as core metrics. Alert *rules* are then attached to those tests and define the conditions that trigger notifications.

**Global vs. location conditions.** Every rule has two layers — a global condition (e.g., "trigger when N agents see the issue") and a per-location condition (the actual metric threshold). This is the key lever for differentiating critical from non-critical sites: you can write one rule per metric and apply it to a *tag* (e.g., `Critical-Sites` vs. `Standard-Sites`) with different thresholds.

**Three alerting modes** are available depending on the metric and test type:

- **Static thresholds** — a fixed number you set (e.g., latency ≥ 150 ms). Best for VoIP and known SLAs.
- **Quantile Dynamic Baselining (QDB)** — automatically adjusts thresholds based on the metric's recent distribution. Available on a subset of metrics.
- **Adaptive Alerting** — a learning model that analyzes historic test data and current network behavior to distinguish noise from actual problems.

**Persistence logic to fight noise.** ThousandEyes recommends configuring rules to trigger only when a condition persists across multiple intervals (e.g., 3 of 5 rounds) rather than alerting on a single spike. This is critical — a single bad round on a 1-minute test is not an incident.

**Severity levels** (Info / Minor / Major / Critical) drive routing and let you do layered detection — Minor as an early-warning tier, Critical for service-impacting outages.

---

## Step 1: Tag your sites first

Before writing any rules, organize agents with tags. Built-in and custom tags allow you to target groups of agents when configuring tests and alerts.

Recommended tags:

- `site-tier:critical` — your 10 critical sites
- `site-tier:standard` — the other 80
- Optional: `region:*` if your offices span continents (latency baselines differ a lot for trans-oceanic paths)

---

## Step 2: Define what tests should exist on every Enterprise Agent

To alert on jitter/loss/latency, you need tests producing those metrics. Baseline tests per agent:

1. **Agent-to-Server (network) → primary data center / SaaS edge** (e.g., your DC VIP, M365 endpoint, Salesforce). Produces loss, latency, jitter. Run every 1 minute for critical sites, 2–5 minutes for standard.
2. **Agent-to-Agent (network) → at least one DC-resident Enterprise Agent.** Gives you a controlled, internal-only path measurement so you can isolate WAN/MPLS/SD-WAN issues from internet/SaaS issues.
3. **HTTP Server → key internal app + key SaaS app.** Gives availability and response time, which is what most users actually feel.
4. **DNS Server → internal resolver(s).** DNS issues masquerade as everything else.

> **Note:** Don't enable bandwidth measurement on every test. ThousandEyes recommends keeping bandwidth measurement disabled for tests using agents on routers or switches.

---

## Step 3: Recommended baseline alert thresholds

These are starting points grounded in widely accepted VoIP/real-time SLAs and ThousandEyes' own examples — **tune them after 2–4 weeks of baseline data**.

Industry reference points:

- VoIP typically tolerates round-trip delays up to ~150 ms before call quality becomes unacceptable.
- Cisco SD-WAN voice SLA classes commonly use `loss 2 / latency 250 / jitter 25` as a reasonable upper bound for voice tunnels.

### Layered thresholds (Minor = early warning, Major = action, Critical = page someone)

#### Latency (Agent-to-Server, round-trip)

| Site tier        | Minor (warn) | Major   | Critical |
| ---------------- | ------------ | ------- | -------- |
| Critical sites   | ≥ 80 ms      | ≥ 120 ms| ≥ 200 ms |
| Standard sites   | ≥ 120 ms     | ≥ 180 ms| ≥ 250 ms |

*Rationale: at the Major threshold for critical sites you're still inside VoIP tolerance; Critical = clearly degraded user experience.*

#### Packet loss

| Site tier        | Minor   | Major  | Critical |
| ---------------- | ------- | ------ | -------- |
| Critical sites   | ≥ 0.5%  | ≥ 1%   | ≥ 2%     |
| Standard sites   | ≥ 1%    | ≥ 2%   | ≥ 5%     |

*Rationale: even 1% packet loss meaningfully degrades TCP throughput and voice; the 2% Major aligns with common SD-WAN voice SLAs. Standard sites get more tolerance because they may include smaller offices on cheaper transport.*

#### Jitter

| Site tier        | Minor   | Major   | Critical |
| ---------------- | ------- | ------- | -------- |
| Critical sites   | ≥ 10 ms | ≥ 20 ms | ≥ 30 ms  |
| Standard sites   | ≥ 15 ms | ≥ 25 ms | ≥ 40 ms  |

*Rationale: 30 ms is the common VoIP-quality cliff; the 25 ms Major for SD-WAN voice classes is a well-established industry number.*

### Persistence (the anti-noise rule)

For all of the above, configure the alert as:

- **Trigger:** condition met **3 of 5 consecutive rounds** by the affected agent
- **Clear:** condition NOT met for **3 consecutive rounds**

This single setting will probably do more to reduce false positives than any threshold tuning.

### Global condition (how many agents must see it)

For network tests targeting the DC/SaaS, set the global condition to **"any 1 agent"** — you want to know about a single branch having a problem. The persistence requirement handles the noise. Don't use "all agents" for branch-side alerts; that's appropriate only for tests that target a service from many vantage points where you're trying to detect a service-side problem.

---

## Step 4: The actual rules to build (10 rules total)

| #  | Rule name                       | Tag scope             | Metric             | Threshold              | Severity |
| -- | ------------------------------- | --------------------- | ------------------ | ---------------------- | -------- |
| 1  | Critical – Latency – Major      | `site-tier:critical`  | Latency            | ≥ 120 ms, 3/5 rounds   | Major    |
| 2  | Critical – Latency – Critical   | `site-tier:critical`  | Latency            | ≥ 200 ms, 3/5 rounds   | Critical |
| 3  | Critical – Loss – Major         | `site-tier:critical`  | Loss               | ≥ 1%, 3/5 rounds       | Major    |
| 4  | Critical – Loss – Critical      | `site-tier:critical`  | Loss               | ≥ 2%, 3/5 rounds       | Critical |
| 5  | Critical – Jitter – Major       | `site-tier:critical`  | Jitter             | ≥ 20 ms, 3/5 rounds    | Major    |
| 6  | Standard – Latency – Major      | `site-tier:standard`  | Latency            | ≥ 180 ms, 3/5 rounds   | Major    |
| 7  | Standard – Loss – Major         | `site-tier:standard`  | Loss               | ≥ 2%, 3/5 rounds       | Major    |
| 8  | Standard – Jitter – Major       | `site-tier:standard`  | Jitter             | ≥ 25 ms, 3/5 rounds    | Major    |
| 9  | All sites – Availability        | both tags             | Availability       | < 100%, 2/3 rounds     | Critical |
| 10 | Agent health – Last contact     | all EAs               | Agent Last Contact | > 10 min               | Major    |

**Notes:**

- Skip a "Minor" jitter rule for now — jitter alone rarely fires without latency or loss also moving, and the Minor tier creates a lot of noise.
- Rule #10 is the agent-health rule from the Notifications tab under `Network & App Synthetics > Agents Settings` — separate from test-based alerts but essential, because a dead agent silently kills all your other alerting.
- Naming follows ThousandEyes' own convention: `[Region] - [Metric] - [Threshold]` — easy to scan in the alert list.

---

## Step 5: Routing

| Severity | Destination                              |
| -------- | ---------------------------------------- |
| Critical | PagerDuty / on-call                      |
| Major    | Team channel (Slack/Teams) + ticket      |
| Minor    | Dashboard only — no page                 |

ThousandEyes natively integrates with PagerDuty, Slack, ServiceNow, and Splunk among others.

---

## Step 6: After 30 days, migrate the right metrics to Adaptive Alerting

For Critical-tier **latency** and **jitter** specifically — where "normal" varies a lot by time of day — strongly consider migrating Rules #1, #2, and #5 to **Adaptive Alerting**.

Reason: a static 120 ms threshold over-alerts on a site whose normal is 90 ms and under-alerts on one whose normal is 40 ms. Adaptive Alerting handles that automatically.

Keep **packet loss on static thresholds** — loss has a hard physical meaning (≥1% is bad, period) and is the metric where a fixed line works best.

---

## Things to validate before locking this in

1. **Are all 10 critical sites equally critical?** In most enterprises you'll find a smaller "Tier 0" (HQ, contact centers, trading floors, manufacturing lines) with truly tight SLAs, and a "Tier 1" of large but tolerant sites. A three-tier scheme (T0 / T1 / T2) often models reality better than critical/non-critical and lets you set tighter T0 thresholds without over-alerting on T1.

2. **What lives on the WAN?** If your critical sites are voice/video heavy, the numbers above are right. If they're primarily heavy file transfer / DB replication, latency matters less but loss and throughput matter more — add Agent-to-Agent throughput tests with their own alert rule.

3. **Transport type matters.** MPLS, SD-WAN, and internet-only sites have different baseline expectations. Consider sub-tagging (e.g., `transport:mpls` / `transport:sdwan` / `transport:internet`) if you see big variance in normal performance.

---

## Implementation checklist

- [ ] Tag all 90 Enterprise Agents with `site-tier:critical` or `site-tier:standard`
- [ ] Deploy baseline tests on every agent (Agent-to-Server, Agent-to-Agent, HTTP, DNS)
- [ ] Verify test intervals (1 min for critical, 2–5 min for standard)
- [ ] Create the 10 alert rules per the table above
- [ ] Configure notification routing per severity
- [ ] Configure agent health notification rule (Rule #10)
- [ ] Let baselines accumulate for 2–4 weeks
- [ ] Review false-positive / false-negative rates and tune thresholds
- [ ] At 30 days, evaluate which Critical-tier rules to migrate to Adaptive Alerting
