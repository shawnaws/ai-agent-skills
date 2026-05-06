---
name: data-transfer-cost-audit
description: Analyzes AWS networking and data transfer costs across public IPv4 addresses, NAT Gateway, Transit Gateway, VPC peering, PrivateLink interface endpoints, inter-AZ, inter-region, and internet egress. Identifies top cost drivers, surfaces VPC endpoint savings opportunities, and guides progressive investigation using Cost Explorer, CloudWatch, and optionally VPC Flow Logs. Use when the user says "data transfer costs", "NAT Gateway charges", "why is my networking bill so high", "inter-AZ transfer costs", "egress costs", "public IPv4 costs", "Transit Gateway charges", "PrivateLink costs", "VPC endpoint charges", "VPC flow logs analysis", "data transfer breakdown", or "what's driving my EC2-Other charges".
version: "0.6.0"
keywords: [aws, data-transfer, nat-gateway, inter-az, egress, cost-optimization, vpc, flow-logs, networking, cost-explorer, privatelink, vpc-peering]
author: shawnsmc@amazon.com
license: MIT
---

# Data Transfer Cost Audit

Identifies and attributes AWS data transfer costs across NAT Gateway processing, inter-AZ transfers, inter-region transfers, and internet egress. Uses a progressive investigation approach — starting with Cost Explorer (always available) and escalating to CloudWatch and VPC Flow Logs for deeper attribution when needed.

## Cost Categories

| Category | Typical Price | Billing Dimension | Common Culprits |
|----------|--------------|-------------------|-----------------|
| Public IPv4 addresses | $0.005/hr per IP | VPC / PublicIPv4 | In-use and idle EIPs, NAT Gateway IPs, ALB IPs |
| NAT Gateway processing | $0.045/GB | EC2-Other / NatGateway-Bytes | EC2 → internet, EC2 → S3 via NAT instead of endpoint |
| NAT Gateway hourly | $0.045/hr | EC2-Other / NatGateway-Hours | Idle or redundant NAT Gateways |
| Transit Gateway | $0.05/hr per attachment + $0.02/GB | VPC / TransitGateway | Unused attachments, cross-region peering |
| VPC Peering | $0.01/GB each way (inter-AZ/cross-region) | EC2-Other / VpcPeering-Bytes | Cross-AZ or cross-region peered traffic |
| Inter-AZ transfer | $0.01/GB each way | EC2-Other / DataTransfer-Regional-Bytes or DataTransfer-xAZ-*-Bytes | Multi-AZ RDS reads, cross-AZ ALB traffic, microservices |
| Inter-region transfer | $0.02–0.09/GB | EC2-Other / `<SRC>-<DST>-AWS-Out-Bytes` | Replication, cross-region APIs, DR traffic |
| Internet egress (EC2) | $0.09/GB (after 100GB free) | EC2-Other / `<REGION>-AWS-Out-Bytes` | Direct EC2 egress, missing CloudFront |
| PrivateLink interface endpoint hours | $0.01/hr per ENI per AZ | VPC / VpcEndpoint-Hours | Endpoints in multiple AZs, unused endpoints |
| **PrivateLink data processing (consumer)** | $0.01/GB (first 1 PB/mo) | VPC / VpcEndpoint-Bytes | High-volume traffic through interface endpoints |
| **PrivateLink service hours (provider)** | $0.01/hr per ENI per AZ | VPC / VpcEndpoint-Service-Hours | Services you publish via PrivateLink |
| Gateway VPC endpoint savings | Avoidable | N/A | S3/DynamoDB traffic routed through NAT (gateway endpoints are free) |

> **Don't overlook PrivateLink data processing.** In high-volume accounts, `VpcEndpoint-Bytes` can be the single largest line item — larger than NAT Gateway processing. See the [usage type reference](references/usage-types.md) for the complete list of patterns seen in real billing data.

## Investigation Workflow

