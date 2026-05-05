---
name: extended-support-audit
description: Audits AWS resources for extended support charges and end-of-life version risk. Scans running resources, cross-references with AWS EOL schedules, and produces a status table showing what's incurring extra charges or approaching EOL. Use when the user says "extended support audit", "EOL check", "show me resources in extended support", "version support status", "find unsupported versions", "extended support costs", or asks about specific services like "show my Redis clusters at risk", "RDS extended support status", "Lambda runtime EOL check", "EKS version support".
version: "0.1.0"
keywords: [aws, extended-support, eol, end-of-life, cost-optimization, elasticache, rds, lambda, eks, opensearch, documentdb, neptune, msk, version-audit, lifecycle]
author: shawnsmc@amazon.com
license: MIT
---

# Extended Support Audit

Scans AWS resources for versions that have entered (or are approaching) extended support / end-of-life, cross-references with official AWS lifecycle documentation, and produces a comprehensive status table with cost impact.

## Supported Services

| Service | What's Checked | EOL Dimension |
|---------|---------------|---------------|
| ElastiCache | Clusters & replication groups | Redis OSS major version |
| RDS | DB instances & clusters | Engine major version (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server) |
| Lambda | Functions | Runtime identifier (python3.8, nodejs16.x, etc.) |
| EKS | Clusters | Kubernetes version |
| OpenSearch | Domains | Engine version |
| DocumentDB | Clusters | Engine version |
| Neptune | Clusters | Engine version |
| MSK | Clusters | Kafka version |

## Workflow

### 1. Identify Target Service

Parse the user's request to determine which service(s) to audit. If ambiguous, ask. If the user says "all" or "full audit", iterate through all supported services.

### 2. Fetch EOL Schedule from AWS Documentation (MUST complete before Step 3)

Use `aws.search_documentation` and `aws.read_documentation` to find the current extended support / EOL schedule for the target service. **This step is mandatory and must complete before scanning the environment.** Store the URLs and extracted dates — they will be rendered in the output's Sources section.

Key search terms per service:

| Service | Search Phrase |
|---------|--------------|
| ElastiCache | "ElastiCache extended support versions lifecycle" |
| RDS | "RDS extended support versions end of standard support" |
| Lambda | "Lambda runtime end of support deprecation policy" |
| EKS | "EKS Kubernetes version end of support lifecycle" |
| OpenSearch | "OpenSearch Service end of life versions" |
| DocumentDB | "DocumentDB end of life version support" |
| Neptune | "Neptune engine version end of life" |
| MSK | "MSK Kafka version end of support" |

Extract and structure:
- Version identifiers
- End of standard support date
- Extended support start/end dates (if applicable)
- Premium tiers and percentages (if applicable)
- Recommended upgrade target

**Always cite the documentation URLs used.**

### 3. Scan the Environment

Use `aws.run_script` to query the account for running resources of the target service. Use the appropriate Describe/List API:

| Service | API Call | Key Fields |
|---------|----------|------------|
| ElastiCache | `DescribeCacheClusters` + `DescribeReplicationGroups` | Engine, EngineVersion, CacheNodeType, Status |
| RDS | `DescribeDBInstances` + `DescribeDBClusters` | Engine, EngineVersion, DBInstanceClass, Status |
| Lambda | `ListFunctions` | Runtime, FunctionName, LastModified |
| EKS | `ListClusters` + `DescribeCluster` per cluster | Version, Status |
| OpenSearch | `ListDomainNames` + `DescribeDomains` | EngineVersion, InstanceType |
| DocumentDB | `DescribeDBClusters` (filter engine=docdb) | EngineVersion, DBClusterIdentifier |
| Neptune | `DescribeDBClusters` (filter engine=neptune) | EngineVersion, DBClusterIdentifier |
| MSK | `ListClustersV2` | KafkaVersion, ClusterName |

For multi-region audits, iterate all opted-in regions. Default to the session's current region unless the user specifies otherwise or says "all regions".

### 4. Cross-Reference Versions

For each discovered resource, classify its support status:

| Status | Meaning | Icon |
|--------|---------|------|
| Standard Support | Version is fully supported, no extra charges | ✅ |
| Approaching EOL | Standard support ends within 6 months | ⚠️ |
| Extended Support Y1 | Past EOL, in year 1 of extended support | 🔴 |
| Extended Support Y2 | Past EOL, in year 2 of extended support | 🔴 |
| Extended Support Y3 | Past EOL, in year 3 (highest premium) | 🚨 |
| Deprecated / Unsupported | Past all support (Lambda runtimes, etc.) | ⛔ |

Use today's date to determine which tier applies.

### 5. Calculate Cost Impact (MANDATORY — never skip)

**Cost estimation is required for every audit, even if the user didn't explicitly ask for it.** Resources in extended support are incurring charges the customer may not be aware of. Always surface the cost.

For services with tiered extended support pricing (ElastiCache, RDS):
- Look up the on-demand hourly rate for the node type via `aws.search_documentation` or the AWS pricing page
- Apply the premium percentage for the current tier:
  - ElastiCache: 20% surcharge in Y1, 50% in Y2+
  - RDS: 20% surcharge in Y1, 50% in Y2+
