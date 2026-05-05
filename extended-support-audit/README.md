# Extended Support Audit

Audits AWS resources for extended support charges and end-of-life version risk. Scans running resources, cross-references with AWS lifecycle documentation, and produces a status table with cost impact and upgrade recommendations.

## Supported Services

ElastiCache, RDS, Lambda, EKS, OpenSearch, DocumentDB, Neptune, MSK

## Requirements

- **AWS MCP Server** — provides `run_script`, `call_aws`, `search_documentation`, and `read_documentation` tools. The agent uses these to query AWS APIs and fetch current EOL schedules.
  - Docs: https://docs.aws.amazon.com/aws-mcp/latest/userguide/what-is-mcp-server.html
- **AWS credentials** — the MCP server must have valid credentials for the target account(s). No specific IAM policy is bundled; the agent needs read access to the services being audited (e.g., `elasticache:DescribeCacheClusters`, `rds:DescribeDBInstances`, etc.).

## Installation

### Kiro

Place the skill directory under your agent's custom skills path:

```
~/.holocron/agents/<agent>/custom-skills/extended-support-audit/
├── SKILL.md
└── README.md
```

Kiro loads custom skills automatically via glob. Ensure the AWS MCP server is configured in your agent's `agent.json` under `mcpServers`.

### Cursor

Add the `SKILL.md` content as a rule file:

```
.cursor/rules/extended-support-audit.mdc
```

Configure the AWS MCP server in `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "aws": {
      "command": "uvx",
      "args": ["awslabs.mcp-server-aws@latest"]
    }
  }
}
```

### Claude Code

Place as a slash-command or project instruction:

```
.claude/commands/extended-support-audit.md
```

Or add to `.claude/CLAUDE.md` as a referenced skill. Configure the AWS MCP server in `.claude/mcp.json` (same format as Cursor).

### AgentCore Runtime

Bundle the skill as a tool definition or instruction set in your agent's configuration. The AWS MCP server should be registered as a tool provider in the agent's runtime environment. Refer to AgentCore documentation for tool registration patterns.

## Usage

Invoke with natural language:

- "Run an extended support audit on ElastiCache"
- "Check my RDS instances for EOL versions"
- "Full extended support audit across all services"
- "Which resources are costing me extra due to EOL?"

## How It Works

1. Fetches current EOL/lifecycle schedules from official AWS documentation
2. Scans the target account for running resources via AWS APIs
3. Cross-references discovered versions against the lifecycle schedule
4. Classifies each resource (Standard Support → Approaching EOL → Extended Support Y1/Y2/Y3 → Deprecated)
5. Produces a report with sources, status table, cost impact, and upgrade recommendations

No static scripts are embedded — the agent generates API calls dynamically from the skill instructions, adapting to the target environment and handling API variations at runtime.
