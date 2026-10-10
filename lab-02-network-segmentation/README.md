# Lab 02 — Azure Network Segmentation and Least-Privilege Traffic Control

## Overview

This phase of the ContosoWorks Azure Security Engineering project focuses on designing network segmentation for a simple application architecture.

The environment contains an application tier and a database tier. The objective is not simply to place the workloads on separate subnets, but to explicitly define and enforce which network paths should exist between them.

The design follows a least-privilege principle:

> Network location alone should not establish trust.

---

## Scenario

ContosoWorks is preparing to deploy its first application in Azure.

The application has the following requirements:

- Customers will eventually access the application through an approved public frontend.
- Application workloads should not require individual public IP addresses.
- The application tier must communicate with a PostgreSQL database.
- PostgreSQL communication uses TCP port 5432.
- The database must not be directly accessible from the Internet.
- Other workloads introduced into the environment should not automatically gain database access.
- Application workloads will eventually require controlled outbound Internet connectivity for updates and external dependencies.

---

## Network Architecture

The environment uses a single Azure Virtual Network with separate subnets for the application and database tiers.

```text
                     Internet
                         |
                         | HTTPS
                         v
                Approved Public Frontend
                    [Planned]
                         |
                         v
+---------------------------------------------------+
|        vnet-contosoworks-dev-eastus              |
|        10.10.0.0/16                              |
|                                                   |
|   +--------------------+                          |
|   | snet-app           |                          |
|   | 10.10.1.0/24       |                          |
|   |                    |                          |
|   | nsg-app            |                          |
|   +---------+----------+                          |
|             |                                     |
|             | TCP 5432                            |
|             v                                     |
|   +--------------------+                          |
|   | snet-database      |                          |
|   | 10.10.2.0/24       |                          |
|   |                    |                          |
|   | nsg-database       |                          |
|   +--------------------+                          |
|                                                   |
+---------------------------------------------------+
             |
             | HTTPS outbound
             v
        NAT Gateway
         [Planned]
```

The public frontend and NAT Gateway are intentionally not deployed in this phase. They will be introduced when workloads require those connectivity paths.

---

## Resources Created

### Virtual Network

**Name:** `vnet-contosoworks-dev-eastus`  
**Address space:** `10.10.0.0/16`

### Application Subnet

**Name:** `snet-app`  
**Address range:** `10.10.1.0/24`

### Database Subnet

**Name:** `snet-database`  
**Address range:** `10.10.2.0/24`

### Network Security Groups

- `nsg-app-dev-eastus`
- `nsg-database-dev-eastus`

Each NSG is associated with the subnet whose security boundary it is intended to control.

---

## Database Segmentation Policy

The primary security requirement was:

> Only the application tier should be able to initiate PostgreSQL connections to the database tier.

The database NSG implements an allowlist-oriented policy.

| Priority | Source | Destination | Protocol | Port | Action |
|---|---|---|---|---|---|
| 100 | `10.10.1.0/24` | `10.10.2.0/24` | TCP | 5432 | Allow |
| 200 | Any | `10.10.2.0/24` | Any | Any | Deny |

This produces a simple security policy:

**Explicitly allow the required application-to-database path and deny other inbound paths to the database tier.**

---

## Accounting for Azure Default Behavior

Separating workloads into different subnets does not automatically isolate them.

Azure Network Security Groups contain default security rules, including `AllowVNetInBound`, which permits traffic matching the `VirtualNetwork` service tag unless a higher-priority rule overrides it.

This means creating an application-to-database allow rule alone would not necessarily create the intended database security boundary.

The design therefore uses higher-priority custom rules:

```text
100  ALLOW  App subnet → Database subnet  TCP 5432

200  DENY   Any → Database subnet         Any

65000  Azure default AllowVNetInBound
```

Because lower priority numbers are evaluated first, traffic matching the custom deny rule does not continue to the default `AllowVNetInBound` rule.

This creates a default-deny posture for inbound database connectivity while preserving the explicitly authorized application path.

---

## NSG Statefulness

Azure NSGs are stateful.

If the application initiates an allowed PostgreSQL connection:

```text
Application → Database : TCP 5432
```

the database's response traffic is automatically permitted as part of the established connection.

A separate reverse allow rule is therefore not required simply to permit response traffic.

A new connection initiated independently by the database would be evaluated separately against the applicable security rules.

---

## Application Tier Security

An NSG was also associated with `snet-app`.

The intended application-tier policy is:

### Inbound

Allow application traffic only through the approved public frontend rather than exposing individual application workloads directly to the Internet.

### Outbound

The application tier will require:

- PostgreSQL connectivity to the database tier on TCP 5432.
- Controlled HTTPS connectivity on TCP 443 for approved updates and external dependencies.

The exact frontend and outbound connectivity controls are intentionally deferred until the corresponding workloads are deployed.

---

## Planned Outbound Architecture

Application workloads will remain private and will not receive individual public IP addresses.

When outbound Internet connectivity is required, a controlled egress architecture will be introduced.

A NAT Gateway is one planned option for providing outbound connectivity from the private application subnet.

The distinction is important:

- **NSG:** Determines whether the network flow is permitted.
- **NAT Gateway:** Provides an outbound Internet path and source network address translation.

The NAT Gateway was not deployed during this phase because no running workload currently requires it and it is a billable Azure resource.

---

## Security Validation

The segmentation policy was reviewed against several expected traffic flows.

| Traffic Flow | Expected Result | Reason |
|---|---|---|
| App → DB TCP 5432 | Allow | Required application dependency |
| App → DB TCP 22 | Deny | Not explicitly authorized |
| Other VNet workload → DB TCP 5432 | Deny | Database access is not inherited from network location |
| Internet → DB TCP 5432 | Deny | Database is not publicly accessible |
| DB response to established App → DB connection | Allow | NSGs are stateful |

---

## Security Design Decisions

### Separate Subnets Do Not Equal Segmentation

Subnets provide network organization and security boundaries, but actual isolation requires traffic controls.

### Protect the Destination Boundary

The database NSG is associated with the database subnet because the primary requirement is to control who can enter the database tier.

An application-side NSG can restrict traffic leaving the application tier, but it does not replace destination-side protection of the database.

### Default Deny Over Implicit Trust

Being inside the VNet does not automatically justify database access.

New workloads should receive database connectivity only when there is an explicit requirement.

### Account for Platform Defaults

Security architecture must account for controls created by the cloud platform itself, not only custom rules created by engineers.

Understanding Azure's default NSG behavior was therefore part of the segmentation design.

---

## Key Takeaways

- Different Azure subnets are not automatically isolated from one another.
- Azure NSGs include default rules that must be considered when designing segmentation.
- NSG rules are evaluated by priority, with lower numbers evaluated first.
- NSGs are stateful, so established connection response traffic does not require mirrored rules.
- Network least privilege means allowing required communication paths rather than broadly trusting internal networks.
- Destination-side controls protect the resource being accessed.
- Private network placement should not be treated as equivalent to trust.
- Security architecture should account for both current requirements and future workload growth.

---

## Evidence

Add the following screenshots to the `images/` directory:

1. `network-architecture.png` — ContosoWorks network architecture.
2. `database-nsg-rules.png` — Database NSG showing the custom priority 100 allow and priority 200 deny rules.
3. `subnet-nsg-associations.png` — Evidence that the application and database NSGs are associated with their intended subnets.

---

## Next Phase

**Lab 03 — Workload Identity**

The next phase introduces Azure workloads and begins moving security decisions from network location toward workload identity.

This will provide the foundation for managed identities, workload-specific authorization, secrets management, and more granular network controls such as Application Security Groups.
