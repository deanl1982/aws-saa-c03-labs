# Lab 15 — VPC Peering & Transit Gateway

| | |
|---|---|
| **Module** | 4 — Networking with Amazon VPC |
| **Exam mapping** | Domain 3 · Task 3.4 (scalable network architectures) · Domain 4 · Task 4.4 (network cost) · Domain 2 · Task 2.2 (hybrid connectivity resilience) |
| **Difficulty** | ★★★ Advanced |
| **Time** | 90 minutes |
| **Estimated cost** | ~$0.30–0.50 (TGW attachments are $0.05/h each, plus 3 × t3.micro) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-14-vpc-endpoints-flowlogs.md) · [Back to index](../README.md) · [Next lab ➡](lab-16-ec2-fundamentals.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Connect two VPCs with **VPC peering** and update the route tables on both sides.
- Show that peering is **non-transitive**.
- Build a **hub-and-spoke** network with **AWS Transit Gateway**, including TGW route tables.
- Choose between peering, Transit Gateway, PrivateLink, and VPC Lattice.
- Explain hybrid options: **Site-to-Site VPN**, **Client VPN**, **Direct Connect**, **DX Gateway**, and **VPN over DX**.

## Topology

```
Part A (peering)                      Part B (Transit Gateway)

 saa-vpc ◄──peer──► vpc-b              saa-vpc   vpc-b   vpc-c
 10.0/16            10.1/16               \        |        /
                      ▲                    \       |       /
                      │ peer                ┌──────▼──────┐
                    vpc-c                   │   TGW hub   │──► VPN / DX (on-prem)
                    10.2/16                 └─────────────┘
 saa-vpc ✗ vpc-c (non-transitive)        everything ↔ everything
```

## Choosing a connectivity option

| Need | Use |
|---|---|
| A few VPCs, lowest latency, no per-GB hub charge | **VPC peering** (inter-AZ/inter-region data charges still apply) |
| Many VPCs and on-prem, transitive routing, central control | **Transit Gateway** |
| Expose a single service, overlapping CIDRs | **PrivateLink** |
| Service-to-service app networking across VPCs/accounts with auth policies | **VPC Lattice** |
| Encrypted on-prem link, quick to set up, over the internet | **Site-to-Site VPN** (2 tunnels, ~1.25 Gbps each) |
| Consistent bandwidth and latency, private link, 1/10/100 Gbps | **Direct Connect** (weeks to provision, **not encrypted** by default) |
| DX resilience on a budget | **DX primary + Site-to-Site VPN backup** |
| Maximum DX resilience | **Two DX connections at two DX locations** |
| One DX to many regions' VPCs | **Direct Connect Gateway** |

---

## Prerequisites

- Lab 11 (`saa-vpc`) and Lab 12 (`saa-ssm-ec2-role`).

---

## Part A — Console: VPC peering

### A1. Create a second VPC

- [ ] Open **VPC → Create VPC → VPC only**, name it `saa-vpc-b`, CIDR `10.1.0.0/16`.
- [ ] Create subnet `b-private-a` (`10.1.1.0/24`, us-east-1a). The VPC's main route table is fine.

### A2. Launch the test instances

- [ ] In `saa-vpc` **saa-public-a**, launch `saa-hub-tester` (t3.micro, profile `saa-ssm-ec2-role`, public IP on, SG allowing **all ICMP from 10.0.0.0/8**). Session Manager works over the IGW.
- [ ] In `saa-vpc-b` **b-private-a**, launch `saa-b-target` (t3.micro, SG `b-icmp` allowing **all ICMP from 10.0.0.0/8**). Note its private IP.

### A3. Create the peering connection

- [ ] Open **Peering connections → Create**.
  - Name `saa-peer-a-b`, requester `saa-vpc`, accepter `saa-vpc-b` (same account, same region)
- [ ] Select it and choose **Actions → Accept request**.
- [ ] Update the routes **on both sides**:
  - `saa-public-rt` (saa-vpc): `10.1.0.0/16` → `pcx-...`
  - `saa-vpc-b` main route table: `10.0.0.0/16` → `pcx-...`

### A4. Test

- [ ] Start a Session Manager session on `saa-hub-tester` and run `ping -c 3 <saa-b-target private IP>`. **It replies.**
- [ ] Enable **DNS resolution** on the peering connection (*Actions → Edit DNS settings*) so private DNS names resolve across the peering.

### A5. Show non-transitivity

- [ ] Create `saa-vpc-c` (`10.2.0.0/16`) with subnet `c-private-a` (`10.2.1.0/24`) and an instance `saa-c-target` with the same ICMP SG rule.
- [ ] Peer **vpc-b ↔ vpc-c** (`saa-peer-b-c`), accept it, and add routes between B and C.
- [ ] Add a route in `saa-public-rt`: `10.2.0.0/16` → **`saa-peer-a-b`**.
- [ ] From `saa-hub-tester`, ping `saa-c-target`. **It fails.** Peering doesn't allow edge-to-edge routing, so A can't reach C through B.
- [ ] Remove that bogus route.

---

## Part B — CLI: Transit Gateway hub-and-spoke

```bash
source ~/saa-vpc.env
VPC_B=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=saa-vpc-b --query 'Vpcs[0].VpcId' --output text)
VPC_C=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=saa-vpc-c --query 'Vpcs[0].VpcId' --output text)
SUB_B=$(aws ec2 describe-subnets --filters Name=tag:Name,Values=b-private-a --query 'Subnets[0].SubnetId' --output text)
SUB_C=$(aws ec2 describe-subnets --filters Name=tag:Name,Values=c-private-a --query 'Subnets[0].SubnetId' --output text)
RT_B=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC_B Name=association.main,Values=true --query 'RouteTables[0].RouteTableId' --output text)
RT_C=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC_C Name=association.main,Values=true --query 'RouteTables[0].RouteTableId' --output text)
```

### B1. Remove the peering connections

- [ ] Delete both peerings (Transit Gateway replaces them):

```bash
for P in $(aws ec2 describe-vpc-peering-connections \
  --filters Name=tag:Name,Values=saa-peer-a-b,saa-peer-b-c Name=status-code,Values=active \
  --query 'VpcPeeringConnections[].VpcPeeringConnectionId' --output text); do
  aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $P
done
```

- [ ] Remove the peering routes in each route table (blackholed routes stay behind otherwise):

```bash
aws ec2 delete-route --route-table-id $PUB_RT --destination-cidr-block 10.1.0.0/16
aws ec2 delete-route --route-table-id $RT_B --destination-cidr-block 10.0.0.0/16
aws ec2 delete-route --route-table-id $RT_B --destination-cidr-block 10.2.0.0/16 2>/dev/null
aws ec2 delete-route --route-table-id $RT_C --destination-cidr-block 10.1.0.0/16 2>/dev/null
```

### B2. Create the Transit Gateway

- [ ] Create it and wait for it to become available:

```bash
TGW=$(aws ec2 create-transit-gateway --description "SAA hub" \
  --options AmazonSideAsn=64512,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable,DnsSupport=enable \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=saa-tgw}]' \
  --query TransitGateway.TransitGatewayId --output text)

while [ "$(aws ec2 describe-transit-gateways --transit-gateway-ids $TGW --query 'TransitGateways[0].State' --output text)" != "available" ]; do
  echo "waiting..."; sleep 15; done
```

### B3. Attach the three VPCs

- [ ] Attach them. In production, attach one subnet **per AZ** (here: `saa-vpc` uses two AZs, B and C one each):

```bash
ATT_A=$(aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TGW --vpc-id $VPC_ID \
  --subnet-ids $APP_A $APP_B --query TransitGatewayVpcAttachment.TransitGatewayAttachmentId --output text)
ATT_B=$(aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TGW --vpc-id $VPC_B \
  --subnet-ids $SUB_B --query TransitGatewayVpcAttachment.TransitGatewayAttachmentId --output text)
ATT_C=$(aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TGW --vpc-id $VPC_C \
  --subnet-ids $SUB_C --query TransitGatewayVpcAttachment.TransitGatewayAttachmentId --output text)

aws ec2 describe-transit-gateway-attachments --filters Name=transit-gateway-id,Values=$TGW \
  --query 'TransitGatewayAttachments[].{Id:TransitGatewayAttachmentId,Vpc:ResourceId,State:State}' --output table
# wait until all are 'available' (≈1–2 min)
```

### B4. VPC route tables → TGW

- [ ] Point a summary route at the TGW from each VPC:

```bash
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 10.0.0.0/8 --transit-gateway-id $TGW
aws ec2 create-route --route-table-id $RT_B   --destination-cidr-block 10.0.0.0/8 --transit-gateway-id $TGW
aws ec2 create-route --route-table-id $RT_C   --destination-cidr-block 10.0.0.0/8 --transit-gateway-id $TGW
```

> The more specific `local` route (e.g. `10.0.0.0/16`) still wins for traffic inside each VPC, because of longest prefix match.

### B5. Inspect the TGW route table (propagated routes)

- [ ] Look at the routes the attachments propagated:

```bash
TGW_RT=$(aws ec2 describe-transit-gateways --transit-gateway-ids $TGW \
  --query 'TransitGateways[0].Options.AssociationDefaultRouteTableId' --output text)
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id $TGW_RT \
  --filters Name=state,Values=active \
  --query 'Routes[].{CIDR:DestinationCidrBlock,Type:Type,Attachment:TransitGatewayAttachments[0].ResourceId}' --output table
```

### B6. Test transitive routing

- [ ] Ping both spoke instances from the hub tester:

```bash
TESTER=$(aws ec2 describe-instances --filters Name=tag:Name,Values=saa-hub-tester Name=instance-state-name,Values=running --query 'Reservations[0].Instances[0].InstanceId' --output text)
IP_B=$(aws ec2 describe-instances --filters Name=tag:Name,Values=saa-b-target Name=instance-state-name,Values=running --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
IP_C=$(aws ec2 describe-instances --filters Name=tag:Name,Values=saa-c-target Name=instance-state-name,Values=running --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

CMD=$(aws ssm send-command --instance-ids $TESTER --document-name AWS-RunShellScript \
  --parameters "commands=[\"ping -c 2 $IP_B\",\"ping -c 2 $IP_C\"]" --query Command.CommandId --output text)
sleep 8
aws ssm get-command-invocation --command-id $CMD --instance-id $TESTER --query StandardOutputContent --output text
```

> Both spokes reply. A ↔ C now works, which peering couldn't do.

### B7. Segmentation with TGW route tables (concept + optional)

To **isolate** VPC-B from VPC-C while both still reach the hub (`saa-vpc`):

1. Create two TGW route tables, `rt-shared` and `rt-spokes`.
2. Associate B and C with `rt-spokes`, which only has a route to `saa-vpc`.
3. Associate `saa-vpc` with `rt-shared`, which has routes to both B and C.

- [ ] *(Optional)* Build it with `create-transit-gateway-route-table`, `disassociate-/associate-transit-gateway-route-table`, and `create-transit-gateway-route`.

### B8. Hybrid: create a VPN attachment (read-through; costs $0.05/h if created)

```bash
# Customer gateway represents your on-prem router
# aws ec2 create-customer-gateway --type ipsec.1 --public-ip <onprem-public-ip> --bgp-asn 65000
# aws ec2 create-vpn-connection --type ipsec.1 --customer-gateway-id cgw-... --transit-gateway-id $TGW
```

- [ ] Read and understand the VPN commands above. Don't run them unless you have a real on-prem device.

---

## Verify

- [ ] Peering worked for A ↔ B, and A ↔ C failed through peering.
- [ ] Transit Gateway gives full mesh connectivity between A, B, and C.
- [ ] You can explain which TGW route table each attachment is associated with and propagates to.

## Exam tips

- **Peering:** non-transitive, no overlapping CIDRs, works cross-account and cross-region, no bandwidth bottleneck, no single point of failure.
- **More than a handful of VPCs, plus on-prem** → **Transit Gateway**. Inter-region → **TGW peering**.
- **Share one TGW across accounts** → **AWS RAM**.
- **VPN throughput beyond 1.25 Gbps** → **ECMP** across several VPN tunnels on a **Transit Gateway**.
- **Direct Connect is not encrypted.** For encryption, run **IPsec VPN over DX** or use MACsec (on dedicated connections).
- **Client VPN** is for individual users (OpenVPN). **Site-to-Site VPN** connects whole networks.

## Challenge (optional)

- [ ] Turn on **TGW Flow Logs** and **Network Manager** to visualise the topology.

## Cleanup

> ⚠️ Transit Gateway attachments cost **$0.05/h each**. Clean up today.

- [ ] Remove the TGW routes and attachments:

```bash
aws ec2 delete-route --route-table-id $PUB_RT --destination-cidr-block 10.0.0.0/8
aws ec2 delete-route --route-table-id $RT_B   --destination-cidr-block 10.0.0.0/8
aws ec2 delete-route --route-table-id $RT_C   --destination-cidr-block 10.0.0.0/8
for A in $ATT_A $ATT_B $ATT_C; do aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id $A; done
```

- [ ] Wait for the attachments to finish deleting (about 2 minutes), then delete the TGW:

```bash
aws ec2 delete-transit-gateway --transit-gateway-id $TGW
```

- [ ] Terminate the instances:

```bash
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters Name=tag:Name,Values=saa-hub-tester,saa-b-target,saa-c-target Name=instance-state-name,Values=running,stopped \
  --query 'Reservations[].Instances[].InstanceId' --output text)
```

- [ ] Once the instances have terminated, delete `saa-vpc-b` and `saa-vpc-c` (console: *Actions → Delete VPC*).
- [ ] **Keep** `saa-vpc`.

---

## Knowledge check

**Q1.** A company has 60 VPCs across 10 accounts, all of which need to talk to each other and to on-premises over a VPN. What should it use?

<details><summary>Answer</summary>

An **AWS Transit Gateway** shared through **AWS RAM**, with the VPN attached to the TGW. A full mesh of peering would need 1,770 connections and still couldn't route on-prem traffic transitively.

</details>

**Q2.** VPC-A is peered with VPC-B, and VPC-B with VPC-C. Can VPC-A instances reach VPC-C?

<details><summary>Answer</summary>

**No.** VPC peering isn't transitive. Peer A ↔ C directly or use a Transit Gateway.

</details>

**Q3.** A company needs a dedicated, consistent 10 Gbps link to AWS, encrypted, with a cost-effective backup path. What should it build?

<details><summary>Answer</summary>

**AWS Direct Connect** (10 Gbps) with an **IPsec Site-to-Site VPN over DX** (or MACsec) for encryption, plus a **Site-to-Site VPN over the internet** as the backup path.

</details>
