# Policy + Automation new-tenant live test (2026-09-27)

**Guide sections 1–8 passed in a brand-new Microsoft Entra tenant and Azure
Free Trial subscription with the same source pins as the
[2026-09-06 record](LIVE-TEST-2026-09-06-policy-automation.md).** Two
privately rendered values changed because the environment required them; no
source, script, definition, initiative or fixture changed. Some guide sections
overlapped instead of running strictly in order; see
[Execution order](#execution-order). Optional extensions probed the Deny
definitions with validation-only requests, mostly right after promotion. After
section 8, they exercised the VM audit policies and the Automation runbook
against one short-lived private VM with real backup protection.

Only tenant-neutral outcomes appear here. Live names, identities, resource,
request and job IDs, and raw responses remain in private evidence.

## Scope and environment

- One brand-new Microsoft Entra tenant with security defaults on, and one Azure
  Free Trial subscription with its spending limit on. The operator was a
  tenant member account holding Owner on that subscription.
- A Linux workstation with Azure CLI 2.90.0, the `automation` extension
  1.0.0b2, Bicep 0.47.16 and Python 3.14. Local PowerShell 7.6 ran the offline
  Automation tests; the deployed runbook still used the PowerShell 7.4
  runtime. A dedicated `AZURE_CONFIG_DIR` kept the workstation's existing
  Azure CLI sign-in untouched.
- Device-code sign-in failed with `AADSTS530035`; browser `az login` worked.
  Microsoft documents that security defaults block device code flow and that,
  starting 2026-07-01, new tenants block it as part of security defaults.
  [Security defaults](https://learn.microsoft.com/en-us/entra/fundamentals/security-defaults)
- Of the guide's nine providers, only `Microsoft.Authorization` was already
  registered. Registering the other eight plus `Microsoft.Compute` (for the
  optional VM) took about 13 minutes of wall-clock time, roughly 75 seconds or
  more each.
- The subscription's regional vCPU quota was 4.

## Sources and validation

- Guide: `ac5e034b5aea4f1c9769e5fa8457c2a6da938d5a` on `main`. Its deployable
  Policy core was diff-equal to the qualified
  `d07fe194b46a4b1df9f20e04f6454dc1f3f81148`.
- Automation helper: `03839a29b0fff02442d88a414d7ac32851d227c7`.
- Unchanged Policy fixture SHA-256:
  `ee6a382443881993fea49ef25f9c6c89ecaff38addb1faa04180b22fc5f0eaad`.
- Unchanged Backup runbook SHA-256:
  `2cef45acc81b04a6bbcd62582db6f974102ae98f2de79231a90907f49a7dd555`.

All offline gates passed: 16 definitions and one initiative validated, 50
Python tests, the public-content privacy gate, 45 of 45 Automation behavioral
scenarios, the static and shell checks, and both Bicep builds.

## Environment-driven deviations

| Guide value | Value used | Reason |
|---|---|---|
| `AUTOMATION_LOCATION="centralus"` | `eastus` | Free Trial and Azure for Students subscriptions can create one Automation account per region, only in an allow-listed set of regions that does not include Central US. The guide's existing `unique` step rendered allowed locations as `eastus` and `eastus2`. [Automation limits](https://learn.microsoft.com/en-us/azure/automation/automation-subscription-limits-faq#service-and-subscription-limits) |
| Example `allowedVmSkus` | Added `Standard_D2als_v7` | In East US 2 this subscription was offered no v5 D-series size, v6 sizes were restricted, and v7 sizes were unrestricted. This matters only for the optional VM extension; the guide itself creates no VM. |

## Execution order

The guide asks for its numbered sections in order in one terminal. This run
executed the guide's blocks from private step scripts, and some steps
overlapped:

- Provider registration continued while sections 3 and 4 ran. Six
  registrations, including Storage, Network and Key Vault, finished after the
  section 4 definitions were published. Recovery Services and Compute finished
  only after the Policy fixture deployment had completed.
- The main validation-only Deny probes and the detached inline-rule NSG ran
  within two minutes of promotion, while section 5 remediation was starting.
  The port-range and port-list child-rule probes ran after section 8.
- Sections 6 and 7 ran while section 5's remediation tasks were still
  processing. The remediations finished eight to nine minutes after section
  7 passed. Section 5's final gate and direct checks ran after that and passed,
  followed by section 8.
- Section 5's final gate therefore evaluated 32 rows, 30 of them compliant,
  instead of the Policy-only shape. It also saw the Automation Account, both
  vaults and the detached NSG. The two non-compliant rows were the intentional
  unprotected share and the detached NSG's child rule. No section 5 gate
  predicate depends on NSG Deny rows. The NSG was deleted after section 5
  passed and before section 8.

## Guide sections 1–8

| Stage | Observed result |
|---|---|
| Fresh-only collision proof (section 4) | Passed; `deploy.sh` then published the 16 definitions and initiative. The 2026-09-06 run adopted existing definitions, so this is the first recorded fresh publication through the current guide. `undeploy.sh` was not invoked. |
| Report-only assignment | `DoNotEnforce`, with exactly the three expected remediation grants |
| Report-only evaluation | 21 rows: 13 compliant, six missing inherited tags, one missing Key Vault diagnostic setting and one intentionally unprotected share, the same shape as 2026-09-06 |
| Promotion and remediation (section 5) | Tag remediation 6/6 and Key Vault diagnostics 1/1 deployments, zero failed; they finished about 17.5 and 18.5 minutes after submission, with no resubmission |
| Section 5 final checks | Final gate (32 rows, 30 compliant; see [Execution order](#execution-order)), six direct `CostCenter` checks and the exact diagnostic setting passed |
| Automation fixture and publication (section 6) | Automation Account in `eastus` and two empty vaults; runbook published by the pinned byte-preserving helper: `Published`, `PowerShell74`, SHA-256 equal to the qualified source |
| Optional pre-RBAC audit job | `Failed` with an authorization (403) exception; no `SUMMARY`; policy unchanged |
| Reader readiness | The first readiness job completed about 35 seconds after the RG-scoped reader grant: `AlreadyCompliant`, one matched policy, zero writes |
| Seed | RG canary `DoNotTier`; subscription-vault canary still `TierRecommended`. The policy GET returned no ETag, so the guide's seed PUT was sent without `If-Match` |
| Audit, apply and repeat (section 7) | `WouldEnableTierRecommended` 1/0/0; `EnabledAndVerified` 1/1/1 on the first attempt, with no 403 retry; `AlreadyCompliant` 0/0/0; writer revoked |
| Final invariants (section 8) | All passed, and passed again after the optional extension was cleaned up |

Counts are candidate/submitted/written, as in the guide.

## Validation-only Deny probes

The guide deliberately ships no negative probe templates. As an extension,
this run submitted `az deployment group validate` requests against the
enforced canary assignment. Validation creates nothing, and a post-check found
no probe resources. Each denial returned `RequestDisallowedByPolicy` naming
exactly the expected reference ID.

| Probe | Expected reference ID | Result |
|---|---|---|
| Positive control: compliant storage, NSG, NIC, Key Vault and allowed-size VM | None | Validated successfully |
| Storage with HTTPS-only disabled | `deny-storage-https-disabled` | Denied |
| Storage with `TLS1_0` | `deny-storage-minimum-tls` | Denied |
| Storage allowing public blob access | `deny-storage-public-blob-access` | Denied |
| Resource in `westus2` | `allowed-locations` | Denied |
| NIC with a public IP | `deny-nic-public-ip` | Denied |
| Key Vault without purge protection | `deny-keyvault-purge-protection-disabled` | Denied |
| `Standard_D4als_v7` VM, a size that was available and within quota | `allowed-vm-skus` | Denied |
| VM with an unmanaged `osDisk.vhd.uri` | `deny-vm-unmanaged-disks` | Denied |

This run did not probe `deny-sql-public-network-access` or
`require-tag-on-resource-groups`; the
[2026-08-31 record](LIVE-TEST-2026-08-31-policy-automation.md) covers them.

## NSG observations

`deny-nsg-open-management-ports` evaluates the child
`Microsoft.Network/networkSecurityGroups/securityRules` type. The results match
both NSG entries in [DESIGN.md Known limitations](DESIGN.md#known-limitations).

| Request shape | Source and destination port | Result |
|---|---|---|
| Rules inline in the parent NSG body | `*` to 22; `::/0` to 3389 | Not denied at validation |
| The same two rules as child `securityRules` resources | `*` to 22; `::/0` to 3389 | Denied |
| Child rule with `destinationPortRanges` `["80","3389"]` | `*` | Denied |
| Child rule with range `1-65535` | `*` | Not denied |
| Child rule with range `20-25` | `Internet` | Not denied |

One inline case was confirmed with a real resource. Shortly after promotion,
while the assignment was enforced, a detached NSG (no subnet and no NIC) with
an inline 22-from-`*` rule was created. Modify added the inherited tag. A
later compliance scan reported its child rule `NonCompliant`; that row was
part of section 5's final state. The NSG was deleted after section 5 passed
and before section 8. No other probe created a resource.

## Optional VM extension: Policy results

A private `Standard_D2als_v7` Ubuntu 24.04 VM with no public IP was created
without tags or identity.

- Request-time Modify added the `CostCenter` tag to the VM, its implicitly
  created OS disk and its NIC.
- Before protection, `allowed-locations`, `allowed-vm-skus`,
  `deny-vm-unmanaged-disks` and `inherit-tag-from-resource-group` reported
  `Compliant`. `audit-vm-backup-protection` and
  `audit-vm-system-assigned-identity` reported `NonCompliant`.
- After backup protection and a system-assigned identity were enabled, both
  audits reported `Compliant` about 16 minutes later, about a minute after a
  further scan was triggered.
- Section 8 had started an on-demand scan before the VM existed. A
  `trigger-scan` issued after the VM was created appeared to join that
  still-running scan. About 23 minutes after creation, the VM still had no
  Policy rows.
  Microsoft documents that a new resource's compliance status usually appears
  around 15 minutes after creation, but no rows had appeared in that window. A
  fresh scan, triggered after the earlier one had finished, produced the VM's
  rows within about two minutes. The likely cause is that the joined scan had
  enumerated the group before the VM existed; that was not observed directly.
  Microsoft documents that a scope already running an on-demand scan does not
  start a new one.
  [Evaluation triggers](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/get-compliance-data#evaluation-triggers),
  [On-demand scan](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/get-compliance-data#on-demand-evaluation-scan-using-rest)

The guide's required-resource-ID gate does not accept such an incomplete
state: it waits for exact resource rows. The guide now describes how to wait
for a running scan and start a fresh one if that gate times out.

## Optional VM extension: Automation summary

A dedicated Recovery Services vault in the VM's region protected the VM with a
V1 daily policy (30 days, 12 weeks, 12 months, 2 years) whose `ArchivedRP`
tiering mode started at `DoNotTier`. Soft delete on the new vault was
`AlwaysOn`, which Microsoft documents as permanently enabled for all newly
created vaults. [Secure by default](https://learn.microsoft.com/en-us/azure/backup/secure-by-default)

The runbook then ran against that policy while it protected one real VM, with
exact vault and policy names, `ExpectedMatches=1` and `MaxChanges=1`:

- `MaxProtectedItemsPerPolicy=0` skipped with
  `SkippedProtectedItemsExceedLimit`, 0/0/0, and made no write.
- `MaxProtectedItemsPerPolicy=1` without the override skipped with
  `SkippedNoConcurrencyToken`, 0/0/0. The `backupPolicies` GET
  (api-version 2025-08-01) returned no ETag in the body or response headers.
- With `AllowWriteWithoutETag=true`, inside an exclusive single-operator window
  on a dedicated policy: audit 1/0/0, apply `EnabledAndVerified` 1/1/1 on the
  first attempt, and repeat `AlreadyCompliant` 0/0/0. The temporary writer
  was revoked and verified absent; only the RG-scoped discovery reader
  remained.
- Non-tiering policy properties were unchanged, `protectedItemsCount` stayed
  1, and the item stayed protected by the same policy. Azure started a
  `ConfigureBackup` job on the item right after the policy write; it completed
  in about 10 seconds.
- An on-demand backup submitted before the write completed afterwards, after
  about 26 minutes, with one recovery point. The item therefore had no
  completed recovery point when the policy was written; a policy whose items
  hold mature recovery points was not tested.

Setup details, Azure CLI quirks and limits are in the
[Automation record](https://github.com/kevo099/azure-backup-smart-tiering-automation/blob/main/docs/LIVE-TEST-2026-09-27.md).

## Cleanup and retained state

- Protection was stopped with delete-data, and the item became soft-deleted.
  The vault's soft-delete retention was left at its default, which Microsoft
  documents as 14 days (configurable from 14 to 180 days). `AlwaysOn` means
  soft delete cannot be disabled.
  [Soft-delete retention](https://learn.microsoft.com/en-us/azure/backup/secure-by-default#soft-delete-retention-period)
- The VM, its OS disk and its NIC were deleted. The detached NSG had been
  deleted earlier, after section 5 passed.
- Section 8 was rerun after this cleanup and passed.
- The guide's showcase remains retained as intended. The extension's vault is
  in the same resource group. It was retained with its soft-deleted item until
  the retention period ends; vault and group deletion were not attempted.
- Microsoft's pages differ on what deleting such a vault does. The
  [vault-deletion article](https://learn.microsoft.com/en-us/azure/backup/backup-azure-delete-vault#before-you-start)
  says a vault holding soft-deleted items can't be deleted. The
  [secure-by-default article](https://learn.microsoft.com/en-us/azure/backup/secure-by-default#soft-delete-for-vaults)
  says such a vault instead moves into a soft-deleted state. This
  repository's [2026-07-28](LIVE-TEST-2026-07-28-file-share-backup.md) and
  [2026-08-25](LIVE-TEST-2026-08-25-file-share-backup-enforcement.md) Azure
  Files runs observed the second behavior: the vault became a soft-deleted
  vault record and group deletion completed. This run tested neither path for
  a VM item.

## Boundaries

- The run was agent-executed; it is not an observed human study.
- It used one Free Trial subscription. Automation regions, VM sizes and vCPU
  quota differ for other offers and subscriptions.
- Some guide sections overlapped; see [Execution order](#execution-order).
- Deny probes were validation-only, except the single detached inline-NSG
  confirmation.
- No archive-tier movement, restore, V2 or hourly policy, multi-item policy,
  or production-scale behavior was qualified.
- `AllowWriteWithoutETag` was exercised only in an exclusive test window, on a
  policy whose single item had no completed recovery point yet. It is not
  evidence of safe concurrent writes.
- Vault and resource-group deletion after soft delete were not tested.
- The storage-account smart-tier repository was not exercised.
