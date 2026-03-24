---
name: aws-service-chaos-research
description: >
  Use when the user asks about chaos engineering, fault injection, resilience testing,
  or HA verification for a SPECIFIC AWS service (e.g., RDS, EKS, MSK, ElastiCache,
  DynamoDB, S3, Lambda, OpenSearch, etc.). Triggers on "chaos testing on [service]",
  "fault injection for [service]", "how to test HA of [service]",
  "FIS scenarios/actions for [service]", "[service] failover testing",
  "[service] resilience testing", "[service] 混沌测试", "[service] 故障注入",
  "[service] 高可用验证", "对 [service] 做混沌实验", "test my [service]",
  "verify my [service] is resilient". Use this skill even when the user phrases
  it casually like "test my RDS" or "how resilient is my MSK cluster".
---

# AWS Service-Specific Chaos & HA Testing Research

Generate comprehensive chaos engineering and high availability testing scenarios for a
specific AWS service. Uses a **Scenario-Library-first** approach: read the latest FIS
Scenario Library documentation for pre-built composite scenarios first, then query
individual FIS actions via `list-actions`, and finally supplement with deep documentation
research.

## Output Language Rule

Detect the language of the user's conversation and use the **same language** for all output.
- Chinese input -> Chinese output
- English input -> English output
- Mixed -> follow the dominant language

## Prerequisites

Required tools (at least one of each group):

**FIS Scenario Library (Group A — documentation-based, always available):**
- `aws___read_documentation` — read FIS Scenario Library pages directly (scenarios are
  console-only and cannot be queried via CLI, so reading the latest docs is the only way
  to discover them)

**FIS Actions Discovery (Group B — use in order of preference):**
1. **AWS CLI** `aws fis list-actions` — definitive, real-time list of FIS actions from user's region
2. **aws___search_documentation** — FIS actions reference page as fallback when CLI is unavailable

**Documentation Research (Group C):**
- `aws___search_documentation` — search AWS official docs
- `aws___read_documentation` — read full doc pages
- `aws___recommend` — discover related pages

All documentation research uses **only** the AWS Knowledge MCP tools above.
Do NOT use SearXNG or other web search tools for documentation research.

## Workflow

### Step 1: Identify Target Service

Extract the target AWS service from the user's message and determine the target region.

#### Region Detection

FIS actions can differ across AWS regions — some actions may be available in
`us-east-1` but not yet in `ap-southeast-1`. Always determine the target region first,
because service keyword resolution depends on it.

**Detection order (use the first one that applies):**

1. **User explicitly specifies** — e.g., "us-west-2", "东京区域", "ap-northeast-1"
2. **Infer from context** — resource ARNs, previous conversation mentioning a region
3. **Check AWS CLI default** — run `aws configure get region` to get the configured default
4. **Ask the user** — if none of the above yields a region, ask:
   "Which AWS region are you targeting? FIS actions and scenarios may vary by region."

Store the resolved region as `TARGET_REGION` for use in subsequent steps.

#### Service Keyword Resolution

FIS action IDs follow the pattern `aws:<service>:<action>`. To map the user's input
to the correct FIS service keyword, use dynamic discovery from the live FIS action list:

```bash
aws fis list-actions --region TARGET_REGION | jq '.actions[].id' | awk -F':' '{print $2}' | sort -u
```

This returns the definitive list of FIS-supported service keywords in that region
(e.g., `ebs`, `ec2`, `ecs`, `eks`, `elasticache`, `fis`, `network`, `rds`, `s3`, `ssm`...).
Match the user's service name against this list. For example, if the user says
"Aurora", match it to `rds`; if "Kubernetes", match to `eks`.

If the AWS CLI is not available, derive the keyword by lowercasing the AWS service name
and removing spaces/hyphens (e.g., "ElastiCache" -> `elasticache`).

If the service is ambiguous, ask the user to clarify (e.g., "RDS MySQL or Aurora MySQL?").

Also determine the deployment architecture if the user mentions it:
- Multi-AZ, Multi-Region, Single-AZ
- Read replicas, Global Tables, Cross-region replication
- This affects which scenarios are relevant

