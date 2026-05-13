# ElastiCache Usage Audit

Inventories ElastiCache clusters across an AWS account, classifying them by engine type (Redis OSS vs Valkey), node architecture (Graviton vs x86), and optionally grouping by a user-specified resource tag. Surfaces modernization opportunities and cost context.

## Common Scenarios

- "Which of my clusters are still on Redis vs Valkey?"
- "How much of my ElastiCache fleet is on Graviton?"
- "Break down ElastiCache clusters by the `team` tag"
- "Find clusters that should migrate to Graviton"
- "What's my ElastiCache spend by node type?"

## What It Produces

| Section | Content |
|---------|---------|
| Inventory summary | Total clusters, nodes, engine split, architecture split |
| Cluster detail table | Per-cluster: engine, version, node type, arch, shards, replicas, region, tag |
| Architecture breakdown | Graviton3 / Graviton2 / x86 with cluster and node counts |
| Engine breakdown | Valkey / Redis OSS / Memcached with counts |
| Tag grouping | Clusters grouped by user-specified tag (e.g., team, project) |
| Cost summary | Last 30 days spend by node type and region |
| Modernization opportunities | Graviton migration, Valkey migration, generation upgrade candidates |

## Requirements

- **AWS credentials** — read access to ElastiCache and Cost Explorer:
  - `elasticache:DescribeReplicationGroups`
  - `elasticache:DescribeCacheClusters`
  - `elasticache:ListTagsForResource`
  - `ce:GetCostAndUsage`
- **AWS MCP Server** — provides `run_script` and `call_aws` tools

## Installation

### Kiro / kiro-cli

```bash
~/.holocron/agents/<agent>/custom-skills/elasticache-usage-audit/SKILL.md
```

### Cursor

```bash
.cursor/rules/elasticache-usage-audit.mdc
```

### Claude Code

```bash
.claude/commands/elasticache-usage-audit.md
```

### Windsurf / Codex / OpenCode

```bash
.windsurf/skills/elasticache-usage-audit/SKILL.md
```

## Usage

```
"Audit my ElastiCache usage"
→ Full inventory: engine, architecture, cost breakdown

"Show me clusters by the team tag"
→ Groups clusters by tag:team, shows Redis/Valkey/Graviton split per group

"Which clusters should move to Graviton?"
→ Lists x86 clusters with recommended Graviton equivalents

"Show me ElastiCache clusters tagged project=payments"
→ Filters to a specific tag value
```

## Typical Findings

- Redis OSS clusters eligible for zero-downtime Valkey migration
- x86 nodes (r5, m5, t3) that could save ~20% by moving to Graviton (r7g, m7g)
- Graviton2 clusters that could upgrade to Graviton3 for better performance
- Untagged clusters with no ownership attribution
