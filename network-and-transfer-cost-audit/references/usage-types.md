# Data Transfer Usage Type Reference

This file catalogs the AWS billing usage types that appear in a data transfer cost audit, with the billing service they live under, pricing, and notes on how they show up in real invoices. Patterns here are derived from production AWS accounts — expect variation based on region activity and account age.

## Region Prefix Convention

Usage types are prefixed with a short region code, with **one important exception**: `us-east-1` line items are frequently unprefixed (the region code is omitted entirely). This catches people off guard when scanning the CUR.

| Prefix | Region |
|---|---|
| _(none)_ | us-east-1 |
| `USE1-` | us-east-1 (some services/newer line items) |
| `USE2-` | us-east-2 |
| `USW1-` | us-west-1 |
| `USW2-` | us-west-2 |
| `EU-` | eu-west-1 (short form) |
| `EUW1-` | eu-west-1 (long form — depending on service) |
| `EUW2-` | eu-west-2 |
| `EUW3-` | eu-west-3 |
| `EUC1-` | eu-central-1 |
| `EUN1-` | eu-north-1 |
| `APN1-` | ap-northeast-1 |
| `APN2-` | ap-northeast-2 |
| `APS1-` | ap-southeast-1 |
| `APS2-` | ap-southeast-2 |
| `CAN1-` | ca-central-1 |
| `SAE1-` | sa-east-1 |
| `MXC1-` | mx-central-1 |
| `UGW1-` | us-gov-west-1 |
| `AFS1-` | af-south-1 |

For a complete list, see AWS docs on [region-to-region pricing](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer).

## Usage Types by Billing Service

Data transfer line items are split across **two** AWS billing services:

- **Amazon Virtual Private Cloud** — public IPv4, Transit Gateway, Interface VPC Endpoints (PrivateLink)
- **Amazon Elastic Compute Cloud - Compute** (aka "EC2-Other") — NAT Gateway, Regional/xAZ transfer, region-to-region egress, VPC peering, internet egress

This split is why a single Cost Explorer query rarely captures everything. Always query both services.

---

## Amazon Virtual Private Cloud service

### Public IPv4 addresses

| Usage Type | Meaning | Price |
|---|---|---|
| `<REGION>-PublicIPv4:InUseAddress` | Hourly charge per in-use public IPv4 (attached to EC2, NAT GW, NLB, etc.) | $0.005/hr |
| `<REGION>-PublicIPv4:IdleAddress` | Hourly charge per idle/unattached EIP | $0.005/hr |

Each IP = $3.60/month running 24/7. A fleet of 30 public IPs = ~$108/mo. Introduced Feb 2024.

### Transit Gateway

| Usage Type | Meaning | Price |
|---|---|---|
| `<REGION>-TransitGateway-Hours` | Per-attachment hourly charge | $0.05/hr ($36/mo each) |
| `<REGION>-TransitGateway-Bytes` | Data processed through the TGW | $0.02/GB |

Flag any attachment in a region with zero `TransitGateway-Bytes` — it may be an abandoned setup still accruing $36/mo.

### Interface VPC Endpoints (PrivateLink)

| Usage Type | Meaning | Price |
|---|---|---|
| `<REGION>-VpcEndpoint-Hours` | Per-ENI hourly for endpoints you consume | $0.01/hr per AZ per endpoint ($7.30/mo) |
| `<REGION>-VpcEndpoint-Service-Hours` | Per-ENI hourly for PrivateLink services you publish | $0.01/hr per AZ ($7.30/mo) |
| `<REGION>-VpcEndpoint-Bytes` | **Data processed through interface endpoints** | $0.01/GB (first 1 PB/mo) |

**PrivateLink data processing is the sleeper cost.** Accounts routing large volumes through interface endpoints (centralized egress patterns, cross-account service consumption, high-volume CloudWatch Logs/S3/Kinesis usage) can see `VpcEndpoint-Bytes` as the #1 line item — exceeding NAT Gateway by 10–20×. Always check this first in high-volume accounts.

Gateway endpoints (S3, DynamoDB) are **free** and do not generate any of these line items. If you see `VpcEndpoint-Bytes` attributed to an S3 or DynamoDB service name, the traffic is going through an interface endpoint when it could go through a gateway endpoint — immediate savings opportunity.

---

## Amazon Elastic Compute Cloud - Compute (EC2-Other) service

### NAT Gateway

| Usage Type | Meaning | Price |
|---|---|---|
| `<REGION>-NatGateway-Hours` | NAT Gateway hourly | $0.045/hr ($32.85/mo each) |
| `<REGION>-NatGateway-Bytes` | Data processed through NAT Gateway | $0.045/GB |

For us-east-1 these often appear **unprefixed**: `NatGateway-Hours`, `NatGateway-Bytes`.

Each NAT Gateway = ~$33/mo before any bytes. A multi-AZ 3-region deployment can easily be $300/mo in hourly charges alone.

### Inter-AZ Transfer

Two different naming conventions exist for the same thing. Accounts may see one or the other depending on age and region:

