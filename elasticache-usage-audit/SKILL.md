---
name: elasticache-usage-audit
description: Audits ElastiCache usage across an AWS account — inventories clusters by engine (Redis vs Valkey), node type (Graviton vs non-Graviton), Extended Support exposure, and optionally groups by a user-specified tag with per-group cost attribution. Surfaces modernization opportunities (Graviton migration at ~20% savings, Valkey migration at ~20% savings, Extended Support elimination) and cost breakdown. Use when the user says "ElastiCache usage", "ElastiCache inventory", "Redis vs Valkey", "Graviton ElastiCache", "ElastiCache costs", "cache cluster audit", "which clusters are still on Redis", "ElastiCache by tag", or "Extended Support charges".
version: "0.2.0"
keywords: [aws, elasticache, redis, valkey, graviton, cost-optimization, inventory, tags, cache, extended-support]
author: shawnsmc@amazon.com
license: MIT
---

# ElastiCache Usage Audit

Inventories ElastiCache clusters across an AWS account, classifying them by engine type (Redis OSS vs Valkey), node architecture (Graviton vs x86), and Extended Support exposure. Optionally groups by a user-specified resource tag with per-group cost attribution derived from Cost Explorer actuals. Surfaces modernization opportunities and quantifies savings.

## Parameters

| Parameter | Source | Default |
|-----------|--------|---------|
| Region(s) | User-specified or session default | All active regions |
| Tag key | User-specified (e.g., `team`, `project`, `environment`) | None (skip tag grouping) |
| Tag value filter | User-specified (optional — filter to specific tag value) | None (show all values) |

## Investigation Workflow

### Phase 1: Cluster Inventory

#### 1a. Discover All ElastiCache Clusters

Query all regions (or user-specified regions) for replication groups and standalone clusters:

```python
import asyncio

# Use replication groups as the primary view — they represent logical clusters
# Standalone cache clusters (non-replicated) are picked up separately
regions = active_regions  # e.g., ['us-east-1', 'us-west-2', 'eu-west-1']

rg_results = await asyncio.gather(*[
    call_boto3(
        service_name='elasticache',
        operation_name='DescribeReplicationGroups',
        region_name=r,
        params={}
    )
    for r in regions
], return_exceptions=True)

# Also get standalone (non-replicated) Memcached clusters if present
cluster_results = await asyncio.gather(*[
    call_boto3(
        service_name='elasticache',
        operation_name='DescribeCacheClusters',
        region_name=r,
        params={'ShowCacheNodeInfo': True}
    )
    for r in regions
], return_exceptions=True)
```

> **Note:** `DescribeReplicationGroups` returns Redis/Valkey clusters (which are always replication groups, even single-node). `DescribeCacheClusters` returns individual nodes — use it to catch Memcached clusters and to get node-level detail for replication groups.

#### 1b. Classify by Engine

ElastiCache engines as of 2024+:

| Engine Value | Meaning | Notes |
|---|---|---|
| `redis` | Redis OSS | Legacy — AWS recommends migration to Valkey |
| `valkey` | Valkey (Redis-compatible fork) | AWS-recommended, no license fees, same API |
| `memcached` | Memcached | Different use case — note but don't compare to Redis/Valkey |

Extract from replication group response:
- `Engine` field → `"redis"` or `"valkey"`
- `EngineVersion` → e.g., `"7.1"`, `"7.2"`, `"8.0"` (Valkey starts at 7.2+)

> **Important:** `DescribeReplicationGroups` does NOT reliably return `EngineVersion` for all clusters. Always cross-reference with `DescribeCacheClusters` which returns per-node engine version. Build a `version_map` from cache cluster responses keyed by `ReplicationGroupId`.

For each cluster, record:
- Replication group ID (logical cluster name)
- Engine + engine version (from cache cluster cross-reference)
- Number of node groups (shards) — from `NodeGroups` array length
- Number of replicas per shard — from `NodeGroups[].NodeGroupMembers` length minus 1
- Node type (e.g., `cache.r7g.xlarge`, `cache.r6g.large`, `cache.m5.large`)
- Multi-AZ enabled
- Encryption at rest / in transit
- Region
- **Extended Support risk** — flag if engine version is in the ES set (see below)

#### Extended Support Version Detection

These Redis engine versions are in Extended Support (incurring surcharges):