### Step 2: Fetch FIS Scenario Library (Scenario-Library-First)

**This step has the highest priority.** The FIS Scenario Library provides AWS-curated
composite scenarios that orchestrate multiple fault injection actions into realistic
failure simulations. These are the most valuable starting point because they represent
AWS's own recommendations for how to test resilience.

Scenario Library scenarios are **console-only** — they cannot be listed or queried via
AWS CLI or API. The only way to discover them is by reading the latest documentation.
This is why documentation fetching is essential and not optional.

#### Required reads (always fetch these first):

```
aws___read_documentation(
  url="https://docs.aws.amazon.com/fis/latest/userguide/scenario-library.html",
  max_length=10000
)
```

```
aws___read_documentation(
  url="https://docs.aws.amazon.com/fis/latest/userguide/scenario-library-scenarios.html",
  max_length=10000
)
```

#### Detailed scenario pages (fetch based on relevance to user's target service):

Read the pages that are most likely to contain scenarios affecting the target service.
For most services, read all of these — they cover different failure domains:

- **AZ Power Interruption** — simulates complete AZ power failure:
  `https://docs.aws.amazon.com/fis/latest/userguide/az-availability-scenario.html`

- **AZ Application Slowdown** — simulates degraded performance within a single AZ:
  `https://docs.aws.amazon.com/fis/latest/userguide/az-application-slowdown-scenario.html`

- **Cross-AZ Traffic Slowdown** — simulates traffic disruption between AZs:
  `https://docs.aws.amazon.com/fis/latest/userguide/cross-az-traffic-slowdown-scenario.html`

- **Cross-Region Connectivity** — simulates cross-region connectivity failure:
  `https://docs.aws.amazon.com/fis/latest/userguide/cross-region-scenario.html`

#### Supplementary (fetch if needed for deeper context):

- **FIS Actions Reference** — full list of individual FIS actions:
  `https://docs.aws.amazon.com/fis/latest/userguide/fis-actions-reference.html`

- **FIS Document History** — check for recently added scenarios or actions:
  `https://docs.aws.amazon.com/fis/latest/userguide/doc-history.html`

#### From the scenario documentation, extract for each relevant scenario:

- **Scenario name and description**
- Which **sub-actions** the scenario orchestrates
- Which sub-actions are **relevant to the target service**
- What **resource tags** are required to target specific resources
- The **default durations** (interruption + recovery phases)
- Any **prerequisites or limitations**
- **Stop condition** recommendations

#### Decision: Which scenarios apply?

After reading the documentation, classify each scenario's relevance:

| Relevance | Criteria |
|---|---|
| **Directly relevant** | Scenario includes sub-actions that explicitly target the service (e.g., "Failover RDS" in AZ Power Interruption) |
| **Indirectly relevant** | Scenario affects infrastructure the service depends on (e.g., network disruption affects any VPC-based service) |
| **Not relevant** | Scenario has no meaningful impact on the target service |

Include both directly and indirectly relevant scenarios in the output.

### Step 3: Query FIS Actions

After the Scenario Library research, query individual FIS actions to discover
service-specific fault injection capabilities that may not be covered by composite
scenarios.

#### Path A: AWS CLI Available (Preferred)

**Step 3a: Fetch ALL FIS actions in the target region:**

```bash
aws fis list-actions --region TARGET_REGION --query 'actions[].{id:id, description:description}' --output json
```

Replace `TARGET_REGION` with the region resolved in Step 1 (e.g., `us-east-1`).
If no region was determined, omit `--region` to use the CLI default, but **warn
the user** that results reflect their default region and may differ in other regions.

**Step 3b: Filter for target service** — from the full list, find actions whose `id`
contains the search keyword(s) from Step 1. The FIS action ID format is
`aws:<service-name>:<action-type>`, so match the keyword against the service-name
segment.

For example, if keyword is `msk`, look for any action ID matching `aws:msk:*`.
If keyword is `rds`, look for `aws:rds:*`.