This skill uses a **three-phase progressive approach**. Start with Phase 1 (always available, no setup). Escalate to Phase 2 and 3 only if more attribution is needed.

---

### Phase 1: Cost Explorer Analysis (Always Available)

**Goal:** Identify top cost categories and time trends. No setup required.

> **Billing note:** Public IPv4 address charges and NAT Gateway hourly charges are billed under the **VPC service** (as `PublicIPv4:InUseAddress`, `PublicIPv4:IdleAddress`, `NatGateway:Hours`), not under EC2 usage type groups. The VPC service query (Query 2) is the authoritative source for these line items. Do not rely on `USAGE_TYPE_GROUP` filters for these — they may return empty results depending on account billing configuration.

#### 1a. Pull Overall Data Transfer and VPC Infrastructure Spend

Use `aws.run_script` to query Cost Explorer for the last 3 months. Run both queries in parallel:

```python
import asyncio
from datetime import datetime, timedelta

end = datetime.today().strftime('%Y-%m-%d')
start = (datetime.today() - timedelta(days=90)).strftime('%Y-%m-%d')

ce_transfer, ce_vpc = await asyncio.gather(
    # Query 1: Data transfer (internet egress, region-to-region, CloudFront, NAT data, inter-AZ)
    call_boto3(
        service_name='ce', operation_name='GetCostAndUsage', region_name='us-east-1',
        params={
            'TimePeriod': {'Start': start, 'End': end},
            'Granularity': 'MONTHLY',
            'Filter': {
                'Dimensions': {
                    'Key': 'USAGE_TYPE_GROUP',
                    'Values': [
                        'EC2: Data Transfer - Internet (Out)',
                        'EC2: Data Transfer - Region to Region (Out)',
                        'EC2: Data Transfer - CloudFront (Out)',
                        'EC2: NAT Gateway - Data Processed',
                        'EC2: Data Transfer - Inter AZ',
                    ]
                }
            },
            'Metrics': ['UnblendedCost', 'UsageQuantity'],
            'GroupBy': [
                {'Type': 'DIMENSION', 'Key': 'SERVICE'},
                {'Type': 'DIMENSION', 'Key': 'USAGE_TYPE'}
            ]
        }
    ),
    # Query 2: VPC infrastructure (public IPv4, NAT hourly, TGW, interface endpoints)
    # This is the authoritative source for public IPv4 and NAT hourly charges.
    call_boto3(
        service_name='ce', operation_name='GetCostAndUsage', region_name='us-east-1',
        params={
            'TimePeriod': {'Start': start, 'End': end},
            'Granularity': 'MONTHLY',
            'Filter': {
                'Dimensions': {
                    'Key': 'SERVICE',
                    'Values': ['Amazon Virtual Private Cloud']
                }
            },
            'Metrics': ['UnblendedCost', 'UsageQuantity'],
            'GroupBy': [
                {'Type': 'DIMENSION', 'Key': 'USAGE_TYPE'},
                {'Type': 'DIMENSION', 'Key': 'REGION'}
            ]
        }
    ),
    return_exceptions=True
)
```

From the VPC query results, look for these usage type patterns. **All are region-prefixed** (e.g., `USE1-`, `USW2-`, `EU-`, `EUW1-`):

| Usage Type Pattern | Meaning | Price |
|---|---|---|
| `<REGION>-PublicIPv4:InUseAddress` | In-use public IPv4 | $0.005/hr each |
| `<REGION>-PublicIPv4:IdleAddress` | Idle/unattached EIPs | $0.005/hr each |
| `<REGION>-TransitGateway-Hours` | TGW attachment hourly | $0.05/hr each |
| `<REGION>-TransitGateway-Bytes` | TGW data processing | $0.02/GB |
| `<REGION>-VpcEndpoint-Hours` | Interface endpoint ENI hours (consumer side) | $0.01/hr per ENI |
| `<REGION>-VpcEndpoint-Service-Hours` | Interface endpoint ENI hours (provider side) | $0.01/hr per ENI |
| `<REGION>-VpcEndpoint-Bytes` | **PrivateLink data processing** | $0.01/GB |

