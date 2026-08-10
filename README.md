# ITS Help Desk Portfolio — Won Seo

## About This Project

I'm a biology major preparing to apply for a campus ITS help desk student worker position. Since I had no prior IT background, I built a self-hosted lab environment from scratch — two virtual machines networked together, with one configured as a Windows Server Domain Controller — to learn and practice the core skills a Tier 1 help desk technician uses every day: account creation, password resets, account unlocks, and group management inside Active Directory.

This repo documents that process step by step, including the mistakes I ran into and how I resolved them, because troubleshooting through real problems is the actual skill being demonstrated here — not just following a tutorial.

**Goal:** Land a help desk student worker role. **Cost:** $0 — everything here runs locally using free evaluation software.

---

## Tools Used

- **VirtualBox** — free hypervisor (virtualization software) used to run virtual machines
- **Windows Server 2022** (free 180-day evaluation) — configured as the Domain Controller
- **Windows 11 Enterprise** (free evaluation) — configured as a domain-joined client machine
- **Active Directory Domain Services (AD DS)** — Microsoft's centralized directory service for managing users, computers, and permissions

---

## Repo Structure

```
/its-helpdesk-portfolio
  /01-lab-setup         → VirtualBox + VM installation screenshots
  /02-active-directory  → AD DS promotion, OU/user/group tasks, troubleshooting
  /03-osticket          → (in progress) ticketing system setup
  /04-integration       → (in progress) full end-to-end support scenarios
  README.md             → this file
```

---

## Phase 1 — Lab Environment Setup

**What I did:** Installed VirtualBox, then built two virtual machines — a Windows Server 2022 VM (`WinServer-DC`, destined to become the Domain Controller) and a Windows 11 Enterprise VM (`Win11-Client`) — and networked them together on an isolated internal network so they could communicate with each other.

| Screenshot | What it shows |
|---|---|
| `1` | Both VMs (`WinServer-DC` and `Win11-Client`) listed and built in the VirtualBox Manager |
| `2.0` / `2.1` | Server VM hardware allocation — 4096 MB RAM, 2 CPUs |
| `3.0` / `3.1` | Client VM hardware allocation — 4096 MB RAM, 2 CPUs |
| `4.0` / `4.1` | Network adapter settings for both VMs, set to **Internal Network** (`LabNetwork`) instead of the default NAT, so the two VMs can see and talk to each other directly |
| `5` (DC) | WinServer-DC desktop after first successful login |
| `5` (Client) | Win11-Client home screen after setup, logged in with a local account |

**Why Internal Network instead of the default (NAT):** By default, VirtualBox gives each VM internet access but keeps it isolated from other VMs — fine for browsing, but it means the two machines can't see each other at all. Switching both VMs to **Internal Network** with the same network name (`LabNetwork`) puts them on their own private "wire," so they can communicate directly. The trade-off is that neither VM has real internet access anymore — which is fine, since Active Directory doesn't need it.

**A real problem I hit and fixed:** The first time I booted the Windows 11 Enterprise VM, it failed to boot at all — the installer showed "No bootable option or device was found." This was because Windows 11 requires **EFI** (the modern BIOS replacement) along with TPM 2.0 and Secure Boot, and the VM briefly shows a "press any key to boot from CD or DVD" prompt during boot that has to be caught within a few seconds — if you miss it, the VM gives up and won't boot from the installer at all. I fixed it by re-mounting the ISO and pressing a key immediately when the prompt appeared.

---

## Phase 2 — Active Directory Deep Dive

**What I did:** Promoted `WinServer-DC` from a plain Windows Server install into an actual **Domain Controller** by installing the AD DS role and running through the promotion wizard, then performed the core Tier 1 tasks: creating organizational units, creating and managing user accounts, resetting passwords, unlocking a locked account, disabling an account, and managing security group membership.

### Promoting the server to a Domain Controller

| Screenshot | What it shows |
|---|---|
| `7` | Server Manager dashboard after the AD DS, DNS, and related roles were installed |
| `8` | The "Before You Begin" screen of the Add Roles and Features Wizard |

**Installing the role vs. becoming a Domain Controller — these are two separate steps.** Installing "Active Directory Domain Services" through Server Manager only adds the *software* to the server; it doesn't actually configure anything yet. The server only becomes a functioning Domain Controller after a second step: running the **Active Directory Domain Services Configuration Wizard** and choosing to promote the server, which is where the actual domain gets created.

**The `lab.local` domain name:** During promotion, I chose **"Add a new forest"** (since this was a brand-new setup with nothing existing yet) and named the root domain `lab.local`. A couple of notes on this:
- A **forest** is just the top-level container that holds one or more domains together — for a single-domain lab like this, choosing "Add a new forest" automatically creates a forest containing just this one domain.
- I used `.local` instead of a real top-level domain like `.com` because that's standard practice for private, internal-only domains — it guarantees the name never conflicts with anything on the real internet, since `.local` isn't a publicly routable domain suffix.
- During promotion, the wizard also warned that "a delegation for this DNS server cannot be created because the authoritative parent zone cannot be found." This is expected and safe to ignore in a lab: the Domain Controller also acts as its own DNS server, and the warning just means there's no larger, internet-connected DNS zone above it to link to — which makes sense, since this lab is deliberately isolated on an internal network with no real internet access.
- The wizard automatically restarts the server once at the end of the promotion process — this is normal and required to finish applying the changes.

