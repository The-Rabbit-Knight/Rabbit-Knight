---
title: "Engineering an Open-Source SIEM: Deploying Wazuh and Building Custom Detections"
date: 2026-06-19
medium_url: https://medium.com/@rabbit_knight/engineering-an-open-source-siem-deploying-wazuh-and-building-custom-detections-98041cfe4a96
---

> **Original Published Article on Medium:** [Engineering an Open-Source SIEM: Deploying Wazuh and Building Custom Detections](https://medium.com/@rabbit_knight/engineering-an-open-source-siem-deploying-wazuh-and-building-custom-detections-98041cfe4a96)

### Engineering an Open-Source SIEM: Deploying Wazuh and Building Custom Detections

![Article Figure](https://cdn-images-1.medium.com/max/800/1*htj0HrDZCBDQsYpmijD4WQ.png)

A field report on standing up Wazuh on hardware I already own, instrumenting the first endpoint, and engineering custom detections against a live Domain Controller.

My last few evenings were eaten by evaluating a SIEM deployment. I’m exploring a small SOC setup on my bare-metal Proxmox server, taking it one tool at a time, and judging each on how much real detection I can extract before the complexity tax comes due. This is the first in a series of those evaluations. The subject is Wazuh: open-source, runs the manager, indexer, and dashboard as separate components, distributed across dedicated hosts at scale, or, as here, co-located on a single VM via the all-in-one installer.

The scope for this part is deliberately narrow and concrete: get the platform live on a Proxmox VM, instrument one endpoint with high-fidelity telemetry, and validate the whole pipeline by engineering a custom rule that catches a password spray against DC. Not a tutorial attack against localhost. A real one, from a Kali box, against a live Windows Server 2025 DC.

Let me walk you through it.

#### The decisions, and why I made them

A few choices got made early that shaped everything after, and they’re worth recording so a future rebuild doesn’t relitigate them:

- Debian 13 (Trixie), minimal, for the server VM was chosen over Ubuntu because Debian stable drifts far less over time, which is what you want in a box meant to run unattended for months. “Minimal” means no desktop: just an SSH server and standard utilities. Small footprint, fast install.

- The native install script over Docker. My first plan was Wazuh-docker - hand-generate certs, docker compose up. Wazuh’s own quickstart recommends a single official script (wazuh-install.sh -a) for exactly this single-host case, and my VM tier (4 vCPU / 8 GB RAM) matches their documented spec for 1–25 agents. One catch worth stating plainly: the script’s supported OS list is Amazon Linux, CentOS Stream, RHEL, and Ubuntu. Debian isn’t on it. I ran it on Debian deliberately, with eyes open: the install mechanics are close enough that the failure modes are predictable, the box is trivial to rebuild, and for an evaluation I’m willing to own being off the support matrix. The tradeoff: if I hit a Debian-specific bug later, that’s mine to debug. For an eval box that’s an acceptable price. For production, I’d land on a supported distro and not think twice. It installed cleanly.

- Networking handled by the router, not the VM. Instead of hardcoding a static IP inside the VM, I added a DHCP reservation on the router tied to the VM’s MAC address. Same fixed IP, nothing to misconfigure inside the guest, and a rebuild inherits the lease automatically.

That last point matters for the rest of the lab, so here’s the map everything plugs into.

#### Network reference

```
[Gateway / Router]
 (
192.168
.
1.1
)
 │-- DHCP & Routing
 │
 ├──> 
[Wazuh Manager VM]
 (
192.168
.
1.25
)  <-- Pinned via MAC
 │      ▲
 │      │ (Telemetry / Agent Dial-out)
 │      │
 ├──> 
[DC02 - Target]
 (
192.168
.
1.65
)      <-- 
[Wazuh Agent #1]
 │      │ Future Primary DNS
 │      ▲
 │      │ (Password Spray Attack)
 │      │
 ├──> 
[Kali Linux]
 (Dynamic IP)           <-- Attacker
 │
 ├──> 
[DC01 - Legacy]
 (
192.168
.
1.5
)       <-- Phasing out
 │
 └──> 
[Windows Client]
 (Dynamic IP)       <-- Future Agent
```

#### Standing up the box

The build itself is straightforward, so I’ll keep the routine parts short.

I pulled the Debian 13 net-install ISO (under 700 MB, it fetches packages live from a mirror during setup rather than shipping them on the disc) straight onto the Proxmox host over SSH, into the ISO storage directory, skipping the browser upload entirely.

The VM in Proxmox:

- General: name wazuh-manager.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*vf1cL6bTRE3MiLuiLfx67g.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*61J_ocNpcO1vxPDT92_UZQ.png)

- System: defaults, QEMU Guest Agent checkbox enabled.
- Disks:40 GB, VirtIO SCSI, Discard on (thin provisioning on SSD-backed storage).

![Article Figure](https://cdn-images-1.medium.com/max/800/1*H-7XiWgZVi58hNoJXfNWNQ.png)

- CPU: 4 cores, type host.
- Memory: 8192 MB, min = max, no ballooning. The indexer needs that RAM reliably present, not negotiated away under host pressure.

That 4 vCPU / 8 GB spec isn’t arbitrary; it’s Wazuh’s documented recommendation for 1–25 agents on a single host, and it’s the same tier that justified using the native installer.

During the Debian text install, the only choice worth slowing down for is Software selection: deselect everything except SSH server and standard system utilities. No desktop. This box is managed entirely over SSH, and its dashboard is a web app you reach from a browser on another machine — the VM never needs to render a pixel locally. When GRUB asks where to install, pick the VM’s own virtual disk (/dev/sda, shown as scsi-0QEMU_QEMU_HARDDISK-…). Each Proxmox VM gets a fully isolated virtual disk, so there’s zero risk of touching another guest, no matter that they share physical storage underneath.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*1n_BbFzrdTOJQxeHtuhe0g.png)

#### The minimal-install tax

This is where “minimal” hands you the first bill. After first boot, sudo was missing: the minimal profile doesn’t always install it or add your user to the sudo group. The symptom is honest enough: sudo: command not found, and a regular user whose PATH doesn’t even include /sbin, so half your admin commands come back as “not found” too.

Log in as root directly (you set that password during install) and fix it at the source:

```
apt update
apt install -y sudo
usermod -aG sudo yourusername
```

Log out and back in as your regular user, and both sudo and the full command PATH behave from then on. I am using sudo here as the primary example, but expect to repeat this exact process. A minimal install ships bare. When you inevitably hit command not found for tools like curl, wget, or gnupg later in the deployment, this is the workflow to fix it. Find the gap, install the missing package, and keep moving.

Then the basic hygiene and the one kernel parameter that actually matters:

```
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent
sudo systemctl 
enable
 qemu-guest-agent --now
sudo sysctl -w vm.max_map_count=262144
echo
 
"vm.max_map_count=262144"
 | sudo 
tee
 -a /etc/sysctl.conf
```

The guest agent is pure quality-of-life — it lets Proxmox cleanly shut the VM down instead of yanking the virtual power cord, and surfaces the real IP on the summary screen. Don’t be thrown by the warnings it throws on Debian: a complaint about ‘no [Install] section’ just means the package wired itself up through its own setup script instead of the standard enable path, and the recurring ‘Failed to query local AF_VSOCK CID’ lines come from an unrelated systemd component looking for a cloud-console feature Proxmox doesn’t expose. Both are cosmetic. The only thing that confirms it’s alive is ‘systemctl status qemu-guest-agent’ reading active (running).

![Article Figure](https://cdn-images-1.medium.com/max/800/1*9TAa9nk5pmaynzPqaD77DA.png)

vm.max_map_count, on the other hand, is not optional. The Wazuh indexer is OpenSearch, which is Lucene, which memory-maps a large number of file segments. Linux’s default cap on those mappings is too low for what OpenSearch wants, and when it hits the ceiling the indexer doesn’t warn; it crashes on startup. Raising it to 262144 is the documented fix and one of the most common first-install failures when skipped. The installer often sets it for you; setting it yourself first costs nothing and removes a variable.

#### Installing Wazuh

The off-matrix bet, running the installer on Debian, was the part I expected to fight. It was the part that gave me the least trouble. As root:

```
curl -s https://packages.wazuh.com/4.14/wazuh-install.sh > /tmp/wazuh-install.sh
chmod
 ugo+x /tmp/wazuh-install.sh
sudo /tmp/wazuh-install.sh -a
```

A note on versions: I pinned to 4.14 here. Whatever you run, the rule that bites later is that the manager and every agent must stay on the same version, so check the current release and use one number consistently rather than copying mine blindly.

The -a flag is “all-in-one”: manager, indexer, and dashboard in a single pass, with its own certificate generation and a freshly minted admin password. It takes several minutes and spits out dense log output along the way — password rotation, internal user setup — none of which is a failure. On success, it prints a summary block with the dashboard URL and that admin password. Copy the password immediately; it is not shown again. If you miss it:

```
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Browse to https://192.168.1.25 , accept the self-signed cert warning, and log in. The dashboard is a web application served on 443 by the install, the Debian box stays headless and does its work in the background while you look at everything from Chrome or Firefox on another machine. That’s how nearly every server on earth runs; almost none of them have a desktop, and plenty serve rich dashboards to whoever browses in.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*8hk30z_aeucOnldW_n9a6Q.png)

#### Pinning the repo so an unrelated upgrade can’t break me

The installer leaves a Wazuh package repo enabled on the system. Left alone, a routine ‘sudo apt upgrade’ months from now could pull a newer version of one component without me meaning to upgrade anything. Since manager, indexer, dashboard, and agents all have to stay on matching versions, that kind of silent partial upgrade is exactly the subtle-breakage trap I want to close now, while I’m thinking about it, not debug later when I’m not:

```
sed -i 
"s/^deb /#deb /"
 /etc/apt/sources.list.d/wazuh.list
apt update
```

Now Wazuh upgrades only happen when I deliberately re-enable the repo and choose to. The honest tradeoff: this also blocks security updates, not just version drift. Disabling the repo trades automatic patching for an upgrade control, which is only safe if you replace it with a deliberate cadence: re-enable, upgrade manager, indexer, dashboard, and agents together as a matched set, then re-pin. Skip that discipline, and pinned for stability silently becomes running unpatched. A cleaner long-term approach is apt-mark hold on the Wazuh packages, or version-pinning via apt preferences, so the repo stays available for visibility while the versions stay frozen until you move them on purpose.

#### A small detour: a file share for config edits

Here’s a real friction point I solved with a hack, and I’m leaving it in because pretending it didn’t happen would be dishonest. This box is CLI-only, and hand-typing XML rule blocks and config snippets straight into vi over SSH is error-prone; one fat-fingered tag and you’re hunting a typo instead of testing a rule. So I stood up a tiny Samba share on the Wazuh VM, write my config snippets into a text file on my workstation, and copy-paste them in cleanly. Windows speaks SMB natively, so any machine on the LAN maps it as a network drive with no extra software.

On the Wazuh VM:

```
sudo apt install -y samba
sudo 
mkdir
 -p /srv/shared
sudo 
chmod
 2775 /srv/shared
sudo 
chown
 yourusername:yourusername /srv/shared
sudo smbpasswd -a yourusername
```

That last command sets a Samba-specific password for your existing Linux user; it’s a separate credential store from your SSH login. Then append a share definition to ‘/etc/samba/smb.conf’:

```
[shared]
   path = /srv/shared
   browseable = yes
   read only = no
   guest ok = no
   valid users = yourusername
```

Restart it with ‘sudo systemctl restart smbd’, then map \\192.168.1.25\shared from any Windows box.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Kb2RwjHxdN7fXawzDGR5EA.png)

The Security Caveat: To be clear, this is a convenience hack for an isolated LAN, not a secure-by-default answer. The moment this environment gets exposed over a mesh VPN to remote endpoints, this share needs locking down or it doesn’t come along for the ride. The standard, safer pattern is using SCP/SFTP (via WinSCP or PuTTY’s pscp), which moves files over the SSH channel that is already open and authenticated with nothing extra listening on the box. Samba won here purely for the frictionless copy-paste-into-a-mapped-drive workflow while iterating on configs.

#### Enrolling the first agent: DC02

I picked DC02 as the first endpoint because it’s the controller staying long-term. Installing a Wazuh agent has no effect on AD; it runs as its own independent Windows service and just reads logs.

From the dashboard: Agents → Deploy new agent → Windows. In this version the fields don’t auto-fill, so type them in:

- Server address: 192.168.1.25 (the manager, not DC02’s own IP ) - Agent name: DC02 - Group: default

The wizard generates an install command at the bottom with the matching MSI version baked in. Copy that exact command rather than one from a tutorial, because the agent and manager versions have to line up. Run it on DC02 as Administrator, then start the service: NET START WazuhSvc

![Article Figure](https://cdn-images-1.medium.com/max/800/1*pXIig6GDRcZsiEumEqJTBw.png)

DC02 showed up as active in the Agents page within a minute.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*AN_hDasAcd1jkNEB16EdRg.png)

#### What you get for free, and what you don’t

Out of the box, with zero extra config, the agent collects three Windows event channels: Application, Security, and System. Through its built-in ‘eventchannel’ collector, and runs periodic Security Configuration Assessment (SCA) scans. Wazuh even auto-filters a handful of especially noisy, low-value Security event IDs for you.

What it does not give you is process-level visibility. Individual command and process execution isn’t something Windows logs by default, and that’s the single biggest blind spot for an endpoint you actually want to hunt on. Two things to understand clearly here, because they’re easy to conflate:

- The Security channel (and the failed-logon events that power the password-spray detection coming up) is collected by default. No Sysmon required. - Process execution: what command ran, with what arguments, spawned by what parent, needs Sysmon.

So the spray detection and the process visibility are two separate capabilities riding two separate pipelines. I built both, but I want them in their own lanes.

#### Adding Sysmon for process visibility

Wazuh is powerful, but a SIEM is only as good as the telemetry you feed it. To get high-fidelity process visibility on the endpoint, I needed Sysinternals Sysmon.

A default installation captures basic process creation, but it floods the pipeline with harmless background chatter while completely missing advanced injection techniques. In a production environment, you have to drive Sysmon with a vetted, high-signal configuration.

For this deployment, I anchored the endpoint using the industry-standard SwiftOnSecurity Sysmon config. The benefit of this configuration is immediate: it aggressively cuts the noise by filtering out known-good Windows executables, while specifically targeting execution anomalies, suspect network connections, and registry modifications.

To automate the drop, I extracted the Sysmon download to C:\Sysmon (which gives you sysmon64.exe and creates the directory), then pulled the configuration and installed it from an administrative PowerShell prompt:

```
# Ensure the working directory exists (the Sysmon zip should already be extracted here)
New-Item -ItemType Directory -Force -Path C:\Sysmon | Out-Null
# Download the SwiftOnSecurity configuration
Invoke-WebRequest -Uri 
"https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml"
 -OutFile 
"C:\Sysmon\sysmonconfig.xml"
# Install Sysmon and enforce the configuration
Set-Location C:\Sysmon
.\sysmon64.exe -accepteula -i sysmonconfig.xml
```

A note on supply-chain trust. Notice that I pulled the XML directly from the master branch of a public GitHub repo. For a proving ground, that’s fast and effective. For production, it’s a real supply-chain risk. If that upstream repo is ever compromised and an attacker slips in a bypass rule or a maintainer pushes a broken regex, every endpoint pulling from it silently ingests the bad config. In a production deployment you’d never pull live from an external URI: you’d download the file, review the exclusion rules by hand, host the vetted copy internally, and push it through GPO or your endpoint management tooling. Worth saying out loud, because the convenience and the risk are the same line of code.

Once Sysmon was logging high-fidelity events locally, I needed to route that telemetry to the SIEM. I opened the Wazuh agent configuration file (C:\Program Files (x86)\ossec-agent\ossec.conf) and appended the specific event channel Sysmon writes to:

```
<
localfile
>
  
<
location
>
Microsoft-Windows-Sysmon/Operational
</
location
>
  
<
log_format
>
eventchannel
</
log_format
>
</
localfile
>
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*mf_SAoF_VWiOWK6M1GFlVQ.png)

Then I bounced the Wazuh agent service to lock in the routing and start the forwarder:

```
Restart-Service -Name WazuhSvc
```

With that single service restart, the endpoint went from a blind spot to a highly tuned sensor, and the telemetry pipeline was fully operational.

#### Proving the process pipeline works

The cheapest possible end-to-end test: run ‘whoami’ in PowerShell on DC02, then go looking for it in the Threat Intelligence -> Threat Hunting -> Events ( or Explore -> Discover):

The Exact Path (Strict Match) If you are searching for the exact execution string, you must double the backslashes to escape them properly in the query:

```
data.win.eventdata.commandLine:\"C:\\WINDOWS\\system32\\whoami.exe\"
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*L8-_-gFykCAqjGGTJesEAg.png)

The Wildcard (Flexible Match) Attackers rarely use full, predictable paths. Using DQL wildcards (the OpenSearch Dashboards query language, not to be confused with WQL, which drives Wazuh’s API-backed module search bars) is the standard way to hunt for the binary execution, regardless of where it was called from or what arguments were attached:

![Article Figure](https://cdn-images-1.medium.com/max/800/1*-Be5KsOG5zW865N35Y1Xvg.png)

So, it was there. That one event confirms the whole chain: Sysmon captured the process creation, the agent shipped it to the manager, the manager decoded it, and the indexer made it searchable. Trivial command, but it’s the heartbeat that tells you the plumbing is sound before you build anything real on top of it.

Worth knowing for capacity planning later: a domain controller’s Security channel is the busiest of the three by a wide margin, because every authentication, group-membership change, and Kerberos ticket request across the whole domain can land there. A DC is not a workstation, and it logs like it.

#### The real test: catching a password spray

A whoami showing up in a search is a pipeline test. It is not detection. Detection is when something attacks and the platform tells me on its own, without me going to look. So I built one.

A password spray is the cleanest attack to detect against, because its signature is so distinct: one source IP, many different usernames, one password attempt each. That’s the inverse of brute force (one username, many passwords), and it’s specifically designed to stay under per-account lockout thresholds. The fingerprint isn’t “an account failed repeatedly” — it’s “multiple accounts failed, from one location.” While advanced threat actors will run these “low and slow” over weeks to evade detection, for the purpose of validating this SIEM pipeline, I am executing a fast, loud spray attack. To generate it, I used CrackMapExec from the Kali box against DC02, with a list of valid usernames and a single wrong password:

```
crackmapexec smb 
192.168
.1
.65
 -u valid-users.txt -p 
'Pass123'
 --
continue
-
on
-success
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*KqcdPAy3wWU8wQJapU_H3A.png)