> **PrivateLink data processing is often the #1 line item** in accounts that route heavy traffic through interface endpoints (e.g., cross-account service consumption, centralized egress architectures). Do not skip over `VpcEndpoint-Bytes` even if `VpcEndpoint-Hours` looks small.

See [references/usage-types.md](references/usage-types.md) for the complete catalog of data transfer usage types, including EC2-Other patterns covered in the next section.

> **Region prefix examples from real billing data:**
> - `USE1-` → us-east-1
> - `USW2-` → us-west-2
> - `EU-` → eu-west-1 (note: shorter prefix than expected)
> - Other regions follow `<REGION_CODE>-` pattern

**NAT Gateway hourly charges appear under EC2-Other, not VPC service.**

**If NAT Gateway hourly charges are missing:** They appear under EC2-Other (not VPC service) with the same region-prefix pattern:
- `NatGateway-Hours` — us-east-1 (no prefix — unique to us-east-1)
- `USW2-NatGateway-Hours` — us-west-2
- `EU-NatGateway-Hours` — eu-west-1
- Other regions: `<REGION_CODE>-NatGateway-Hours`

Run this diagnostic query if NAT hourly costs appear missing:

```python
ce_nat_ec2other = await call_boto3(
    service_name='ce', operation_name='GetCostAndUsage', region_name='us-east-1',
    params={
        'TimePeriod': {'Start': start, 'End': end},
        'Granularity': 'MONTHLY',
        'Filter': {
            'And': [
                {'Dimensions': {'Key': 'SERVICE',
                    'Values': ['Amazon Elastic Compute Cloud - Compute']}},
                {'Dimensions': {'Key': 'USAGE_TYPE_GROUP',
                    'Values': ['EC2: NAT Gateway - Hours']}}
            ]
        },
        'Metrics': ['UnblendedCost', 'UsageQuantity'],
        'GroupBy': [
            {'Type': 'DIMENSION', 'Key': 'USAGE_TYPE'},
            {'Type': 'DIMENSION', 'Key': 'REGION'}
        ]
    }
)
```

Sum across all returned usage types to get total NAT hourly spend. At $0.045/hr per NAT Gateway, 6 NAT GWs running 24/7 = ~$194/mo — a significant cost if not surfaced by the VPC query.

#### 1b. Identify EC2-Other Data Transfer (Inter-AZ, Inter-Region, Internet Egress, Peering)

EC2-Other contains the bulk of data transfer line items. Rather than relying solely on `USAGE_TYPE_GROUP` filters (which have gaps — e.g., region-to-region pairs may not be fully classified), query the entire EC2 service grouped by usage type and filter the long tail in code.