### Organizational Units, users, and groups

| Screenshot | What it shows |
|---|---|
| `10` | Active Directory Users and Computers (ADUC) console, showing the `lab.local` domain now exists |
| `11` | Creating a new user, "Jane Doe," including setting her user logon name |
| `12` | Organizational Units (OUs) created — Students, Staff, and IT — with Jane Doe placed inside the Students OU |
| `23` | The Helpdesk-Staff security group, located inside the IT OU |
| `25` | Creating the new group object, named "Helpdesk-Staff" |
| `26` | Jane Doe's account showing Helpdesk-Staff listed under her group memberships |
| `27` | Jane Doe now appearing in the same OU (IT) as the Helpdesk-Staff group |

**OU vs. Security Group — these serve different purposes.** An **OU (Organizational Unit)** is a folder-like container used purely to *organize* accounts logically (e.g., separating Students from Staff from IT) and to apply settings like Group Policy to a specific set of people. A **Security Group** is a different kind of object entirely — it's a way to *grant permissions or access* to a defined set of users, regardless of what OU they happen to live in. That's why Jane Doe can be physically located in the Students OU while still being a *member* of the Helpdesk-Staff security group — location (OU) and permissions (group membership) are independent of each other.

### Password reset and account lockout/unlock

| Screenshot | What it shows |
|---|---|
| `13` | Resetting a password on an account, with "User must change password at next logon" checked |
| `14` | The forced password-change prompt appearing on the client machine when John Brown (a new test user) logs in |
| `15` | The "account locked" prompt appearing after simulating too many failed login attempts |
| `16` | John Brown's account properties in ADUC, with "Unlock account" checked to resolve the lockout |
| `17` | Back on the client machine — John Brown successfully logged in after the unlock |

**Why "must change password at next logon" matters:** This is standard help desk practice — when you reset someone's password, you set a temporary one, but you check this box so the user is forced to set their own private password the moment they log in. That way, the help desk technician (or anyone else) never actually knows the user's real, ongoing password.

### Enabling/disabling accounts and Group Policy refresh

| Screenshot | What it shows |
|---|---|
| `18` | John Brown's account shown as disabled in ADUC |
| `19.0` | Running `gpupdate /force` from Command Prompt on the client machine |
| `19.1` | The client machine rejecting John Brown's login attempt after the account was disabled |

**What `gpupdate /force` does:** Domain-joined computers don't check in with the Domain Controller constantly for every single change — policy and account-status updates normally sync on a periodic schedule. Running `gpupdate /force` manually tells the client machine to immediately re-check with the Domain Controller and pull down the latest settings right now, rather than waiting for the next scheduled sync. This is a genuinely useful real-world command — if you disable a user's account and they're somehow still able to log in, forcing a policy refresh on their machine is one of the first things to try.

### Troubleshooting: client couldn't see the Domain Controller

| Screenshot | What it shows |
|---|---|
| `20` | Output of `ipconfig /all`, used to diagnose the issue |
| `21` | Manually configuring a static IP address, subnet mask, and preferred DNS server on the client to match the DC |

**The problem:** The client machine wasn't able to communicate properly with the Domain Controller. Running `ipconfig /all` showed the issue — the client wasn't on the same IP network as the server, and more importantly, it wasn't pointed at the Domain Controller for DNS. Since Active Directory relies on DNS to locate the Domain Controller and process logins, the client needs to be manually configured to use the DC's IP address as its **preferred DNS server**, rather than any default/automatic DNS setting.

**The fix:** I went into the client's network adapter settings (Control Panel → Network and Sharing Center → adapter properties → Internet Protocol Version 4) and manually set a static IP address on the same subnet as the server, along with the correct subnet mask, and pointed the "Preferred DNS server" field directly at the Domain Controller's IP address. Once that matched, the client could properly authenticate against `lab.local`.

---

## Phase 3 — Ticketing System (osTicket)
*In progress.*

## Phase 4 — Integration Project
*In progress — will combine osTicket tickets with the AD lab above to simulate real end-to-end help desk workflows (e.g., a "user" submits a lockout ticket, I resolve it in AD, then reply and close the ticket in osTicket).*

---

## Key Terms Used in This Project

Tier 1/2/3 support, Incident vs. Service Request vs. Problem, SLA, Escalation, DNS, DHCP, LAN/WAN, RDP, VPN, Domain vs. Domain Controller, OU (Organizational Unit), Security Group, GPO (Group Policy Object), EFI/UEFI, static IP configuration.
