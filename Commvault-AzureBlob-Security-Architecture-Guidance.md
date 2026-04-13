# Securing Commvault Backups on Azure Blob Storage
### A Repeatable Architecture Guide

**Scenario:** A customer moves their Commvault primary backup copy from Air Gap storage to Azure Blob Storage to cut costs — but wants to make sure that data is just as safe.

---

## The Problem in Plain English

Moving backups to Azure Blob is a smart cost move. But there is a catch: if the storage account lives inside your main Azure environment, anyone with enough admin rights — or anyone who compromises an admin account — can potentially reach it. Network restrictions are helpful but they only go so far. What the customer needs is a design where even a fully compromised admin in their main organization has no path to the backup data.

There are also secondary risks to think about: what if the backup tool itself gets compromised? What if someone with legitimate access makes a mistake and deletes something? The architecture below addresses all of these, layer by layer.

---

## The Approach at a Glance

The architecture is built on a single core idea: **put the backup storage account in a completely separate Azure environment that your main organization has no visibility into, then add platform-native controls so that even authorized credentials can't permanently destroy data.**

There are four layers:

| Layer | What it does |
|---|---|
| 1 — Tenant Isolation | Cuts off your main organization from even seeing the backup environment |
| 2 — Resource Lock | Prevents the storage account from being deleted, even by an admin |
| 3 — Blob Immutability (WORM) | Makes backup data untouchable for a defined retention window |
| 4 — Soft Delete | Provides a recovery safety net for anything outside the WORM window |

Each layer is independent and defends against a different type of failure. Together, they create a posture where no single compromised credential — inside or outside your organization — can result in permanent data loss.

---

## Layer 1 — Tenant Isolation

### What it is and why it works

An Entra ID tenant is a completely isolated directory. Microsoft treats each tenant as a separate organizational boundary. There is no inheritance, no cross-visibility, and no shared trust — unless you explicitly configure one.

The idea here is to create a second Entra ID tenant and place the backup storage account inside it. Your primary organization simply has no way to see it, reach it, or touch it.

**What you are explicitly NOT doing (and why that matters):**

- **No AD Connect or Cloud Sync** — these would sync identities from your main organization into the backup tenant, defeating the isolation
- **No B2B guest invitations** — inviting any user from the primary tenant creates a cross-tenant trust link
- **No external IdP federation** — this would allow any identity from a linked provider to authenticate

The result: the only way anyone gets into the backup tenant is through a locally created account inside it. No back door through your main AD, no inherited admin rights, nothing.

### How to set it up

**Step 1 — Create the backup tenant**

Create a new Entra ID tenant from the Azure portal: `portal.azure.com` → search "Entra ID" → "Manage tenants" → "Create". Give it a meaningful name (e.g. `contoso-backup.onmicrosoft.com`).

Do not connect it to your existing directory in any way.

**Step 2 — Create a small number of break-glass admin accounts**

Provision only the accounts that genuinely need access. These should be treated like physical keys to a server room — used only when necessary, never for day-to-day work.

Harden each account with:
- **FIDO2 hardware security key** — this is the highest MFA assurance level available in Entra ID. Unlike TOTP or SMS codes, FIDO2 keys are phishing-resistant.
- **Conditional Access policy** — require the FIDO2 key and restrict sign-in to trusted IP ranges only. Any authentication attempt from an unrecognized IP is blocked outright.

**Step 3 — Register the Commvault service principal in the backup tenant**

In the backup tenant, go to Entra ID → App registrations → New registration. Name it something like `sp-commvault-backup`. Note the Application (client) ID and Tenant ID — you will need these in Commvault.

Create a client secret (or preferably a certificate) under Certificates & secrets.

**Step 4 — Create the storage account and assign minimum RBAC**

Create your storage account and backup container in the backup tenant's subscription. Then assign the service principal the minimum permissions it needs — scoped to the container, not the whole storage account:

```bash
# Assign Storage Blob Data Contributor on the container only
az role assignment create \
  --assignee "<appId-of-the-service-principal>" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<storageAccount>/blobServices/default/containers/<containerName>"
```

`Storage Blob Data Contributor` allows Commvault to read and write blobs. It does not allow deleting the storage account itself, modifying network rules, or accessing other containers.

### Billing — keeping everything under one agreement

The backup tenant can be linked to your existing Enterprise Agreement (EA) or Microsoft Customer Agreement (MCA) billing account. This is purely a financial relationship — linking a tenant to a billing account does not create any identity trust between the two tenants.

Once linked, shared-scope reservations automatically apply to eligible subscriptions in the backup tenant. No changes to identity configuration, RBAC, or tenant settings are needed.

> Confirm the exact linking steps with your Microsoft account team or CSP before proceeding — the process varies slightly between EA and MCA agreements. See the references section at the end of this document.