```python
ce_ec2_transfer = await call_boto3(
    service_name='ce', operation_name='GetCostAndUsage', region_name='us-east-1',
    params={
        'TimePeriod': {'Start': start, 'End': end},
        'Granularity': 'MONTHLY',
        'Filter': {
            'Dimensions': {
                'Key': 'SERVICE',
                'Values': ['Amazon Elastic Compute Cloud - Compute']
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

Then filter the returned usage types in Python to the data-transfer patterns below. **Skip** `EBS:*`, `EBSOptimized:*`, `CPUCredits:*`, `BoxUsage:*`, `SpotUsage:*` — these are compute/storage, not data transfer.

**Data transfer patterns to include** (all may carry a `<REGION>-` prefix except us-east-1, which is often unprefixed):

| Pattern | Meaning | Price | Notes |
|---|---|---|---|
| `NatGateway-Bytes` | NAT Gateway data processing | $0.045/GB | Top culprit for NAT overspend |
| `NatGateway-Hours` | NAT Gateway hourly | $0.045/hr | Sums independently — see 1a diagnostic |
| `DataTransfer-Regional-Bytes` | Inter-AZ (older/common naming) | $0.01/GB each way | Bidirectional — bytes going both directions are billed |
| `DataTransfer-xAZ-In-Bytes` | Inter-AZ ingress (newer naming) | $0.01/GB | Some accounts see `xAZ` instead of `Regional` |
| `DataTransfer-xAZ-Out-Bytes` | Inter-AZ egress (newer naming) | $0.01/GB | — |
| `DataTransfer-AZ-In-Bytes` | Intra-AZ (same AZ) | $0.00/GB | Free — confirm value is $0 |
| `DataTransfer-AZ-Out-Bytes` | Intra-AZ (same AZ) | $0.00/GB | Free — confirm value is $0 |
| `<REGION>-AWS-Out-Bytes` | Internet egress from that region | $0.09/GB (after 100 GB free) | `USE1-AWS-Out-Bytes`, `USW2-AWS-Out-Bytes`, etc. |
| `<REGION>-AWS-In-Bytes` | Inbound from internet | $0.00/GB | Free — confirm value is $0 |
| `<SRC>-<DST>-AWS-Out-Bytes` | Inter-region egress | $0.02–$0.09/GB | e.g. `USW2-USE1-AWS-Out-Bytes`, `EU-USE1-AWS-Out-Bytes` — dozens of pairs possible |
| `VpcPeering-In-Bytes` | VPC peering ingress | $0.01/GB (inter-AZ/cross-region) | Free within same AZ |
| `VpcPeering-Out-Bytes` | VPC peering egress | $0.01/GB | — |

Recommended filter in code:

```python
import re
DT_PATTERNS = re.compile(
    r'(NatGateway-(Bytes|Hours)|'
    r'DataTransfer-(Regional|xAZ|AZ)-(In|Out)?-?Bytes|'
    r'AWS-(In|Out)-Bytes|'
    r'VpcPeering-(In|Out)-Bytes)$'
)
# Match against the part after an optional region prefix like "USW2-" or "USE1-USW2-"
```

> **USAGE_TYPE_GROUP has gaps.** CE's `EC2: Data Transfer - Region to Region (Out)` group does not always classify every `<SRC>-<DST>-AWS-Out-Bytes` line item, especially for newer regions. Querying the whole EC2 service and filtering by pattern catches the long tail.

#### 1c. Idle Elastic IPs and Public IPv4 Inventory

Scan active regions for idle EIPs ($0.005/hr each = $3.60/mo per idle IP):

```python
active_regions = ['us-east-1', 'us-west-2', 'eu-west-1']  # adjust to your active regions

eip_results = await asyncio.gather(*[
    call_boto3(service_name='ec2', operation_name='DescribeAddresses', region_name=r)
    for r in active_regions
], return_exceptions=True)

for i, region in enumerate(active_regions):
    data = eip_results[i]
    if isinstance(data, dict):
        addresses = data.get('Addresses', [])
        idle = [a for a in addresses if not a.get('AssociationId')]
        # idle EIPs: $0.005/hr × 720 hrs = $3.60/mo each
