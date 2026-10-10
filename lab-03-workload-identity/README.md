# Lab 03 — Workload Identity and Secretless Authentication

## Overview

This phase of the ContosoWorks Azure Security Engineering project focuses on workload identity and eliminating unnecessary long-lived credentials from application authentication.

The design question was:

> How should an Azure-hosted application authenticate to Azure services without storing client secrets in application configuration?

Rather than embedding service principal credentials in the workload, the design uses Microsoft Entra managed identities and least-privilege authorization.

---

## Scenario

The ContosoWorks application needs to access Azure services at runtime.

One proposed approach was to configure the application with service principal credentials:

```text
AZURE_CLIENT_ID=...
AZURE_CLIENT_SECRET=...
AZURE_TENANT_ID=...
```

Although this would allow the application to authenticate, it introduces a long-lived credential that must be stored, protected, rotated, and monitored.

The credential could potentially be exposed through:

- Source control
- Application configuration
- CI/CD pipelines
- Logs
- Backups
- Developer workstations
- Compromised application hosts

The preferred design is to avoid creating an application-managed credential when the platform supports identity-based authentication.

---

## Security Design

The application uses a Microsoft Entra managed identity.

```text
Azure Workload
      |
      | Managed Identity
      v
Microsoft Entra ID
      |
      | Short-lived access token
      v
Azure Resource
```

Azure manages the underlying identity credential.

The application does not need to store a client secret in its configuration.

---

## System-Assigned vs. User-Assigned Managed Identity

Two managed identity models were evaluated.

### System-Assigned Managed Identity

A system-assigned identity is tied directly to the lifecycle of an Azure resource.

```text
Azure Resource
      |
      +---- System-Assigned Identity

Delete Resource
      |
      +---- Identity is deleted
```

For the ContosoWorks application, a system-assigned identity was selected as the preferred design because:

- The identity is intended for one workload.
- The identity does not currently need to be shared.
- Its lifecycle should follow the application resource.
- Deleting the workload also removes the associated identity.

### User-Assigned Managed Identity

A user-assigned identity exists independently from the workload.

```text
User-Assigned Managed Identity
          |
          +---- Workload A
          |
          +---- Workload B
```

This model can be useful when an identity requires an independent lifecycle or needs to be attached to multiple compatible Azure resources.

A user-assigned managed identity was also explored in Azure to understand how identities can be created independently before workloads are deployed.

---

## Authentication Does Not Equal Authorization

Creating or enabling a managed identity answers:

> Who is this workload?

It does not answer:

> What is this workload allowed to do?

The identity must still receive explicit authorization to the resources it needs.

For example:

```text
Application
     |
     | Managed Identity
     v
Microsoft Entra ID
     |
     | Authentication succeeds
     v
Azure Key Vault
     |
     | Authorization check
     v
ALLOW or DENY
```

A valid managed identity with no Key Vault permissions should still be denied access to protected Key Vault data.

---

## Least-Privilege Authorization

If the application needs only to retrieve existing secrets from a specific Key Vault, the authorization design should be:

```text
Principal:
Application Managed Identity

Role:
Key Vault Secrets User

Scope:
Specific ContosoWorks Key Vault
```

The application should not receive broader roles such as:

- Owner
- Contributor
- Key Vault Administrator

simply because those roles would also make the application functional.

The requirement determines the permission.

---

## Role and Scope

Least privilege applies to both the **role** and the **scope**.

Giving the application `Key Vault Secrets User` at subscription scope would provide unnecessary access to applicable Key Vault resources throughout that scope.

Instead:

```text
Correct Role
      +
Smallest Practical Scope
      =
Reduced Blast Radius
```

A compromised workload should expose only the resources required for that workload's function.

---

## Control Plane vs. Data Plane

This phase also reinforces the distinction between Azure's control and data planes.

Managing the Key Vault Azure resource is different from accessing the data stored inside the vault.