```bash
aws fis list-actions --region TARGET_REGION --query 'actions[?starts_with(id, `aws:KEYWORD:`)].{id:id, description:description}' --output json
```

Also scan the description field for the service name, because some actions may
reference a service in their description even if the action prefix is different
(e.g., an SSM action that mentions RDS in its description).

**Step 3c (Optional): Collect cross-cutting actions** — these affect services
indirectly and can be useful for building custom experiments. Include them if the
user's service would benefit from network, API, or infrastructure-level fault
injection testing:

```bash
aws fis list-actions --region TARGET_REGION --query 'actions[?starts_with(id, `aws:network:`) || starts_with(id, `aws:fis:inject`) || starts_with(id, `aws:ssm:`) || starts_with(id, `aws:ec2:stop`) || starts_with(id, `aws:ec2:terminate`)].{id:id, description:description}' --output json
```

Cross-cutting actions and when they're useful:
- `aws:network:disrupt-connectivity` — useful for any VPC-based service
- `aws:network:disrupt-vpc-endpoint` — useful for services accessed via PrivateLink
- `aws:fis:inject-api-internal-error` — useful to test app handling of AWS API failures
- `aws:fis:inject-api-throttle-error` — useful to test backoff/retry logic
- `aws:fis:inject-api-unavailable-error` — useful to test graceful degradation
- `aws:ec2:stop-instances` / `terminate-instances` — useful for services running on EC2
- `aws:ssm:send-command` / `start-automation-execution` — useful for custom fault scripts

Whether to include cross-cutting actions depends on context:
- **Include** when the service runs on EC2, uses VPC networking, or the user is
  interested in infrastructure-level failure testing
- **Skip** when the user is focused only on service-native failures, or the service
  is fully managed with no user-accessible infrastructure layer

#### Path B: AWS CLI Not Available

Search the FIS actions reference documentation:
```
aws___search_documentation(
  search_phrase="AWS FIS actions [SERVICE_NAME] fault injection",
  topics=["reference_documentation"],
  limit=10
)
```

Then read the FIS actions reference page:
```
aws___read_documentation(
  url="https://docs.aws.amazon.com/fis/latest/userguide/fis-actions-reference.html",
  max_length=10000
)
```

Scan the page content for the search keyword(s) from Step 1.

#### Decision Point: FIS Actions Found?

Count the number of **service-specific** actions found (exclude cross-cutting actions).

- **YES (1+ service-specific actions found)** -> Continue to Step 4 (FIS-Enriched Path)
- **NO (zero service-specific actions)** -> Jump to Step 5 (Documentation-Only Path)

### Step 4: FIS-Enriched Path

When FIS has native actions for the target service, the output should combine
Scenario Library findings with FIS-action-specific details.

#### 4a: Organize FIS Actions into Testing Scenarios

Map each FIS action to a **testing scenario** with a clear HA verification purpose:

```markdown
## FIS Native Fault Injection Scenarios

| # | Testing Scenario | FIS Action | What It Does | HA Verification Purpose |
|---|---|---|---|---|
| 1 | [scenario name] | `aws:xxx:yyy` | [action description] | [what resilience property it validates] |
```

Group scenarios by failure domain:
1. **Instance/Task Level** — individual resource failure
2. **Storage Level** — disk/volume failure or degradation
3. **Network Level** — connectivity disruption
4. **AZ Level** — availability zone failure simulation
5. **Region Level** — cross-region failover
6. **API/Control Plane** — AWS API errors

#### 4b: Enrich with Service-Specific Capabilities

Some services have **built-in fault injection** beyond FIS. Search for these:

```
aws___search_documentation(
  search_phrase="[SERVICE_NAME] fault injection testing failover simulation",
  topics=["general", "reference_documentation"],
  limit=10
)
```

Known built-in capabilities to check:
- **Aurora MySQL/PostgreSQL**: `ALTER SYSTEM CRASH`, `ALTER SYSTEM SIMULATE READ REPLICA FAILURE`,
  `ALTER SYSTEM SIMULATE DISK FAILURE`, `ALTER SYSTEM SIMULATE DISK CONGESTION`
