
# Phase 1 — Rebuild commands (runbook)

Source-of-truth commands to recreate the lab. Free resources rarely need
rebuilding, but documenting them makes the environment fully reproducible.
Metered resources (NAT, VPC endpoints) are torn down and rebuilt each session.

Region: us-east-1

## Current resource IDs (reference)
| Resource | Name | ID |
|----------|------|----|
| VPC | lab1-vpc | vpc-0e294eb2f741b62db |
| Public subnet 1a | lab1-public-1a | subnet-009a8803f96809ad3 |
| Public subnet 1b | lab1-public-1b | subnet-0f356456d9b186457 |
| Private subnet 1a | lab1-private-1a | subnet-01efc5725e9bcfca3 |
| Private subnet 1b | lab1-private-1b | subnet-0b8165f6f1cea13d4 |
| Internet Gateway | lab1-igw | igw-089e4069df2044a08 |
| Public route table | lab1-public-rt | rtb-03cacedef39d522d7 |
| Private route table | lab-private-rt | rtb-03d18178801dcc387 |
| Bastion SG | lab1-bastion-sg | sg-0dd00fe08d2c90886 |
| App SG | lab1-app-sg | sg-07a7893d2f0bfffcd |
| DB SG | lab1-db-sg | sg-05a0876448d5807ad |

## Order of operations (full rebuild from scratch)
1. VPC -> 2. Subnets -> 3. IGW + attach -> 4. Route tables + routes + associations
-> 5. Security groups + rules -> 6. Bastion -> 7. [Metered] NAT -> 8. [Metered] SSM endpoints

---

## 1. VPC
\```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab1-vpc}]'
\```

## 2. Subnets
\```bash
# Public 1a
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab1-public-1a}]'
# Public 1b
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab1-public-1b}]'
# Private 1a
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.10.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab1-private-1a}]'
# Private 1b
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.11.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab1-private-1b}]'
\```

## 3. Internet Gateway
\```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=lab1-igw}]'
aws ec2 attach-internet-gateway --internet-gateway-id <IGW_ID> --vpc-id <VPC_ID>
\```

## 4. Route tables
\```bash
# Public RT + internet route + associations
aws ec2 create-route-table --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab1-public-rt}]'
aws ec2 create-route --route-table-id <PUBLIC_RT_ID> \
  --destination-cidr-block 0.0.0.0/0 --gateway-id <IGW_ID>
aws ec2 associate-route-table --route-table-id <PUBLIC_RT_ID> --subnet-id <PUBLIC_1A_ID>
aws ec2 associate-route-table --route-table-id <PUBLIC_RT_ID> --subnet-id <PUBLIC_1B_ID>

# Private RT (local route only) + associations
aws ec2 create-route-table --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab-private-rt}]'
aws ec2 associate-route-table --route-table-id <PRIVATE_RT_ID> --subnet-id <PRIVATE_1A_ID>
aws ec2 associate-route-table --route-table-id <PRIVATE_RT_ID> --subnet-id <PRIVATE_1B_ID>
\```

## 5. Security groups
\```bash
# Bastion SG — SSH from my IP only
aws ec2 create-security-group --group-name lab1-bastion-sg \
  --description "SSH access to bastion from admin IP only" --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=lab1-bastion-sg}]'
aws ec2 authorize-security-group-ingress --group-id <BASTION_SG_ID> \
  --protocol tcp --port 22 --cidr <YOUR_IP>/32

# App SG — SSH from bastion SG
aws ec2 create-security-group --group-name lab1-app-sg \
  --description "App tier - SSH from bastion" --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=lab1-app-sg}]'
aws ec2 authorize-security-group-ingress --group-id <APP_SG_ID> \
  --protocol tcp --port 22 --source-group <BASTION_SG_ID>

# DB SG — MySQL from app SG only
aws ec2 create-security-group --group-name lab1-db-sg \
  --description "DB tier - database port from app tier only" --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=lab1-db-sg}]'
aws ec2 authorize-security-group-ingress --group-id <DB_SG_ID> \
  --protocol tcp --port 3306 --source-group <APP_SG_ID>
\```

## 6. Bastion instance
# Key pair (create once, save the .pem securely - cannot re-download)
aws ec2 create-key-pair --key-name lab1-bastion --key-type ed25519 \
  --query 'KeyMaterial' --output text > lab1-bastion.pem

# Launch bastion into public subnet 1a with bastion SG
aws ec2 run-instances \
  --image-id <AL2023_AMI_ID> \
  --instance-type t2.micro \
  --key-name lab1-bastion \
  --subnet-id subnet-009a8803f96809ad3 \
  --security-group-ids sg-0dd00fe08d2c90886 \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-bastion}]'

# Note: AL2023 AMI ID changes per region/update. Get the latest with:
aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text

  ### Bastion access verified
- Connected via SSH from admin IP -> confirms lab1-bastion-sg chain works
- Key: lab1-bastion (ED25519), connected from Windows via MobaXterm
- Windows lesson: quote paths with spaces; MobaXterm maps C: to /drives/c/
- Operational note: stopping the instance changes its public IP on restart

## 7. [METERED] NAT Gateway — create at session start, DELETE at session end
(added when we build it)

## 8. [METERED] SSM VPC endpoints — create at session start, DELETE at session end
(added when we build the SSM lab)
