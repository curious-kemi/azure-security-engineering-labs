
# Lab 04 — Secrets Management and Azure Key Vault

## Overview

This lab focused on securing application secrets that cannot be eliminated through identity-based authentication.

The goal was not simply to store a secret in Azure Key Vault. The goal was to design the security layers around that secret, including identity, authorization, network exposure, lifecycle management, recovery, and monitoring.

The scenario involved a ContosoWorks application that integrates with a third-party payment provider requiring an API key.

Because the third-party provider requires a credential, the secret cannot currently be replaced with Microsoft Entra authentication or Managed Identity.

---

## Scenario

The ContosoWorks application requires the following third-party credential:

```text
PAYMENT_API_KEY
```

The security requirements were:

- Do not store the API key in application code.
- Do not store the API key in application configuration.
- Do not expose the credential through source control.
- Centralize secret storage.
- Apply least-privilege access.
- Establish an expiration and rotation strategy.
- Protect against accidental or malicious deletion.
- Consider network exposure.
- Maintain visibility into secret access.

---

## Security Design

The first design decision was determining whether the secret could be eliminated.

```text
Application requires credential
          |
          v
Can the credential be eliminated?
       /        \
     YES         NO
      |           |
Managed Identity  Secret must exist
Entra ID          |
                  v
             Azure Key Vault
                  |
             Managed Identity
                  |
          Runtime retrieval
```

For Azure services that support Microsoft Entra authentication, Managed Identity should be preferred over long-lived credentials.

In this scenario, however, the payment provider requires an API key. Therefore, the credential still has to exist and must be securely managed.

---

## Key Vault Deployment

A Standard Azure Key Vault was deployed in the existing ContosoWorks development environment.

### Resource Configuration

| Setting | Configuration |
|---|---|
| Resource Group | `rg-contosoworks-dev-eastus` |
| Region | East US |
| Pricing Tier | Standard |
| Permission Model | Azure RBAC |
| Public Network Access | Enabled |
| Network Access | All networks |
| Private Endpoint | Not configured |

Public network access was intentionally left enabled during this phase.

Private connectivity will be addressed separately so that identity, authorization, secret lifecycle, and networking can be evaluated as independent security layers.

---

## Resource Tags

The following tags were applied:

```text
Environment = Dev
Project     = ContosoWorks
ManagedBy   = Manual
Purpose     = SecurityLab
```

Tags do not provide security enforcement by themselves, but they provide useful metadata for ownership, environment identification, cost management, automation, and governance.

A future phase can use Azure Policy to enforce required tagging standards.

---

## Permission Model

Azure RBAC was selected instead of the legacy Key Vault access-policy model.

This allows Key Vault permissions to align with the broader Azure RBAC authorization model:

```text
Principal + Role + Scope
```

No additional resource-access options were enabled because the current architecture did not require:

- Azure Virtual Machines for deployment
- Azure Resource Manager for template deployment
- Azure Disk Encryption

This followed the principle of enabling only capabilities required by the workload.

---

## Control Plane vs. Data Plane

One of the most important lessons from this lab occurred while creating the first secret.

Although my user could access and manage the Azure Key Vault resource, attempting to create a secret initially failed with an RBAC authorization error.

This demonstrated the separation between the Azure control plane and Key Vault data plane.

```text
CONTROL PLANE

Manage Key Vault resource
View configuration
Configure networking
Configure resource settings


DATA PLANE

Create secrets
Read secret values
Update secrets
Delete secrets
```

Being able to manage or view the Key Vault resource does not automatically provide access to the secrets stored inside it.

---

## Resolving the RBAC Failure

My user required permission to create and manage secrets during the lab.

Instead of assigning a broader Key Vault administrative role, I used:

```text
Role:
Key Vault Secrets Officer

Scope:
Specific ContosoWorks Key Vault
```

This provided the permissions required to manage secrets without unnecessarily granting broader key and certificate administration capabilities.

After the role assignment was applied, the secret was successfully created.

This reinforced an important principle:

> Authentication does not equal authorization, and control-plane access does not automatically provide data-plane access.

---

## Secret Creation

The following lab secret was created:

```text
Name:
payment-api-key

Value:
Fake lab credential

Content Type:
API Key

Status:
Enabled

Expiration:
Approximately 90 days
```