```text
Key Vault Resource
       |
       +---- Control Plane
       |     Manage Azure resource configuration
       |
       +---- Data Plane
             Access secrets, keys, and certificates
```

A workload should receive only the data-plane permissions required for its application function.

---

## Runtime Authentication Flow

When the application runs on a supported Azure compute service, the authentication flow can operate without an application-managed client secret.

```text
Application
      |
      | Requests Azure access
      v
Managed Identity
      |
      | Requests token
      v
Microsoft Entra ID
      |
      | Short-lived access token
      v
Target Azure Service
      |
      | Authorization check
      v
Requested operation
```

Azure SDKs can use credential mechanisms such as `DefaultAzureCredential` to obtain the appropriate identity credential at runtime.

The important security property is that the application does not need to contain a long-lived Entra client secret.

---

## Managed Identity vs. Key Vault

Managed identity and Key Vault solve different security problems.

**Managed Identity**

Answers:

> How does the workload authenticate without storing its own Azure credential?

**Key Vault**

Answers:

> Where should credentials, keys, certificates, or secrets that still need to exist be securely stored?

Where a downstream service supports Entra ID authentication directly, managed identity may eliminate the need for an application password entirely.

Where a downstream service still requires a password, API key, or other secret, managed identity can be used to authenticate the workload to Key Vault so the required secret can be retrieved securely at runtime.

---

## Security Maturity

Three possible application authentication patterns were compared.

### 1. Embedded Credential

```text
Application
     |
     +---- Long-lived password/client secret
```

Highest credential exposure and operational burden.

### 2. Runtime Secret Retrieval

```text
Application
     |
     | Managed Identity
     v
Key Vault
     |
     | Retrieve credential
     v
Target Service
```

The credential is removed from application configuration but still exists and must be managed.

### 3. Passwordless Workload Authentication

```text
Application
     |
     | Managed Identity
     v
Microsoft Entra ID
     |
     | Short-lived token
     v
Target Service
```

Preferred when the target service supports identity-based authentication because the application-managed secret can be eliminated entirely.

---

## Validation Strategy

When a workload is deployed later in the project, the managed identity design should be validated rather than assumed to work.

The intended test is:

1. Enable the managed identity.
2. Grant the required least-privilege role.
3. Verify the application can access the authorized resource.
4. Remove the role assignment.
5. Verify authentication still identifies the workload but authorization fails.
6. Restore the required role.
7. Confirm access succeeds again.

This validates the separation between authentication and authorization.

---

## Implementation Decision

A compute workload was intentionally **not deployed during this phase**.

Deploying compute solely to demonstrate the managed identity configuration would introduce unnecessary cost without materially changing the architecture decision.

The system-assigned identity will be implemented and tested when an application workload is introduced later in the ContosoWorks environment.

This reflects an additional engineering principle:

> Design and understand the required control before provisioning infrastructure that has no current operational requirement.

---

## Key Takeaways

- Avoid long-lived application credentials when identity-based authentication is available.
- Managed identities allow Azure workloads to authenticate without application-managed client secrets.
- System-assigned identities share the lifecycle of their associated resource.
- User-assigned identities have independent lifecycles.
- Authentication does not automatically provide authorization.
- Least privilege requires both the correct role and the smallest practical scope.
- Control-plane access and data-plane access are different authorization concerns.
- Key Vault and managed identity solve different problems.
- Runtime secret retrieval reduces credential exposure but does not eliminate the underlying secret.
- Passwordless identity-based authentication is preferable when the target service supports it.
- Security controls should be validated by testing both successful and denied access paths.

---

## Next Phase

**Lab 04 — Secrets Management**

The next phase introduces Azure Key Vault for credentials that cannot be eliminated through managed identity.

The focus shifts from:

**Can this secret be eliminated?**

to:

**If the secret must exist, how should it be stored, accessed, scoped, rotated, and protected?**