```

#### 1d. Transit Gateway Inventory

Check for TGW attachments — each costs $0.05/hr ($36/mo) regardless of traffic:

```python
tgw_results = await asyncio.gather(*[
    call_boto3(service_name='ec2', operation_name='DescribeTransitGatewayAttachments',
               region_name=r, params={'Filters': [{'Name': 'state', 'Values': ['available']}]})
    for r in active_regions
], return_exceptions=True)
```

Flag any TGW attachments in regions with low or zero TGW data transfer costs — the attachment may be unused.

#### 1e. Check for VPC Endpoint Savings Opportunity

Query whether S3 or DynamoDB traffic is flowing through NAT Gateway by checking if VPC endpoints exist:

```python
ec2 = boto3.client('ec2')
endpoints = ec2.describe_vpc_endpoints(
    Filters=[{'Name': 'service-name', 'Values': ['*s3*', '*dynamodb*']}]
)
```

If no S3 or DynamoDB gateway endpoints exist in VPCs that have NAT Gateways, flag as a high-priority savings opportunity. Gateway endpoints for S3 and DynamoDB are **free** — routing this traffic through NAT Gateway is pure waste.

#### 1f. Investigate PrivateLink Data Processing

If `<REGION>-VpcEndpoint-Bytes` is a top cost (e.g., >$1000/mo in any region), drill into which interface endpoints are responsible. PrivateLink data processing is billed at $0.01/GB (first 1 PB/mo), so $1000 = 100 TB/mo of traffic — enough to investigate.

Inventory interface endpoints and their ENIs:

```python
vpce_results = await asyncio.gather(*[
    call_boto3(service_name='ec2', operation_name='DescribeVpcEndpoints', region_name=r,
               params={'Filters': [{'Name': 'vpc-endpoint-type', 'Values': ['Interface']}]})
    for r in active_regions
], return_exceptions=True)
```

For each interface endpoint, pull CloudWatch metrics from the `AWS/PrivateLinkEndpoints` namespace to rank by data processed:

```python
from datetime import datetime, timedelta
end = datetime.utcnow()
start_cw = end - timedelta(days=30)

# Per endpoint, per ENI
for endpoint in interface_endpoints:
    for eni in endpoint['NetworkInterfaceIds']:
        response = await call_boto3(
            service_name='cloudwatch', operation_name='GetMetricStatistics',
            region_name=endpoint['region'],
            params={
                'Namespace': 'AWS/PrivateLinkEndpoints',
                'MetricName': 'BytesProcessed',
                'Dimensions': [
                    {'Name': 'VPC Endpoint Id', 'Value': endpoint['VpcEndpointId']},
                    {'Name': 'VPC Id', 'Value': endpoint['VpcId']},
                    {'Name': 'Endpoint Type', 'Value': 'Interface'},
                    {'Name': 'Service Name', 'Value': endpoint['ServiceName']}
                ],
                'StartTime': start_cw, 'EndTime': end,
                'Period': 2592000,  # 30 days
                'Statistics': ['Sum']
            }
        )
```

Present as: service name, endpoint ID, region, GB processed (30 days), estimated monthly cost. Common findings:
- Chatty SDK clients (e.g., logs endpoint receiving high-volume log streaming)
- Endpoints in multiple AZs with unbalanced traffic (consolidate to fewer AZs if latency allows)
- Cross-account service consumption generating heavy traffic (consider direct VPC peering for high-volume flows)
- Unused endpoints still provisioned in multiple AZs

**Savings levers:**
- Consolidate endpoints to fewer AZs (each ENI is $0.01/hr = $7.30/mo)
- For very high volumes, evaluate whether PrivateLink is still cheaper than alternatives (VPC peering at $0.01/GB each way, Transit Gateway at $0.02/GB, direct Direct Connect)
- Remove unused endpoints (Service-Hours billing accrues even with zero bytes)

---

### Phase 2: CloudWatch NAT Gateway Metrics

**Goal:** Identify which NAT Gateways are processing the most traffic and whether traffic is inbound or outbound.

Use when Phase 1 shows significant NAT Gateway costs and you need to know which specific NAT GWs are the culprits.

```python
import boto3

ec2 = boto3.client('ec2')
cw = boto3.client('cloudwatch')

# Get all NAT Gateways
nat_gws = ec2.describe_nat_gateways(
    Filters=[{'Name': 'state', 'Values': ['available']}]
)['NatGateways']

# For each NAT GW, pull 30-day byte metrics
from datetime import datetime, timedelta

end = datetime.utcnow()
start = end - timedelta(days=30)

