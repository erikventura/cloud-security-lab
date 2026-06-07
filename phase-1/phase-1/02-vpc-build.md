# Phase 1 — VPC build

## Overview
Building a secure 3-tier VPC from scratch instead of using the default VPC.
The goal is to understand every networking component and how they map to
traditional network security concepts.

## Architecture
- VPC CIDR: 10.0.0.0/16
- 2 public subnets (10.0.1.0/24, 10.0.2.0/24) across us-east-1a and 1b
- 2 private subnets (10.0.10.0/24, 10.0.11.0/24) across us-east-1a and 1b
- Internet Gateway for public subnet internet access

## Mapping to traditional networking
| AWS concept | Traditional equivalent |
|-------------|------------------------|
| VPC | VDOM / isolated network |
| Subnet | VLAN bound to one AZ |
| Internet Gateway | WAN uplink interface |
| Route table | Static routing table |
| Security Group | Stateful firewall policy |
| NACL | Stateless ACL |

## Steps completed
### VPC
- Created lab-vpc with CIDR 10.0.0.0/16
- Why: defines the private network boundary; nothing enters or leaves
  until explicitly allowed

### Subnets
- Created 2 public + 2 private subnets across two AZs
- Why: public/private separation is the core of secure cloud design;
  multi-AZ provides resilience
- Note: AWS reserves 5 IPs per subnet (not the usual 2)

### Internet Gateway
- Created and attached lab-igw to lab-vpc
- Why: single controlled door to the internet; does nothing until
  referenced in a route table
  
  ### Route tables
- Created lab-public-rt with route 0.0.0.0/0 -> lab-igw
- Associated both public subnets with lab-public-rt
- Left private subnets on the main route table (local route only)
- Why: routing is what actually enforces the public/private boundary.
  Private subnets have no internet route, so instances there cannot
  reach the internet directly regardless of security group settings
  — this is defense through architecture (layered protection)

### AWS subnet IP reservations
- AWS reserves 5 IPs per subnet, not the usual 2:
  - .0 network address
  - .1 VPC router (implicit gateway)
  - .2 Amazon DNS resolver
  - .3 reserved for future use
  - .255 broadcast (unused but reserved)
- Security note: the .2 DNS address is a key point for DNS query
  logging and detecting DNS-based exfiltration
  
### Route tables (production best practice)
- Created lab-public-rt: route 0.0.0.0/0 -> lab-igw, associated both public subnets
- Created lab-private-rt: local route only (NAT route added later),
  explicitly associated both private subnets
- Left the VPC main route table unused
- Why explicit associations: makes the architecture intentional and
  auditable; prevents private subnets from silently inheriting changes
  to the main route table

### Why multiple route tables in one VPC
- Route tables apply per-subnet, not per-VPC
- Public and private tiers have different egress needs:
  - Public: 0.0.0.0/0 -> Internet Gateway (inbound + outbound)
  - Private: 0.0.0.0/0 -> NAT Gateway (outbound only) or no internet route
- This per-segment routing is what enables tiered segmentation /
  microsegmentation within a single VPC

### Default VPC security note
- Account has a default VPC (172.31.0.0/16) with all-public subnets
- Common security finding: default VPCs expose resources unintentionally
- Best practice: delete unused default VPCs or document their existence;
  CSPM tools (e.g. Prowler) flag them
- Lab decision: leave default VPC untouched, launch nothing into it,
  always target lab1-vpc explicitly

  ### Security groups (tiered, production pattern)

Built a strict trust chain: internet -> bastion -> app -> database.
Each tier only trusts the tier directly above it.

| SG | Inbound rule | Source | Purpose |
|----|-------------|--------|---------|
| lab-bastion-sg | SSH (22) | My IP (/32) | Single hardened entry point |
| lab-app-sg | SSH (22) | lab-bastion-sg | App mgmt only via bastion |
| lab-db-sg | MySQL (3306) | lab-app-sg | DB reachable only from app tier |

### Key security group concepts
- Stateful: return traffic auto-allowed (like FortiGate policies)
- Allow-only: no explicit deny; implicit deny on anything not listed
  (use NACLs for explicit deny / IP blocking)
- SG-to-SG references: source can be another security group, not just
  an IP. This is identity-based access and the foundation of zero-trust
  segmentation inside AWS — rules stay correct as instances scale/change

### Operational notes
- Bastion SSH pinned to my IP as /32. Residential/CGNAT IP changes will
  break this rule and require updating it (real operational consideration)
- DB SG has NO SSH and NO internet exposure — exposed databases are a
  critical finding behind many major breaches
- Modern alternative to bastion: SSM Session Manager (no open SSH port
  at all) - to explore later

## Status
- [x] VPC created
- [x] Subnets created
- [x] Internet Gateway attached
- [X] Route tables
- [X] Security groups
- [ ] EC2 + bastion host test
