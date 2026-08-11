# ITS Help Desk Portfolio (Won Seo)

## About This Project

I built a self-hosted lab environment from scratch to learn and practice the core skills a Tier 1 help desk technician uses every day: account creation, password resets, account unlocks, and group management inside Active Directory. The setup is two virtual machines networked together, with one configured as a Windows Server Domain Controller and the other as a domain-joined client.

This repo documents that process step by step, including the problems I ran into along the way and how I worked through them. Troubleshooting through real issues is most of what's actually being demonstrated here, not just following a tutorial start to finish.

---

## Tools Used

- **VirtualBox**: hypervisor used to run the virtual machines
- **Windows Server 2022**: configured as the Domain Controller
- **Windows 11 Enterprise**: configured as a domain-joined client machine
- **Active Directory Domain Services (AD DS)**: Microsoft's directory service for managing users, computers, and permissions

---

## Repo Structure

```
/its-helpdesk-portfolio
  /01-lab-setup         → VirtualBox + VM installation screenshots
  /02-active-directory  → AD DS promotion, OU/user/group tasks, troubleshooting
  /03-osticket          → ticketing system setup and configuration
  /04-integration       → (in progress) full end-to-end support scenarios
  README.md             → this file
```

---

## Phase 1: Lab Environment Setup

I installed VirtualBox and built two virtual machines: a Windows Server 2022 VM (`WinServer-DC`, meant to become the Domain Controller) and a Windows 11 Enterprise VM (`Win11-Client`). Then I networked them together on an isolated internal network so they could talk to each other.

| Screenshot | What it shows |
|---|---|
| `1` | Both VMs (`WinServer-DC` and `Win11-Client`) listed in the VirtualBox Manager |
| `2.0` / `2.1` | Server VM hardware allocation: 4096 MB RAM, 2 CPUs |
| `3.0` / `3.1` | Client VM hardware allocation: 4096 MB RAM, 2 CPUs |
| `4.0` / `4.1` | Network adapter settings for both VMs, set to **Internal Network** (`LabNetwork`) instead of the default NAT, so the two VMs can see each other |
| `5` (DC) | WinServer-DC desktop after first successful login |
| `5` (Client) | Win11-Client home screen after setup, logged in with a local account |

**Why Internal Network instead of the default NAT:** By default, VirtualBox gives each VM internet access but keeps it isolated from other VMs. That's fine for browsing, but it means the two machines can't see each other at all. Switching both to **Internal Network** with the same network name (`LabNetwork`) puts them on their own private wire so they can communicate directly. The trade-off is neither VM has real internet access anymore, which doesn't matter here since AD doesn't need it.

**A problem I hit and fixed:** The first time I booted the Windows 11 VM, it failed with "No bootable option or device was found." Windows 11 needs EFI along with TPM 2.0 and Secure Boot, and the VM briefly shows a "press any key to boot from CD or DVD" prompt during boot. Miss that window by even a couple seconds and it gives up and won't boot from the installer at all. Fixed it by re-mounting the ISO and catching the prompt immediately.

---

## Phase 2: Active Directory Deep Dive

I promoted `WinServer-DC` from a plain Windows Server install into an actual Domain Controller by installing the AD DS role and running the promotion wizard, then worked through the core Tier 1 tasks: creating organizational units, creating and managing user accounts, resetting passwords, unlocking a locked account, disabling an account, and managing security group membership.

### Promoting the server to a Domain Controller

| Screenshot | What it shows |
|---|---|
| `7` | Server Manager dashboard after the AD DS, DNS, and related roles were installed |
| `8` | The "Before You Begin" screen of the Add Roles and Features Wizard |

Installing the role and becoming a Domain Controller are two separate steps. Adding "Active Directory Domain Services" through Server Manager only installs the software. It doesn't configure anything. The server only becomes a functioning Domain Controller after running the AD DS Configuration Wizard and promoting it, which is where the domain actually gets created.

