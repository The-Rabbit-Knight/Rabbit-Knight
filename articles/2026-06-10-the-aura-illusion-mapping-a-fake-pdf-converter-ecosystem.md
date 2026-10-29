---
title: "The Aura Illusion: Mapping a Fake PDF Converter Ecosystem"
date: 2026-06-10
medium_url: https://medium.com/@rabbit_knight/the-aura-illusion-mapping-a-fake-pdf-converter-ecosystem-b61730291506
---

> **Original Published Article on Medium:** [The Aura Illusion: Mapping a Fake PDF Converter Ecosystem](https://medium.com/@rabbit_knight/the-aura-illusion-mapping-a-fake-pdf-converter-ecosystem-b61730291506)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*BwOZ0-iyKuT9uG5Yz-1PtA.png)

### The Aura Illusion: Mapping a Fake PDF Converter Ecosystem

![Article Figure](https://cdn-images-1.medium.com/max/800/1*wjyTzZpRVCB46b5DaY7Vjg.png)

Over the last few months, a pattern emerged in endpoint telemetry. Fake PDF and document-converter installers: different names, different numeric suffixes, different publishers, different branding. Individually, each looked like another routine PUP caught by a standard download-from-browser rule. But when I started mapping the relationships between them, a more structured picture appeared.

This article documents the current state of that research. The goal is not to attribute the activity to a specific actor or make claims beyond the evidence. The goal is to present what connects to what, how those connections were established, and where the boundaries of the evidence are.

#### The Pattern

Every sample in this ecosystem follows the same structural blueprint:

```
Stage-1 EXE (downloaded via browser)
  → Stage-2 .tmp (Inno Setup child from AppData\Local\Temp\is-*.tmp\)
    → Installed Electron/Nativefier app (AppData\Local\Programs\<brand>\)
      → Network callback to shared infrastructure
```

The stage-1 installer doesn’t perform the installation itself. It extracts a temporary child process into ‘AppData\Local\Temp\is-*.tmp\’ and launches it with the ‘/SL5’ parameter, a standard Inno Setup behavior indicating the .tmp was started internally by the parent.

```
"C:\Users\USER~1\AppData\Local\Temp\is-#.tmp\ConvertPDF_*.tmp" 
  /SL5="$HANDLE,SIZE,SIZE,C:\Users\user\Downloads\ConvertPDF_*.exe"
```

Despite its .tmp extension, this is a full PE executable. It performs the actual installation: writing files, dropping DLL components, creating registry entries, and initiating network callbacks.

#### The Toolchain

VirusTotal metadata (DetectItEasy / TrID) confirms a consistent compiler fingerprint across the stage-2 children:

```
| Hash        | Role | Identification   |
|-------------|-------------------------|------------------------------------------------|
| `83ad86b2…` | ConvertPDF_*.tmp        | Embarcadero Delphi 11.x Alexandria, Inno Setup |
| `0921f5ef…` | ConvertPDF_*.tmp        | Embarcadero Delphi 11.x Alexandria, Inno Setup |
| `ce8df791…` | FlipGrip_*.tmp          | Embarcadero Delphi 11.x Alexandria, Inno Setup |
| `4a668d16…` | SetupPDF_*.tmp          | Embarcadero Delphi 11.x Alexandria, Inno Setup |
| `afdd70b3…` | SetupMyPDFConvert_*.tmp | Embarcadero Delphi 11.x Alexandria, Inno Setup |
| `9497472f…` | SetupPDF_*.tmp          | Inno Setup Module 6.4.3, Delphi 11.x           |
```

Every confirmed stage-2 child across all observed publishers uses the same Delphi compiler version and the same installer framework. This is a factual observation; it does not prove common authorship, but it is consistent with a shared build environment or packaging pipeline.

The PdfTransform cluster breaks the Inno Setup pattern. Endpoint telemetry for PdfTransform_* (hash 8b9311a0…) shows the installer EXE itself directly dropping NSIS artifacts: nsDialogs.dll, Banner.dll, embedhtml.dll, and multiple ns*.tmp files alongside pz.dll. There is no separate .tmp child process in this chain. The PdfTransform EXE performs all installation steps directly. This is structurally different from the staged Inno Setup model seen in every other cluster.

But a different PdfTransform variant, PdfTransform_* (hash 533c701c…, from Joe Sandbox ) does use Inno Setup staging with a PdfTransform_897340.tmp child and /SL5 parameters. So the PdfTransform family spans both NSIS and Inno Setup builds. Whether these represent different generations, different operators, or a deliberate toolkit split is unknown.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*n5N-vQrDg5HhnTTB7lN2Mg.png)

The final Electron payloads use a different toolchain entirely. Two payload hashes are confirmed: Payload Group A ( b58c6084b2621e9759d0c4e1ab38208ec3b4e1bb04f83c93f9a31bfd9269fc8d ) identified by TrID as a Win64 Executable. Confirmed as an Electron application by runtime structure. Payload Group B ( e6eddee12d08ca8c7f4746a963cb992c6f9fd0d7c8c5108ae5923d73d264506f ) PE64, identified by DIE as an “Electron package,” built with Microsoft Visual C/C++ 16.00, Microsoft Linker 14.0. Confirmed as an Electron application by runtime structure. Both are Electron/Nativefier applications verified by their process architecture (gpu-process, renderer, utility children) and resource layout (resources\app\lib\main.js, preload.js, static\login.js). The compiler details matter less than the structural confirmation; these are rebranded web-wrapper apps, not native binaries.

#### The Publishers

Four publisher strings appear across the confirmed samples: ONLINE INVESTMENT OPPORTUNITIES LLC signs the ConvertPDF_* family, with display names “Aura Document Studio Setup” and “Aura Docs Workspace Setup.” PRIVATELY OWNED ENTERPRISE LINUSOFTECH signs PdfTransform_* (“CORE-X-Convert”), FlipGrip_* (“ComfyMorph Setup”), and SetupPDF_* (“Comfy Morpher Setup”). DISPLAY NETWORK REVOLUTIONS LLC signs SetupMyPDFConvert_* and PDFFlip_* (OmniTransform). APP CRAFTSMEN LLC signs SetupPDF_* (“Optimal Converter Setup”) and ConvertPDF_* (“Optima PDF”). The distinction between “installer brand” and “payload brand” matters. The installer may carry one display name while the installed application carries another. In at least two confirmed cases, an installer from one publisher delivers a payload brand originally associated with a different publisher (detailed below).

#### Cluster 1: ConvertPDF / Aura

The first cluster to surface. Installer names follow ConvertPDF_#.exe`, signed by ONLINE INVESTMENT OPPORTUNITIES LLC.

Four samples were found. Two share the same stage-1 hash (fd9f432d…), same binary, different filename, with display name “Aura Document Studio Setup.” A third sample (5282530b…) is a different build but shares the same stage-2 child hash (83ad86b2…) with both. The fourth sample (5da14629…) carries the display name “Aura Docs Workspace Setup” and is a distinct branch with its own stage-2 child (0921f5ef…).

![Article Figure](https://cdn-images-1.medium.com/max/800/1*hwZL_7kA5dcC75P8vYqmgw.png)

DLL artifacts:

The three “Aura Document Studio” branches drop libzfse.dll (c5c1f111…). Detections include PUP, Adware, and Trojan labels from VT. The name references Apple’s LZFSE compression library, a macOS/iOS-focused project with no official Windows distribution and no obvious role in a PDF converter, making it plausible-looking camouflage rather than a genuine dependency.

The “Aura Docs Workspace” branch drops a different artifact: zfse.dll (a657f88f…). Similar naming, different hash. Telemetry shows the stage-2 .tmp creating, modifying, and then deleting this file; this pattern is consistent with temporary extraction and cleanup during installation.

Important caveat on DLL loading: Telemetry confirms file creation/modification/deletion events for these DLLs. Actual image-load events (confirming the DLL was loaded into a process) were not captured in the reviewed data window. It is possible these are installer-stage artifacts that are extracted and cleaned up without being loaded as modules.

Network activity observed from this cluster: - softwareinstallercdn[.]com — repeated POST requests (including to /session) - auradocstudio[.]com/ file.auradocstudio[.]com - auradocinstaller[.]com (Aura Docs Workspace branch only) - versioncurrent[.]com

#### Cluster 2: PdfTransform / CORE-X Convert

Installer names: PdfTransform_*.exe, signed by PRIVATELY OWNED ENTERPRISE LINUSOFTECH. Display name: “CORE-X-Convert.”

Two observed samples share the same hash (8b9311a0…) — same binary, different filenames. This is identical to the ConvertPDF pattern above.

DLL artifact: These drop pz.dll (6d236af6…). DIE identifies it as PE64, Microsoft Visual C/C++ 16.00, Microsoft Linker 14.0 — the same compiler/linker version as the Group B Electron payload (e6eddee1…), and notably not the Delphi/Inno Setup toolchain of the installers.

Cross-Publisher Link: PdfTransform → Aura Document Studio

A public Joe Sandbox report for a PdfTransform sample (533c701c…) provides the critical cross-publisher evidence:

Process chain: PdfTransform_*.exe → PdfTransform_*.tmp → Aura Document Studio.exe

Installed payload: C:\Users\user\AppData\Local\Programs\Aura Document Studio\Aura Document Studio.exe( b58c6084b2621e9759d0c4e1ab38208ec3b4e1bb04f83c93f9a31bfd9269fc8d )

Sandbox-observed behaviors: - Dropped PE files - Memory allocation in foreign processes - Writes to foreign memory regions - Process creation in suspended mode - HTTP GET/POST without user agent - Registry uninstall key: Aura Document Studio_is1

This is the pivotal finding: a LINUSOFTECH-signed installer (PdfTransform) delivers a payload branded as “Aura Document Studio” — a name associated with ONLINE INVESTMENT OPPORTUNITIES LLC in Cluster 1. One publisher’s installer delivers another publisher’s payload brand.

The same hash (b58c6084…) carries the following submission names on VirusTotal: Aura Document Studio.exe, CORE-X Convert.exe, Omni Transform.exe, Snap Convert.exe, ComfyMorph.exe, electron.exe. These are VT community-submitted names — not all are confirmed from internal telemetry. What is confirmed: the hash appears in endpoint telemetry as “Aura Document Studio.exe” and in the Joe Sandbox report under the same name.

#### Cluster 3: FlipGrip / ComfyMorph

Also signed by PRIVATELY OWNED ENTERPRISE LINUSOFTECH.

One confirmed FlipGrip sample (hash c69e0bda…) with display name “ComfyMorph Setup.” The stage-2 .tmp (ce8df791…) drops liblzma.dll (199c4042…). VT confirms it is a genuine DLL (File type: Win32 DLL; magic: PE32 executable (DLL) (GUI)), built with Microsoft Visual C/C++ 19.16. VT detections align with trojan/downloader/AdLoad behavior. liblzma is the name of a legitimate, widely-used LZMA compression library. The lib prefix is a standard naming convention and not itself suspicious. But there is no obvious role for an LZMA library in a Windows PDF converter, so the name reads as plausible-looking camouflage rather than a genuine dependency as we saw with libzfse.dll above. The VT detections, not the filename, are what flag it.

Infrastructure overlap: This FlipGrip sample makes POST/GET requests to softwareinstallercdn[.]com. This domain is also contacted by ConvertPDF (ONLINE INVESTMENT OPPORTUNITIES) and SetupPDF (APP CRAFTSMEN). This is the cross-publisher infrastructure link that anchors the ecosystem map.

Cross-Publisher Link: FlipGrip → OmniTransform

A second Joe Sandbox report for a different FlipGrip sample (SHA256: 13fcb6ca…) shows: Process chain: FlipGrip_*.exe → FlipGrip_*.tmp (with /SL5) Installed payload: C:\Users\user\AppData\Local\Programs\OmniTransform\Omni Transform.exe. With dropped Electron runtime DLLs (d3dcompiler_47.dll, ffmpeg.dll, libEGL.dll).

This is the second confirmed cross-publisher payload delivery: a LINUSOFTECH-signed installer (FlipGrip) delivering OmniTransform — a brand associated with DISPLAY NETWORK REVOLUTIONS. The pattern repeats.

#### SetupPDF: Comfy Morpher

A more recent sample in this lineage (observed in June):

A SetupPDF sample (314a8752…) with display name “Comfy Morpher Setup,” LINUSOFTECH publisher. The stage-2 child resolves mp[.]firstrunpdf[.]com and installs: C:\Users\…\AppData\Local\Programs\Comfy Morpher\Comfy Morpher.exe SHA256: e6eddee12d08ca8c7f4746a963cb992c6f9fd0d7c8c5108ae5923d73d264506f This is Payload Group B. Has the same Electron child-process pattern: gpu-process, renderer, network service, utility — standard Chromium/Electron multi-process architecture.

DLL artifact: libmpg.dll (83aa8e1f…). File magic confirms PE32 executable DIE: Microsoft Visual C/C++. Telemetry shows File Modification → File Deletion — consistent with temporary extraction during installation. The name follows the same pattern as the other dropped DLLs: libmpg references libmpg123, the open-source MPEG audio decoding library from the mpg123 project — open-source compression/decoding libraries with no obvious role in a PDF converter. Three of the four DLL artifacts in this ecosystem (libzfse, liblzma, libmpg) borrow names from legitimate Unix-native libraries, suggesting a deliberate camouflage strategy.

```
DLL dropped  | Real library it mimics   | Purpose of real library     | Native platform
-------------|--------------------------|-----------------------------|------------------
liblzma.dll  | liblzma (XZ Utils)       | LZMA compression            | Cross-platform; common on Linux/Unix
libzfse.dll  | libzfse / Apple LZFSE    | LZFSE compression           | macOS/iOS
libmpg.dll   | libmpg123 (mpg123)       | MPEG audio decoding         | Cross-platform; common on Linux/Unix and Windows
Cluster 4: OmniTransform / Optimal Converter
```

#### Cluster 4: OmniTransform / Optimal Converter

Two publishers:

![Article Figure](https://cdn-images-1.medium.com/max/800/1*E5kw3ACciCWKqnE3iA9Baw.png)

DISPLAY NETWORK REVOLUTIONS LLC

Signs the SetupMyPDFConvert_*.exe and PDFFlip_*.exe families.

Two SetupMyPDFConvert samples share the same hash (4fdd9059…) — same binary, different filenames. A separate PDFFlip/ConvertPDF pair (4cc1b7c8…) also shares one hash across two different name prefixes. This confirms the naming rotation is deliberate rather than coincidental.

An AnyRun analysis of a PDFSnap sample (same ecosystem, same naming conventions) confirms the full chain: - InnoSetup installer compiled with Borland Delphi - Environmental checks (computer name, machine GUID, language settings, IE security settings) - Final payload: C:\Users\admin\AppData\Local\Programs\OmniTransform\Omni Transform.exe - Persistence: Start Menu shortcut, Desktop shortcut (Omni Transform.lnk), registry uninstall key (OmniTransform_is1) - Network: contacts omnitransformapp[.]com (registered 2026–04–10, registrar SRSPlus), triggering AnyRun’s “unwanted program domain” behavioral detection

APP CRAFTSMEN LLC

Signs a SetupPDF sample (941f7ef6…) with display name “Optimal Converter Setup. The stage-2 child includes /verysilent in the command line: “C:\Users\USER\AppData\Local\Temp\is-*.tmp\SetupPDF_*.tmp” /SL5=”$HANDLE,SIZE,SIZE,C:\Users\user\Downloads\SetupPDF_*.exe” /verysilent This installs Optimal Converter.exe with hash: e6eddee12d08ca8c7f4746a963cb992c6f9fd0d7c8c5108ae5923d73d264506f This is the same hash as Comfy Morpher.exe from Cluster 3 — a finding significant enough to break out on its own (see below).

Network: contacts softwareinstallercdn[.]com (with long encoded query strings, could be session tracking, campaign attribution, or configuration exchange), appsetup24[.]com, rel.appsetup24[.]com, versioncurrent[.]com.

A separate ConvertPDF sample (b747b457…) — also APP CRAFTSMEN, display name “Optima PDF” — resolves oneinternetbrowser[.]com and softwareinstallercdn[.]com. The oneinternetbrowser[.]com domain is associated with OneBrowser/OBUpdateService infrastructure observed in a separate campaign. This is a potential link to adjacent activity but is not elaborated here.

#### The Same Binary, Two Publishers

The single most direct piece of evidence in this dataset sits at the boundary between Cluster 3 and Cluster 4:

SetupPDF (LINUSOFTECH) → Comfy Morpher.exe → e6eddee12d08… SetupPDF (APP CRAFTSMEN) → Optimal Converter.exe → e6eddee12d08… Two installers, signed by two different publishers, carrying two different brand names, install a payload with the identical SHA256 (e6eddee12d08ca8c7f4746a963cb992c6f9fd0d7c8c5108ae5923d73d264506f). In isolation, this could have benign explanations: two companies could license the same white-label Electron application from a shared vendor. But in context, alongside sandbox-confirmed cross-publisher payload deliveries, shared C2 infrastructure across three publishers, and a uniform installer toolchain, the convergence of evidence makes the shared-vendor explanation less parsimonious than a common operator. The reader can weigh these factors independently.

#### The Two Payload Groups

The installed Electron applications converge on two confirmed payload hashes:

Payload Group A (b58c6084…): - Confirmed deployment names: Aura Document Studio.exe (endpoint telemetry + Joe Sandbox), Omni Transform.exe (Joe Sandbox FlipGrip report) - Additional names from VT submissions (not independently confirmed): CORE-X Convert.exe, Snap Convert.exe, ComfyMorph.exe, electron.exe

Payload Group B(e6eddee1…): - Confirmed deployment names: Comfy Morpher.exe (SetupPDF/LINUSOFTECH chain), Optimal Converter.exe (SetupPDF/APP CRAFTSMEN chain) - Additional names from VT submissions (not independently confirmed): Optima Convert.exe, Vertex Documents.exe, Aura Docs Workspace.exe, Optima PDF.exe, electron.exe

Shared characteristics for both payload groups: - Are PE64 binaries (Group B confirmed as MSVC 16.00) - Run as Electron/Nativefier applications with resources\app\lib\main.js, preload.js, and static\login.js - Share Chromium runtime DLLs (ffmpeg.dll, libEGL.dll, d3dcompiler_47.dll, vulkan-1.dll, vk_swiftshader.dll, dxcompiler.dll, dxil.dll, libGLESv2.dll) - Install to AppData\Local\Programs\<brand name>\ - Spawn standard Electron child processes ( — type=renderer, — type=gpu-process, — type=utility)

The application-specific JavaScript resources (main.js, preload.js, login.js) have different hashes across brands even within the same payload group. This means the apps are not byte-for-byte identical rebrands. They share a common base but carry brand-specific application logic.

#### The Infrastructure

RDAP lookups reveal a pattern: every campaign domain shares the same registrar and the same Cloudflare nameserver pair.

All nine confirmed campaign domains were registered through TLDS L.L.C. d/b/a SRSPlus and point to the same nameservers: nina.ns.cloudflare.com and noah.ns.cloudflare.com.

- internationalcdn[.]com — registered 2025–10–01 (earliest known) - omnitransformapp[.]com— registered 2026–04–10 - softwareinstallercdn[.]com — registered 2026–04–17 - versioncurrent[.]com— registered 2026–04–17 (21 minutes after softwareinstallercdn) - auradocstudio[.]com — registered 2026–05–04 - auradocument[.]com — registered 2026–05–04 (3 minutes before auradocstudio) - auradocinstaller[.]com — registered 2026–05–22 - appsetup24[.]com — registered 2026–05–29 - firstrunpdf[.]com— registered 2026–05–29 (34 seconds before appsetup24)

The registration timestamps reveal batched infrastructure provisioning — pairs registered within minutes or seconds of each other. SRSPlus alone is not distinctive because it’s a common registrar. The registration timeline (October 2025 → April → May 2026) shows the infrastructure was built progressively over approximately 8 months.

softwareinstallercdn[.]com remains the strongest infrastructure pivot. It receives traffic from samples signed by at least three publishers and appears in both sandbox reports and endpoint telemetry.

#### Conclusion

The evidence supports a structured fake PDF/converter ecosystem built around staged Inno Setup installers, rebranded Electron applications, and shared network infrastructure. The signing identities change. The brand names change. The DLL names change. The numeric suffixes change. But the architecture underneath is mechanically consistent.

The strongest evidence for a connected ecosystem is the cross-publisher payload behavior: sandbox-confirmed deliveries crossing brand boundaries, and the shared payload binary observed under two different signing identities. These are observations about files, signatures, and infrastructure, not findings about the legal entities whose names appear in the signing certificates. Code-signing identities can be purchased, abused, stolen, or fraudulently obtained, and the presence of a company name in a certificate is not evidence that the named company authored, distributed, or had knowledge of the software. Where this report refers to “publishers,” it means the signing strings observed in file metadata, not an assertion about the conduct of any named entity. No allegation of wrongdoing against any named company is made or implied.

What the evidence supports is narrow: the samples are technically interrelated, they share infrastructure, and the payloads cross signing boundaries. Whether this reflects a single operator, multiple parties using a shared platform, abused signing credentials, or another arrangement is beyond what the current data can establish, and is not asserted here. Scope and limitations: This analysis is based on endpoint telemetry, public sandbox reports, and RDAP records at the time of lookup. A few boundaries are worth stating plainly: the dropped DLLs (libzfse, liblzma, libmpg, pz) were observed as file events, not confirmed loaded as modules, and may be installer-stage artifacts rather than active components. The “Aura Docs Workspace” branch is confirmed by branding but its final payload hash was not captured. All hashes, domains, and signing strings are published so that others can independently verify these findings.

Stage-1 Installer Hashes:

Publisher shorthand: OIO LLC = ONLINE INVESTMENT OPPORTUNITIES LLC; DNR LLC = DISPLAY NETWORK REVOLUTIONS LLC; LINUSOFTECH = PRIVATELY OWNED ENTERPRISE LINUSOFTECH.

```
| SHA256                                                             | Family                   | Publisher     |
|--------------------------------------------------------------------|--------------------------|---------------|
| `fd9f432df97f47d91abd462a408156a442b231cb7aa91b434fe3f4a1fc4b9f1c` | ConvertPDF_*             | OIO LLC       |
| `5282530b94e69e0b55e4608de3bfa5f230d3e3f4cb02c8e30b007eeb8cf00847` | ConvertPDF_*             | OIO LLC       |
| `5da14629d48611eeea8c2171c3f5cfe82df0c6f2c42ad6b0fe74a0289d18efee` | ConvertPDF_*             | OIO LLC       |
| `8b9311a09e26659deeeab648a6a96d598d7777c42a44da379665dcfe603cde72` | PdfTransform_*           | LINUSOFTECH   |
| `533c701c9c46d4f45407b76e86d4e5fe5af9e0cbf4201dcffb633b3a7f16d379` | PdfTransform_*           | LINUSOFTECH   |
| `13fcb6ca3add60428bd147826d274ea07037602ac7f4bd62e84be6ff08eb3d8a` | FlipGrip_*               | LINUSOFTECH   |
| `c69e0bda68ce6f72f3e9e6097e364e7c735fb9f6f64f5a1d743bbf9cd77e8b87` | FlipGrip_*               | LINUSOFTECH   |
| `314a87526a55330da8b0fbae2c9d1c8ffe110fb8fbd6dc9db0dadd8ea882ae65` | SetupPDF_*               | LINUSOFTECH   |
| `4fdd9059cceda195b734f548209647700cbc7d0a15cd8e62f5159be2d2125235` | SetupMyPDFConvert_*      | DNR LLC       |
| `4cc1b7c878006e708f913660358598c71907455a270ac5e9fd21bb2922e3dc51` | PDFFlip_* / ConvertPDF_* | DNR LLC       |
| `941f7ef682d0c676af62923bd2210166238469dd290b8a3ce2f785c1613faa08` | SetupPDF_*               | APP CRAFTSMEN |
| `b747b457564dc6edc27b37c2db1886e77440ff086463035ef32acfd05bcc3bc5` | ConvertPDF_*             | APP CRAFTSMEN |
```

Stage-2 .tmp Hashes (Inno Setup children)

```
| SHA256                                                             | Parent Family      | Publisher     |
|--------------------------------------------------------------------|--------------------|---------------|
| `83ad86b2be4f8e5581fdce4e2a85d7f01b7b09c0d6cad64a1986c2d2d390059d` | ConvertPDF_*       | OIO LLC       |
| `0921f5efd322a4d500670d3f3c451b3a6600d89fdca5d68b225c353f08025672` | ConvertPDF_*       | OIO LLC       |
| `ce8df791c5a6ff85b721272d9d1fcf58febb176552533833558c2d19036d8955` | FlipGrip_*         | LINUSOFTECH   |
| `4a668d161ebf2127dfe5eb45f535aefe323095ae903840886ae9602cfb5a99ec` | SetupPDF_*         | LINUSOFTECH   |
| `afdd70b3810208661ff955ee55ec7f74aef97b8eae62599c17177371518bcd94` | SetupMyPDFConvert_*| DNR LLC       |
| `9497472ff0611a35f28a24838c99b79984c4250f3687d2310647e8339038ecbf` | SetupPDF_*         | APP CRAFTSMEN |
```

Payload Hashes:

```
| SHA256                                                             | Names                                        | Group |
|--------------------------------------------------------------------|----------------------------------------------|-------|
| `b58c6084b2621e9759d0c4e1ab38208ec3b4e1bb04f83c93f9a31bfd9269fc8d` | Aura Document Studio.exe, Omni Transform.exe | A     |
| `e6eddee12d08ca8c7f4746a963cb992c6f9fd0d7c8c5108ae5923d73d264506f` | Comfy Morpher.exe, Optimal Converter.exe     | B     |
```

DLL Artifacts:

```
| SHA256                                                             | Name        |
|--------------------------------------------------------------------|-------------|
| `c5c1f111994267cbcfe33ff5fa6c46f1f682c3aff13b60a30df31aaf208679e3` | libzfse.dll | 
| `a657f88f597463838a3c821092967f02221d390cbcf87df6065241957b6e4cc9` | zfse.dll    | 
| `199c4042177e24d3436d18d56cbce3aeaf17a9aea3e9d9507fcdaed8cd3156ed` | liblzma.dll |
| `6d236af665ffd1fa01c903107eafb0729ae8b3f933525fce5f5edaabc960c4d9` | pz.dll      | 
| `83aa8e1fc35d43c5f2e2260e74ad518b2e7ff6d79bb106120093dbc019db0f5a` | libmpg.dll  |
```