for ngw in nat_gws:
    ngw_id = ngw['NatGatewayId']
    subnet_id = ngw['SubnetId']

    for metric in ['BytesInFromDestination', 'BytesOutToDestination',
                   'BytesInFromSource', 'BytesOutToSource']:
        response = cw.get_metric_statistics(
            Namespace='AWS/NATGateway',
            MetricName=metric,
            Dimensions=[{'Name': 'NatGatewayId', 'Value': ngw_id}],
            StartTime=start,
            EndTime=end,
            Period=2592000,  # 30 days
            Statistics=['Sum']
        )
        # Sum datapoints and convert bytes to GB
```

Present as a ranked table: NAT Gateway ID, subnet, AZ, total GB processed (30 days), estimated monthly cost.

---

### Phase 3: VPC Flow Logs (Deepest Attribution)

**Goal:** Identify exactly which source IPs, destination IPs, and ports are generating the most cross-AZ or NAT Gateway traffic.

Use when Phase 1 and 2 confirm high costs but you need to attribute them to specific workloads or services.

#### 3a. Check if Flow Logs Are Already Enabled

```python
ec2 = boto3.client('ec2')
flow_logs = ec2.describe_flow_logs()
```

If flow logs exist and are recent, proceed to 3c. If not, guide the user through temporary enablement.

#### 3b. Enable Flow Logs Temporarily (if not already enabled)

Present this as a guided step — flow logs have a cost (~$0.50/GB ingested to CloudWatch Logs) so temporary enablement for investigation is the right approach.

**Recommended approach: Enable for 24–48 hours on the highest-cost VPC only.**

```
📋 To enable VPC Flow Logs for investigation:

1. Go to VPC Console → Your VPCs → select the VPC with highest NAT Gateway costs
2. Actions → Create Flow Log
   - Filter: All (captures ACCEPT and REJECT)
   - Destination: CloudWatch Logs (easier to query) or S3 (cheaper for large volumes)
   - Log format: Custom — include: ${srcaddr} ${dstaddr} ${srcport} ${dstport} 
     ${protocol} ${bytes} ${action} ${flow-direction} ${traffic-path}
   - IAM Role: Create new or use existing flow-logs role
3. Let it run for 24–48 hours during normal business hours
4. Return here and say "analyze my flow logs" to continue

⚠️ Cost note: CloudWatch Logs ingestion is ~$0.50/GB. For a busy VPC, 
24 hours may generate 1–5GB. Disable flow logs after analysis is complete.
```

Offer to create the IAM role policy if needed:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents",
      "logs:DescribeLogGroups",
      "logs:DescribeLogStreams"
    ],
    "Resource": "*"
  }]
}
```

#### 3c. Query Flow Logs for Top Talkers

Once flow logs are available (CloudWatch Logs Insights):

```
fields @timestamp, srcAddr, dstAddr, bytes, trafficPath
| filter trafficPath = 4  # 4 = through NAT Gateway
| stats sum(bytes) as totalBytes by srcAddr, dstAddr
| sort totalBytes desc
| limit 20
```

For inter-AZ traffic (requires knowing AZ CIDR ranges):
```
fields @timestamp, srcAddr, dstAddr, bytes, azId
| filter flowDirection = "ingress" or flowDirection = "egress"
| stats sum(bytes) as totalBytes by srcAddr, dstAddr, azId
| sort totalBytes desc
| limit 20
```

#### 3d. Disable Flow Logs After Analysis

Remind the user to disable flow logs once analysis is complete:

```
✅ Analysis complete. Remember to disable flow logs to avoid ongoing charges:
VPC Console → Flow Logs tab → select the flow log → Actions → Delete
```

---

### Phase 4: Produce Output

Render a comprehensive report with ALL of the following sections:

1. **Cost Summary (Last 90 Days)** — total data transfer spend by category, month-over-month trend

