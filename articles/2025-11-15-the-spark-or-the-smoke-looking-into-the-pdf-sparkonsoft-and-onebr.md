---
title: "The Spark or the Smoke: Looking into the PDF SparkOnSoft and OneBrowser Malware"
date: 2025-11-15
medium_url: https://medium.com/@rabbit_knight/the-spark-or-the-smoke-looking-into-the-pdf-sparkonsoft-and-onebrowser-malware-3a7a19dafdba
---

> **Original Published Article on Medium:** [The Spark or the Smoke: Looking into the PDF SparkOnSoft and OneBrowser Malware](https://medium.com/@rabbit_knight/the-spark-or-the-smoke-looking-into-the-pdf-sparkonsoft-and-onebrowser-malware-3a7a19dafdba)

### The Spark or the Smoke: Looking into the PDF SparkOnSoft and OneBrowser Malware

![Article Figure](https://cdn-images-1.medium.com/max/800/1*_jkO-c7baiw33O0g2Rogrg.jpeg)

For anyone who has followed the waves of fake PDF utilities, repackaged browser add-ons, and the recycled branding of gray-market software, “Spark” has become something of a calling card.

So when new reports started appearing with installers labeled PDFSpark, PDFSparkOnSoft, PDFSparkWare, PDFSparkApp, and very “helpful” OneBrowser, there was no suspense about whether these were benign. The real story was not if they were malicious, but how far the operation went, and how deeply it tried to root itself into the system.

What my investigation uncovered was not a single installer but a sophisticated digital ecosystem: a rotating family of repackaged binaries, inconsistent publishers, and a persistence mechanism designed to hide inside Windows.

### The Installers That Pretended Not to Exist

At the surface, the installers looked ordinary. A double-click launched a familiar Inno Setup wizard, the same installer framework used by everything from open-source tools to commercial utilities. Inno Setup’s legitimacy is exactly what threat actors relied on: its behavior blends seamlessly into Windows ecosystems.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*heHjPEYSP1Y7i5J2CnRvvA.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*59nJUN8RcNabUuEBOJrJ5Q.png)

The real activity began after execution, when the installer quietly unpacked itself into disposable folders like:

```
C:\Users\<User>\AppData\Local\Temp\is-####.tmp\PDFSparkOnSoft_####.tmp
```

These locations, which were meant for short-lived extraction, instead became the onset of the malware’s entire payload chain. Inside these folders were the payloads, often named setup.exe, PDFSparkOnSoft_*.tmp, or similarly generic names.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*WS6gOswQYX678hPaIqRj8g.png)

Those variants were signed under different publishers:

• PDFSpark Setup — Publisher: CROWD SYNC LLC • PDF Spark, PDF Spark Setup, PDF_Spark Setup, PDFSpark Setup — Publisher: MAINSTAY CRYPTO LLC • OneBrowser Installer — Publisher: WORK PRODUCT INC

This inconsistency is intentional. Block one name, another slides through. Try building detections based on metadata, and the artifacts suddenly look unrelated.

Here is the growing list of hashes: • 4779db5eb5116033e0f2e5d7fda041a4887830c778319d55b0bbbef55a6a0e7f • 9b0a20c640efdc33a542c58a1ccc9120e6fb3dda256095505d03557e347f4f04 • a89119306a39c1cba6055548507ad3140d04288b4a2b19087d242e68a94548b0 • c6761ea547845e0ae37d6bc42eacdb10b1e14bcc8d6909363ea8bbc3ae2b67d0 • d6118f28991ca22e70ad9d056c7e9b21ad18acf1ce40a573b117c4f5ca62b142 • 415b6d1bb78cb74a468b29e7af09e885999cfcabf2c413f3bf533c2191d4e626 • 27d3bf2bb8b86283af214e9c60363421297ed1d7e8702c0faf1d80616d3ac216 • 717c2728b957a7faecb7d1ac057bb03053f6397bbc369092c493daf6d45dc67c • 239a3e38c467c6539d58d0f76352e502a9e5a697bc36ea2329da14a234236486 • 4617e321a142d4ed35d71d3532be764deec63dafe8bdec010a8ace4ea5bba5b4 • 97806d3aad8850f391298f7ca2a12504ed66619588dce05efe29d3e946375c66 • 38d37ae2a3b016c5172568bc708e2890fc8e413e7502e84b706725a6915bb95e • 9a35f60880b9551af2fd4a170a1e8f85b8216a2322b98497b6b5d1d9f3109288 • c1f686082eb39db8cd58f36247e894b22c95d10672e9d50380b824ab9f2e2f46 • e92b3fee05d6681def5776f8ca3d01afc9b0751812ab758dfabc2cd4432eca6c • 60fe51310585d95f48fc5fed54d27c78d409290b3d77866b35772c253219ea26

