---
title: VPC & Networking — Multi-Region
service: vpc
tags: [service, multi-region, vpc, networking, ipam, nat-gateway, foundation]
status: researched
replication: manual
rpo_achievable: N/A — no data plane state to replicate
rto_achievable: "< 1 min if the standby VPC, subnets, NAT and endpoints are pre-provisioned; 5–15 min and failing RTO if NAT/endpoints are created at failover"
meets_targets: conditional
updated: 2026-09-17
---

# VPC & Networking — Multi-Region

## TL;DR

- **CIDR planning is the single irreversible decision in this entire programme.** `cidr_block` on `aws_vpc` is `ForceNew` and the primary CIDR can never be disassociated. If the three existing production VPCs already overlap (they probably do — they were built as three independent islands and nobody needed them to be unique), you can still fix it by choosing non-overlapping CIDRs *for the three new standby VPCs*, but only if you do it **before the first standby VPC exists**. Do the global allocation plan first, in [[cidr-allocation-plan]], before any other work in this vault.
- **AZ count check, done properly (this was the plan-changing question).** `ca-west-1` has **3** AZs — the same as `ca-central-1`. Subnet-per-AZ templating at 3 AZs is safe for the CA pair. But the AZ *IDs* do not line up: `ca-central-1` is `cac1-az1 / cac1-az2 / cac1-az4` (**there is no `cac1-az3`**), while `ca-west-1` is `caw1-az1 / caw1-az2 / caw1-az3`. Any module that constructs an AZ ID by string-formatting an index (`"${prefix}-az${i}"` for `i` in 1..3) produces `cac1-az3`, which does not exist, and fails in the *primary*, not the standby. `eu-west-2` has **4** AZs vs Ireland's 3 (standby is bigger, fine). `us-west-2` has **4** vs `us-east-1`'s **6** — if `us-east-1` prod spans more than 4 AZs today, `us-west-2` cannot mirror it 1:1.
- **Idle-cost floor.** With three standbys at 3 AZs each, NAT-per-AZ + 15 interface endpoints ≈ **$1,400/month (~$16.8k/year) for a region that serves zero traffic.** The endpoints cost roughly 3× the NAT gateways. Details and levers below.
- **Regional NAT Gateway (`availability_mode = "regional"`, GA Nov 2025, provider v6.24.0) is the right primitive for a warm standby** — it bills per AZ it is *actually expanded into*, and in auto mode it only expands into an AZ once an ENI appears there. A scaled-down standby pays for one AZ, and pays for three the moment the standby scales up. That is exactly the cost curve you want.
- **The thing that will bite:** security group IDs are region-local and **cannot be referenced from another region under any connectivity option** — not peering, not Transit Gateway, not (cross-region) Cloud WAN. Every `source_security_group_id` in your templates re-derives to a different ID in the standby, and any rule that genuinely needs to allow the *other region* must be rewritten as a CIDR or a managed prefix list.

---

## The loudest point: CIDR planning

### Why it is irreversible

`aws_vpc.cidr_block` forces replacement. AWS itself will not let you disassociate the primary IPv4 CIDR of a VPC ([Add or remove a CIDR block from your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/add-ipv4-cidr.html)), and you cannot resize an existing block. You can *add* up to five secondary IPv4 CIDRs live via `aws_vpc_ipv4_cidr_block_association` — that is the only escape hatch, and it does not help you if the primary overlaps a region you later want to reach.

Consequence: **overlapping CIDRs permanently foreclose VPC peering and Transit Gateway between primary and standby.** Neither supports overlapping address space; there is no NAT-your-way-out that is worth operating. The remedy is "recreate the VPC", which means recreating everything inside it.

### Why this bites *this* company specifically

The brief states there is no data sharing between the three current regions. Estates built that way almost always reuse the same CIDR in every region, because it was free to do so and it made the cookiecutter template simpler (one `vpc_cidr` default, three environments). That is fine right up to the moment a second VPC in the same "family" appears.

So: **before the first standby VPC is created, audit the three existing primaries' CIDRs.** Three outcomes:

| Finding | Consequence | Action |
|---|---|---|
| The three primaries already use distinct CIDRs | Best case. Extend the existing scheme to the three standbys. | Write down the scheme formally, then move to IPAM. |
| The three primaries overlap each other, but each *pair* can be made disjoint | Acceptable. EU-primary and EU-standby must not overlap; EU-standby and US-standby overlapping is harmless because they will never be connected. | Allocate standby CIDRs from a fresh, globally-unique block. |
| The team wants a single globally-unique plan | Costs one VPC recreation per primary (disruptive) or a multi-year drift-correct-on-rebuild policy. | Do **not** recreate live primaries for this. Allocate standbys globally-uniquely and let the primaries drift; document the exceptions. |

**Recommendation: allocate all six VPCs (and future ones) from one globally-unique plan, implement the plan in IPAM, and accept that one or more existing primaries are pre-IPAM legacy allocations that get imported as static allocations rather than re-CIDRed.** IPAM supports exactly this via `aws_vpc_ipam_pool_cidr_allocation` on an existing CIDR.

### A concrete plan to argue against

Nothing here is sacred; the point is to have *a* written plan before anyone runs `terraform apply`.

```
10.0.0.0/8                       corporate AWS space
├── 10.0.0.0/12    eu-west-1     (EU primary)
├── 10.16.0.0/12   eu-west-2     (EU standby)
├── 10.32.0.0/12   us-east-1     (US primary)
├── 10.48.0.0/12   us-west-2     (US standby)
├── 10.64.0.0/12   ca-central-1  (CA primary)
├── 10.80.0.0/12   ca-west-1     (CA standby)
└── 10.96.0.0/12 … 10.240.0.0/12  reserved (10 spare region slots)

within each region /12:  /16 per environment  (16 environments)
within each env /16:     /18 per AZ           (4 AZ slots — see AZ section)
within each AZ /18:      /20 private app, /22 public, /22 data, remainder spare
```

