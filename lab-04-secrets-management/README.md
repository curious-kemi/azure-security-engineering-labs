# Lab 04 — Secrets Management and Azure Key Vault

## Overview

In this lab, I focused on how I would secure application secrets in Azure when the secret cannot be eliminated completely.

My main takeaway from this lab was that putting a secret in Azure Key Vault does not automatically make the entire design secure. Key Vault solves the storage problem, but I still have to think about who is accessing the secret, what permissions they have, how the application authenticates, how the traffic reaches Key Vault, how the secret is rotated, and what happens if something gets compromised.

The scenario for this lab involved a ContosoWorks application that needs an API key to communicate with a third-party payment provider.

---

## Scenario

The application requires:

```text
PAYMENT_API_KEY
```

My first question was:

> Does this secret actually need to exist?

If the application was connecting to an Azure service that supported Microsoft Entra authentication, I would prefer to eliminate the credential and use Managed Identity.

In this case, however, the third-party provider requires an API key.

That means the secret cannot currently be eliminated.

So the design becomes:

```text
Third-Party Provider
        |
   Issues API Key
        |
        v
Azure Key Vault
        ^
        |
Managed Identity
        |
ContosoWorks Application
```

The goal is to keep the API key out of application code, configuration files, source control, and CI/CD variables where possible.

---

# 1. Secure Secret Storage

I created an Azure Key Vault in the existing ContosoWorks development environment.

### Configuration

| Setting | Configuration |
|---|---|
| Resource Group | `rg-contosoworks-dev-eastus` |
| Region | East US |
| Pricing Tier | Standard |
| Permission Model | Azure RBAC |
| Public Network Access | Enabled |
| Network Access | All networks |
| Private Endpoint | Not configured |

I intentionally left public network access enabled during this phase because I wanted to focus first on identity, authorization, and secret management.

I already know this is something I would revisit for a production environment.

If the application only needs private connectivity, there is no reason to leave an unnecessary public network path available.

That becomes the focus of the next lab.

---

## Resource Tags

I added:

```text
Environment = Dev
Project     = ContosoWorks
ManagedBy   = Manual
Purpose     = SecurityLab
```

The tags themselves are not security controls.

I am using them to identify the environment, project, purpose, and how the resource is currently being managed.

Later, instead of relying on someone to remember to add the correct tags, I can use Azure Policy and Infrastructure as Code to enforce the standard.

---

# 2. Authentication vs. Authorization

One of the best hands-on lessons from this lab happened when I tried to create the secret.

I could see and manage the Key Vault resource in Azure, but when I tried to create a secret, Azure denied the operation because of RBAC.

That immediately demonstrated something important:

> Being able to manage the Key Vault resource does not automatically mean I can access or manage the secrets inside it.

This is the difference between the **control plane** and the **data plane**.

```text
CONTROL PLANE
-------------------------
Manage Key Vault resource
View configuration
Configure networking
Manage resource settings


DATA PLANE
-------------------------
Create secrets
Read secret values
Update secrets
Delete secrets
```

I had access to the resource, but I did not yet have the data-plane permission required to create the secret.

---

# 3. Applying Least Privilege

To create and manage secrets for the lab, I needed an additional Key Vault role.

I could have assigned myself a very broad administrative role, but I did not need permissions to manage everything in Key Vault.

I only needed to manage secrets.

I therefore used:

```text
Principal:
My lab user

Role:
Key Vault Secrets Officer

Scope:
ContosoWorks Key Vault
```

After the role assignment was applied, I was able to create the secret.

This reinforced another security principle for me:

> Least privilege is not just about whether someone gets access. It is also about how much access they receive and where that access applies.

---

# 4. Creating the Secret

I created:

```text
Name:
payment-api-key

Value:
Fake lab API credential

Content Type:
API Key

Status:
Enabled

Expiration:
Approximately 90 days
```

Only a fake credential was used for this lab.

Real passwords, API keys, tokens, or production secrets should never be committed to this repository.

---

# 5. How the Application Should Authenticate

I would not want the application using something like:

```text
AZURE_CLIENT_ID
AZURE_CLIENT_SECRET
AZURE_TENANT_ID
```

with a long-lived Azure client secret just so that it can retrieve another secret.

That creates another credential I now have to protect.

Instead, I would use Managed Identity.

```text
Application
     |
Managed Identity
     |
     v
Microsoft Entra ID
     |
Short-Lived Token
     |
     v
Azure Key Vault
```

Azure manages the workload identity, and the application can request a token at runtime.

This removes the need for me to store and rotate an Azure client secret inside the application.

However, Managed Identity answers only one question:

> Who is this workload?

It does not answer:

> What is this workload allowed to access?

That still requires authorization.

