---
title: "Surviving a Split-Brain Domain Controller Promotion"
date: 2026-06-17
medium_url: https://medium.com/@rabbit_knight/surviving-a-split-brain-domain-controller-promotion-79e235259561
---

> **Original Published Article on Medium:** [Surviving a Split-Brain Domain Controller Promotion](https://medium.com/@rabbit_knight/surviving-a-split-brain-domain-controller-promotion-79e235259561)

### Surviving a Split-Brain Domain Controller Promotion

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Bf8sSx08-RbaW6XTIQ8Dig.png)

In security operations, building a realistic testing ground means rapidly changing configurations to validate different defenses. Recently, during a purple teaming exercise focused on creating custom detection rules for Active Directory, I pushed a Windows Server 2025 Domain Controller promotion to its absolute limit. The result was a catastrophic split-brain Netlogon failure, a fractured trust relationship, and a completely severed secure channel.

Here is a technical breakdown of what happens when Active Directory trips over its own ghost records, why the “nuke and pave” methodology is the standard enterprise response, and how to automate a flawless, secure Domain Controller deployment from scratch using PowerShell

#### The Trap: Ghost Records and The Nuke & Pave

Active Directory has a long memory. The entire cascade failure started with something nearly invisible: a lingering DNS A record from a virtual machine I had deleted a while ago.

When I fired off the command to rename the new server and promote it, AD checked the network, saw that old ghost record squatting on the hostname, and immediately slammed the brakes. Access Denied. But the process didn’t just fail gracefully; it fractured.

The server got caught in the middle of a transition. The local Windows OS had already committed the new name to its registry. However, the AD database and the underlying Kerberos Service Principal Names (SPNs) were still tethered to the old identity.

This created a textbook “split-brain” scenario. When the Netlogon service attempted to start, it checked its cryptographic keys against the domain and realized the server was suffering from a massive identity crisis. Operating exactly as designed, Netlogon refused to run. Every attempt to test the secure channel just returned a fatal wall of red text: The RPC server is unavailable. The server was effectively deaf and blind to its own domain.

#### The Solution: Nuke, Pave, and Automate

At a certain point in troubleshooting, you have to recognize when a virtual machine is a lost cause. Spending hours trying to manually reconstruct SPNs and force Kerberos password resets on a fundamentally corrupted OS isn’t just frustrating; in a production environment, it’s a massive waste of time. (Snapshots are a very useful tool)

The server was a dead end. So, I burned it down. But you can’t just delete a broken Domain Controller from your hypervisor and pretend it never happened. My primary controller (DC01) still held the master database, and its internal records were screaming into the void, trying to sync with a partner that no longer existed.

Before spinning up a replacement, I had to manually scrub the dead server’s ghost records out of DC01’s brain. This meant diving into Active Directory Users and Computers to forcefully delete the computer object, routing through Sites and Services to kill the broken replication links, and finally sweeping out the old Host (A) and Name Server (NS) records in DNS.

Once DC01 was pristine, the board was clear for a fresh, completely controlled deployment.

#### The Clean Build

To guarantee the new VM wouldn’t fall into the same trap, the deployment order had to be flawless.

1. The Network Anchor Before the new server ever attempted to talk to the domain, I hardcoded its DNS. The Preferred DNS pointed directly to the IP of DC01, and the Alternate DNS pointed to 127.0.0.1. If a server doesn’t know exactly who holds the authoritative keys to the network before it joins, the promotion is doomed to fail.

2. The Isolated Rename Next, I renamed the machine to DC02 while it was still safely isolated in a standard Workgroup. This ensured Active Directory never even caught a glimpse of the ugly, auto-generated WIN- computer name.

3. The PowerShell Promotion Clicking “Next” through Server Manager is fine, but configuration-driven deployment is how you actually scale. Once the server was joined to the domain and I was authenticated as a Domain Admin, I bypassed the GUI entirely. I used PowerShell to install the AD DS roles, securely inject the Safe Mode (DSRM) password, and force the database to build.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*qPr9ri2RkskZ78N060gwRA.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*vAJEOgrGPtPWeUSk0kkJBg.png)

```
# 1. Install the Active Directory Role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
# 2. Define the Target Domain
$domainName
 = Read-Host 
"Enter your Target Domain Name (e.g., example.com)"
# 3. Prompt for the Safe Mode (DSRM) Password securely
$dsrmPass
 = Read-Host 
"Enter a strong DSRM (Safe Mode) Password"
 -AsSecureString
# 4. Setup the Domain Admin Credentials for the promotion
$user
 = Read-Host 
"Enter your Domain Admin Username (e.g., DOMAIN\Administrator)"
$pass
 = Read-Host 
"Enter your Domain Admin Password"
 -AsSecureString
$cred
 = New-Object System.Management.Automation.PSCredential (
$user
, 
$pass
)
# 5. Fire the Promotion Command
Install-ADDSDomainController -DomainName 
$domainName
 -NoGlobalCatalog:
$false
 -CreateDnsDelegation:
$false
 -Credential 
$cred
 -SafeModeAdministratorPassword 
$dsrmPass
 -CriticalReplicationOnly:
$false
 -DatabasePath 
"C:\Windows\NTDS"
 -LogPath 
"C:\Windows\NTDS"
 -SysvolPath 
"C:\Windows\SYSVOL"
 -Force:
$true
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*xtvpoashw4-bkhsSOdaIJQ.png)

Note: PowerShell will likely output two yellow WARNING messages regarding DNS delegation. If you are building an isolated lab or an internal domain without an authoritative external parent zone, this is completely expected behavior. Active Directory is simply noting that it cannot register its DNS with the outside world. You can safely ignore these warnings. The only metric that matters is at the very bottom of the terminal: as long as the final output reads Operation completed successfully and Status: Success, your database was built securely and the server will auto-reboot to finalize its new identity.

Successfully promoting the server is only half the battle. At this point, DC02 is just a partner. The DC01 still technically holds the FSMO (Flexible Single Master Operations) roles: the master keys to the domain. If I were to decommission DC01 right now, the entire environment would eventually collapse.

First, I verified that both servers were successfully communicating as partners:

```
netdom query dc
```

Next, I checked who currently held the master roles:

```
netdom query fsmo
```

To execute the final handover, I dropped back into PowerShell. You can transfer these through the GUI using Active Directory Users and Computers, but a single command is much cleaner:

```
# Transfer all 5 FSMO roles to the new Domain Controller
Move-ADDirectoryServerOperationMasterRole -Identity 
"NewDC"
 -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster -Confirm:
$false
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*I7VyCzZfCXoMCW3fWdRy6w.png)

#### Trust, But Verify.

A script finishing without a wall of red text is nice, but it isn’t proof. I needed hard confirmation that the AD database actually replicated and that the new DC successfully generated its own SPNs. I pulled up the built-in diagnostic tools:

repadmin /showrepl: Confirmed that the Schema, Configuration, and Domain partitions pulled perfectly from the primary controller without a single RPC error.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*eM91cBAxUlxA5VS1-UbegA.png)

setspn -X: Scanned the entire domain for duplicate SPNs. It returned exactly zero, proving the Kerberos secure channel was rock solid.

setspn -Q: Queried my custom service accounts (like my SQL and Web SPNs). They were all sitting happily in the replicated database.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*_46gBdzCuQrpyvFmWEB_vg.png)

At the end of the day, building infrastructure isn’t just about following the happy path until it works. It’s about knowing when the wreckage isn’t worth saving. Sometimes, clearing the board and rebuilding from the ground up is the only honest way to finish the job.
