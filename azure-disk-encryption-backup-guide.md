# Securing Azure Disk Exports While Enabling Backup Deduplication

**A technical guide to replacing Guest OS–level encryption with infrastructure-layer controls**

---

## The Use Case

Your VMs hold sensitive data. To protect it, you've enabled Guest OS–level disk encryption — tools like LUKS or Azure Disk Encryption that encrypt data from inside the operating system. It's a solid security choice.

But there's a catch: your enterprise backup system hates it.

When backup software tries to read an encrypted disk, it doesn't see your files — it sees a wall of scrambled, random-looking bytes. It has no way to tell which blocks are new, which are unchanged, and which are duplicates. So instead of smartly transferring only what's changed, it copies everything, every time. **Deduplication breaks. Compression breaks. Backups get slow, expensive, and hard to manage.**

---

## The Challenge

The instinct is to find a way to make backup software and OS-level encryption coexist — but that's fighting against the design of both systems. There's a cleaner answer: move encryption out of the Guest OS entirely and push it down to the infrastructure layer, where Azure handles it transparently using **Server-Side Encryption with Customer-Managed Keys (SSE-CMK)** and **Encryption at Host**.

With this model, Azure encrypts your data at the physical hypervisor level, before it ever touches the storage fabric. Your backup software reads clean, unencrypted blocks — just like it was designed to. Deduplication and compression work again. Your keys stay yours.

Problem solved. Except it opens a new one.

---

## The New Risk

When OS-level encryption is in place, it acts as a second security gate. Even if someone compromises an administrator account, the data on disk is still scrambled — they'd need the OS encryption key to do anything useful with it.

Remove that gate, and a different attack becomes possible. With infrastructure-layer encryption, Azure handles decryption transparently as data is read. That means anyone who can read the disk — or generate a download link for it — gets the decrypted data automatically.

Here's what a compromised Contributor account could do in just a few steps:

1. Take a snapshot of a VM's data disk
2. Ask Azure to generate a download link (a SAS URL) for that snapshot
3. Download the fully decrypted data through that link — no extra credentials needed

Azure decrypts silently as the data streams out. There's no alarm, no confirmation prompt. The data just leaves.

---

## The Approach: Trust the Control Plane, Not the OS

Instead of relying on the Guest OS as the last line of defense, we shift trust to the **Azure Control Plane** — the layer that governs what operations are even allowed to happen.

Two controls work together to close this gap:

| Control | What it does |
|---|---|
| **Azure Policy** | Blocks anonymous download links entirely — no SAS URL can be generated without Entra ID authentication |
| **Entra ID Custom RBAC** | Defines exactly who is allowed to export data — the automated backup system gets access, human administrators do not |

The result: your backup system runs nightly, exports data securely via authenticated tokens, and works perfectly. A compromised admin account — even a highly privileged one — hits a hard wall the moment it tries to generate an export link.

This guide walks through exactly how to build and verify this setup, step by step.

---

## Phase 1 — Enable Infrastructure-Layer Encryption

First, enable Encryption at Host and attach a Customer-Managed Key. This ensures data is encrypted at the physical hypervisor before it reaches the storage fabric.

```bash
# 1. Create a Disk Encryption Set linked to your Key Vault key
az disk-encryption-set create \
  --name $DES_NAME \
  --key-url $KEY_URL

# 2. Enable Encryption at Host on the VM
az vm update \
  --name $VM_NAME \
  --set securityProfile.encryptionAtHost=true

# 3. Apply the Customer-Managed Key to each disk
az disk update \
  --name $OS_DISK \
  --encryption-type EncryptionAtRestWithCustomerKey \
  --disk-encryption-set $DES_ID

az disk update \
  --name $DATA_DISK \
  --encryption-type EncryptionAtRestWithCustomerKey \
  --disk-encryption-set $DES_ID
```

---

## Phase 2 — Create the Backup Snapshot

Generate a point-in-time snapshot of the data disk. Because the source disk is encrypted with the Customer-Managed Key, the snapshot automatically inherits the same encryption.

```bash
az snapshot create \
  --resource-group $RG_NAME \
  --name $SNAPSHOT_NAME \
  --source $DATA_DISK_ID
```

---

## Phase 3 — Block Anonymous Download Links with Azure Policy

Deploy a custom Azure Policy that denies any attempt to create or modify a disk or snapshot without enforcing Entra ID authentication. If anyone tries to generate an old-style anonymous SAS URL, the ARM API call is rejected immediately.

**Policy definition:**

```json
{
  "if": {
    "anyOf": [
      {
        "allOf": [
          { "field": "type", "equals": "Microsoft.Compute/disks" },
          { "field": "Microsoft.Compute/disks/dataAccessAuthMode", "notEquals": "AzureActiveDirectory" }
        ]
      },
      {
        "allOf": [
          { "field": "type", "equals": "Microsoft.Compute/snapshots" },
          { "field": "Microsoft.Compute/snapshots/dataAccessAuthMode", "notEquals": "AzureActiveDirectory" }
        ]
      }
    ]
  },
  "then": {
    "effect": "deny"
  }
}
```

**Assign the policy and bring the new snapshot into compliance:**

```bash
# Assign the policy globally to the subscription or resource group
az policy assignment create \
  --name "require-aad-disk-auth" \
  --policy $POLICY_ID

# Update the snapshot to comply with the Entra ID requirement
az snapshot update \
  --resource-group $RG_NAME \
  --name $SNAPSHOT_NAME \
  --set dataAccessAuthMode=AzureActiveDirectory
```

---

## Phase 4 — Enforce Identity Guardrails with Custom RBAC

Define exactly who is permitted to export data. Two custom roles are needed.

### Role 1: Backup System Operator (Allowed to Export)