Only a fake credential was used.

Real API keys, passwords, tokens, or production secrets should never be committed to this repository.

---

## Application Authentication

The application should not use an Azure client secret to authenticate to Key Vault.

Instead, the intended production design uses a Managed Identity.

```text
Application
     |
Managed Identity
     |
     v
Microsoft Entra ID
     |
Short-lived access token
     |
     v
Azure Key Vault
```

This removes the need to store a long-lived Azure credential inside the application.

Managed Identity solves the application's authentication problem.

It does **not**, however, automatically authorize the application to retrieve secrets.

---

## Application Authorization

The application only requires permission to retrieve existing secrets.

The appropriate Azure RBAC role is:

```text
Key Vault Secrets User
```

A broader role such as Contributor, Owner, Key Vault Administrator, or Key Vault Secrets Officer would provide unnecessary permissions.

The intended authorization model is:

```text
Principal:
Application Managed Identity

Role:
Key Vault Secrets User

Scope:
Required Key Vault or individual secret
```

No application identity was deployed during this lab because compute resources were intentionally avoided to keep the lab cost minimal.

The application-side role assignment therefore remained an architectural design rather than a deployed configuration.

---

## Vault-Level vs. Secret-Level Authorization

If a Key Vault contains secrets for multiple unrelated applications, assigning `Key Vault Secrets User` at the entire vault scope may give a compromised workload access to secrets it does not require.

For example:

```text
Key Vault
├── payment-api-key
├── sendgrid-api-key
└── payroll-api-key
```

The Payment application should not automatically gain access to the Notification or HR application's credentials.

Two designs can be considered.

### Separate Vault Boundaries

```text
kv-payments-prod
└── payment-api-key

kv-notification-prod
└── sendgrid-api-key

kv-hr-prod
└── payroll-api-key
```

### Fine-Grained RBAC

```text
Shared Key Vault

payment-api-key
     ↑
Payment Managed Identity

sendgrid-api-key
     ↑
Notification Managed Identity

payroll-api-key
     ↑
HR Managed Identity
```

Azure RBAC can be scoped to an individual Key Vault secret when finer-grained authorization is justified.

The appropriate design depends on isolation requirements, operational complexity, scale, and manageability.

Infrastructure as Code can make fine-grained RBAC significantly easier to deploy consistently, but automation does not remove the need for good security boundaries.

---

## Managed Identity vs. Key Vault

Managed Identity and Key Vault solve different security problems.

### Managed Identity

Answers:

> How does the workload authenticate to Azure without storing an Azure credential?

### Key Vault

Answers:

> Where do credentials, keys, and certificates that still need to exist get securely stored and managed?

The stronger design is always to eliminate a secret where possible.

```text
Least Preferred

Application
    |
Hardcoded credential


Better

Application
    |
Managed Identity
    |
Key Vault
    |
Third-party credential


Strongest where supported

Application
    |
Managed Identity
    |
Entra token
    |
Target Service

No application-managed password
```

---

## Secret Rotation

Adding an expiration date to a Key Vault secret does not automatically rotate the credential at the third-party provider.

For a third-party API key, rotation requires coordination between the credential issuer and Key Vault.

A safe rotation process could be:

```text
Generate new credential at provider
          |
          v
Store new secret version in Key Vault
          |
          v
Application retrieves new credential
          |
          v
Validate application functionality
          |
          v
Revoke old credential at provider
```

Where the provider supports overlapping credentials, the new credential should be validated before the previous credential is revoked.

At larger scale, this workflow should be automated where provider APIs and operational requirements allow.

---

## Secret Versioning

Azure Key Vault supports multiple versions of a secret.

For example:

```text
payment-api-key
├── Version 1
├── Version 2
└── Version 3
```

Applications can generally retrieve the current version without hardcoding a specific version.

This helps support controlled secret rotation while maintaining a history of secret versions.

---

## Recovery Protection

Two important recovery controls were reviewed:

### Soft Delete

Deleted Key Vault objects remain recoverable during the configured retention period.

This helps protect against accidental deletion.

### Purge Protection

Purge protection prevents a deleted Key Vault object from being permanently destroyed before the retention period expires.