### What this looks like if something goes wrong

> **Scenario: A Global Admin account in the primary organization is compromised.**
>
> The attacker logs in with the stolen credentials and reviews every Azure subscription visible to the primary tenant. The backup storage subscription does not appear — it belongs to a completely separate tenant that the primary organization has zero connection to. They try to switch directories and are denied. They find the Commvault service principal credentials and attempt to authenticate, but their IP is not on the Conditional Access trusted list. The token request is blocked. They have no way in.

---

## Layer 2 — Azure Resource Lock

### What it is and why it works

A resource lock adds a hard stop to any deletion of the storage account or resource group. Even an account with full Owner-level RBAC cannot delete a locked resource — it has to explicitly remove the lock first. That lock removal is a separate, independently logged event you can alert on.

This does not require complex configuration. It is a single command and it applies immediately.

### How to set it up

```bash
az lock create \
  --name "BackupDeleteLock" \
  --lock-type CanNotDelete \
  --resource-group <backup-resource-group>
```

Apply the lock at the **resource group level** so it covers the storage account, network rules, and any other resources inside it.

Set up an Azure Monitor alert to notify your security team any time a lock removal is attempted. In a well-run environment, that should never happen outside of a formal change request.

### What this looks like if something goes wrong

> **Scenario: An operator with legitimate access accidentally runs a cleanup script targeting the wrong resource group.**
>
> The deletion hits the lock and is refused. Azure returns an error. The storage account and all its contents remain intact. The attempted lock removal is logged as a separate auditable event, and the alert fires before anything is deleted.

---

## Layer 3 — Blob Immutability (WORM)

### What it is and why it works

WORM stands for Write Once, Read Many. When you apply a time-based immutability policy to a storage container and lock it, Azure physically prevents any blob inside that container from being deleted or overwritten for the duration of the retention period. This is enforced at the storage platform level — not at the application layer, not at the RBAC layer. It does not matter who is asking or what permissions they have.

This is the most important control for cyber resilience. It is what stops a ransomware operator — even one using legitimate, stolen Commvault credentials — from destroying backup data.

### How to set it up

```bash
# Step 1 — Create a time-based retention policy on the backup container (period is in days)
az storage container immutability-policy create \
  --account-name <storage-account> \
  --container-name <backup-container> \
  --period 30

# Step 2 — Retrieve the ETag of the policy you just created (needed to lock it)
az storage container immutability-policy show \
  --account-name <storage-account> \
  --container-name <backup-container> \
  --query etag \
  --output tsv

# Step 3 — Lock the policy (IRREVERSIBLE — do this only after thorough lab testing)
az storage container immutability-policy lock \
  --account-name <storage-account> \
  --container-name <backup-container> \
  --if-match "<etag from step 2>"
```

> **Before locking:** A locked immutability policy can only be extended — never shortened or removed. This is by design; it is what makes the protection meaningful. Test thoroughly in a lab first. For demos, leave the policy unlocked — you can still demonstrate the behavior, and a locked policy on a demo account is a permanent commitment.

### The Commvault compatibility check you must do before going to production

Commvault's backup writes are compatible with WORM — it appends new blobs and does not overwrite existing ones. However, Commvault's **pruning jobs** (which delete aged backup data) will be blocked for any blob still within its retention window.

Before enabling WORM, confirm:
- Your WORM retention period is **shorter** than Commvault's pruning schedule for this storage tier, OR
- Automated pruning is disabled for this storage tier

Test in a lab environment before applying a locked policy in production.

### What this looks like if something goes wrong

> **Scenario: Ransomware compromises a Commvault media agent and uses the stored Service Principal credentials to wipe backup data.**
>
> The ransomware authenticates successfully — it is operating from the media agent, which is on a trusted network. It starts calling the blob delete API. Azure refuses every single request. The blobs are within their retention window and the immutability policy is locked. The ransomware cannot delete, overwrite, or encrypt any of them. Backup data survives intact.

---

## Layer 4 — Soft Delete

### What it is and why it works

Soft delete is your recovery backstop for anything that falls outside the WORM window. When a blob or container is deleted, Azure quietly retains a hidden copy for a configurable grace period. It can be fully restored through the portal or API at any point during that window.

**How is this different from WORM?**

| | WORM (Immutability) | Soft Delete |
|---|---|---|
| Role | Prevention — the delete API call is refused entirely | Recovery — the delete happens but a copy is kept |
| When it helps | During the active retention window | After the WORM window expires, or before WORM is enabled |
| Protection level | Absolute — no one can bypass it | Requires someone to notice and restore within the window |

Think of WORM as a vault that cannot be opened during the retention period. Think of soft delete as a recycle bin — the file appears gone, but it can still be retrieved if you catch it in time. Both should be enabled; they cover different time windows.

### How to set it up