- Calculate: `monthly_surcharge = hourly_rate × 730 × premium_pct × node_count`
- If exact pricing is unavailable, provide a range estimate and note the assumption

For Lambda: no direct extended support charge, but deprecated runtimes cannot be updated and may lose security patches. Flag as **operational risk** with estimated remediation effort.

For EKS: extended support is **$0.60/cluster/hour** after standard support ends. Calculate: `monthly_surcharge = 0.60 × 730 × cluster_count`.

For OpenSearch, DocumentDB, Neptune, MSK: note whether extended support pricing applies and look it up if so.

**If pricing data is unavailable:** state the formula and note what data is needed to complete the estimate. Never omit the cost section — an incomplete estimate is better than no estimate.

### 6. Produce Output

Render a comprehensive report with ALL of the following sections (do not skip any):

1. **Sources & Lifecycle Reference** — MANDATORY. For each service audited, include:
   - The official AWS documentation URL(s) consulted
   - The key EOL dates extracted from those pages
   - Format as a table: `| Service | Source URL | Key Dates |`
   - This section must appear before the environment scan results so the reader can verify the classification logic

2. **Environment Status Table** — every resource with version, node type, status classification, and monthly cost impact column. The cost column is required — use "N/A" only for services with no extended support charge (Lambda), never leave it blank.

3. **Cost Summary** — MANDATORY. Total monthly surcharge across all resources in extended support. Format:
   ```
   💰 Estimated Monthly Extended Support Charges
   ├── ElastiCache: $X/month (N nodes)
   ├── RDS: $X/month (N instances)
   ├── EKS: $X/month (N clusters)
   └── Total: $X/month (~$X/year)
   ```
   If estimates are incomplete, show what's known and flag what's missing.

4. **Summary** — counts by status category, timeline pressure points (resources approaching EOL in next 90 days)

5. **Recommendations** — upgrade paths, quick wins (delete unused resources), timeline urgency ranked by cost impact

**The Sources & Lifecycle Reference and Cost Summary sections are non-negotiable.** If documentation lookup fails or pricing is unavailable, state that explicitly rather than omitting the section.

### 7. Offer Follow-ups

After presenting results, offer:
- "Want me to check another service?"
- "Want me to run this across all regions?"
- "Want me to save this as a report?" (see Export Options below)
- "Want me to estimate upgrade effort?"

## Export Options (Optional)

When the user asks to save or export results, or says "give me the CSV", "export as JSON", "save this report", produce the output in the requested format and write it to a file.

### Formats

**JSON** — full structured data, suitable for programmatic processing:
```json
{
  "audit_date": "YYYY-MM-DD",
  "region": "us-east-1",
  "services_audited": ["elasticache", "rds"],
  "resources": [
    {
      "service": "elasticache",
      "resource_id": "my-cluster",
      "version": "6.2",
      "node_type": "cache.r6g.large",
      "status": "Extended Support Y1",
      "eol_date": "2024-12-31",
      "monthly_surcharge_usd": 45.20,
      "recommendation": "Upgrade to Redis 7.x"
    }
  ],
  "cost_summary": {
    "total_monthly_usd": 45.20,
    "total_annual_usd": 542.40,
    "by_service": {"elasticache": 45.20}
  }
}
```

**CSV** — flat table, suitable for spreadsheets:
```
service,resource_id,version,node_type,status,eol_date,monthly_surcharge_usd,recommendation
elasticache,my-cluster,6.2,cache.r6g.large,Extended Support Y1,2024-12-31,45.20,Upgrade to Redis 7.x
```

**Markdown** — formatted report, suitable for sharing or pasting into docs (default inline format).

### File Writing

When writing to a file:
1. Ask for the output path if not specified, or suggest a sensible default: `./extended-support-audit-YYYY-MM-DD.<ext>`
2. Use the `write` tool to write the file
3. Confirm the file was written and show the full path

If the user has a project configured with an OneDrive path, offer to save there instead.

## Parameters

| Parameter | Source | Default |
|-----------|--------|---------|
| Service | Parsed from user query | Ask if ambiguous |
| Region | User-specified or session default | Current region |
| Multi-region | User says "all regions" | Single region |
| Output format | User request | Markdown (inline) |
| Output path | User request or suggested default | `./extended-support-audit-YYYY-MM-DD.<ext>` |

## Tool Dependencies

- `aws.search_documentation` — find EOL schedule pages
- `aws.read_documentation` — extract version tables and dates
- `aws.run_script` — scan environment (parallel API calls)
- `aws.call_aws` — single targeted queries if needed
- `write` — save exported output to a file (JSON, CSV, or Markdown)

## Example Invocations

- "Show me my Redis clusters that are in extended support"
- "RDS extended support status"
- "Find Lambdas using unsupported runtimes"
- "EKS version check"
- "Full extended support audit across all services"
- "What's costing me extra due to EOL versions?"
- "Which resources are approaching end of standard support?"
- "Export the results as CSV"
- "Save the audit as JSON to ./audit-results.json"
- "Give me a full audit and save it as a report"