### Persistence Through the Windows Registry & Root Certificate Abuse

The strongest signs of malicious behavior came not from the installer itself, but from what it left behind.

#### 1. Subverting Trust: The Malicious Root Certificate

One of the most significant discoveries was its manipulation of Windows’ Trusted Root Certification Authorities, essentially the list of entities Windows automatically trusts for encrypted connections and signed code. Normally, this store contains only vetted, legitimate certificate authorities. Malware has no business being here.

Yet during analysis, multiple entries appeared within:

```
HKLM\SOFTWARE\Microsoft\SystemCertificates\AuthRoot\Certificates\<Thumbprint>\Blob
```

The Blob value, which holds the binary representation of the certificate, had been modified from its expected state, likely the injection of an unknown, unauthorized certificate.

The implication is severe: If attackers insert their own certificate into AuthRoot, Windows will trust anything signed with the corresponding private key. That means: malicious executables appear cryptographically “trusted”, HTTPS interception becomes possible without browser warnings, C2 traffic looks legitimate, and security products relying on certificate validation can be bypassed. This is not adware behavior. This is a high-risk trust subversion technique used by more advanced campaigns.

#### 2. Deep Camouflage: Masquerading and Uninstallation

In parallel with subverting system trust, the malware executed a deliberate campaign to appear legitimate to the end-user and administrative tools.

The installer created extensive entries within the user-level Uninstall registry keys ( HKCU\...\Uninstall\PDF_Spark_is1 ), mimicking standard software installation metadata:

- False Publisher and Versioning: The entries were filled with legitimate-looking details, such as the DisplayVersion (1.0.0.0) and attributing the Publisher to Mainstay Crypto LLC.
- Disabled Removal: To ensure persistence, the malware set the NoModify and NoRepair values to 1 . This simple but effective change removes the user’s ability to easily remove or modify the malicious software using the standard "Add/Remove Programs" function in the Control Panel.

#### 3. Execution Trace and Anti-Forensic Cleanup

The final set of artifacts provides clear forensic evidence of the payload’s launch and the actor’s subsequent cleanup effort:

- Execution Confirmation (BAM): Multiple execution traces recorded under the Background Activity Moderator (BAM) service state. This is critical as it confirms the execution of the initial payload, revealing its temporary path:

```
HKLM\SYSTEM\ControlSet001\Services\bam\
State\UserSettings\<SID>\Device\Users\<user>\AppData\Local\Temp\is-
####.tmp\PDFSparkOnSoft_####.tmp
```

- Shell Activity: BAM logs also recorded chrome.exe, cmd.exe, and conhost.exe, pointing to direct command-line operations, often used for post-install tasks, cleanup, or staging additional payloads. But this campaign did something unusual. It didn’t merely leave BAM traces — it modified BAM entries directly. Through OBUpdater.exe, the malware attempted to overwrite or manipulate BAM data, effectively trying to control what Windows would remember about its execution. Instead of BAM reporting what happened, the attacker tried to tell BAM what to report.

```
MACHINE\SYSTEM\ControlSet001\Services\bam\State\UserSettings\S-
#\\Device\ProgramFiles\Google\Chrome\Application\chrome.exe
```

- Self-Cleanup Command: One of the clearest signs of cleanup behavior came from a captured command sequence: nested cmd.exe invocations were used to execute a timed self-deletion command:

```
choice /C Y /N /D Y /T 5 & del /Q "C:\Users\user\AppData\Local\OneBrowser\Update\OBUpdater.exe"
```

This is not usual behavior. It’s a scheduled, timed removal designed to suppress prompts and silently clean up traces of an auxiliary persistence component, something often seen in loaders or update stagers that want to erase their setup trail.