2. **Top Cost Drivers** — ranked table:
   ```
   | Rank | Category | Monthly Cost | GB/Month | % of Total |
   |------|----------|-------------|----------|------------|
   | 1    | PrivateLink data processing | $X | X GB | X% |
   | 2    | NAT Gateway processing | $X | X GB | X% |
   | 3    | Inter-AZ transfer | $X | X GB | X% |
   ```

3. **NAT Gateway Detail** (if Phase 2 run) — per-NAT-GW breakdown with subnet, AZ, traffic volume

4. **PrivateLink Endpoint Detail** (if 1f run and VpcEndpoint-Bytes is material) — per-endpoint breakdown with service name, region, GB processed, monthly cost

5. **Quick Wins — Savings Opportunities**:
   - VPC gateway endpoints for S3/DynamoDB (free, immediate savings)
   - Idle NAT Gateways (zero traffic but hourly charges)
   - Unused interface endpoints (Service-Hours with zero bytes)
   - Consolidate PrivateLink endpoints to fewer AZs
   - Single-AZ architecture candidates (services that don't need cross-AZ)
   - CloudFront for internet egress (if direct EC2 egress is high)

6. **Flow Log Findings** (if Phase 3 run) — top source/destination pairs, workload attribution

7. **Recommendations** — ranked by estimated monthly savings, with implementation complexity

8. **Investigation Status** — which phases were completed, what additional data would improve attribution

### Phase 5: Offer Follow-ups

After presenting results:
- "Want me to enable flow logs and analyze traffic patterns?"
- "Want me to calculate savings from adding VPC endpoints?"
- "Want me to identify which workloads are generating cross-AZ traffic?"
- "Want me to export this as CSV or JSON?"
- "Want me to save this report to a file?"

## Export Options (Optional)

When the user asks to save or export results, produce output in the requested format.

**JSON** — full structured data:
```json
{
  "audit_date": "YYYY-MM-DD",
  "period": "90 days",
  "total_data_transfer_cost_usd": 1250.00,
  "categories": [
    {
      "category": "nat_gateway_processing",
      "monthly_cost_usd": 800.00,
      "gb_per_month": 17778,
      "nat_gateways": [...]
    }
  ],
  "savings_opportunities": [
    {
      "type": "vpc_endpoint_s3",
      "estimated_monthly_savings_usd": 400.00,
      "effort": "low",
      "description": "Add S3 gateway endpoint to 3 VPCs"
    }
  ]
}
```

**CSV** — flat table of cost drivers, suitable for spreadsheets.

Write to file using the `write` tool. Suggest default filename: `./data-transfer-audit-YYYY-MM-DD.<ext>`

## Parameters

| Parameter | Source | Default |
|-----------|--------|---------|
| Lookback period | User request | 90 days |
| Region | User-specified or session default | All regions |
| VPC | User-specified | All VPCs |
| Phase depth | User request or cost threshold | Phase 1 always (including 1f PrivateLink if VpcEndpoint-Bytes > $1K/mo); Phase 2 if NAT > $100/mo; Phase 3 if user confirms |
| Output format | User request | Markdown (inline) |

## Tool Dependencies

- `aws.run_script` — Cost Explorer queries, NAT Gateway inventory, VPC endpoint checks, CloudWatch metrics
- `aws.call_aws` — targeted single queries
- `aws.search_documentation` — current pricing lookups
- `write` — save exported output to file

## Example Invocations

- "Why is my data transfer bill so high?"
- "Break down my NAT Gateway costs"
- "Show me inter-AZ transfer charges"
- "What's driving my EC2-Other charges?"
- "What's driving my VPC service charges?"
- "Break down my PrivateLink / VPC endpoint costs"
- "Find VPC endpoint savings opportunities"
- "Analyze my data transfer costs for the last 3 months"
- "Enable flow logs and analyze my top traffic sources"
- "Export the data transfer audit as CSV"