| Version | ES Status | Surcharge |
|---|---|---|
| `5.0.6` | Year 2+ (deep ES) | ~$0.10/node-hr |
| `6.0.5` | Year 1 | ~$0.10/node-hr |
| `6.2.6` | Year 1 | ~$0.10/node-hr |
| `7.0.7` | Entered ES Jan 2025 | ~$0.10/node-hr |

```python
ES_VERSIONS = {'5.0.6', '6.0.5', '6.2.6', '7.0.7'}
```

Extended Support charges appear in Cost Explorer as `<REGION>-ElastiCache:ExtendedSupport-NodeUsage:cache.<type>`. These are **additive** to the base node cost — a cluster on ES pays both the node-hour rate AND the ES surcharge. In the test run, ES charges were 18% of total spend ($910 out of $5,170).

#### 1c. Classify by Architecture (Graviton vs x86)

Node type naming convention:

| Pattern | Architecture | Generation |
|---|---|---|
| `cache.r7g.*`, `cache.m7g.*`, `cache.c7gn.*` | Graviton3 | Current |
| `cache.r6g.*`, `cache.m6g.*`, `cache.c6gn.*`, `cache.t4g.*` | Graviton2 | Previous gen |
| `cache.r6gd.*` | Graviton2 (data-tiering) | Previous gen |
| `cache.t3g.*` | Graviton2 (burstable) | Previous gen |
| `cache.r5.*`, `cache.m5.*`, `cache.r4.*`, `cache.m4.*`, `cache.t3.*`, `cache.t2.*` | x86 (Intel) | Previous gen |

Classification logic:

```python
def classify_architecture(node_type: str) -> dict:
    """Classify a cache node type by architecture and generation."""
    # node_type format: cache.<family>.<size>
    parts = node_type.split('.')
    family = parts[1] if len(parts) >= 2 else ''

    # Graviton families contain 'g' after the generation number
    # e.g., r7g, m7g, c7gn, r6g, m6g, c6gn, r6gd
    is_graviton = bool(re.match(r'^[a-z]+[0-9]+g', family))

    # Extract generation number
    gen_match = re.search(r'[0-9]+', family)
    generation = int(gen_match.group()) if gen_match else 0

    return {
        'is_graviton': is_graviton,
        'architecture': 'Graviton' if is_graviton else 'x86',
        'family': family,
        'generation': generation,
        'graviton_gen': 3 if generation >= 7 and is_graviton else (2 if generation >= 6 and is_graviton else None),
    }
```

#### 1d. Tag-Based Grouping (Optional)

If the user provides a tag key (e.g., `team`, `project`, `cost-center`), fetch tags for each cluster and group results:

```python
# ElastiCache ARN format: arn:aws:elasticache:<region>:<account>:replicationgroup:<rg-id>
# Tags are fetched per-resource via ListTagsForResource

tag_results = await asyncio.gather(*[
    call_boto3(
        service_name='elasticache',
        operation_name='ListTagsForResource',
        region_name=region,
        params={'ResourceName': arn}
    )
    for region, arn in cluster_arns
], return_exceptions=True)

# Group clusters by the specified tag key
from collections import defaultdict
grouped = defaultdict(list)
for cluster, tags in zip(clusters, tag_results):
    tag_value = next(
        (t['Value'] for t in tags.get('TagList', []) if t['Key'] == user_tag_key),
        '(untagged)'
    )
    grouped[tag_value].append(cluster)
```

If the user also provides a tag value filter, only show clusters matching that value.

---

### Phase 2: Cost Context

#### 2a. Cost Explorer Breakdown

Pull ElastiCache spend grouped by usage type to see cost by node type:

```python
from datetime import datetime, timedelta

end = datetime.today().strftime('%Y-%m-%d')
start = (datetime.today() - timedelta(days=30)).strftime('%Y-%m-%d')

ce_elasticache = await call_boto3(
    service_name='ce', operation_name='GetCostAndUsage', region_name='us-east-1',
    params={
        'TimePeriod': {'Start': start, 'End': end},
        'Granularity': 'MONTHLY',
        'Filter': {
            'Dimensions': {
                'Key': 'SERVICE',
                'Values': ['Amazon ElastiCache']
            }
        },
        'Metrics': ['UnblendedCost', 'UsageQuantity'],
        'GroupBy': [
            {'Type': 'DIMENSION', 'Key': 'USAGE_TYPE'},
            {'Type': 'DIMENSION', 'Key': 'REGION'}
        ]
    }
)
```

Usage type patterns for ElastiCache:

| Pattern | Meaning |
|---|---|
| `<REGION>-NodeUsage:cache.<type>` | On-demand node hours |
| `<REGION>-ElastiCache:ExtendedSupport-NodeUsage:cache.<type>` | **Extended Support surcharge** (additive to base node cost) |
| `<REGION>-ReservedCacheNodeUsage:cache.<type>` | Reserved node hours (applied) |
| `<REGION>-BackupUsage` | Backup storage |
| `<REGION>-DataTransfer-*` | Data transfer (cross-AZ replication, etc.) |
| `<REGION>-DataTiering-Data-Stored` | Data tiering (r6gd) SSD storage |

#### 2b. Per-Cluster Cost Attribution (Preferred Method)

Rather than relying on cost allocation tags (which most accounts don't have enabled), derive per-cluster cost from CE actuals:

1. From the CE response, extract the $/node-hr rate for each `(region, node_type)` pair by dividing `UnblendedCost` by `UsageQuantity` for `NodeUsage:cache.<type>` line items
2. Do the same for `ExtendedSupport-NodeUsage:cache.<type>` to get the ES surcharge rate
3. Multiply each cluster's node count × rate × 730 hrs to get monthly cost
4. Roll up by tag group

```python
# Derive $/node-hr from CE actuals
rate_map = {}   # (region, node_type) → $/node-hr
es_rate_map = {}  # (region, node_type) → ES surcharge $/node-hr

for period in ce_resp.get('ResultsByTime', []):
    for g in period.get('Groups', []):
        ut = g['Keys'][0]
        cost = float(g['Metrics']['UnblendedCost']['Amount'])
        usage_hrs = float(g['Metrics']['UsageQuantity']['Amount'])
        if usage_hrs == 0:
            continue

        is_es = 'ExtendedSupport' in ut
        nt_match = re.search(r'NodeUsage:(cache\.[a-z0-9]+\.[a-z0-9]+)', ut)
        if not nt_match:
            continue

        node_type = nt_match.group(1)
        # Determine region from usage type prefix
        region = determine_region_from_prefix(ut)
        rate = cost / usage_hrs
        key = (region, node_type)

        if is_es:
            es_rate_map[key] = rate
        else:
            rate_map[key] = rate

# Apply to each cluster
for c in clusters:
    key = (c['region'], c['node_type'])
    c['monthly_cost'] = rate_map.get(key, 0) * c['total_nodes'] * 730
    c['monthly_es_cost'] = es_rate_map.get(key, 0) * c['total_nodes'] * 730
```

This gives accurate per-cluster cost without requiring cost allocation tags, and enables per-tag-group cost rollup.

#### 2c. Tag-Based Cost Allocation (If Available)

If the user's account has cost allocation tags activated for the tag they specified, query CE with a tag filter:

```python
ce_by_tag = await call_boto3(
    service_name='ce', operation_name='GetCostAndUsage', region_name='us-east-1',
    params={
        'TimePeriod': {'Start': start, 'End': end},
        'Granularity': 'MONTHLY',
        'Filter': {
            'Dimensions': {
                'Key': 'SERVICE',
                'Values': ['Amazon ElastiCache']
            }
        },
        'Metrics': ['UnblendedCost'],
        'GroupBy': [
            {'Type': 'TAG', 'Key': user_tag_key}
        ]
    }
)
```

> **Note:** This only works if the tag is activated as a cost allocation tag in the billing console. If it returns empty, fall back to the inventory-based grouping from Phase 1d and note that cost allocation tags are not enabled for this key.

---

### Phase 3: Produce Output

Render a report with the following sections:

#### 1. Inventory Summary

```
| Metric | Count |
|--------|-------|
| Total clusters (replication groups) | X |
| Redis OSS clusters | X |
| Valkey clusters | X |
| Memcached clusters | X |
| Graviton nodes | X |
| x86 nodes | X |
| Total nodes | X |
```

#### 2. Cluster Detail Table

```
| Cluster ID | Engine | Version | Node Type | Arch | Shards | Replicas/Shard | Region | Multi-AZ | <Tag> |
|---|---|---|---|---|---|---|---|---|---|
```

Sort by: engine (Redis first — migration candidates), then node count descending.

#### 3. Architecture Breakdown

```
| Architecture | Clusters | Nodes | % of Fleet |
|---|---|---|---|
| Graviton3 (r7g/m7g) | X | X | X% |
| Graviton2 (r6g/m6g) | X | X | X% |
| x86 (r5/m5/r4/t3) | X | X | X% |
```

#### 4. Engine Breakdown

```
| Engine | Clusters | Nodes | % of Fleet |
|---|---|---|---|
| Valkey | X | X | X% |
| Redis OSS | X | X | X% |
| Memcached | X | X | X% |
```

#### 5. Tag Grouping (if tag key provided)

```
| <Tag Key> | Clusters | Nodes | Redis | Valkey | Graviton | x86 | Monthly Cost | ES Charges ⚠️ | ES Risk |
|---|---|---|---|---|---|---|---|---|---|
| team-alpha | X | X | X | X | X | X | $X | $X | X |
| team-beta | X | X | X | X | X | X | $X | $X | X |
| (untagged) | X | X | X | X | X | X | $X | $X | X |
```

Include a "Modernization Priority by Group" narrative section ranking groups by urgency (highest ES-to-compute ratio first, then largest x86 node count).

#### 6. Cost Summary (Last 30 Days)

Total ElastiCache spend, broken down by:
- Node type and region (from CE usage types)
- Extended Support charges called out separately with ⚠️ indicator
- Per-tag cost (derived from per-cluster cost attribution in Phase 2b)

Flag any group where ES charges exceed 10% of base node cost — these are paying more in penalties than necessary.

#### 7. Modernization Opportunities

Present as a savings table with estimated monthly savings per opportunity:

- **Extended Support elimination** — Clusters on Redis 5.x/6.x/7.0 are paying ES surcharges (~$0.10/node-hr) on top of base node costs. Upgrading engine version to 7.2+ (or migrating to Valkey) eliminates these immediately. This is the highest-ROI, lowest-risk action — pure penalty elimination with no architectural change.
- **Valkey migration** — Redis OSS → Valkey saves **~20% on node-hour costs** (AWS prices Valkey 20% lower than Redis OSS for self-designed clusters, 33% lower for serverless). API-compatible, zero downtime migration. All Redis clusters are eligible regardless of version.
- **Graviton migration** — x86 → Graviton3 saves **~20% on node-hour costs** with equal or better performance. List each with current type → recommended type (e.g., `cache.m5.large` → `cache.m7g.large`, `cache.t2/t3.*` → `cache.t3g.*`).
- **Generation upgrade candidates** — Graviton2 clusters that could move to Graviton3 for improved performance.

> **Overlap note:** Valkey and Graviton savings are partially stackable. Migrating a Redis m5.large to Valkey on m7g.large captures both the 20% Valkey discount and the Graviton price reduction simultaneously (~35–38% combined). Present them independently in the table but note the combined potential.

#### 8. Savings Summary Table

```
| Opportunity | Est. Monthly Savings | Notes |
|---|---|---|
| Extended Support elimination | $X | Upgrade Redis 5.x/6.x/7.0 clusters |
| Valkey migration (all Redis) | $X | 20% off Redis node-hours |
| Graviton migration (x86 → Graviton3) | $X | ~20% off x86 node-hours |
| Total | $X | (some overlap if done together) |
```

---

### Phase 4: Offer Follow-ups

After presenting results:
- "Want me to filter to a specific tag value?"
- "Want me to estimate savings from migrating x86 → Graviton?"
- "Want me to check which Redis clusters are eligible for Valkey migration?"
- "Want me to check reserved instance coverage for these clusters?"
- "Want me to export this as CSV or JSON?"

## Export Options

**JSON** — full structured data with cluster details, groupings, and cost.

**CSV** — flat table of all clusters with columns for engine, version, node type, architecture, region, tag value, estimated monthly cost.

Write to file. Suggested filename: `./elasticache-audit-YYYY-MM-DD.<ext>`

## Tool Dependencies

- `aws.run_script` — ElastiCache inventory, Cost Explorer queries, tag lookups
- `aws.call_aws` — targeted single queries
- `write` — save exported output to file

## Example Invocations

- "Audit my ElastiCache usage"
- "Show me which clusters are still on Redis vs Valkey"
- "Which ElastiCache clusters are not on Graviton?"
- "Break down ElastiCache by the `team` tag"
- "Show me ElastiCache clusters tagged `project=payments`"
- "What's my ElastiCache spend by node type?"
- "Find Graviton migration candidates in ElastiCache"
- "Which clusters are paying Extended Support charges?"
- "What's the full savings potential if we migrate everything to Valkey + Graviton?"
- "Group by cluster_id and show cost per group"