Together, these controls help reduce the impact of both accidental and malicious deletion.

They should not be confused with a complete backup or disaster-recovery strategy.

---

## Network Security Review

During this phase, the Key Vault remained configured with:

```text
Public network access: Enabled
Network access: All networks
```

This does **not** mean anonymous Internet users can retrieve secrets.

Authentication and authorization are still required.

However, if the application only requires private connectivity, leaving an unnecessary public network path increases exposure.

The production architecture should therefore evaluate:

```text
Application
     |
Private network
     |
Private Endpoint
     |
Azure Key Vault
```

After private connectivity and DNS resolution are validated, unnecessary public network access can be restricted or disabled.

This design will be addressed in the next lab.

---

## Monitoring and Detection

Secure secret storage is not enough.

The environment should provide visibility into:

- Successful secret retrieval
- Failed authorization attempts
- Unexpected access patterns
- Key Vault configuration changes
- RBAC changes
- Secret deletion
- Secret lifecycle and rotation failures

Key Vault diagnostic and audit telemetry can be centralized and used with the organization's monitoring and detection platform.

This allows the security architecture to move beyond prevention and support detection and investigation.

---

## Defense-in-Depth Model

The final design can be viewed as multiple security layers:

```text
1. Secret Elimination
        |
        v
2. Secure Storage
        |
        v
3. Workload Identity
        |
        v
4. Least-Privilege Authorization
        |
        v
5. Network Isolation
        |
        v
6. Lifecycle + Recovery
        |
        v
7. Monitoring + Detection
```

No individual control provides complete protection.

For example, Managed Identity removes the application's long-lived Azure credential, but a compromised workload may still use its identity.

Least-privilege authorization therefore limits the blast radius.

Network isolation reduces unnecessary reachability.

Rotation limits credential lifetime.

Recovery controls protect against destructive actions.

Monitoring provides visibility when preventative controls fail.

---

## Security Design Review

The final architecture identified the following controls and remaining improvements:

| Security Area | Design |
|---|---|
| Secret elimination | Prefer Managed Identity / Entra authentication where supported |
| Secret storage | Azure Key Vault |
| Workload authentication | Managed Identity |
| Authorization | Key Vault Secrets User |
| Scope | Vault or secret scope based on security boundary |
| Network | Private connectivity planned |
| Secret lifecycle | Expiration + coordinated rotation |
| Recovery | Soft delete + purge protection |
| Detection | Audit logging + centralized monitoring |
| Automation | Terraform planned for future phases |

---

## Key Security Takeaways

1. **Putting a secret in Key Vault only solves the storage problem.**

2. Authentication and authorization must be designed separately.

3. Control-plane access does not automatically provide data-plane access.

4. Managed Identity removes application-managed Azure credentials but does not eliminate the impact of workload compromise.

5. Least privilege should consider both the role and its scope.

6. Secret-level RBAC can reduce blast radius when multiple applications share a vault, but security boundaries and operational manageability must also be considered.

7. Expiration is not the same as rotation.

8. Third-party credential rotation must be coordinated with the credential issuer.

9. Soft delete and purge protection provide deletion recovery, not a complete backup strategy.

10. Publicly reachable does not mean publicly accessible, but unnecessary network exposure should still be removed.

11. Infrastructure as Code can make fine-grained authorization and secure-by-default configurations more scalable.

12. Secret management requires multiple layers: identity, authorization, networking, lifecycle, recovery, and detection.

---

## Interview Takeaway

When reviewing secrets architecture, I should not immediately ask:

> “Which Azure service should I use?”

Instead, I should ask:

> Can the secret be eliminated?

> Who needs access?

> What is the minimum access required?

> At what scope should that access exist?

> Does the service need public network connectivity?

> How will the credential be rotated?

> What happens if the workload is compromised?

> How will deletion be recovered?

> How will suspicious access be detected?

This turns secrets management from a product-selection question into a security architecture problem.

---

## Next Phase

**Lab 05 — Private Connectivity**

The next phase will address the known network-security gap by examining:

- Azure Private Endpoints
- Private IP addressing for Azure PaaS services
- Private DNS
- Public vs. private service access
- Disabling unnecessary public network access
- Connectivity validation
- Private Endpoint architecture and trade-offs
