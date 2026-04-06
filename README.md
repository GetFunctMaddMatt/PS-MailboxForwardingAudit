# Get-MailboxForwardingAudit

![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-blue?logo=powershell)
![Platform](https://img.shields.io/badge/Platform-Exchange%20Online-blue?logo=microsoft)
![License](https://img.shields.io/badge/License-MIT-green)

Finds who configured email forwarding on an Exchange Online mailbox and when. Searches the Unified Audit Log the right way — because just searching by email address quietly misses half the records.

---

## The problem this solves

Someone reports that emails are being forwarded out of a mailbox. Maybe it's a compromised account, maybe it was set intentionally, maybe nobody knows. Your job is to find out who made the change and when.

You run a search against the audit log. Nothing comes back. The forwarding is clearly configured — you can see it on the mailbox right now — but the audit log says it never happened.

The reason: when forwarding is configured through the Exchange Admin Center GUI, Exchange logs the mailbox as an Azure AD Object ID (a GUID), not an email address. If your search is filtering by email, those records are invisible. You need to resolve the mailbox to its `ExternalDirectoryObjectId` first, then search by that GUID. Most scripts and most manual searches never do this.

This script does both — searches by GUID to catch GUI-originated changes, searches by email to catch PowerShell-originated changes, deduplicates the results, and tells you exactly what changed, who changed it, and where they were when they did it.

One more thing worth knowing: `Search-MailboxAuditLog` and `New-MailboxAuditLogSearch` were deprecated by Microsoft in March 2025. If you have existing scripts built around those cmdlets, they are broken. This script uses `Search-UnifiedAuditLog` which is the correct replacement.

---

## Requirements

- **ExchangeOnlineManagement module** — the script checks for it and offers to install if missing
- Exchange Online permissions: View-Only Audit Logs role or higher
- PowerShell 5.1+
- Unified Audit Log ingestion enabled for your tenant (the script checks this and warns you if it is off)

---

## Usage

```powershell
# Basic — prompts for mailbox and days to search
.\Get-MailboxForwardingAudit.ps1 -Mailbox "user@contoso.com"

# Open a save dialog to export results to CSV
.\Get-MailboxForwardingAudit.ps1 -Mailbox "user@contoso.com" -SaveToCsv

# Specify the output path directly
.\Get-MailboxForwardingAudit.ps1 -Mailbox "user@contoso.com" -OutputCsv "C:\Temp\audit.csv"

# Search further back (E5 licensing required for beyond 90 days)
.\Get-MailboxForwardingAudit.ps1 -Mailbox "user@contoso.com" -DaysBack 180
```

---

## Output

Results print as formatted blocks — one per change found:

```
=== FORWARDING CHANGES FOUND ===

Date         : 3/18/2026 5:00:36 PM
Performed By : admin@contoso.com
Client IP    : 20.228.89.52
Operation    : Set-Mailbox
Source       : GUI/EAC
Changes      :
               ForwardingSmtpAddress = external@gmail.com
               DeliverToMailboxAndForward = True
------------------------------------------------------------
```

`Source` tells you whether the change was made through the Exchange Admin Center or via PowerShell — useful when you are trying to determine whether something was done interactively or scripted.

If a `ForwardingAddress` is stored as an internal GUID (which Exchange does for internal recipients), the script resolves it to a display name and SMTP address so you are not left staring at a random GUID trying to figure out who it is.

---

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `-Mailbox` | Required | Email address of the mailbox to investigate |
| `-DaysBack` | Prompted (default 90) | Days back to search. Max 90 standard, 180 with E5 |
| `-SaveToCsv` | Off | Opens a save dialog to export results |
| `-OutputCsv` | None | Direct path to save CSV output |

---

## Notes

- Read-only. Nothing is changed on any mailbox.
- If the audit log comes back empty, the script explains why — licensing, audit not enabled when the change was made, or the change predates your retention window.
- Changes made via the Outlook desktop client are not always captured in the audit log. This is a Microsoft limitation, not a script limitation.
- The script only opens a new Exchange Online session if one does not already exist. If you are already connected it uses your existing session and disconnects cleanly when finished.

---

## License

MIT — use it, modify it, share it.
