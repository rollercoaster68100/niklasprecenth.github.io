---
title: "The Case of the Missing AdvancedSettings: A Container Label Mystery"
date: 2026-05-18
draft: false
tags: ["Microsoft Purview", "Sensitivity Labels", "PowerShell", "Container Labels", "MembersCanShare", "Copilot Governance", "Oversharing"]
summary: "How a routine container label configuration uncovered an undocumented change in ExchangeOnlineManagement 3.9.x — and why it matters for Copilot governance."
author: "Niklas Précenth"
---

> **TL;DR:** Default Teams/SharePoint sharing is often *too generous* for Copilot-era governance. Container sensitivity labels + `MembersCanShare="MemberShareNone"` fix that **at creation time**. The gotcha: in ExchangeOnlineManagement **3.9.x** the values write fine via `-AdvancedSettings`, but you must verify them via **`.Settings`** (not `.AdvancedSettings`).

## Why this matters (especially with Copilot)

When you create a new Team in Microsoft 365, the default sharing posture often allows **every member** to share files/folders broadly. In a world without Copilot, that was already risky. In a world **with** Copilot, it becomes a governance gap.

Copilot searches across what users *can access*. If collaboration spaces are overshared by default, Copilot can surface information much more easily than a human would ever discover it.

At **Atea**, this is part of how we help customers prepare for Copilot: lock down collaboration defaults **before** AI amplifies whatever is already shared.

---

## Proof: The label enforces sharing (screenshot)

Below is the real-world outcome after creating a Team with the **Secure Team** container sensitivity label. SharePoint shows that **sharing permissions are managed by your organization**, and the options are **greyed out** — confirming the label is enforcing the setting.

{{< figure src="/images/memberscanshare-proof.png" alt="Site sharing settings: Sharing permissions are managed by your organization. Only site owners can share." caption="Label-enforced site sharing: Users cannot override sharing permissions. (MembersCanShare = MemberShareNone)" >}}

**What this demonstrates:**

- The site UI explicitly states: *"Sharing permissions are managed by your organization."*
- The three sharing choices are **locked**.
- The effective posture is **Only site owners can share files, folders, and the site**.

> **Tip (Hugo/PaperMod):** Place the image at `static/images/memberscanshare-proof.png` so it is served as `/images/memberscanshare-proof.png`.

---

## The goal

I needed a container sensitivity label for Microsoft Teams that would:

- Set the Team to **Private**
- Block guest access
- Restrict external sharing
- Ensure **only site owners can share** files, folders, and the site

The first three are configurable in the Purview GUI. The last one is **PowerShell-only** via `MembersCanShare`.

Microsoft Learn documents this pattern for container labels (Teams/Groups/Sites).

### MembersCanShare values

| SharePoint option (UI wording) | PowerShell value |
|---|---|
| Site owners and members can share files, folders, and the site | `MemberShareAll` |
| Members can share files/folders, only owners can share the site | `MemberShareFileAndFolder` |
| Only site owners can share files, folders, and the site | `MemberShareNone` |

---

## The problem

I did what the docs (and most blogs) suggest:

```powershell
Set-Label -Identity <GUID> -AdvancedSettings @{MembersCanShare="MemberShareNone"}
```

The command executed without errors.

Then I verified like everyone does:

```powershell
(Get-Label -Identity <GUID>).AdvancedSettings
```

…and it was empty. Every time.

At face value it looked like the setting was not persisting — which would be a serious issue if you are relying on this to enforce secure-by-default sharing.

---

## The investigation

I went through the usual suspects:

- Tried a **new label** vs an old label
- Tested multiple advanced settings (`MembersCanShare`, `DefaultSharingScope`, `Color`)
- Verified module version (ExchangeOnlineManagement 3.9.2)
- Checked the label object output with `Get-Label | fl *`

Everything *looked* like a silent failure.

---

## The “aha” moment

The breakthrough came from reading the label object more closely.

In ExchangeOnlineManagement **3.9.x**, the advanced settings are **stored in `.Settings`** (and not in `.AdvancedSettings`).

So this:

```powershell
(Get-Label -Identity <GUID>).Settings
```

…returned what I expected all along:

```text
[memberscanshare, MemberShareNone]
[defaultsharingscope, SpecificPeople]
[color, #FF0000]
```

**In other words:** the write commands work exactly as documented — it’s the read-back property that has changed.

---

## Complete working configuration (copy/paste)

### 1) Create the label

```powershell
Connect-IPPSSession

New-Label -DisplayName "Secure Team" \
  -Name "SecureTeam" \
  -Tooltip "Private team with restricted sharing (owners only)" \
  -ContentType "Site, UnifiedGroup"

$labelId = (Get-Label -Identity "SecureTeam").Guid.ToString()
```

### 2) Configure container behaviour (privacy + guests)

> Note: `-LabelActions` expects JSON.

```powershell
Set-Label -Identity $labelId -LabelActions '{"Type":"protectgroup","SubType":null,"Settings":[{"Key":"privacy","Value":"private"},{"Key":"allowemailfromguestusers","Value":"false"},{"Key":"allowaccesstoguestusers","Value":"false"},{"Key":"disabled","Value":"false"}]}'
```

### 3) Configure SharePoint sharing restrictions (owners only)

```powershell
Set-Label -Identity $labelId -AdvancedSettings @{MembersCanShare="MemberShareNone"}
```

### 4) Verify (critical!)

```powershell
# In EXO Mgmt 3.9.x, verify via .Settings
(Get-Label -Identity $labelId).Settings

# Optional: search for the key
(Get-Label -Identity $labelId).Settings | Out-String | Select-String -Pattern "memberscanshare"
```

### 5) Publish the label

```powershell
New-LabelPolicy -Name "Secure Team Policy" \
  -Labels $labelId \
  -Comment "Publishes Secure Team container label" \
  -ExchangeLocation "All"
```

---

## Key takeaways

1. **Default sharing is often too generous** for Copilot-era governance.
2. Container sensitivity labels let you apply the right controls **at creation time**.
3. `MembersCanShare="MemberShareNone"` delivers the *"only owners can share"* posture.
4. In ExchangeOnlineManagement **3.9.x**, verify advanced settings via **`.Settings`** (not `.AdvancedSettings`).
5. A single screenshot like the one above is the best way to prove enforcement to stakeholders.

---

## References

- Microsoft Learn: Sensitivity labels for Teams/Groups/Sites (container labels): https://learn.microsoft.com/en-us/purview/sensitivity-labels-teams-groups-sites
- Tony Redmond (Office 365 IT Pros) on the setting (historical background): https://office365itpros.com/2022/04/27/sensitivity-label-setting-spo/
- Nikki Chapple (container labels + governance): https://nikkichapple.com/configure-container-sensitivity-labels-microsoft-365/
- Simon Ågren (AgrenPoint): https://www.agrenpoint.com/container-labels-in-powershell/

---

*Troubleshooting and blog post created with the help of Microsoft 365 Copilot — my AI pair-debugger for the afternoon.*