This role grants the `beginGetAccess` action — the specific API permission required to generate a secure download link.

```json
{
  "Name": "Backup System Operator",
  "IsCustom": true,
  "Actions": [
    "Microsoft.Compute/disks/read",
    "Microsoft.Compute/snapshots/read",
    "Microsoft.Compute/disks/beginGetAccess/action",
    "Microsoft.Compute/snapshots/beginGetAccess/action",
    "Microsoft.Compute/disks/endGetAccess/action"
  ],
  "NotActions": []
}
```

Assign this role directly to the backup system's Service Principal, scoped to the target resource group:

```bash
az role assignment create \
  --assignee <Backup-Service-Principal-AppID> \
  --role "Backup System Operator" \
  --scope "/subscriptions/<Sub-ID>/resourceGroups/<RG-Name>"
```

### Role 2: VM Admin No Export (Denied from Exporting)

This role covers IT staff. It allows full VM management but uses `NotActions` to explicitly prohibit generating export links.

```json
{
  "Name": "VM Admin No Export",
  "IsCustom": true,
  "Actions": [
    "Microsoft.Compute/virtualMachines/*",
    "Microsoft.Compute/disks/read",
    "Microsoft.Compute/snapshots/read"
  ],
  "NotActions": [
    "Microsoft.Compute/disks/beginGetAccess/action",
    "Microsoft.Compute/snapshots/beginGetAccess/action"
  ]
}
```

> **Note:** `NotActions` in Azure RBAC cannot be overridden by additional role assignments at the same scope. A human admin assigned this role cannot gain export access by also being assigned a broader role at the same level.

---

## Verification

### Scenario 1 — Backup System (Expected: Success)

Log in as the restricted Service Principal to simulate a scheduled backup job:

```bash
# 1. Request a secure SAS URI (Entra ID auth enforced)
az snapshot grant-access \
  --resource-group $RG_NAME \
  --name $SNAPSHOT_NAME \
  --access-level Read \
  --duration-in-seconds 3600

# 2. Obtain an Entra ID bearer token for the storage fabric
export ACCESS_TOKEN=$(az account get-access-token \
  --resource "https://storage.azure.com/" \
  --query accessToken \
  -o tsv)

# 3. Download the VHD blocks using the token as the Authorization header
curl -sS \
  --request GET \
  --url "${SAS_URI}" \
  --header "Authorization: Bearer ${ACCESS_TOKEN}"
```

**Result:** The storage fabric validates the Entra ID token and returns the VHD data. The download succeeds.

---

### Scenario 2 — Compromised Admin Account (Expected: Blocked)

Log in as a human administrator and attempt to strip the Entra ID requirement from the snapshot — the prerequisite for generating an anonymous link:

```bash
# Attempt to downgrade dataAccessAuthMode to None
az snapshot update \
  --name $SNAPSHOT_NAME \
  --set dataAccessAuthMode=None
```

**Result:** Azure Resource Manager returns a `PolicyViolation` error. The request is denied before it completes.

---

## Conclusion

This guide walked through a complete, end-to-end solution to a problem that often gets dismissed as a forced trade-off: you can have strong disk encryption, or you can have a performant backup system — but not both.

That trade-off is a false choice.

Here's what was covered and what was achieved:

- **Moved encryption to the right layer.** By replacing Guest OS encryption with SSE-CMK and Encryption at Host, the backup software can once again read blocks natively — restoring deduplication, compression, and sane backup windows — while your encryption keys remain fully under your control.

- **Closed the exfiltration gap.** Shifting encryption to the infrastructure layer doesn't weaken security, but it does change where the security boundary lives. An Azure Policy was deployed to ensure that boundary holds: no disk or snapshot can be accessed without Entra ID authentication. Anonymous download links are gone entirely.

- **Locked down who can export data.** Two custom RBAC roles were defined — one that explicitly allows the backup system to generate secure, token-gated export links, and one that explicitly blocks human administrators from doing the same. This is enforced at the control plane level, not by convention or trust.

- **Proved it works.** Both scenarios were verified in the lab. The backup system succeeded, completing a full authenticated download. The compromised admin account was blocked at the ARM layer with a `PolicyViolation` error before it could do anything useful.

The end state is a system where security and operational needs are no longer competing. Your backup software runs efficiently, your data exfiltration risk is controlled by Azure's own enforcement layer, and no amount of compromised credentials — short of your Key Vault itself — can silently walk data out the door.

---

## Key Trade-offs and Considerations

| Topic | Detail |
|---|---|
| **Key Vault availability** | SSE-CMK depends on your Key Vault being reachable. Plan for geo-redundancy or key backup accordingly. |
| **Snapshot inheritance** | Snapshots automatically inherit the CMK from the source disk — no additional configuration needed. |
| **Policy scope** | Apply the policy at the subscription or management group level to prevent bypasses via resource group moves. |
| **Service Principal rotation** | The Backup System Operator Service Principal credentials should follow your standard secret rotation policy. |
| **Existing disks** | Azure Policy with `deny` effect only applies on create/update calls. Run a remediation task to bring existing non-compliant disks into compliance. |

---

## Official Documentation

| Topic | Link |
|---|---|
| Encryption at Host | [End-to-end encryption using encryption at host](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption#encryption-at-host---end-to-end-encryption-for-your-vm-data) |
| Entra ID for Disk Access | [Secure Azure Managed Disk Downloads and Uploads](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/disks-enable-customer-managed-keys-portal) |
| Custom RBAC for Snapshot Export | [Azure resource provider operations — Microsoft.Compute/snapshots](https://learn.microsoft.com/en-us/azure/role-based-access-control/resource-provider-operations#microsoftcompute) |
