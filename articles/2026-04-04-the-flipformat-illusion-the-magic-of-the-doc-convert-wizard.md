---
title: "The FlipFormat_# Illusion: The Magic of the Doc Convert Wizard"
date: 2026-04-04
medium_url: https://medium.com/@rabbit_knight/the-flipformat-illusion-the-magic-of-the-doc-convert-wizard-10036623d0e8
---

> **Original Published Article on Medium:** [The FlipFormat_# Illusion: The Magic of the Doc Convert Wizard](https://medium.com/@rabbit_knight/the-flipformat-illusion-the-magic-of-the-doc-convert-wizard-10036623d0e8)

### The FlipFormat_# Illusion: The Magic of the Doc Convert Wizard

![Article Figure](https://cdn-images-1.medium.com/max/800/1*MvYMLhLqmQZD5Hz_QK4aNg.png)

Doc Convert Wizard (originating from the installer FlipFormat_947850.exe) looks like a useful document conversion tool. But with so many fake utilities out there still waiting to be brought to light, what started as a routine check quickly revealed some highly unconventional behaviors for a standard desktop app. It utilizes customized NSIS scripts, hidden network requests, and leverages legitimate open-source tools to set the perfect stage for a classic DLL sideloading hijack.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*a-7X-7Tsov0HD2wH-DGjXQ.png)

Here is a step-by-step breakdown of how this application was unpacked to expose its deceptive execution flow.

Files analyzed:

- b68415932c4757897def578e356edec75838475faedaf5ad420b660acf0f27bf PDFGrip_650632.exe (FlipFormat_947850.exe)
- d94322a4eeb72e8afbfd020f691bb9f8dd9da099c86ff794c0c571b550fe555b Doc Convert Wizard.exe
- c21eb14ba63e943db5ea9ab64af02a50a17260c7d8538a133c4f6e0957d36f47 main.js

#### The Smokescreen Dropper and the Fat Tail

The binary is an NSIS (Nullsoft Scriptable Install System) executable, but its size immediately raises eyebrows. The total size of the executable is around 8.5 MB, yet the actual LZMA-compressed NSIS installation archive only accounts for 5 MB. The remaining 3.5 MB of data is appended directly to the end of the file as a high-entropy “tail.”

![Article Figure](https://cdn-images-1.medium.com/max/800/1*h3ezE500huUa1kPlcsNO_A.png)

One theory I have after trying to extract this tail is that it acts as garbage padding. Adding megabytes of random, high-entropy bytes to the end of an executable is a common tactic to artificially inflate file size. This can successfully bypass certain cloud sandboxes that have strict file-size limits for automatic scanning, while also altering the file hash without breaking any digital signatures. While it is possible this block contains a secondary custom-packed payload, static analysis of the NSIS script has not revealed any instructions designed to read or decrypt this specific appended data. However, it’s just a theory.

#### Interrogating the Installer’s Network Flow

If the malware isn’t hidden in the 3.5 MB tail, where is it? Carving out and decompressing the internal NSIS script bytecode reveals the dropper’s true mechanism: it acts as a stager.

The script utilizes a bundled plugin called NScurl.dll to make a direct, silent HTTPS request to an external Command and Control (C2) server.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*_wWNST8jjov-7S_pC-5NWg.png)

The installer explicitly reaches out to https://tool[.]securedocumentinstaller[.]com/appsectools, setting a connection timeout of 7000 milliseconds. If the download succeeds, it saves the file locally as “\appsectools”.

Crucially, the script then calls the “nsis7z::Extract” function to unpack it. Because no password is provided to the extraction command, it's possible that the downloaded file is a standard, unencrypted 7z archive containing the core application.

#### The DLL Sideloading Moment

During the extraction phase, the installer drops two peculiar files into the target directory: “d3dcompiler_47.dll” ( which I have seen frequently in fake PDF tools) and “libEGL.dat”.

In the context of modern web-based desktop apps (like Electron or CEF), “libEGL.dll’ is an essential graphics rendering library. However, this installer deliberately drops it with a .dat extension.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*w24X9gzUgtJboXWsHE0hpA.png)

Reverse-engineering libEGL.dat reveals a surprising fact: despite the .dat extension, this file is a legitimate, unmodified, safe Windows DLL from Google’s ANGLE project. It contains standard exports like eglInitialize and eglMakeCurrent.

Why would an attacker drop a perfectly clean DLL but rename its extension?

The theory is that this is preparation for a sophisticated DLL Proxying attack (a form of DLL Sideloading). Given that the final application requires the 145 MB Electron runtime (which is downloaded from the C2 server), I theorize that the downloaded appsectools archive contains a malicious file named libEGL.dll.

When the user launches the application, Windows will load the malicious libEGL.dll into memory first. The malicious DLL executes its spyware payload, and then silently forwards (proxies) all legitimate graphics requests to the renamed, clean libEGL.dat. This ensures the application runs smoothly, keeping the user completely unaware. Without intercepting the downloaded appsectools archive to confirm the presence of this malicious DLL, this remains a highly probable, yet unproven, hypothesis based on the artifacts present.

#### Living Off The Land

The ultimate goal of this execution chain is to launch a Nativefier (Electron) bundle governed by a script named main.js.

Initial analysis of main.js might raise massive red flags: the script features an IPC “backdoor” (session-interaction), explicitly disables the Chromium sandbox (contextIsolation: false), and automatically grants all system permissions (camera, microphone, geolocation) without prompting the user.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*mgmrhLTf6Lia22o4QdDhYQ.png)

At first glance, this looks like a custom-coded, highly weaponized spyware script. However, a deeper dive into the source code reveals a brilliant, deceptive reality: main.js is standard, unmodified open-source Nativefier boilerplate code. However, it was probably compiled for specific purposes. This specific hash of main.js has been identified in three distinct PUP-associated directories: “DocConvertWizard”, “FoodFormula”, and “ProfessionalConverter”, “PDFPolish”.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*GyC3H1HtG2CFjS0cuz1fjQ.png)

The attacker didn’t write a custom, highly weaponized Electron app. Instead, they took a completely legitimate open-source tool (Nativefier), intentionally built a generic app with its natively loose security defaults, and used it as a perfectly clean Trojan Horse.

By utilizing a standard, widely trusted application framework, the scripts (main.js) will fly right past static antivirus scanners. However, the main executable (doc-wizard.exe) itself has been compiled with custom metadata (Company Name: “Jia Hao”, Product Name: “Doc Convert Wizard”). While the internal engine of the executable is the standard benign Electron framework, this custom branding makes its file hash unique to this specific campaign, providing an Indicator of Compromise (IoC) for defenders.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*uYkaoTK8msBMPt4Odtq6Uw.png)

#### The Takeaway

If there is one interesting architectural choice to take away from this analysis, it is this: a supposedly legitimate document converter acts as a lightweight stager, reaching out to a remote server to download an unencrypted archive, and carefully positioning a renamed, legitimate graphics library to facilitate a silent DLL sideloading hijack. Furthermore, by “Living off the Land” and relying on the security defaults of an open-source Electron wrapper, the attacker constructed a pristine, undetectable host application. And don’t forget the present in the chain of events, our old friend One Browser .

The reader can decide whether this is simply an eccentric software engineering choice or a highly effective evasion strategy designed to mask a stealthy, modular malware.