---

# 6. Application Authorization

If the application only needs to retrieve an existing secret, I would not give it Key Vault Secrets Officer, Contributor, Owner, or Key Vault Administrator.

The application does not need those permissions.

It only needs to retrieve the secret.

The intended role is:

```text
Principal:
Application Managed Identity

Role:
Key Vault Secrets User

Scope:
Required Key Vault or Secret
```

I did not deploy compute during this phase because I wanted to keep the lab cost as low as possible.

Because there is no application workload yet, I did not create or attach an application Managed Identity. This part of the architecture remains a design that I will implement when the workload exists.

---

# 7. Thinking About Blast Radius

This became one of the most important parts of the lab for me.

Suppose I eventually have:

```text
Key Vault

├── payment-api-key
├── sendgrid-api-key
└── payroll-api-key
```

And I have three applications:

```text
Payment App
Notification App
HR App
```

Before giving all three applications `Key Vault Secrets User` at the entire vault scope, I would first ask:

> Does the Payment application actually need access to the Notification or HR application's secrets?

If the answer is no, then I don't want compromising the Payment application to automatically expose credentials belonging to unrelated applications.

I have a few ways I could design around that.

### Option 1 — Separate Security Boundaries

```text
kv-payments-prod
└── payment-api-key

kv-notification-prod
└── sendgrid-api-key

kv-hr-prod
└── payroll-api-key
```

Each application gets access only to its own vault.

### Option 2 — More Granular Authorization

If there is a reason to use a shared vault, Azure RBAC can also be scoped more granularly.

Conceptually:

```text
Payment Managed Identity
        |
Key Vault Secrets User
        |
payment-api-key only
```

The Payment application could retrieve:

```text
payment-api-key ✓
```

but not:

```text
sendgrid-api-key ✗
payroll-api-key ✗
```

I would not automatically choose one design for every environment.

I would look at the isolation requirements, number of applications, operational complexity, and how permissions are going to be managed.

For workloads with stronger separation requirements, such as HR and Payments, separate vault boundaries may make more sense.

For cases where finer-grained permissions are justified, Infrastructure as Code can make those RBAC assignments easier to manage consistently.

The security goal stays the same:

> If one workload is compromised, I want to reduce what the attacker can access with that workload's identity.

---

# 8. Managed Identity Does Not Eliminate Every Secret

Another important distinction from this lab is that Managed Identity does not automatically mean the application is completely secretless.

In this scenario:

```text
Application
     |
Managed Identity
     |
     v
Azure Key Vault
     |
     v
PAYMENT_API_KEY
     |
     v
Third-Party Provider
```

Managed Identity eliminates the Azure credential the application would otherwise need to authenticate to Key Vault.

But the third-party API key still exists.

So I think about secret management as a maturity model:

```text
BEST WHERE SUPPORTED

Application
     |
Managed Identity / Federation
     |
Short-Lived Token
     |
Target Service

No long-lived application secret


BETTER

Application
     |
Managed Identity
     |
Key Vault
     |
Third-Party Secret


WORST

Application
     |
Hardcoded Secret
```

My first choice is always to ask whether I can eliminate the secret.

If I cannot eliminate it, then I protect it properly.

---

# 9. Secret Expiration Is Not Secret Rotation

I configured an expiration period for the API key.

But one thing I learned is that expiration does not mean Azure Key Vault automatically rotates a third-party credential.

Key Vault stores the credential.

The third-party provider is still the system that issues the credential.

So a real rotation process would look more like:

```text
Generate new key at provider
        |
        v
Store new version in Key Vault
        |
        v
Application retrieves new key
        |
        v
Validate application
        |
        v
Revoke old key at provider
```

If the provider allows two credentials to exist temporarily, I would prefer to validate the new credential before revoking the old one.

That reduces the chance of causing an outage during rotation.

At scale, I would automate as much of this process as the provider allows.

---

# 10. Secret Versions

Key Vault can maintain multiple versions of a secret.

For example:

```text
payment-api-key

Version 1
Version 2
Version 3
```

This means an application can normally retrieve the current version instead of having a particular secret version permanently hardcoded into the application.

Versioning also helps support controlled rotation.

---

# 11. Recovery — Soft Delete and Purge Protection

I also reviewed what happens if someone deletes a secret.

There are two important protections:

### Soft Delete

If a secret is deleted, it can remain recoverable during the configured retention period.

### Purge Protection

Purge protection prevents someone from permanently destroying the deleted object before that retention period ends.

I think about this from both an accidental and malicious perspective.

Someone could delete a secret by mistake.

But if an attacker gains privileged access, they may also intentionally try to destroy important secrets or keys.

Soft delete and purge protection make that type of destructive action harder to turn into permanent data loss.