- **RDS Multi-AZ**: Reboot with Failover option
- **ElastiCache**: Test Failover API
- **DynamoDB**: On-demand backup/restore testing

If found, add a separate section:

```markdown
## Service Built-in Fault Injection

| # | Method | Command/API | What It Simulates | Notes |
|---|---|---|---|---|
```

#### 4c: Deep Documentation Research via AWS Knowledge MCP

Use **only** `aws___search_documentation` and `aws___read_documentation` for this step.
Run **5 sequential searches** across different topic dimensions to get comprehensive coverage
of official docs, reference pages, blogs, and troubleshooting guides:

```
Search 1 (blogs & best practices):
  search_phrase: "[SERVICE_NAME] chaos engineering fault injection best practices"
  topics: ["general"]
  limit: 10
  — Targets: AWS Architecture Blog, DevOps Blog, Database Blog articles

Search 2 (official docs & user guides):
  search_phrase: "[SERVICE_NAME] high availability failover testing Multi-AZ"
  topics: ["general"]
  limit: 10
  — Targets: service user guide, HA/DR documentation pages

Search 3 (API & CLI reference):
  search_phrase: "[SERVICE_NAME] resilience testing AWS FIS"
  topics: ["reference_documentation"]
  limit: 10
  — Targets: FIS action reference, API reference, CLI reference pages

Search 4 (troubleshooting & failure modes):
  search_phrase: "[SERVICE_NAME] failover failure troubleshooting recovery"
  topics: ["troubleshooting"]
  limit: 10
  — Targets: repost.aws knowledge center, troubleshooting guides, known issues

Search 5 (Well-Architected & current awareness):
  search_phrase: "[SERVICE_NAME] resilience reliability Well-Architected"
  topics: ["general"]
  limit: 10
  — Targets: Well-Architected reliability pillar, new features, Resilience Hub
```

After searches complete, read the top 3-5 most relevant pages using
`aws___read_documentation` with `max_length: 8000`. Priority order:
1. Service-specific chaos engineering / FIS blog post
2. Official HA / failover documentation page
3. Well-Architected guidance for this service
4. Troubleshooting / known failure mode pages

Also use `aws___recommend` on the most relevant page found to discover
related content that may not appear in search results (especially "New"
and "Similar" recommendations).

### Step 5: Documentation-Only Path (No FIS Actions)

When FIS has no native actions for the target service, fall back to comprehensive
documentation research to find alternative testing approaches. Note that Scenario
Library findings from Step 2 still apply — composite scenarios may affect this service
indirectly even without dedicated FIS actions.

#### 5a: Deep Documentation Search via AWS Knowledge MCP

Since FIS has no native actions, thorough documentation research is critical.
Use **only** `aws___search_documentation` and `aws___read_documentation`.

Run **6 sequential searches** to cover all documentation dimensions:

```
Search 1 (HA & failover):
  search_phrase: "[SERVICE_NAME] high availability failover testing"
  topics: ["general"]
  limit: 10
  — Targets: HA architecture docs, failover procedures, Multi-AZ guides

Search 2 (DR & resilience):
  search_phrase: "[SERVICE_NAME] resilience disaster recovery Multi-AZ Multi-Region"
  topics: ["general"]
  limit: 10
  — Targets: DR strategy docs, cross-region replication, backup/restore

Search 3 (chaos & fault injection):
  search_phrase: "[SERVICE_NAME] chaos engineering fault injection testing"
  topics: ["general"]
  limit: 10
  — Targets: any chaos/FIS blog posts mentioning this service

Search 4 (best practices & reliability):
  search_phrase: "[SERVICE_NAME] best practices reliability availability"
  topics: ["general"]
  limit: 10
  — Targets: Well-Architected guidance, service-specific best practices

Search 5 (troubleshooting & failure modes):
  search_phrase: "[SERVICE_NAME] failure troubleshooting recovery error"
  topics: ["troubleshooting"]
  limit: 10
  — Targets: repost.aws knowledge center, known failure modes, recovery steps

Search 6 (API & configuration reference):
  search_phrase: "[SERVICE_NAME] reboot failover replication configuration API"
  topics: ["reference_documentation"]
  limit: 10
  — Targets: API actions that can trigger failover/reboot, config parameters
```

