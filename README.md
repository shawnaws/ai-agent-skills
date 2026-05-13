# AI Agent Skills

A collection of reusable AI agent skills for AWS practitioners, solutions architects, and builders. Each skill gives your AI agent specialized knowledge and workflows for common cloud and field tasks — from auditing AWS environments to analyzing customer workloads.

Skills are plain markdown files. Drop them into any AI agent that supports skill/instruction files and they work immediately.

## What are Skills?

Skills are structured markdown files that give AI agents specialized context, workflows, and decision logic for specific tasks. When a skill is loaded, the agent knows:

- When to activate (trigger phrases and scenarios)
- What steps to follow (workflow)
- What tools to use (API calls, documentation lookups)
- How to format output (tables, reports, recommendations)

No code to deploy. No infrastructure to manage. Just markdown that makes your agent smarter.

## Available Skills

| Skill | Description |
|-------|-------------|
| [extended-support-audit](./extended-support-audit) | Audit AWS resources for extended support charges and EOL version risk across ElastiCache, RDS, Lambda, EKS, OpenSearch, DocumentDB, Neptune, and MSK |
| [elasticache-usage-audit](./elasticache-usage-audit) | Inventory ElastiCache clusters by engine (Redis vs Valkey), architecture (Graviton vs x86), and tag — surfaces modernization opportunities and cost breakdown |
| [network-and-transfer-cost-audit](./network-and-transfer-cost-audit) | Audit AWS networking and data transfer costs — NAT Gateway, public IPv4, Transit Gateway, PrivateLink, VPC peering, inter-AZ, inter-region, and internet egress — with progressive investigation through Cost Explorer, CloudWatch, and VPC Flow Logs |

## Installation

### Kiro / kiro-cli

Place the skill's `SKILL.md` under your agent's custom skills directory:

```bash
~/.holocron/agents/<agent>/custom-skills/<skill-name>/SKILL.md
```

Skills are loaded automatically via glob on agent spawn.

### Cursor

Copy `SKILL.md` to your Cursor rules directory:

```bash
.cursor/rules/<skill-name>.mdc
```

### Claude Code

Add to your project instructions or as a slash command:

```bash
.claude/commands/<skill-name>.md
# or reference from .claude/CLAUDE.md
```

### Windsurf / Codex / OpenCode

Place in the appropriate skills directory for your tool:

```bash
.windsurf/skills/<skill-name>/SKILL.md
.codex/skills/<skill-name>/SKILL.md
.opencode/skills/<skill-name>/SKILL.md
```

### AgentCore / Strands

Bundle the skill as a tool definition or instruction set in your agent's configuration. The skill's workflow section maps directly to agent instructions.

## Skill Structure

Each skill lives in its own directory:

```
<skill-name>/
├── SKILL.md     # The skill definition (required)
└── README.md    # Human-readable docs, requirements, examples (recommended)
```

`SKILL.md` follows a standard frontmatter format:

```markdown
---
name: skill-name
description: One-sentence description the agent uses to decide when to load this skill.
---

# Skill Title

Workflow, tool dependencies, output format, examples.
```

The `description` field is critical — it's what the agent reads to decide whether to activate the skill for a given request. Write it to match the natural language a user would use.

## Tool Dependencies

Skills in this repo use standard tool interfaces. Check each skill's README for its specific requirements.

Common dependencies:

| Tool | Purpose |
|------|---------|
| AWS MCP Server | AWS API calls, documentation lookup, environment scanning |
| Web search | Current pricing, EOL schedules, external documentation |
| File read/write | Saving reports to local paths |

## Contributing

Skills should be:

- **Generic** — no customer-specific data, paths, or configurations
- **Self-contained** — all context the agent needs is in the skill file
- **Tool-agnostic** — use logical tool names or document dependencies clearly
- **Documented** — include a README with requirements, installation, and usage examples

To add a skill:

1. Create a directory: `<skill-name>/`
2. Add `SKILL.md` with frontmatter and workflow
3. Add `README.md` with requirements and installation instructions
4. Open a PR

## License

MIT — use, share, and adapt freely.