These controls provide deletion protection and recovery, but I would not treat them as a complete backup or disaster-recovery strategy.

---

# 12. Network Exposure

At the end of this phase, my Key Vault still has:

```text
Public Network Access:
Enabled

Network Access:
All Networks
```

That does **not** mean someone on the internet can automatically retrieve my secrets.

They would still need to authenticate and have the required authorization.

However, I still ask:

> If my application is internal, why should the internet be able to reach the Key Vault endpoint in the first place?

If public connectivity is not required, I would rather remove that unnecessary network path.

The target architecture becomes:

```text
Application
     |
Private Azure Network
     |
Private Endpoint
     |
Azure Key Vault
```

After validating private connectivity and DNS resolution, I could restrict or disable unnecessary public network access.

This reduces network exposure while identity and RBAC continue protecting the resource.

---

# 13. Monitoring and Detection

Even with all of these preventative controls, I still need visibility.

I would want to know:

- Who or what is retrieving secrets?
- Are there unexpected authorization failures?
- Are secrets being accessed at unusual times or by unexpected identities?
- Did someone change Key Vault configuration?
- Did someone modify RBAC?
- Was a secret deleted?
- Did a rotation process fail?

I would enable the appropriate Key Vault diagnostic and audit logging and centralize the security telemetry so that suspicious activity can be detected and investigated.

My assumption is not that prevention will always work.

My assumption is that if something does go wrong, I need enough visibility to know about it.

---

# My Defense-in-Depth Model

By the end of this lab, I stopped thinking about Key Vault as a single security control.

I think about secrets security in layers:

```text
1. Can I eliminate the secret?
             |
             v
2. If not, where do I store it?
             |
             v
3. How does the workload authenticate?
             |
             v
4. What is it authorized to access?
             |
             v
5. What network path can reach it?
             |
             v
6. How is the secret rotated and recovered?
             |
             v
7. How do I detect suspicious activity?
```

Or more simply:

```text
Secret Elimination
        ↓
Secure Storage
        ↓
Workload Identity
        ↓
Least-Privilege Authorization
        ↓
Network Isolation
        ↓
Lifecycle + Recovery
        ↓
Monitoring + Detection
```

The biggest lesson for me was:

> Putting a secret in Key Vault solves the storage problem. It does not solve the entire security problem.

I still have to design identity, authorization, network exposure, lifecycle, recovery, and detection around it.

---

# Security Review

| Security Layer | Current Design |
|---|---|
| Secret Elimination | Preferred where supported |
| Secret Storage | Azure Key Vault |
| Application Authentication | Managed Identity planned |
| Authorization | Azure RBAC |
| Human Secret Management | Key Vault Secrets Officer |
| Application Secret Access | Key Vault Secrets User planned |
| Authorization Scope | Vault or secret depending on boundary |
| Network Isolation | Not yet implemented |
| Expiration | Configured |
| Rotation | Provider-coordinated process required |
| Recovery | Soft delete + purge protection |
| Monitoring | Planned |
| Automation | Terraform planned for later phase |

---

# What I Would Improve for Production

The lab intentionally does not represent the final production architecture.

Before considering this production-ready, I would:

- Implement the application Managed Identity.
- Apply the minimum required Key Vault data-plane permissions.
- Review whether separate vaults or finer-grained RBAC provide the appropriate security boundary.
- Implement private connectivity where public access is unnecessary.
- Restrict or disable unnecessary public network access.
- Establish and test the third-party credential rotation process.
- Enable centralized logging and monitoring.
- Move repeatable configuration and RBAC into Infrastructure as Code.
- Validate both allowed and denied access instead of assuming the configuration works.

---

# Key Takeaways

This lab reinforced several principles for me:

**Authentication is not authorization.**

**Control-plane access is not the same as data-plane access.**

**Managed Identity eliminates one credential, but it does not automatically eliminate every secret.**

**Least privilege is about both role and scope.**

**A compromised workload should not automatically expose unrelated application secrets.**

**Expiration is not rotation.**

**Publicly reachable does not mean publicly accessible, but unnecessary public reachability should still be removed.**

**Recovery controls matter because security also has to account for destructive actions.**

**Prevention alone is not enough. I also need detection and visibility.**

And most importantly:

> I don't want to choose a security product first. I want to understand the requirement and risk first, and then choose the control that addresses it.

---

# Next Phase — Private Connectivity

The biggest known gap remaining in the current Key Vault design is network exposure.

In the next lab, I will focus on:

- Azure Private Endpoints
- Private IP connectivity to Azure PaaS services
- Private DNS
- Public vs. private access
- Restricting unnecessary public network access
- Validating that private connectivity actually works