#### 5b: Read Key Pages and Discover Related Content

From the combined search results, read the **top 5 most relevant pages** using
`aws___read_documentation` with `max_length: 8000`. Priority order:

1. **Service HA/resilience overview page** — architecture, failure domains
2. **Failover/recovery documentation** — how the service handles failures
3. **Best practices page** — official recommendations for reliability
4. **Troubleshooting guide** — known failure modes and recovery procedures
5. **Any chaos/FIS blog post** mentioning this service

Then use `aws___recommend` on the service's main documentation page to discover:
- **"New" recommendations** — recently added features (may include new resilience capabilities)
- **"Similar" recommendations** — related HA/DR content
- **"Journey" recommendations** — what other users read next (often testing/monitoring guides)

Extract from all pages:
- **Failure modes** the service can experience
- **Built-in HA mechanisms** (automatic failover, replication, etc.)
- **Testing approaches** documented in official guides
- **Monitoring/metrics** to watch during tests

#### 5c: Compile Alternative Testing Approaches

Since FIS cannot directly inject faults into this service, organize alternatives:

```markdown
## Testing Approaches (No Native FIS Actions)

> **Note:** AWS FIS does not currently have dedicated actions for [SERVICE].
> The following approaches use FIS Scenario Library composite scenarios,
> indirect FIS actions, AWS APIs, and manual methods.

### FIS Scenario Library (from Step 2)

[Include relevant scenarios discovered in Step 2 — these may affect the service
indirectly through network disruption, EC2 stoppage, or other sub-actions]

### Service API / Console Methods

| # | Testing Scenario | Method | What It Validates |
|---|---|---|---|
| 1 | [scenario from docs] | [API/Console/CLI command] | [what it tests] |
```

### Step 6: Compile Output

Regardless of which path was taken, always produce the following sections:

#### Section 1: Executive Summary

A 2-3 sentence overview:
- Target service and its HA architecture
- **Target region** and whether FIS has native support in that region (and how many actions)
- Which FIS Scenario Library scenarios are relevant
- Key testing recommendation

Always state the region in the summary so the user knows the results are region-specific.
Example: "In **us-east-1**, AWS FIS provides 2 native actions for Amazon RDS. The FIS
Scenario Library offers 3 composite scenarios relevant to RDS, including AZ Power
Interruption which directly triggers RDS failover..."

#### Section 2: FIS Scenario Library Scenarios (from Step 2)

Present the relevant scenarios from the Scenario Library. This section has the highest
priority because these are AWS-curated, production-tested composite scenarios.

```markdown
## FIS Scenario Library

| # | Scenario | Relevance | Sub-Actions for [SERVICE] | Resource Tags Required | Duration |
|---|---|---|---|---|---|
| 1 | AZ Power Interruption | Direct — triggers [SERVICE] failover | [list sub-actions] | [tags] | [duration] |
| 2 | AZ Application Slowdown | Indirect — degrades network for [SERVICE] | [list sub-actions] | [tags] | [duration] |
```

For each scenario, include:
- **Why it matters for [SERVICE]**: explain the specific impact
- **Prerequisites**: what the user needs to prepare (tags, IAM roles, etc.)
- **Limitations**: any caveats from the documentation

If no Scenario Library scenarios are relevant to the target service, state this
explicitly and explain why.

#### Section 3: Testing Scenario Matrix (from Step 4 or Step 5)

The organized scenario tables of individual FIS actions or alternative approaches.

#### Section 4: Recommended Test Priority

Rank all scenarios by priority:

```markdown
## Recommended Test Priority

| Priority | Scenario | Reason |
|---|---|---|
| **P0 Must Test** | [scenario] | [why it's critical] |
| **P1 High** | [scenario] | [why it's important] |
| **P2 Medium** | [scenario] | [good to have] |
| **P3 Optional** | [scenario] | [edge case or advanced] |
```