Sparkling Goals:

- Look legitimate (fake installation metadata and versioning)
- Become trusted (inserted certificates into the system’s root store)
- Prove execution and leave minimal evidence (temporary paths recorded only in BAM)
- Clean up selectively (deleting its own components after setup)

### Scheduled Task Persistence — The OBUpdate Mechanism

One of the most consistent and revealing artifacts was the scheduled task named “OBUpdate”. This task acted as the primary persistence method for OneBrowser components.

How the Persistence Worked:

1. Installer Stage. The malware registered the scheduled task using either setup.exe or OBUpdateService.exe .
2. Archive/Dropper Stage. The installer executed a temporary setup file that silently unpacked a 7-Zip archive named: “ CHROME.PACKED.7Z” : "C:\Users\<user>\AppData\Local\Temp\T.tmp\G.tmp\setup.exe" --install-archive="C:\Users\<user>\AppData\Local\Temp\T.tmp\G.tmp\CHROME.PACKED.7Z" /S --data=# --inst_path="C:\Users\user\AppData\Local\Temp\T.tmp\onebrowser.exe" .
3. Execution Stage. The scheduled task ran through signed Windows processes for stealth: svchost.exe, taskhostw.exe. This blending technique makes malicious activity appear like routine Windows operations.

### DLL Activity: A Sign of Repackaging and Possible Side-Loading

Two DLLs repeatedly appeared during execution:

- d3dcompiler_47.dll — a legitimate Microsoft DirectX component
- turbojpeg.dll — part of the libjpeg-turbo library

Under normal circumstances, these libraries wouldn’t be modified or created inside temp installer directories. Their presence suggests either:

- The attacker repackaged legitimate DLLs to look trustworthy
- The malware attempted DLL side-loading by pairing trusted libraries with malicious executables

Either way, the behavior reinforces a pattern of concealment rather than functionality.

### Network Activity and Evasion Footprint

Outbound HTTP(S) activity revealed a network footprint characteristic of adware-downloaders and modular malware.

The initial focus of the malware’s activity centered on maintaining contact with its operational endpoints. Communication relied on frequent HTTP(S) POST and GET requests, primarily directed at two domains:

- oneinternetbrowser[.]com/service/update2
- crowdsourcesoftware[.]com

The threat actor’s choice to use standard web traffic (ports 80 and 443) for this communication is a fundamental evasion technique designed to bypass basic port-based network filtering.

The malware systematically employed a “blending” strategy, mixing its C2 beaconing with connections to benign or trusted network services to complicate detection:

- Cloud Hosting: The malware made direct connections to Amazon Web Services (AWS) S3 buckets ( pdfsparkcomponent[.]s3[.]us-east-2[.]amazonaws[.]com and pdfsparkcomponents[.]s3[.]us-east-2[.]amazonaws[.]com ). This use of trusted cloud infrastructure is a modern evasion tactic, as it leverages infrastructure that is rarely blocked and provides an effective, high-bandwidth channel for payload retrieval.
- Certificate Trust: Connections were observed to trusted Certificate Revocation List (CRL) providers, such as c[.]pki[.]goog and o[.]pki[.]goog . These connections mimic normal Windows certificate validation checks, adding a layer of legitimacy to the overall network profile.
- Supporting Domains: A large volume of traffic was directed to various advertising, telemetry, and service domains ( appsecuredata[.]com , getappsgroup[.]com , internationalcdn[.]com , supportcrc[.]com ), further obscuring the true malicious purpose among common background noise.

The observed network traffic spans several generic and cloud-associated IP ranges, including:

- 104.x.x.x and 142.x.x.x
- 3.5.x.x (Identified as a range within the AWS infrastructure)

This reliance on common, non-blacklisted IP ranges and the persistent use of standard ports (80, 443) collectively elevates the stealth profile of the malware. This strategic choice makes differentiation between legitimate host communication and malicious beaconing a significant challenge for network monitoring systems.

### Conclusion:

The PDF SparkOnSoft / OneBrowser family is not a simple adware nuisance — it is a coordinated distribution ecosystem that blends into normal user behavior, leverages legitimate installation frameworks, abuses Windows trust mechanisms, and establishes layered persistence.