**The `lab.local` domain name:** During promotion I chose "Add a new forest," since this was a brand-new setup, and named the root domain `lab.local`. A few notes on this:
- A forest is the top-level container holding one or more domains. For a single-domain lab like this, "Add a new forest" just creates a forest with one domain in it.
- I used `.local` instead of something like `.com` because that's standard for private, internal-only domains. `.local` isn't publicly routable, so the name can't ever collide with anything on the real internet.
- The wizard also warned that "a delegation for this DNS server cannot be created because the authoritative parent zone cannot be found." That's expected in a lab: the Domain Controller is also acting as its own DNS server, and the warning just means there's no larger internet-connected DNS zone above it. That makes sense since the network is deliberately isolated.
- The wizard restarts the server once at the end to finish applying the changes. Normal, not a bug.

### Organizational Units, users, and groups

| Screenshot | What it shows |
|---|---|
| `10` | ADUC console, showing the `lab.local` domain now exists |
| `11` | Creating a new user, "Jane Doe," including her logon name |
| `12` | OUs created (Students, Staff, and IT), with Jane Doe placed inside Students |
| `23` | The Helpdesk-Staff security group, located inside the IT OU |
| `25` | Creating the Helpdesk-Staff group object |
| `26` | Jane Doe's account showing Helpdesk-Staff under her group memberships |
| `27` | Jane Doe appearing in the same OU (IT) as the Helpdesk-Staff group |

**OU vs. security group:** these do different jobs. An OU is a folder-like container that organizes accounts (separating Students from Staff from IT) and lets you apply things like Group Policy to a specific set of people. A security group grants permissions or access to a defined set of users, regardless of which OU they're sitting in. That's why Jane Doe can live in the Students OU while still being a member of the Helpdesk-Staff group. Location and permissions aren't tied together.

### Password reset and account lockout/unlock

| Screenshot | What it shows |
|---|---|
| `13` | Resetting a password, with "User must change password at next logon" checked |
| `14` | The forced password-change prompt on the client machine when John Brown (a test user) logs in |
| `15` | The "account locked" prompt after simulating too many failed login attempts |
| `16` | John Brown's account properties, with "Unlock account" checked to resolve the lockout |
| `17` | Back on the client: John Brown logged in successfully after the unlock |

**Why "must change password at next logon" matters:** when you reset someone's password, you set a temporary one, but checking this box forces the user to set their own private password the moment they log in. That way the technician never actually knows the user's real, ongoing password.

### Enabling/disabling accounts and Group Policy refresh

| Screenshot | What it shows |
|---|---|
| `18` | John Brown's account shown as disabled in ADUC |
| `19.0` | Running `gpupdate /force` from Command Prompt on the client |
| `19.1` | The client rejecting John Brown's login after the account was disabled |

**What `gpupdate /force` does:** domain-joined computers don't check in with the Domain Controller for every single change. Updates normally sync on a periodic schedule. `gpupdate /force` tells the client to re-check immediately instead of waiting. Genuinely useful in practice: if a disabled account can still log in somewhere, forcing a policy refresh is one of the first things worth trying.

### Troubleshooting: client couldn't see the Domain Controller

| Screenshot | What it shows |
|---|---|
| `20` | Output of `ipconfig /all`, used to diagnose the issue |
| `21` | Manually configuring a static IP, subnet mask, and preferred DNS server on the client to match the DC |

**The problem:** the client couldn't communicate properly with the Domain Controller. `ipconfig /all` showed why: the client wasn't on the same IP network as the server, and wasn't pointed at the DC for DNS. AD relies on DNS to locate the Domain Controller and process logins, so the client needs to use the DC's IP as its preferred DNS server rather than whatever default it picked up.

**The fix:** went into the client's network adapter settings (Control Panel, Network and Sharing Center, adapter properties, IPv4) and manually set a static IP on the same subnet as the server, matched the subnet mask, and pointed "Preferred DNS server" at the Domain Controller's IP. Once that matched, the client authenticated against `lab.local` without issue.

---

## Phase 3: Ticketing System (osTicket)

I installed osTicket on top of XAMPP (Apache, MySQL, and PHP running locally) and configured it the way a real help desk would: ticket categories, an SLA rule, and canned responses. Then I ran a full ticket through the system end to end, from submission as a "user" to resolution as staff, to prove the setup actually works and not just that the install completed.

### Install

