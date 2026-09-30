# Lab 01: Azure Identity, RBAC, and Least-Privilege Access

## Scenario

ContosoWorks is a fictional SaaS company building its Azure environment. As the Cloud Security Engineer, my goal was to establish an initial access-control model that allows developers, security engineers, and IT administrators to perform their responsibilities without giving everyone unrestricted access.

The focus of this lab was implementing **least privilege, group-based access, separation of duties, and constrained RBAC delegation**.

## Environment

**Resource Group:** `rg-contosoworks-dev-eastus`  
**Region:** East US

The environment uses Microsoft Entra ID security groups rather than assigning permissions directly to individual users.

## Identity Design

I created three security groups representing different job functions:

| Security Group | Persona | Responsibility |
|---|---|---|
| `GRP-Azure-Developers` | Alice / Bob | Deploy and manage development resources |
| `GRP-Azure-Security` | Sarah | Review resources and security configuration |
| `GRP-Azure-IT` | David | Manage approved Azure RBAC assignments |

Using groups makes access easier to manage as the organization grows because permissions can be assigned based on job function instead of maintained separately for every user.

## RBAC Design

Permissions were assigned at the development resource-group scope.

```text
rg-contosoworks-dev-eastus
│
├── GRP-Azure-Developers
│      └── Contributor
│
├── GRP-Azure-Security
│      └── Reader
│
└── GRP-Azure-IT
       └── Role Based Access Control Administrator
            + constrained delegation
```

### Developers — Contributor

Developers need to create and manage application infrastructure, so `GRP-Azure-Developers` received the **Contributor** role.

The assignment is scoped only to the development resource group rather than the entire subscription.

This allows developers to manage resources within their environment without granting unrestricted access across the Azure subscription.

### Security — Reader

`GRP-Azure-Security` received the **Reader** role.

The security team can inspect resources and their configuration but cannot modify the infrastructure.

This supports separation of duties between the engineers operating the environment and the security team reviewing it.

### IT — Constrained RBAC Administration

IT needs the ability to fulfill access requests, but unrestricted role-management permissions could create a privilege-escalation path.

Instead of giving IT unrestricted access administration, `GRP-Azure-IT` received **Role Based Access Control Administrator** with conditions restricting delegated role assignments.

David can delegate only approved roles:

- Reader
- Contributor

And only to approved principals:

- `GRP-Azure-Developers`
- `GRP-Azure-Security`

`GRP-Azure-IT` was intentionally excluded as an allowed target.

The resulting authorization model is:

```text
David
 │
 └── RBAC Administrator
        │
        ├── Reader ──────────┐
        ├── Contributor ─────┤── Approved roles
        │                    │
        ├── Developers ──────┤
        └── Security ────────┘── Approved principals

        IT / David ───────── X
        Owner ────────────── X
```

This limits the ability to use delegated access administration to increase the privileges of David's own group.

## Authorization Testing

I tested the design using separate user identities rather than assuming the configured role assignments worked as intended.

### Test 1 — Developer Write Access

Alice, a member of `GRP-Azure-Developers`, attempted to modify a tag on the development resource group.

**Expected:** Allowed  
**Result:** Allowed

This confirmed that the Contributor assignment provided the required resource-management permissions at the resource-group scope.

### Test 2 — Security Write Access

Sarah, a member of `GRP-Azure-Security`, attempted the same modification.

**Expected:** Denied  
**Result:** Denied

Sarah could inspect the resource group but could not modify it, confirming that the Reader assignment was operating as intended.

## Key Security Concepts

### Authentication vs. Authorization

Microsoft Entra ID establishes the user's identity.

Azure RBAC determines what that authenticated identity can do.

```text
Authentication
"Who are you?"
       ↓
Microsoft Entra ID

Authorization
"What are you allowed to do?"
       ↓
Azure RBAC
```

### Role + Scope

A role assignment does not exist in isolation.

I evaluated permissions using:

```text
Security Principal + Role + Scope
       WHO?          WHAT?   WHERE?
```

For example:

```text
GRP-Azure-Developers
        +
    Contributor
        +
rg-contosoworks-dev-eastus
```

This allows developers to manage resources within the development resource group without granting the same access across the entire subscription.

### Control Plane vs. Data Plane

Another important distinction from this lab was that permission to manage an Azure resource does not necessarily provide permission to access the data stored inside that resource.

For example, being able to inspect a Key Vault resource does not automatically mean an identity can retrieve its secrets.

This distinction will become important in later labs when implementing Key Vault, Storage, managed identities, and workload access.

## Security Takeaways

The biggest lesson from this lab was that least privilege is not simply choosing a less-powerful role.

It requires considering:

**Who needs access?**

**What actions do they need to perform?**

**Where should those permissions apply?**

**Can those permissions be used to obtain additional privileges?**

Using security groups, scoped RBAC assignments, separation of duties, and constrained delegation provides a stronger foundation than assigning broad permissions directly to individual users.

## Evidence

Screenshots in the `/images` directory demonstrate:

1. Resource-group RBAC assignments
2. Constrained RBAC delegation configuration
3. Successful Contributor authorization test
4. Denied Reader write attempt

> Sensitive tenant, subscription, identity, and object information has been removed from screenshots.

## Next Lab

**Lab 02 — Azure Network Segmentation**

The next phase of the ContosoWorks environment will introduce Azure Virtual Networks, subnets, Network Security Groups, and traffic-flow controls to separate application and data tiers.
