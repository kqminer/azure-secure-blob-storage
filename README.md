# Azure Secure Blob Storage

## Overview

I built this lab to get hands-on with securing Azure Blob Storage beyond just creating a storage account and uploading a file.

I wanted to work through a few things I expect to deal with as an Azure administrator:

- controlling who can read or change blob data
- understanding the difference between Azure RBAC and storage data permissions
- protecting files from accidental deletion or changes
- moving older data to cheaper storage automatically
- restricting where storage can be accessed from
- testing the controls instead of assuming they worked

The main resources were a storage account, a blob container called `labdata`, and a test file called `report.txt`.

---

## Architecture

```text
Microsoft Entra ID
        |
        +-- grp-lab-compliance
        |      |
        |      +-- Reader
        |      +-- Storage Blob Data Reader
        |
Azure Subscription
        |
rg-storage-lab
        |
stlab43084998
        |
        +-- Blob container: labdata
        |      |
        |      +-- report.txt
        |
        +-- Lifecycle policy
        +-- Soft delete
        +-- Blob versioning
        +-- Storage firewall
```

---

## 1. Identity and Data-Plane RBAC

One thing I wanted to understand better was the difference between being able to see an Azure resource and being able to access the data inside it.

I assigned `grp-lab-compliance`:

- **Reader** at the resource-group level
- **Storage Blob Data Reader** at the storage-account level

Reader lets the group view the Azure resource, while Storage Blob Data Reader gives it read access to the actual blob data.

![RBAC role assignments](screenshots/blob-rbac-role-assignments.png)

I tested this with the auditor account. The account could read the blob, but a write attempt failed because I did not give the group a write-capable storage role.

![Auditor write denied](screenshots/auditor-write-denied.png)

<details>
<summary>Why this mattered</summary>

During the lab, I ran into the difference between the Azure management plane and the storage data plane.

Having Reader access to the storage account does not automatically mean a user can open the files stored inside it. Blob data requires its own Storage data role.

</details>

---

## 2. Storage Security Settings

I changed several of the default access settings on the storage account.

I configured:

- anonymous blob access: **Disabled**
- storage account key access: **Disabled**
- minimum TLS version: **1.2**

I wanted access to rely on Microsoft Entra ID and RBAC instead of storage account keys.

![Storage security baseline](screenshots/storage-security-baseline.png)

---

## 3. Lifecycle Management

I created a lifecycle rule called `age-out-labdata` for blobs stored under `labdata/`.

The rule moves older files through cheaper storage tiers over time.

| Blob age | Action |
|---|---|
| 30 days | Move to Cool |
| 90 days | Move to Archive |
| 365 days | Delete |

![Lifecycle policy](screenshots/lifecycle-policy.png)

This was useful for understanding how storage costs can be managed automatically instead of manually moving old files.

---

## 4. Data Protection and Recovery

I enabled:

- soft delete for blobs
- soft delete for containers
- blob versioning

![Data protection settings](screenshots/data-protection-settings.png)

I also tested version recovery instead of stopping at the configuration step.

After changing `report.txt`, I restored an earlier version and confirmed that the previous copy could be recovered.

![Blob version restored](screenshots/blob-version-restored.png)

That helped make the difference between **versioning** and **soft delete** much clearer to me. Versioning protects previous versions of a file, while soft delete helps recover something that was deleted.

---

## 5. Network Restrictions

Next, I restricted the storage account to selected networks.

![Storage firewall](screenshots/storage-firewall-selected-networks.png)

I tested access from two different places.

The browser session from my approved client could still reach `report.txt`, while Azure Cloud Shell was blocked by the storage firewall.

![Firewall validation](screenshots/firewall-validation.png)

This was one of the more useful tests in the lab because it showed me that RBAC and network access are two separate checks.

A user can have the correct permissions and still be blocked if the request comes from a network that is not allowed.

---

## Validation Summary

| What I configured | How I tested it |
|---|---|
| Compliance group RBAC | Verified Reader and Storage Blob Data Reader assignments |
| Read-only blob access | Auditor write attempt was denied |
| Anonymous access | Disabled |
| Storage account keys | Disabled |
| TLS | Minimum version set to TLS 1.2 |
| Lifecycle management | Verified 30/90/365-day rule |
| Versioning | Restored a previous version of `report.txt` |
| Soft delete | Enabled for blobs and containers |
| Storage firewall | Approved client worked while Cloud Shell was blocked |

---

## Troubleshooting

### Management Plane vs. Data Plane

This was probably the biggest lesson from the lab.

At first, it was easy to think that Reader or Owner access to the Azure resource should also mean access to the files inside the storage account.

It doesn't.

Azure resource permissions and blob-data permissions are separate. Once I understood that, the role assignments made much more sense.

### RBAC vs. Network Access

I also saw that having the right role does not guarantee that a request will reach the storage account.

The storage firewall can block the connection before the storage service allows the data operation.

That is why the Cloud Shell test failed even though the identity itself could have valid Azure permissions.

---

## What I Learned

The main things I took away from this lab were:

- Azure RBAC and Storage data roles are not the same thing.
- Reader lets someone view an Azure resource but does not automatically let them read blob data.
- Storage firewall rules and RBAC solve different problems.
- Versioning and soft delete work together but protect against different types of mistakes.
- Lifecycle rules can reduce storage costs without manual cleanup.
- Testing failed access is just as useful as testing successful access.

---

## Scope

This was a focused portfolio lab, so I did not try to turn it into a full production storage environment.

Some things I would add in a larger project are:

- Private Endpoints
- private DNS
- customer-managed encryption keys
- Azure Monitor / diagnostic logs
- Azure Policy
- Infrastructure as Code

---

## Cleanup

After I finished testing, I removed the lab resources so they would not continue generating Azure charges.

---

## Repository Structure

```text
azure-secure-blob-storage/
├── README.md
└── screenshots/
    ├── blob-rbac-role-assignments.png
    ├── auditor-write-denied.png
    ├── storage-security-baseline.png
    ├── lifecycle-policy.png
    ├── data-protection-settings.png
    ├── blob-version-restored.png
    ├── storage-firewall-selected-networks.png
    └── firewall-validation.png
```

---

## Skills Used

- Azure Blob Storage
- Microsoft Entra ID
- Azure RBAC
- Storage Blob Data Reader
- Storage Blob Data Contributor
- Blob lifecycle management
- Blob versioning
- Soft delete
- Storage firewall rules
- Azure Portal
- Azure CLI
