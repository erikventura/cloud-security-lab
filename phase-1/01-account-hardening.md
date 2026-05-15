# Phase 1 — Account hardening

## Overview
First step in building a cloud security home lab. 
Goal is to lock down the AWS account before touching any resources.

## What I configured

### IAM admin user
- Created an IAM user with AdministratorAccess policy
- Disabled root account for day-to-day use
- Enabled MFA on both root and IAM admin user
- Why this matters: root account has unrestricted access to everything 
  including billing and account closure — it should never be used for 
  normal work

### AWS Budgets
- Created a zero spend budget with email alert
- Why this matters: prevents surprise charges during lab work

### CloudTrail
- Created a management trail logging all API calls to S3
- Enabled log file validation
- Enabled CloudWatch Logs integration
- Why this matters: every action in AWS generates an API call — 
  CloudTrail is your audit log and the first place you look 
  during an incident

### AWS Config
- Enabled full resource recording across all resource types
- Why this matters: tracks configuration state and changes over time — 
  essential for compliance and detecting drift from secure baselines

### GuardDuty
- Enabled threat detection (pending account activation)
- Why this matters: continuously monitors for malicious activity, 
  compromised credentials, and unusual behavior using ML and 
  threat intelligence feeds

## Key security principles applied
- Least privilege: IAM user has only what's needed, root locked away
- Defense in depth: multiple overlapping security controls
- Audit logging: every action recorded from day one

## Status
- [x] IAM admin user + MFA
- [x] Billing protection
- [x] CloudTrail
- [x] AWS Config
- [ ] GuardDuty (pending account activation)