| Usage Type | Meaning | Price |
|---|---|---|
| `<REGION>-DataTransfer-Regional-Bytes` | Inter-AZ transfer (older/most common naming) | $0.01/GB each way |
| `<REGION>-DataTransfer-xAZ-In-Bytes` | Inter-AZ ingress (newer naming) | $0.01/GB |
| `<REGION>-DataTransfer-xAZ-Out-Bytes` | Inter-AZ egress (newer naming) | $0.01/GB |

**Both directions are billed.** A cross-AZ 1 GB transfer costs $0.02 total ($0.01 from the sender, $0.01 to the receiver).

### Intra-AZ Transfer (Free)

| Usage Type | Meaning | Price |
|---|---|---|
| `<REGION>-DataTransfer-AZ-In-Bytes` | Same-AZ ingress | $0.00/GB |
| `<REGION>-DataTransfer-AZ-Out-Bytes` | Same-AZ egress | $0.00/GB |

These should always show $0. If they don't, something changed in AWS pricing — re-verify.

### Internet Egress

| Usage Type | Meaning | Price |
|---|---|---|
| `<REGION>-AWS-Out-Bytes` | Egress to internet from that region | $0.09/GB (first 10 TB), tiered down to $0.05/GB |
| `<REGION>-AWS-In-Bytes` | Inbound from internet | $0.00/GB |

First 100 GB/mo per region is free (account aggregate across all AWS regions). AWS-In should always be $0.

### Region-to-Region Egress

| Usage Type | Meaning | Price |
|---|---|---|
| `<SRC>-<DST>-AWS-Out-Bytes` | Egress from SRC region to DST region | $0.02/GB (intra-continent) to $0.09/GB (cross-continent) |

Examples from real billing:
- `USW2-USE1-AWS-Out-Bytes` — us-west-2 → us-east-1 ($0.02/GB)
- `EU-USE1-AWS-Out-Bytes` — eu-west-1 → us-east-1 ($0.02/GB)
- `EUC1-APS1-AWS-Out-Bytes` — eu-central-1 → ap-southeast-1 ($0.02/GB)
- `USE1-EU-AWS-Out-Bytes` — us-east-1 → eu-west-1 ($0.02/GB)

**Gotcha:** CE's `USAGE_TYPE_GROUP` of `EC2: Data Transfer - Region to Region (Out)` does not reliably classify every pair, especially for newer regions. Always query EC2-Other grouped by `USAGE_TYPE` and filter for the `<SRC>-<DST>-AWS-Out-Bytes` pattern to catch the long tail.

### VPC Peering

| Usage Type | Meaning | Price |
|---|---|---|
| `<REGION>-VpcPeering-In-Bytes` | Peering ingress (inter-AZ or cross-region) | $0.01/GB (same-region inter-AZ); varies for cross-region |
| `<REGION>-VpcPeering-Out-Bytes` | Peering egress (inter-AZ or cross-region) | $0.01/GB (same-region inter-AZ); varies for cross-region |

Intra-AZ VPC peering (same-AZ, same-region) is free. Cross-region peering is priced as inter-region data transfer.

---

## Usage Types to **Ignore** in a Data Transfer Audit

These appear in the EC2-Other CUR export but are compute/storage, not data transfer. Filter them out:

- `EBS:VolumeUsage.gp2`, `EBS:VolumeUsage.gp3`, `EBS:VolumeUsage.piops`, `EBS:VolumeP-IOPS.*`, `EBS:VolumeP-Throughput.*`, `EBS:SnapshotUsage`
- `EBSOptimized:<instance-type>` — EBS-optimized instance surcharge
- `CPUCredits:t2`, `CPUCredits:t3`, `CPUCredits:t4g` — T-series burstable credits
- `BoxUsage:<instance-type>` — EC2 instance hours (appears in some groupings)
- `SpotUsage:<instance-type>` — Spot instance hours
- `HostBoxUsage:*` — Dedicated Host hours

## Quick Filter Regex

Use this to extract data-transfer line items from an EC2-Other usage type list:

```python
import re

DT_PATTERN = re.compile(
    r'^(?:[A-Z0-9]+-)?'                              # optional region prefix
    r'(?:[A-Z0-9]+-)?'                               # optional second region (for <SRC>-<DST>)
    r'('
    r'NatGateway-(?:Bytes|Hours)|'
    r'DataTransfer-(?:Regional|xAZ|AZ)(?:-(?:In|Out))?-Bytes|'
    r'AWS-(?:In|Out)-Bytes|'
    r'VpcPeering-(?:In|Out)-Bytes'
    r')$'
)

# Also match VPC service usage types:
VPC_PATTERN = re.compile(
    r'^(?:[A-Z0-9]+-)?'
    r'('
    r'PublicIPv4:(?:InUseAddress|IdleAddress)|'
    r'TransitGateway-(?:Hours|Bytes)|'
    r'VpcEndpoint-(?:Hours|Bytes|Service-Hours)'
    r')$'
)
```

## References

- [EC2 on-demand pricing (data transfer section)](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer)
- [VPC pricing (PrivateLink, TGW)](https://aws.amazon.com/privatelink/pricing/)
- [Public IPv4 address pricing](https://aws.amazon.com/vpc/pricing/)
- [NAT Gateway pricing](https://aws.amazon.com/vpc/pricing/) — under "NAT Gateway pricing"
