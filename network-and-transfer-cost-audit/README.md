# Network and Transfer Cost Audit

Identifies and attributes AWS networking and data transfer costs — NAT Gateway, public IPv4 addresses, Transit Gateway, PrivateLink interface endpoints, VPC peering, inter-AZ and inter-region transfer, and internet egress. Uses a progressive investigation approach — starting with Cost Explorer (no setup required) and escalating to CloudWatch and VPC Flow Logs for deeper workload attribution.

## Common Scenarios

- "Why is my networking bill so high?"
- NAT Gateway charges that are hard to attribute
- Inter-AZ transfer costs appearing under "EC2-Other"
- PrivateLink / VPC endpoint data processing as a top cost driver
- S3/DynamoDB traffic routing through NAT Gateway instead of free VPC endpoints
- Idle public IPv4 addresses or Transit Gateway attachments
- Identifying which workloads are generating the most cross-AZ traffic

## Investigation Phases

| Phase | Data Source | Setup Required | Depth |
|-------|------------|----------------|-------|
| 1 | Cost Explorer | None | Category-level costs, 90-day trends, inventory checks |
| 2 | CloudWatch NAT Gateway metrics | None | Per-NAT-GW traffic breakdown |
| 3 | VPC Flow Logs | Temporary enablement (24–48 hrs) | Source/destination workload attribution |

The skill guides you through each phase progressively. Phase 1 is always run first. Phases 2 and 3 are escalated based on cost thresholds or when deeper attribution is needed.

## Requirements

- **AWS credentials** — read access to Cost Explorer, EC2, VPC, and CloudWatch. Specific permissions:
  - `ce:GetCostAndUsage`
  - `ec2:DescribeNatGateways`, `ec2:DescribeAddresses`, `ec2:DescribeTransitGatewayAttachments`
  - `ec2:DescribeVpcEndpoints`
  - `ec2:DescribeFlowLogs`
  - `cloudwatch:GetMetricStatistics`
  - `logs:StartQuery`, `logs:GetQueryResults` (for Flow Logs analysis)
- **AWS MCP Server** — provides `run_script` and `call_aws` tools

## VPC Flow Logs Note

Phase 3 (flow log analysis) requires VPC Flow Logs to be enabled. The skill includes step-by-step guidance for temporary enablement (24–48 hours) on the highest-cost VPC. Flow logs have an ingestion cost (~$0.50/GB to CloudWatch Logs) — the skill reminds you to disable them after analysis.

## Installation

### Kiro / kiro-cli

```bash
~/.holocron/agents/<agent>/custom-skills/network-and-transfer-cost-audit/SKILL.md
```

### Cursor

```bash
.cursor/rules/network-and-transfer-cost-audit.mdc
```

### Claude Code

```bash
.claude/commands/network-and-transfer-cost-audit.md
```

### Windsurf / Codex / OpenCode

```bash
.windsurf/skills/network-and-transfer-cost-audit/SKILL.md
```

## Usage

```
"Why is my data transfer bill so high?"
→ Runs Phase 1 Cost Explorer analysis, identifies top categories

"Break down my NAT Gateway costs"
→ Runs Phase 1 + Phase 2 CloudWatch metrics, ranks NAT GWs by traffic

"What's driving my PrivateLink costs?"
→ Runs Phase 1 + 1f, ranks interface endpoints by bytes processed

"Enable flow logs and find what's generating cross-AZ traffic"
→ Guides through Phase 3 flow log enablement and analysis

"Export the results as CSV"
→ Produces CSV output and writes to file
```

## Typical Findings

- S3/DynamoDB traffic routing through NAT Gateway (fix: free gateway VPC endpoints)
- PrivateLink data processing as the #1 line item in high-volume accounts
- Idle NAT Gateways with hourly charges but no traffic
- Unattached Elastic IPs accruing $3.60/mo each
- Orphaned Transit Gateway attachments at $36/mo with zero bytes
- Cross-AZ database reads from application tier in a different AZ
- Direct EC2 internet egress that could be served cheaper via CloudFront