Two notes on this shape:

1. **Four AZ slots, not three.** `eu-west-2` and `us-west-2` have four AZs today. Reserving a fourth /18 costs nothing and avoids a second CIDR conversation in 2028. `us-east-1` has six — if prod there ever needs 5 or 6 AZs it gets a secondary CIDR association, which is live-addable.
2. **EKS pod space.** If the VPC CNI is used with a secondary CIDR out of `100.64.0.0/10` (the common pattern to avoid burning RFC1918), that space must also be in the plan. It is not routable cross-region today, but "not routable today" is exactly the assumption that created this problem the first time. See [[aws-eks]].

### IPAM as the disciplined answer

[Amazon VPC IPAM](https://docs.aws.amazon.com/vpc/latest/ipam/what-it-is-ipam.html) turns the table above from a wiki page into an API that refuses to hand out an overlapping CIDR. That is the entire value: it makes the irreversible mistake impossible to make.

Practical facts that shape the design:

- IPAM has a **home Region**, set at creation, and *operating Regions* declared on the IPAM (`operating_regions`). A pool has a **locale** — the Region it can allocate into. A pool with no locale is a container; only locale-scoped pools can be consumed by a VPC in that Region.
- **Tiers**: the [VPC pricing page](https://aws.amazon.com/vpc/pricing/) says the Free Tier is for "resources in a single AWS Region and account". Cross-Region pools, Organization-wide sharing and BYOIP need the **Advanced Tier**, billed at **$0.00027 per active managed IP per hour** — $0.197/IP/month. Note that is *active* IPs (allocated/in-use), not the size of the pool, so a /16 that is 5% used costs ~$645/month… which is not nothing. Sanity-check the active-IP count against real usage before committing; for six VPCs at a few thousand live IPs each this lands in the low hundreds of dollars per month, which is cheap insurance against an unfixable CIDR collision.
- IPAM pools are shared across accounts with **RAM**. In a multi-account estate the pool lives in the network account and is shared to the workload accounts.

```hcl
# ── ipam/main.tf — lives once, in the network account, home region eu-west-1 ──
resource "aws_vpc_ipam" "this" {
  description = "Corporate IPv4 address plan"
  tier        = "advanced" # required for cross-region pools

  dynamic "operating_regions" {
    for_each = toset([
      "eu-west-1", "eu-west-2",
      "us-east-1", "us-west-2",
      "ca-central-1", "ca-west-1",
    ])
    content { region_name = operating_regions.value }
  }
}

resource "aws_vpc_ipam_pool" "top" {
  address_family = "ipv4"
  ipam_scope_id  = aws_vpc_ipam.this.private_default_scope_id
  description    = "10.0.0.0/8 — all AWS"
  # no locale: this is a container pool
}

resource "aws_vpc_ipam_pool_cidr" "top" {
  ipam_pool_id = aws_vpc_ipam_pool.top.id
  cidr         = "10.0.0.0/8"
}

locals {
  region_blocks = {
    "eu-west-1"    = "10.0.0.0/12"
    "eu-west-2"    = "10.16.0.0/12"
    "us-east-1"    = "10.32.0.0/12"
    "us-west-2"    = "10.48.0.0/12"
    "ca-central-1" = "10.64.0.0/12"
    "ca-west-1"    = "10.80.0.0/12"
  }
}

resource "aws_vpc_ipam_pool" "region" {
  for_each            = local.region_blocks
  address_family      = "ipv4"
  ipam_scope_id       = aws_vpc_ipam.this.private_default_scope_id
  source_ipam_pool_id = aws_vpc_ipam_pool.top.id
  locale              = each.key
  description         = "Region pool ${each.key}"

  allocation_default_netmask_length = 16
  allocation_min_netmask_length     = 16
  allocation_max_netmask_length     = 16 # every VPC is a /16, no exceptions
  depends_on                        = [aws_vpc_ipam_pool_cidr.top]
}

resource "aws_vpc_ipam_pool_cidr" "region" {
  for_each     = local.region_blocks
  ipam_pool_id = aws_vpc_ipam_pool.region[each.key].id
  cidr         = each.value
}
```

and the VPC module then stops taking a CIDR at all:

```hcl
resource "aws_vpc" "this" {
  ipv4_ipam_pool_id    = var.ipam_pool_id
  ipv4_netmask_length  = 16
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags                 = { Name = "${var.env}-${var.region_role}" } # region_role = primary|standby
}
```

`ipv4_ipam_pool_id` + `ipv4_netmask_length` are mutually exclusive with `cidr_block`, and IPAM picks a non-overlapping block. The cookiecutter template loses a variable rather than gaining one — which is the right direction of travel for a mature templated monorepo.

**Importing the live primaries:** for each existing VPC, `aws_vpc_ipam_pool_cidr_allocation` with an explicit `cidr` records the allocation in IPAM without touching the VPC. No plan diff on the VPC itself, no replacement.

---

## AZ names vs AZ IDs, and the actual counts

### The mechanic

`eu-west-2a` is an *AZ name* and is mapped per-account. Two accounts' `eu-west-2a` may be different physical zones. `euw2-az1` is an *AZ ID* and is the same physical location in every account. Source: [AWS Availability Zones](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-availability-zones.html).

Two changes worth knowing about:

- **Accounts created from November 2025 onward get the same AZ name→ID mapping as each other.** The randomisation only applies to older accounts in older Regions. If this estate's accounts predate that, the randomisation is live for you.
- **Constrained AZs.** AWS may stop new zonal resource creation in a capacity-constrained AZ, and may remove it from the list for new accounts. So *the AZ list is account-specific*, not just region-specific. `us-west-1` already carries a footnote that newer accounts only see two of its three AZs.

Both facts point the same way: **key templated subnet modules on AZ IDs, and read the AZ list from the account you are deploying into rather than hardcoding.**

### The actual counts — verified

From the AWS AZ ID table (link above), for the six regions in scope:

| Region | Role | AZ count | AZ IDs |
|---|---|---|---|
| `eu-west-1` | EU primary | 3 | `euw1-az1`, `euw1-az2`, `euw1-az3` |
| `eu-west-2` | EU standby | **4** | `euw2-az1`, `euw2-az2`, `euw2-az3`, `euw2-az4` |
| `us-east-1` | US primary | **6** | `use1-az1` … `use1-az6` (a Maryland zone is listed as "Coming in 2026") |
| `us-west-2` | US standby | **4** | `usw2-az1` … `usw2-az4` |
| `ca-central-1` | CA primary | 3 | `cac1-az1`, `cac1-az2`, **`cac1-az4`** |
| `ca-west-1` | CA standby | 3 | `caw1-az1`, `caw1-az2`, `caw1-az3` |

**Verdict on the `ca-west-1` hypothesis: it is false, and the real finding is more interesting.** `ca-west-1` has three AZs, exactly matching `ca-central-1`, so a 3-AZ subnet-per-AZ template mirrors cleanly. The trap is that `ca-central-1`'s third AZ ID is `cac1-az4`, not `cac1-az3` — the sequence has a hole. A template that does either of these is broken:

```hcl
# WRONG — assumes contiguous numbering, blows up in ca-central-1
az_ids = [for i in range(1, 4) : "${local.az_prefix}-az${i}"]

# WRONG — assumes suffix letters line up across regions
azs = ["${var.region}a", "${var.region}b", "${var.region}c"]
```

The second one is wrong in `ca-central-1` for a different reason too: the zone letters there are historically `a`, `b`, `d`.

The **US pair is the one with a real structural mismatch**: 6 AZs in the primary, 4 in the standby. If `us-east-1` production genuinely spans 5 or 6 AZs the standby cannot be a 1:1 mirror and the failover capacity plan has to fold 6 AZs of workload into 4. Most estates run 3 and this is a non-issue — but it is a question to put to the team ([[#Open questions]]).

### The pattern that works

```hcl
data "aws_availability_zones" "this" {
  state = "available"
  filter {
    name   = "zone-type"
    values = ["availability-zone"] # excludes Local Zones / Wavelength
  }
}

locals {
  # Sort by AZ ID so the ordering is stable across accounts and re-runs.
  # Never sort by name: names are per-account randomised.
  az_ids = slice(sort(data.aws_availability_zones.this.zone_ids), 0, var.az_count)

  # zone_ids[i] corresponds to names[i]; build the map from the data source,
  # do not construct AZ IDs by string arithmetic.
  az_id_to_name = zipmap(
    data.aws_availability_zones.this.zone_ids,
    data.aws_availability_zones.this.names,
  )
}

resource "aws_subnet" "private" {
  for_each = toset(local.az_ids) # <-- map key is the AZ ID: stable

  vpc_id               = aws_vpc.this.id
  availability_zone_id = each.key
  cidr_block           = cidrsubnet(aws_vpc.this.cidr_block, 4, index(local.az_ids, each.key) * 4)

  tags = {
    Name  = "${var.env}-private-${local.az_id_to_name[each.key]}"
    AZ-ID = each.key
  }
}
```

Keying `for_each` on the AZ ID rather than the name means the Terraform address of a subnet (`aws_subnet.private["euw2-az1"]`) is stable and meaningful. Keying on the *name* means a subnet's resource address depends on an account-specific mapping, and comparing plans between the primary and standby workspace becomes guesswork.

`var.az_count` (default 3) is the variable that makes the template survive `eu-west-2`'s four and `us-east-1`'s six.

---

## NAT Gateways — the biggest idle line item you control

### Verified prices

Hourly rate and per-GB data processing, from the AWS Price List bulk API (`AmazonEC2` offer, `NatGateway-Hours` / `NatGateway-Bytes` usage types, price file version `20260910195514`), cross-checkable on the [VPC pricing page](https://aws.amazon.com/vpc/pricing/):

| Region | NAT GW $/hour | NAT GW $/GB processed | 1 NAT, 730 h | 3 NAT, 730 h |
|---|---|---|---|---|
| `eu-west-1` | 0.048 | 0.048 | $35.04 | $105.12 |
| `eu-west-2` | 0.050 | 0.050 | $36.50 | $109.50 |
| `us-east-1` | 0.045 | 0.045 | $32.85 | $98.55 |
| `us-west-2` | 0.045 | 0.045 | $32.85 | $98.55 |
| `ca-central-1` | 0.050 | 0.050 | $36.50 | $109.50 |
| `ca-west-1` | 0.050 | 0.050 | $36.50 | $109.50 |

Add the **public IPv4 address charge of $0.005/hour ($3.65/month) per EIP** — this is billed separately from the NAT gateway hourly rate and applies to in-use addresses including those on NAT gateways ([Identify and optimize public IPv4 address usage on AWS](https://aws.amazon.com/blogs/networking-and-content-delivery/identify-and-optimize-public-ipv4-address-usage-on-aws/)). A 3-AZ NAT deployment is therefore **$109.50 + $10.95 = $120.45/month** in `eu-west-2` and `ca-west-1`, **$109.50/month** in `us-west-2`.

**Three standbys, NAT-per-AZ, zero traffic: $350.40/month = $4,204/year.** Data processing is ~$0 while idle, which is the whole point — you are paying for presence, not use.

### The options

| Option | Idle cost / standby / month (3 AZ, eu-west-2 rates) | AZ resilience | RTO impact | Verdict |
|---|---|---|---|---|
| Zonal NAT per AZ | $120.45 | Full | 0 | Correct for the primary, wasteful for an idle standby |
| Single zonal NAT, all AZs routed to it | $40.15 | **Cross-AZ SPOF**: lose that AZ and every private subnet loses egress | 0 | Cheap, but it silently downgrades the standby's own resilience at the exact moment you are relying on it |
| **Regional NAT Gateway, auto mode** | $40.15 while the standby is scaled down to one AZ; $120.45 once it scales out | Full (AWS expands it) | 0, **but** expansion into a newly-populated AZ can take up to 60 min | **Recommended** — see below |
| NAT instances | ~$5–10 (t4g.nano/micro) + your own HA | You build it | 0 | Only worth it if someone already owns the AMI/ASG pattern; otherwise you have invented a pager |
| No NAT until failover | $0 | — | **Creation takes minutes and is on the critical path.** Fails the 15-minute RTO as soon as anything else goes slightly wrong | **Reject** |
| No NAT ever (endpoints only, no internet egress) | $0 | N/A | 0 | Viable *only* if every outbound dependency has a VPC endpoint. Almost never true — OS/package mirrors, third-party APIs, OIDC/JWKS fetches, certificate OCSP |

### Why Regional NAT Gateway is the right primitive here

Announced 20 November 2025 ([AWS NAT Gateway now supports regional availability](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-nat-gateway-regional-availability), available in all commercial Regions except GovCloud and China). One NAT gateway object at the VPC level; it "automatically expands and contracts across availability zones". In **auto mode** it expands into an AZ when it detects an ENI there and manages the EIPs itself; in **manual mode** you pin EIPs per AZ and auto-expansion is off ([re:Post explainer](https://repost.aws/articles/AR77CnVH2zR6KjoD4SvgNVAQ/understanding-amazon-vpc-regional-nat-gateway)).

Billing is unchanged in shape: the [VPC pricing page](https://aws.amazon.com/vpc/pricing/) states you are "charged for each hour that the NAT Gateway is configured in each availability zone". **So it is not a discount — it is a cost curve that follows the workload.** That happens to be exactly the property a warm standby wants: while the standby's node groups are scaled down to a single AZ, you pay for one AZ; when the failover runbook scales them out, NAT follows automatically.

Two caveats, both real:

1. **Up to 60 minutes to expand into a newly-populated AZ.** That is four times your RTO. If the failover plan is "scale from 1 AZ to 3 AZs", the NAT in AZs 2 and 3 may not be there in time. Mitigation: keep a **minimal warm ENI in every AZ** — a single tiny node or even a dummy ENI per private subnet — so the regional NAT is already expanded to all AZs. You then pay the full 3-AZ rate, and the benefit is operational (one ID in one route table, no public subnets) rather than financial. **Decide this explicitly**; it is the difference between $40 and $120 per standby per month and it is bought with RTO risk.
2. `connectivity_type` must be `public`, and `subnet_id`/`allocation_id`/`private_ip` are zonal-only arguments.

Provider support landed in **hashicorp/aws v6.24.0 (2 December 2025)** — `availability_mode`, `availability_zone_address`, `vpc_id` on `aws_nat_gateway`, plus `regional_nat_gateway_id` on `aws_flow_log` in the same era. It requires the `ec2:DescribeAvailabilityZones` permission on the Terraform role, which many pipeline roles do not have today. Latest provider at time of writing is 6.65.0.

```hcl
# Standby: one regional NAT, auto mode. No public subnets needed for it.
resource "aws_nat_gateway" "standby" {
  provider          = aws.standby
  vpc_id            = aws_vpc.standby.id
  availability_mode = "regional"
  connectivity_type = "public"
  tags              = { Name = "${var.env}-standby-nat" }
}

# One route per private route table, all pointing at the same ID.
resource "aws_route" "standby_default" {
  provider               = aws.standby
  for_each               = aws_route_table.standby_private
  route_table_id         = each.value.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.standby.id
}
```

Manual mode accepts `availability_zone_id` inside `availability_zone_address`, which is the AZ-ID-keyed form you want if you ever pin it:

```hcl
resource "aws_nat_gateway" "standby_pinned" {
  provider          = aws.standby
  vpc_id            = aws_vpc.standby.id
  availability_mode = "regional"
  connectivity_type = "public"

  dynamic "availability_zone_address" {
    for_each = { for az in local.az_ids : az => aws_eip.nat[az].id }
    content {
      availability_zone_id = availability_zone_address.key
      allocation_ids       = [availability_zone_address.value]
    }
  }
}
```

⚠️ Switching between auto and manual mode (adding or removing `availability_zone_address`) **recreates the NAT gateway**, per the provider docs. Pick a mode per environment and do not flip it casually; a recreate means new public IPs, which matters if any third party allowlists your egress addresses.

**Recommendation: regional NAT gateway in auto mode in all six VPCs.** In the standby, run one warm node per AZ so the NAT is pre-expanded and the 60-minute expansion window is never on the failover path; accept ~$120/month per standby. If cost pressure wins, run warm nodes in one AZ only, accept ~$40/month, and add "confirm NAT expanded to all AZs" as an explicit, timed step in the runbook with a fallback of a pre-created zonal NAT in the other AZs.

---

## VPC endpoints

### Gateway endpoints are free — use them unconditionally

The VPC pricing page is explicit: "There are no data processing or hourly charges for using Gateway Type VPC endpoints." Gateway endpoints exist for **S3 and DynamoDB only**. In every VPC, primary and standby, create both. There is no cost argument and no reason to template them as optional.

```hcl
resource "aws_vpc_endpoint" "gateway" {
  for_each          = toset(["s3", "dynamodb"])
  provider          = aws.standby
  vpc_id            = aws_vpc.standby.id
  service_name      = "com.amazonaws.${var.standby_region}.${each.key}"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = values(aws_route_table.standby_private)[*].id
}
```

They cost nothing *and* they remove S3/DynamoDB traffic from the NAT gateway's data-processing meter, which is the single largest saving available on NAT data processing at $0.045–0.050/GB.

### Interface endpoints are per-hour, per-AZ, and they add up

Verified from the `AmazonVPC` price list (`VpcEndpoint-Hours`, `VpcEndpoint-Bytes`):

| Region | Interface endpoint $/hour per AZ | $/GB (first 1 PB) |
|---|---|---|
| `eu-west-1` | 0.011 | 0.010 |
| `eu-west-2` | 0.011 | 0.010 |
| `us-east-1` | 0.010 | 0.010 |
| `us-west-2` | 0.010 | 0.010 |
| `ca-central-1` | 0.011 | 0.010 |
| `ca-west-1` | 0.011 | 0.010 |

The unit is **per endpoint, per AZ, per hour**. Fifteen interface endpoints across three AZs:

| Standby | 15 endpoints × 3 AZ | × 2 AZ | × 1 AZ |
|---|---|---|---|
| `eu-west-2` | **$361.35/mo** | $240.90 | $120.45 |
| `us-west-2` | **$328.50/mo** | $219.00 | $109.50 |
| `ca-west-1` | **$361.35/mo** | $240.90 | $120.45 |
| **All three** | **$1,051.20/mo** ($12,614/yr) | $700.80 | $350.40 |

**This is the counter-intuitive result and it deserves to be said plainly: in an idle standby, interface endpoints cost roughly three times what the NAT gateways cost.** The usual "endpoints are cheaper than NAT" argument is a *data-processing* argument — $0.010/GB via an endpoint vs $0.045–0.050/GB via NAT. Break-even for one endpoint across 3 AZs in `eu-west-2` is `3 × 0.011 × 730 / (0.050 − 0.010)` ≈ **602 GB/month through that one service**. An idle standby moves approximately zero GB. Every interface endpoint in it is pure loss until failover.

You still cannot simply delete them: creating an interface endpoint takes minutes and flips private DNS for a whole service name, which is not something to do during a 15-minute failover.

**Recommendation:** pre-create interface endpoints in the standby, but **in two AZs, not three**, and prune the list to the services that are genuinely required for the standby to start serving (typically: `sts`, `ecr.api`, `ecr.dkr`, `secretsmanager`, `ssm`, `kms`, `logs`, `monitoring`, `sqs`, `sns`, `elasticloadbalancing`, `eks`, `autoscaling`). `subnet_ids` on `aws_vpc_endpoint` is updatable in place, so the failover runbook can add the third AZ with a fast, non-replacing apply, or you can simply leave it at two — a standby serving from two AZs is still multi-AZ. That turns $1,051/month into $701/month across the three standbys and buys back ~$4.2k/year.

```hcl
variable "interface_endpoints" {
  description = "Service short-names for interface endpoints. Standby list is deliberately shorter."
  type        = set(string)
}

variable "endpoint_az_count" {
  description = "How many AZs to place interface endpoints in. 3 for primary, 2 for standby."
  type        = number
  default     = 3
}

resource "aws_vpc_endpoint" "interface" {
  for_each = var.interface_endpoints

  vpc_id              = aws_vpc.this.id
  service_name        = "com.amazonaws.${var.region}.${each.key}"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  security_group_ids  = [aws_security_group.endpoints.id]

  subnet_ids = [
    for az in slice(local.az_ids, 0, var.endpoint_az_count) :
    aws_subnet.private[az].id
  ]
}
```

⚠️ `private_dns_enabled = true` overrides the public DNS name of the service **for the whole VPC**. In the standby that is usually what you want, but it interacts badly with anything that deliberately talks to the *primary* region's endpoint — the override is per-service-name-per-region, so `com.amazonaws.eu-west-2.sts` does not shadow `sts.eu-west-1.amazonaws.com`. Worth knowing before someone "helpfully" turns it on for a global service.

---

## Security groups do not cross regions

### The hard constraint

Security group IDs are region-scoped. You cannot put `sg-0abc…` from `eu-west-1` into a rule in `eu-west-2`. This is not a Terraform limitation, it is the API.

Current state of cross-region SG referencing, verified:

| Connectivity | SG referencing supported? |
|---|---|
| Intra-region VPC peering | **Yes** — since [March 2016](https://aws.amazon.com/about-aws/whats-new/2016/03/announcing-support-for-security-group-references-in-a-peered-vpc) |
| **Inter-region VPC peering** | **No.** The [peering security-group docs](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-security-groups.html) state you cannot reference the security group of a peer VPC in a different Region; use the peer's CIDR |
| Transit Gateway | **Same-Region only** — [Introducing security group referencing for AWS Transit Gateway](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-security-group-referencing-for-aws-transit-gateway/) is scoped to VPCs attached to a TGW *within the same Region* |
| Cloud WAN | SG referencing went GA in [June 2025](https://aws.amazon.com/about-aws/whats-new/2025/06/aws-cloud-wan-network-operations-security-referencing-dns), again for VPCs connected by Cloud WAN — do not assume it spans Regions without testing it in your own account |

So: **for the foreseeable future, any rule that must allow traffic from the other region of a pair is a CIDR rule or a managed prefix list rule.** AWS's own answer to this is a Lambda that keeps a prefix list in sync with the membership of a security group — [Automated VPC prefix list population for cross-Region and in-Region security group referencing](https://aws.amazon.com/blogs/networking-and-content-delivery/automated-vpc-prefix-list-population-for-cross-region-and-in-region-security-group-referencing/). That is real, it is AWS-authored, and it is also a Lambda you now operate. For an active/passive pair where the two regions mostly do not talk (see [[cross-region-connectivity]]), you probably do not need it at all.

### What this means for the templates

Most of the pain evaporates once you notice that **the standby's security groups reference the standby's own security groups**. The module is instantiated twice with two provider aliases; inside each instantiation, `source_security_group_id = aws_security_group.app.id` resolves to that region's SG. Nothing special is required.

The two cases that *do* need care:

1. **A rule whose source is a hardcoded `sg-` literal** — from a `tfvars` file, an SSM parameter, or a data source lookup by name. These are the ones that silently apply into the standby pointing at a non-existent-in-this-region ID (the API will reject it, loudly — which is the good outcome) or, worse, at an ID that happens to exist in the standby and means something completely different. **Grep the repo for `sg-` literals before starting.**
2. **A rule that must allow the other region.** Rewrite as CIDR. Since you now have a global CIDR plan, `allow 10.16.0.0/12` *is* "allow everything in eu-west-2", which is coarser than an SG reference but honest and stable.

```hcl
resource "aws_vpc_security_group_ingress_rule" "from_peer_region" {
  count = var.peer_region_cidr == null ? 0 : 1

  security_group_id = aws_security_group.app.id
  cidr_ipv4         = var.peer_region_cidr # e.g. "10.16.0.0/12" — never an sg- id
  from_port         = 443
  to_port           = 443
  ip_protocol       = "tcp"
  description       = "Cross-region: SG references do not work across regions"
}
```

Use `aws_vpc_security_group_ingress_rule` / `..._egress_rule` (singular resources) rather than inline `ingress` blocks or the legacy `aws_security_group_rule`. They have stable IDs, they diff cleanly, and in a two-region module the plan output actually tells you which region a rule belongs to.

---

## Route tables, NACLs and flow logs

**Route tables** mirror structurally but never by ID. Every `*_id` in a route — `nat_gateway_id`, `transit_gateway_id`, `vpc_endpoint_id`, `gateway_id` — is region-local. In a templated module this is automatic; the only failure mode is a route table entry sourced from a variable or a remote state output of the *other* region. The `0.0.0.0/0` route is the one to check: in the standby it points at the standby's NAT, and if the standby is intentionally internet-egress-free it may not exist at all, which changes behaviour in ways that only show up under load.

**NACLs** are stateless and mirror trivially. The realistic gotcha is rule *numbering* drift: if the primary's NACL has been hand-edited during an incident (it always has), the standby built from the template is not a mirror of the primary, it is a mirror of the template. Reconcile once, then forbid console edits. Worth a `terraform plan` drift check in CI on the primary specifically.

**Flow logs.** Two independent decisions:

- *Destination.* CloudWatch Logs or S3, **in the same region as the VPC**. Do not cross-region a flow log destination: it is not supported for CloudWatch Logs, and for S3 it costs inter-region transfer for data you will read approximately never. Ship flow logs locally, then aggregate centrally on whatever schedule the observability story uses — see [[cloudwatch-observability]].
- *Cost while idle.* A standby VPC with no traffic generates almost no flow log volume, so leaving flow logs on in the standby is cheap and means the logs exist when you need them at 3am. Leave them on.
- If using a regional NAT gateway, note the provider grew `regional_nat_gateway_id` on `aws_flow_log` — flow logs can target the regional NAT object directly.

---

## Warm standby shape

What exists in the standby VPC while the primary is healthy:

| Resource | Present? | Costs while idle |
|---|---|---|
| VPC, subnets, route tables, NACLs, IGW | Yes | $0 |
| Gateway endpoints (S3, DynamoDB) | Yes | $0 |
| Interface endpoints (pruned list, 2 AZs) | Yes | ~$220–240/mo |
| NAT gateway (regional, auto mode) | Yes | $40–120/mo depending on AZ spread |
| EIPs for NAT | Yes | $3.65/mo each |
| Flow logs | Yes | pennies |
| Security groups, prefix lists | Yes | $0 |
| Transit Gateway attachment | **Probably not** — see [[cross-region-connectivity]] | $0 if absent |

Nothing in the VPC layer is "scaled to zero"; VPC objects are either there or not, and they all need to be there. The levers are AZ count on endpoints and AZ spread on NAT.

---

## Migration path from single-region

The good news: **none of this touches the live primary's data plane.** The standby VPC is new resources in a new region.

1. **Audit CIDRs.** `aws ec2 describe-vpcs --query 'Vpcs[].[VpcId,CidrBlock,Tags]'` in all three primaries. Record in [[cidr-allocation-plan]]. Do this before anything else.
2. **Stand up IPAM** in the network account, advanced tier, with all six operating regions. Record the three existing primary CIDRs as explicit `aws_vpc_ipam_pool_cidr_allocation` entries. *No change to the live VPCs.*
3. **Refactor the VPC module to be AZ-ID-keyed.** This is the one step with replacement risk: changing `for_each` keys on `aws_subnet` from names to AZ IDs will show as destroy+create. Use `moved` blocks (or `terraform state mv`) to re-key without replacement:
   ```hcl
   moved {
     from = aws_subnet.private["eu-west-1a"]
     to   = aws_subnet.private["euw1-az1"]
   }
   ```
   Generate these from a one-off `describe-availability-zones` per account. **Verify the plan shows zero destroys before applying to a primary.** If in doubt, apply the refactor to the standby first and leave the primaries on the old keying until a quiet window.
4. **Add the `aws.standby` provider alias and instantiate the VPC module a second time.** Nothing exists in the standby yet that anything depends on, so this is a pure create.
5. **Regional NAT + gateway endpoints + pruned interface endpoints in the standby.**
6. **Only then** does anything else in this vault get deployed into the standby.

Things that force replacement, flagged loudly:
- `aws_vpc.cidr_block` — **ForceNew.** Never change it on a live VPC.
- `aws_subnet.cidr_block`, `availability_zone`, `availability_zone_id` — **ForceNew.**
- `aws_nat_gateway.availability_mode`, and switching auto↔manual by adding/removing `availability_zone_address` — **ForceNew**, and it changes your public egress IPs.
- `aws_vpc_endpoint.vpc_endpoint_type`, `service_name` — ForceNew. `subnet_ids` and `security_group_ids` are in-place.

---

## Failover procedure

The VPC layer does essentially nothing at failover, which is the correct design. The steps that are genuinely networking-shaped:

1. **Nothing to promote.** The standby VPC is already live and routable from within its own region.
2. **Confirm NAT coverage** if you took the one-AZ-warm option: check `regional_nat_gateway_address` covers every AZ the workload is scaling into, or that the expansion has completed. This is the one VPC-layer item that can eat RTO.
3. **Optionally widen interface endpoints to the third AZ** — a non-replacing update, ~1–2 minutes, safe to skip if the standby serves from two AZs.
4. Everything else — DNS, ALBs, target registration — belongs to [[route53]] and [[alb-nlb]].

**Human decision:** none at the VPC layer. If a human is being asked a networking question during failover, the design is wrong.

## Failback

Also a non-event at this layer, with one exception: **if the standby's regional NAT expanded during the incident, it will contract again when the workload scales back down, and your egress IPs may change.** Anything that allowlists your egress addresses by IP (payment gateways, partner APIs, SFTP endpoints) must have *both* regions' NAT EIPs registered permanently — and with a regional NAT in auto mode, AWS chooses the IPs. If any third party allowlists you, use **manual mode with pinned EIPs** in both regions so the address set is stable and declarable. This is the strongest argument for manual mode and it is worth asking about early.

---

## Gotchas

1. **`cac1-az3` does not exist.** `ca-central-1` runs `cac1-az1`, `cac1-az2`, `cac1-az4`. Any string-built AZ ID breaks in the CA primary.
2. **`us-east-1` has 6 AZs; `us-west-2` has 4.** If the US primary uses more than four, the standby is structurally not a mirror.
3. **AZ lists are per-account, not just per-region.** Constrained AZs can be withheld from newer accounts. Read them at plan time.
4. **AZ name↔ID mapping is randomised per account** for accounts older than November 2025. `eu-west-2a` in the dev account and `eu-west-2a` in prod may be different buildings. This matters the day someone compares latency or capacity numbers between environments.
5. **Interface endpoints in an idle standby cost ~3× the NAT gateways.** Break-even is ~600 GB/month per endpoint. Prune the list and cut the AZ count.
6. **Gateway endpoints are free and there is no excuse for not having them** in all six VPCs — they also remove S3/DynamoDB bytes from the NAT data-processing meter.
7. **EIPs are billed separately from NAT gateways** at $0.005/hr. Easy to miss in a cost model that only counts `NatGateway-Hours`.
8. **Regional NAT expansion can take up to 60 minutes** — four RTOs.
9. **Flipping regional NAT between auto and manual mode recreates it** and changes your public egress IPs.
10. **Security group IDs never cross regions**, under any connectivity option available today. Grep for `sg-` literals.
11. **`private_dns_enabled` on an interface endpoint changes DNS for the entire VPC** for that service name.
12. **IPAM's home Region is fixed at creation.** Choose it deliberately — and note that putting IPAM's home in `eu-west-1` means a hard `eu-west-1` dependency for *provisioning* new CIDRs everywhere. That does not affect running workloads, but it does affect your ability to build during an Ireland event.
13. **Terraform's role needs `ec2:DescribeAvailabilityZones`** for regional NAT gateways, and for the `aws_availability_zones` data source. Many CI roles do not have it.
14. **VPC quotas are per-region and default low** — 5 VPCs, 200 route tables, 50 routes per table, 5 CIDRs per VPC. New standby regions start at defaults. Request increases *before* the first apply, not during it; see [[quotas-and-limits]].

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| CIDR plan | Globally unique across all 6 VPCs, enforced by IPAM | Unique per-pair only; pairs may overlap each other | **A.** The marginal cost is a spreadsheet; the marginal cost of B is discovering in 2029 that you cannot connect two regions you now need to connect |
| Fix overlapping primaries? | Recreate the offending primary VPCs | Leave them, allocate standbys from clean space, document exceptions | **B.** Recreating a live VPC to satisfy a plan you may never exercise is a bad trade |
| IPAM tier | Advanced ($0.197/active IP/mo) | Free tier + a wiki page | **Advanced.** This is the one control that makes the irreversible mistake impossible. Validate the active-IP count against the bill in month one |
| Subnet module keying | AZ ID (`euw2-az1`) | AZ name (`eu-west-2a`) | **AZ ID.** Stable across accounts; makes primary/standby plans comparable |
| AZ count in standby | Match the primary | Fixed 3 everywhere | **Fixed 3**, with `var.az_count` to allow 4 where it helps. Mirroring `us-east-1`'s 6 into `us-west-2` is impossible anyway |
| NAT in standby | Regional NAT, auto mode, warm ENI per AZ (~$120/mo) | Regional NAT, auto mode, one AZ warm (~$40/mo) | **A for prod standbys, B for non-prod.** The $80/month difference is not worth a 60-minute expansion window on the failover path |
| NAT mode | Auto (AWS manages EIPs) | Manual (pinned EIPs per AZ) | **Auto**, *unless* any third party allowlists your egress IPs — then Manual, unconditionally |
| Interface endpoints in standby | Full primary list × 3 AZ ($361/mo) | Pruned list × 2 AZ (~$220/mo) | **B.** Widening AZs is an in-place update; you can buy the third AZ back in two minutes if you ever need it |
| Flow logs in standby | On | Off until failover | **On.** Near-zero cost at zero traffic and you cannot retroactively enable them |

---

## Cost

Idle monthly cost of the **three standby VPCs**, assuming 3 AZs, regional NAT expanded to all AZs, and 15 interface endpoints × 3 AZs (the naive mirror):

| Line | eu-west-2 | us-west-2 | ca-west-1 | Total |
|---|---|---|---|---|
| NAT gateway hours | $109.50 | $98.55 | $109.50 | $317.55 |
| NAT EIPs (3 × $3.65) | $10.95 | $10.95 | $10.95 | $32.85 |
| Interface endpoints | $361.35 | $328.50 | $361.35 | $1,051.20 |
| Gateway endpoints | $0 | $0 | $0 | $0 |
| VPC / subnets / RT / NACL | $0 | $0 | $0 | $0 |
| **Total** | **$481.80** | **$438.00** | **$481.80** | **$1,401.60/mo** |

**≈ $16,819/year to have three empty networks.** With the recommended shape (pruned endpoints × 2 AZs, regional NAT):

| Line | Total/mo |
|---|---|
| NAT gateway hours + EIPs | $350.40 |
| Interface endpoints (2 AZ) | $700.80 |
| **Total** | **$1,051.20/mo (~$12,614/yr)** |

Further levers, in order of payback:
1. **Prune the endpoint list harder.** Every endpoint removed saves $16–24/month per standby. Ten endpoints instead of fifteen at 2 AZs across three standbys: $467/month.
2. **Drop to one AZ of endpoints in non-prod standbys.**
3. **Do not build a Transit Gateway** unless [[cross-region-connectivity]] concludes you need one — that is another $36.50/attachment/month per region plus data processing, for a link you may never send a byte over.
4. IPv6-only subnets for egress-heavy workloads remove the public IPv4 charge entirely. Interesting, out of scope for a mirroring exercise, note it for [[cost-modelling]].

Data transfer is a [[cross-region-connectivity]] question, not a VPC one — but note that intra-region VPC peering is billed at **$0.01/GB in each direction** in all six regions (verified from the `AmazonVPC` price list, `VpcPeering-In-Bytes` / `VpcPeering-Out-Bytes`), and cross-AZ traffic inside a VPC is billed at the usual $0.01/GB each way.

---

## Open questions

1. **What are the actual CIDRs of the three existing production VPCs, and do they overlap?** Everything else waits on this answer.
2. **How many AZs does `us-east-1` production actually use?** If it is more than four, the US pair needs a capacity conversation before anything else.
3. **Does any third party allowlist our NAT egress IPs?** Determines auto vs manual regional NAT mode, in both regions, permanently.
4. **Are the AWS accounts older than November 2025?** Determines whether AZ name randomisation is live for this estate (it almost certainly is).
5. **Does the estate use one account per region, one per environment, or a single account?** IPAM RAM sharing and the SCP work in [[aws-iam]] both depend on the account topology.
6. **How many interface endpoints are actually in use in the primaries today, and what is the per-service data volume?** Needed to prune the standby list on evidence rather than instinct.
7. **Does the VPC CNI use a secondary CIDR (100.64.0.0/10)?** If so it belongs in the global plan. See [[aws-eks]].
8. **Who owns the network account, and can the Terraform CI role be granted `ec2:DescribeAvailabilityZones`?**

---

## Sources

- [AWS Availability Zones — AZ IDs by Region](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-availability-zones.html) — the authoritative AZ ID table. Source of the verified AZ counts, of the `cac1-az4` gap, and of the November 2025 change to name↔ID mapping and the constrained-AZ behaviour.
- [Amazon VPC Pricing](https://aws.amazon.com/vpc/pricing/) — gateway endpoints free; IPAM Free vs Advanced tier and the $0.00027/IP/hr advanced rate; the statement that a regional NAT gateway is charged per AZ it is configured in.
- AWS Price List bulk API, `AmazonEC2` offer, per-region index `20260910195514` (`https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonEC2/current/<region>/index.csv`) — machine-readable source of the per-region NAT gateway hourly and per-GB rates in the table above.
- AWS Price List bulk API, [`AmazonVPC` offer](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonVPC/current/index.json) — interface endpoint per-AZ-hour rates, endpoint data-processing tiers, intra-region VPC peering per-GB rates, and the $0.005/hr public IPv4 rate.
- [AWS NAT Gateway now supports regional availability](https://aws.amazon.com/about-aws/whats-new/2025/11/aws-nat-gateway-regional-availability) (20 Nov 2025) — the GA announcement and region availability.
- [Understanding Amazon VPC Regional NAT Gateway](https://repost.aws/articles/AR77CnVH2zR6KjoD4SvgNVAQ/understanding-amazon-vpc-regional-nat-gateway) — auto vs manual mode, ENI-triggered expansion, the up-to-60-minute expansion window.
- [terraform-provider-aws CHANGELOG, v6.24.0 (2 Dec 2025)](https://github.com/hashicorp/terraform-provider-aws/blob/main/CHANGELOG.md) — `availability_mode`, `availability_zone_address`, `vpc_id` on the NAT gateway resource; the `ec2:DescribeAvailabilityZones` requirement.
- [`aws_nat_gateway` resource documentation](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/nat_gateway.html.markdown) — the argument semantics quoted above, including that auto↔manual transitions recreate the resource and that `availability_zone_address` accepts `availability_zone_id`.
- [Add or remove a CIDR block from your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/add-ipv4-cidr.html) — primary CIDR cannot be disassociated; five CIDRs per VPC by default.
- [What is IPAM?](https://docs.aws.amazon.com/vpc/latest/ipam/what-it-is-ipam.html) — scopes, pools, locales, operating Regions.
- [Update your security groups to reference peer security groups](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-security-groups.html) — the explicit statement that SG referencing does not work across Regions over peering.
- [Introducing security group referencing for AWS Transit Gateway](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-security-group-referencing-for-aws-transit-gateway/) — TGW SG referencing is same-Region only.
- [Automated VPC prefix list population for cross-Region and in-Region security group referencing](https://aws.amazon.com/blogs/networking-and-content-delivery/automated-vpc-prefix-list-population-for-cross-region-and-in-region-security-group-referencing/) — AWS's own workaround for the cross-Region SG gap.
- [Announcing Support for Security Group References in a Peered VPC](https://aws.amazon.com/about-aws/whats-new/2016/03/announcing-support-for-security-group-references-in-a-peered-vpc) — the 2016 baseline, intra-Region only.
- [Identify and optimize public IPv4 address usage on AWS](https://aws.amazon.com/blogs/networking-and-content-delivery/identify-and-optimize-public-ipv4-address-usage-on-aws/) — public IPv4 charging model and how to find idle addresses.

## Related

[[cidr-allocation-plan]] · [[cross-region-connectivity]] · [[aws-iam]] · [[aws-eks]] · [[route53]] · [[alb-nlb]] · [[cloudwatch-observability]] · [[cost-modelling]] · [[quotas-and-limits]] · [[terraform-repo-structure]] · [[region-pair-selection]]
