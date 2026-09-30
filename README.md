# Azure Security Engineering Labs

A hands-on Azure security engineering project focused on designing and securing a fictional company's cloud environment using identity, network security, least privilege, Zero Trust, policy enforcement, monitoring, and infrastructure as code.

## Project Scenario

**ContosoWorks** is a fictional SaaS company building its Azure environment.

In these labs, I take the role of a Cloud Security Engineer responsible for designing security controls that allow engineering teams to operate while reducing unnecessary access and exposure.

Rather than treating each lab as an isolated exercise, the project progressively builds and secures the same Azure environment.

## Security Engineering Approach

The labs follow a practical workflow:

**Design → Deploy → Test → Break → Troubleshoot → Automate**

Controls are first implemented manually in Azure to understand how they work. Where appropriate, they are then recreated using Terraform to make the environment repeatable and secure by default.

The environment is intentionally tested to verify that security controls actually enforce the expected behavior.

## Lab Roadmap

| Lab | Topic | Status |
|---|---|---|
| 00 | Azure Environment & Cost Controls | ✅ Complete |
| 01 | Identity, RBAC & Least Privilege | ✅ Complete |
| 02 | VNet, Subnets & Network Segmentation | ⏳ Next |
| 03 | Virtual Machines & Managed Identity | Planned |
| 04 | Azure Key Vault | Planned |
| 05 | Private Endpoints & Private DNS | Planned |
| 06 | Azure Policy & Governance | Planned |
| 07 | Defender for Cloud & JIT Access | Planned |
| 08 | Conditional Access & Zero Trust | Planned |
| 09 | Microsoft Sentinel & KQL | Planned |
| 10 | Secure Azure Application Capstone | Planned |

## Architecture

The environment will evolve throughout the project as new security controls are introduced.

```text
Microsoft Entra ID
        │
        ▼
Azure Subscription
        │
        ▼
rg-contosoworks-dev-eastus
        │
        ├── Identity & RBAC
        ├── Network Segmentation
        ├── Workload Identities
        ├── Secrets Management
        ├── Private Connectivity
        ├── Policy Enforcement
        └── Security Monitoring
```

## Current Progress

### Lab 01 — Identity, RBAC & Least Privilege

Implemented a group-based Azure authorization model for developers, security engineers, and IT administrators.

Key concepts:

- Microsoft Entra ID security groups
- Azure RBAC
- Role assignment scope
- Least privilege
- Separation of duties
- Control plane vs. data plane
- Constrained role delegation
- RBAC conditions
- Authorization testing

**[View Lab 01 →](./lab-01-identity-rbac/README.md)**

## Technologies

- Microsoft Azure
- Microsoft Entra ID
- Azure RBAC
- Azure Policy
- Microsoft Defender for Cloud
- Microsoft Sentinel
- Terraform
- KQL

Additional Azure security services will be introduced as the environment develops.

## Objective

The goal of this project is not simply to deploy Azure resources, but to understand and demonstrate the security decisions behind them.

Each lab documents:

- The security problem
- Architecture and design decisions
- Implementation
- Security testing
- Troubleshooting
- Lessons learned
- Infrastructure-as-code implementation where appropriate