Step 1. Find the base rule

Before writing anything, I clicked into one of the resulting failed-logon alerts in Discover and read its rule.id field directly. The base SID Wazuh assigns to a single Windows Logon Failure (Event ID 4625) varies by ruleset version, so hardcoding a number from someone else’s blog is how you write a rule that silently never fires. In my 4.14 ruleset it was 60122. Confirm yours; don’t inherit mine.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*GVce7ITb5wAL0B359VFH1g.png)

Step 2. The correlation rule

On the manager, in ‘/var/ossec/etc/rules/local_rules.xml’:

```
<
group
 
name
=
"local,authentication_failures,"
>
  
<
rule
 
id
=
"100050"
 
level
=
"12"
 
frequency
=
"5"
 
timeframe
=
"60"
>
    
<
if_matched_sid
>
60122
</
if_matched_sid
>
    
<
same_field
>
win.eventdata.ipAddress
</
same_field
>
    
<
description
>
Possible password spray: 5+ failed logons from the same source IP within 60s
</
description
>
    
<
mitre
>
      
<
id
>
T1110.003
</
id
>
    
</
mitre
>
    
<
group
>
authentication_failures,
</
group
>
  
</
rule
>
</
group
>
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*GAYfcz496_rIcqXS1A6VMw.png)

Reading it the way I’d want a teammate to read it:

- if_matched_sid 60122 — only start counting events that are already single 4625 failures. - frequency=”5" timeframe=”60" — fire when 5 of them land inside 60 seconds. Tune to taste; an aggressive spray trips this instantly, a slow-and-low one needs a wider window. - same_field win.eventdata.ipAddress— and here’s the actual signature. Tie all 5 back to one source IP. Without this, five unrelated failed logins from five different machines would trip a false alarm. With it, the rule only cares about the one-source-many-accounts pattern that defines a spray. I confirmed the ipAddress field was genuinely populated on my 4625 events with the Kali box’s address before trusting it. - level=”12" — high enough to mean “look at this now.” - The MITRE mapping (`T1110.003`, Password Spraying) — costs one line and makes every resulting alert speak ATT&CK, which is what turns a hobby rule into something a SOC lead recognizes.

Restart the manager to load it:

```
sudo systemctl restart wazuh-manager
```

Step 3. The alert fired.

Rule 100050, level 12, landed in the dashboard. The pipeline I’d built end-to-end now did its actual job: an attack happened and the platform told me, with the source IP and a MITRE technique attached, without me searching for anything.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*f_TWjL9Jg646v8P5A-krxQ.png)

But the alert showed exactly one username, even though I’d used seven. The reflexive read is that something’s broken. It isn’t, and understanding why is the most operationally useful detail in this entire evaluation.

A Wazuh composite rule counts its matching events silently in the background. It tallies each individual 60122 failure as it streams in, and the instant the 5th one arrives inside the 60-second window, it emits one alert. That single alert inherits its fields, including targetUserName from the event that tripped the threshold, the 5th 4625. So of course, it shows one name. The rule’s job is to be a tripwire: ”a spray attack is happening right now, from this IP.” It is deliberately not trying to be the full account list.

The other six usernames were never lost. They’re sitting in the underlying rule.id:60122 alerts, one per attempted account.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*1NzRLAHRyyUBs9EoHJgJIg.png)

Which means the right mental model is two layers:

- The tripwire— rule.id:100050. One high-severity alert that says an attack is happening and where from. This is what you want to be paged on. - The forensic detail — the rule.id:60122 events underneath. This is where you pivot to answer which accounts were targeted.

One boundary worth naming, because a reviewer will ask: this rule keys on a single source IP via same_field. That’s the right call for the classic spray: one box, many accounts, but it’s a known evasion gap. An attacker who sprays one attempt per account from rotating source IPs (a botnet, a proxy pool, password-spray-as-a-service) never trips the five-from-one-IP threshold and walks straight past this detection. Closing that gap means correlating on the targeted side instead: many distinct targetUserName values failing across the domain in a short window, regardless of source, which is a separate rule with a very different false-positive profile (a misconfigured service account can light it up). I’m scoping v1 to the single-source case on purpose and treating the distributed variant as its own detection-engineering problem for a later part.

Step 4. Making the dashboard

A custom rule above level 0 lands in the same wazuh-alerts-* index as every built-in detection, so it’s already visible in Discover (rule.id:100050) and in Security events, with no extra setup.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*27X--lVcsb3kaFiOzrafUA.png)

So, I built a small visualization (Visualization -> Create visualization)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*YMZiEV_okWYKanZGMgkXOQ.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*dQk92M5tDI36jQGulw-iVA.png)

Pointed at the wazuh-alerts-* index pattern, filtered to the underlying data.win.eventdata failures, a data table does the job: split rows by a Terms aggregation on data.win.eventdata.ipAddress (the attacking source) as the primary bucket, then sub-bucket on data.win.eventdata.targetUserName (the accounts they tried).

That single table that appears answers both questions at once: who’s attacking and who they’re after, which the tripwire alert intentionally doesn’t. Saved as “Password spray alerts” and dropped onto a dashboard (Explore -> Dashboards -> Create Dashboard -> Add an existing -> Panel’s name ( “Password spray alerts”) -> Save), it’s a standing panel I can pull up the next time 100050 fires.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*dg12Q4IhlrBeLrkRDRVEiQ.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*x5YZkky4HbpBxwB_3RkWWQ.png)

If you don’t need the full visual, the lazy version still works: filter Discover to rule.id:100050, hit Save, name it. One-click saved search, thirty seconds, good enough most of the time.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*O-XIqoYeoezO89ZhOU5P2w.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*mhuA-7fptD9R8q4jL6LtNA.png)

#### What’s next, and one capability I’m leaving switched off for now

Two things are worth flagging before I wrap, both deliberately deferred.

Storage. Wazuh’s sizing guidance suggests 50 GB for 90 days of retained alert data at the 1–25 agent tier. My VM has 40 GB total, OS included, so I’m slightly under spec, which is fine for one DC reporting, but something I’m watching with ‘df -h’ on ‘/var’, where the indexer lives. If it tightens, growing a Proxmox disk is live and painless: bump the virtual disk, then ‘growpart’ and ‘resize2fs’ inside the guest. No downtime, no reinstall.

Full event archiving. By default, Wazuh only indexes alerts: events that matched a rule and crossed a threshold. Everything else is processed and discarded. You can flip that with <logall> / <logall_json> on the manager and by enabling archives in the Filebeat module, which turns the platform from a SIEM into something closer to a data lake — every event retained and queryable, rule-match or not. It’s genuinely useful for hunting. It also grows storage faster than anything else here, and given I’m already under the disk recommendation, I’m leaving it off until I’ve sorted retention properly (likely an Index State Management policy capping the archive index well short of 90 days). Mentioning it now so the option is on the record; building it out belongs in its own part.

That’s Part 1: a bare VM turned into a working detection platform that caught a real attack against a real DC, with the rough edges left visible because that’s where the operational truth of a tool actually shows. The platform’s live, one endpoint’s instrumented, and the first custom rule earns its keep for now, tuning later.

Next, I start fleeting out agents and stacking more detections on top of this foundation, and bringing in the next tool in the SOC. Stay tuned.