| Screenshot | What it shows |
|---|---|
| `28` | XAMPP Control Panel with Apache and MySQL both running |
| `29` | osTicket installer prerequisites screen, confirming PHP 8.2.12 and the MySQLi extension both pass |
| `30.0` | "Configuration file missing" error partway through install |
| `30.1` | The fix: `ost-config.php` created next to `ost-sampleconfig.php` in the include folder |
| `31` | osTicket Basic Installation form |
| `32.0` | Creating the `osticket` database in phpMyAdmin |
| `32.1` | Running an `ALTER USER` / `FLUSH PRIVILEGES` query in the SQL tab to set a password on the MySQL root account |
| `33` | Install success screen |

**The config file error:** osTicket needs write access to `include/ost-config.php`, but that file doesn't exist until you create it. The installer ships a template, `ost-sampleconfig.php`, that just needs to be duplicated and renamed. Once both files existed side by side, the installer moved on.

**The MySQL password chain:** the install form kept rejecting the database connection with "Access denied for user 'root'@'localhost' (using password: YES)." Checking Apache's error log confirmed the real cause, the form was sending a password that the MySQL root account didn't actually have set yet. I fixed it directly with an `ALTER USER` statement in phpMyAdmin's SQL tab, which set the password without needing to hunt through several menus for the right UI option.

| Screenshot | What it shows |
|---|---|
| `34` | phpMyAdmin's own login now failing with "Access denied (using password: NO)" |
| `35` | The follow-up fix, phpMyAdmin's `config.inc.php` updated with the new password |
| `36` | phpMyAdmin loading successfully again, now showing the fully populated `osticket` database with all of osTicket's tables |

Setting the MySQL root password fixed the install, but it broke something else I wasn't expecting: phpMyAdmin itself, which had been logging in with a blank password by default. Its config file needed the same new password added before it could connect again. One fix created a second, smaller problem downstream, which is a pretty realistic shape for IT troubleshooting to take.

### Configuration

| Screenshot | What it shows |
|---|---|
| `37` | Staff Control Panel login screen |
| `38` | Help Topics list with all 5 categories created: Password Reset, Account Lockout, Software Issue, Hardware Issue, Network Issue |
| `39` | A help topic's configuration screen, showing topic-level settings like parent topic and priority |
| `40` | A new SLA plan added, with a 1-hour grace period |
| `41` | That SLA plan assigned directly to the Password Reset topic |
| `42` | Canned Responses list |
| `43` | One canned response opened, showing the actual reply text |

**Priority vs. SLA:** these sound similar but do different jobs. Priority (Low/Normal/High/Emergency) is about how urgent a ticket is relative to others in the queue. SLA is a concrete time commitment, how long a ticket of a given type is allowed to sit before it needs a response. Password resets are common and quick to resolve, so I gave that topic its own 1-hour SLA instead of leaving it on the system default of 18 hours.

**Canned responses** exist so agents aren't retyping the same answer to the same common issue over and over. I wrote two: one for completed password resets, one for account unlocks.

### Proving it works end to end

| Screenshot | What it shows |
|---|---|
| `44` | Ticket submission form filled out as a test user (Jane Doe) reporting a lockout |
| `45` | Confirmation screen after the ticket was submitted |
| `46` | The new ticket showing up in the Staff Control Panel queue |
| `47` | The ticket opened, with a reply typed using the "Account Unlocked" canned response |
| `48` | The ticket after being marked resolved |

This is the piece that actually matters: not just that the categories and SLA and canned responses exist, but that a ticket can move through the whole lifecycle, submitted, picked up, answered, and closed, the same way a real one would.

## Phase 4: Integration Project
*In progress. Will combine osTicket tickets with the AD lab above to simulate end-to-end support workflows: a "user" submits a lockout ticket, I resolve it in AD, then reply and close the ticket in osTicket.*

---

## Key Terms Used in This Project

Tier 1/2/3 support, Incident vs. Service Request vs. Problem, SLA, Escalation, DNS, DHCP, LAN/WAN, RDP, VPN, Domain vs. Domain Controller, OU (Organizational Unit), Security Group, GPO (Group Policy Object), EFI/UEFI, static IP configuration.
