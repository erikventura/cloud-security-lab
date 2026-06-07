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

## Status
- [x] VPC created
- [x] Subnets created
- [x] Internet Gateway attached
- [X] Route tables
- [ ] Security groups
- [ ] EC2 + bastion host test