```bash
az storage account blob-service-properties update \
  --account-name <storage-account> \
  --resource-group <backup-resource-group> \
  --enable-delete-retention true \
  --delete-retention-days 30 \
  --enable-container-delete-retention true \
  --container-delete-retention-days 30
```

Recommended retention window: **30–90 days.** Soft-deleted blobs still incur storage costs during the grace period, so size the window to match your actual recovery needs.

Also consider enabling **blob versioning** (`--enable-versioning true` in blob service properties) — this creates a new version on every overwrite, protecting against accidental content changes as well as deletions.

### What this looks like if something goes wrong

> **Scenario: A backup blob ages out of its WORM window. During the brief gap before the next Air Gap copy cycle, it is accidentally deleted.**
>
> Soft delete quietly retained a copy. The blob is restored through the Azure portal. No data is lost.

---

## Commvault Authentication — Service Principal vs. Access Key

Always use a Service Principal. Avoid storage access keys if at all possible.

| | Service Principal | Access Key |
|---|---|---|
| RBAC scope | Scoped to a single container | Full storage account |
| Conditional Access | Enforced — IP restrictions apply | Not enforced — keys are not identity-bound |
| Audit trail | Full: IP, timestamp, and application per token | Activity logs only — no user attribution |
| Credential exposure | Short-lived tokens; access expires automatically | A leaked key grants full access until manually rotated |

**If Access Keys are unavoidable** (e.g. older Commvault versions that do not support service principal auth):
- Store the key exclusively in Commvault's encrypted credential vault — never in a config file or script
- Enable automated key rotation via Azure Key Vault
- Ensure the storage firewall is locked to known VNet IPs
- Enable Microsoft Defender for Storage for anomaly alerting

---

## Threat Coverage Summary

| Threat | How it is handled |
|---|---|
| Compromised admin in primary org | Layer 1 — they cannot see or reach the backup subscription at all |
| Compromised Commvault service principal | Layer 3 — WORM refuses blob deletions within the retention window |
| Accidental deletion by an authorized operator | Layers 2, 3, 4 — lock blocks resource deletion; WORM and soft delete protect blobs |
| Ransomware on Commvault infrastructure | Layer 3 — SP credentials are present but WORM refuses all delete calls |
| Long-term retention beyond WORM window | Commvault Air Gap copy — already in the customer's architecture |

**One residual risk worth noting:** WORM cannot protect data after the retention window expires. The customer's Air Gap copy should cover long-term retention requirements beyond that window.

---

## Implementation Checklist

| # | Action | Priority |
|---|---|---|
| 1 | Create second Entra ID tenant — no IdP links, no B2B, no federation | Required |
| 2 | Create break-glass admin accounts with FIDO2 + Conditional Access (IP restriction) | Required |
| 3 | Register Commvault as a service principal in the backup tenant | Required |
| 4 | Assign Storage Blob Data Contributor scoped to the container only | Required |
| 5 | Link backup tenant subscription to existing EA or MCA billing account | Required |
| 6 | Apply CanNotDelete resource lock at resource group level + lock removal alert | Strongly recommended |
| 7 | Enable blob soft delete and container soft delete (30–90 day window) | Strongly recommended |
| 8 | Test Commvault write and pruning behavior in a lab environment | Required before WORM |
| 9 | Apply WORM immutability policy; lock in production only after successful lab test | Strongly recommended |
| 10 | Enable Microsoft Defender for Storage for real-time anomaly detection | Recommended |

---

## Microsoft Documentation References

**Tenant isolation and identity:**
- [What is an Entra ID tenant?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)
- [Register an application in Microsoft Entra ID (service principals)](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app)
- [Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)
- [FIDO2 security key authentication](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-authentication-passwordless-security-key)

**Billing linkage across tenants:**
- [Associate or add a subscription to your Entra ID tenant](https://learn.microsoft.com/en-us/entra/fundamentals/how-subscriptions-associated-directory)
- [EA — Add accounts and subscriptions across tenants](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/direct-ea-administration#add-another-enterprise-administrator)
- [MCA — Transfer billing ownership of a subscription](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/mca-request-billing-ownership)
- [Azure Reservations — shared scope across subscriptions](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/manage-reserved-vm-instance#change-the-reservation-scope)

**Data protection controls:**
- [Blob immutability policies (WORM)](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview)
- [Lock time-based retention policies](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-policy-configure-time-based-retention)
- [Soft delete for blobs](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview)
- [Soft delete for containers](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-container-overview)
- [Blob versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-overview)
- [Lock resources to prevent deletion (resource locks)](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources)

**Monitoring and threat detection:**
- [Microsoft Defender for Storage](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-storage-introduction)

---

*Last updated: April 2026 — Applies to Azure Blob Storage, Microsoft Entra ID, Commvault v11.x and later*