Priority guidelines:
- **P0**: Scenario Library composite scenarios that directly affect the service — the
  most realistic end-to-end resilience tests
- **P0**: Failover / primary failure — directly impacts RTO
- **P1**: AZ-level failure, network isolation — impacts multi-AZ resilience
- **P2**: Performance degradation, read replica failure — impacts read availability
- **P3**: API throttling, cross-region DR, cross-cutting actions — advanced scenarios

#### Section 5: Implementation Best Practices

Include these guidelines tailored to the specific service:
- **Stop Conditions**: Which CloudWatch metrics/alarms to use as guardrails
  (be specific to the service, e.g., `DatabaseConnections`, `ReplicaLag` for RDS)
- **Steady State Definition**: What metrics define "normal" for this service
- **DNS/Connection Handling**: Service-specific reconnection considerations
  (e.g., JVM TTL for RDS, connection pool recycle for databases)
- **Blast Radius**: How to start small and expand
- **Monitoring**: Which CloudWatch dashboards/metrics to watch during experiments

#### Section 6: Reference Materials

```markdown
## Reference Materials

| # | Type | Title | Link |
|---|---|---|---|
```

Only include URLs from actual search results or documentation pages read. Never
fabricate links.

#### Section 7: Next Steps

Suggest 3-4 actionable next steps tailored to the service, for example:
- "I can generate a complete FIS experiment template (JSON) for [specific scenario]."
- "I can walk you through setting up the [Scenario Library scenario] in the FIS console."
- "I can help design CloudWatch alarms to use as Stop Conditions."
- "I can compare Multi-AZ vs Multi-Region HA approaches for [SERVICE]."

## Important Guidelines

- **Scenario Library first, always.** The FIS Scenario Library represents AWS's own
  curated resilience testing scenarios. These composite experiments are the most
  realistic and valuable because they simulate real-world failure patterns (e.g., an
  AZ power outage affects compute, network, and database simultaneously). Always read
  the latest Scenario Library documentation before anything else.
- **Scenario Library is documentation-based.** Unlike FIS actions which can be queried
  via `list-actions`, Scenario Library scenarios are console-only and can only be
  discovered by reading AWS documentation. Step 2 is therefore a documentation fetch
  step, not a CLI step. This makes it especially important — if you skip it, you miss
  the most valuable testing scenarios.
- **Region matters.** Always resolve the target region before querying FIS actions.
  FIS action availability varies by region — an action available in `us-east-1` may
  not exist in `ap-southeast-1`. Always pass `--region` to the AWS CLI and clearly
  state the region in the output so the user knows the results are region-specific.
- **Don't fabricate FIS actions.** If an action doesn't exist, say so clearly. The
  fallback path exists precisely for services FIS doesn't cover.
- **Don't fabricate links.** Only include URLs from actual search results or known
  documentation pages you've read.
- **Be specific about the service.** Generic "test your database" advice is not useful.
  Every recommendation should reference the specific service, its HA mechanisms, and
  its specific metrics.
- **Cross-cutting actions are optional context.** Network disruption, API fault
  injection, and EC2 stop actions can affect almost any service indirectly. Include
  them when they add value for the user's specific scenario, but don't treat them as
  mandatory — focus on service-specific actions and Scenario Library scenarios first.
- **AWS Knowledge MCP only for docs research.** Do NOT use SearXNG or other web search.
  All documentation research (Steps 4c and 5a) must go through `aws___search_documentation`,
  `aws___read_documentation`, and `aws___recommend`. These tools cover official docs,
  blogs, reference pages, troubleshooting guides, and What's New — which is sufficient.
- **Search across multiple topics.** Use different `topics` values (`general`,
  `reference_documentation`, `troubleshooting`) sequentially to cover blogs, user guides,
  API references, and troubleshooting content.
- **Use aws___recommend for discovery.** After reading a key page, call `aws___recommend`
  to find related content that keyword search may miss.
- **Run searches sequentially.** AWS Knowledge MCP tools do not support parallel
  calls. Execute all queries in Steps 4c and 5a one at a time.
- **Respect language.** Output in the same language as the user's conversation.
