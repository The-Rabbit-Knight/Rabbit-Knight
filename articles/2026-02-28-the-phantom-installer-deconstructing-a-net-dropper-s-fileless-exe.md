---
title: "The Phantom Installer: Deconstructing a .NET Dropper’s Fileless Execution"
date: 2026-02-28
medium_url: https://medium.com/@rabbit_knight/the-phantom-installer-deconstructing-a-net-droppers-fileless-execution-7ec5b3bfaf8d
---

> **Original Published Article on Medium:** [The Phantom Installer: Deconstructing a .NET Dropper’s Fileless Execution](https://medium.com/@rabbit_knight/the-phantom-installer-deconstructing-a-net-droppers-fileless-execution-7ec5b3bfaf8d)

### The Phantom Installer: Deconstructing a .NET Dropper’s Fileless Execution

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Ucv3-guDIRoyuBp5Vqezrw.png)

PDFriend.exe (SHA256: 8ab984ac08c2cd4c79d2c3ee8e6da60269fd32bfb0fc8749e25174b41b78fcbd) looks like a useful PDF tool. But with so many fake PDF tools out there still waiting to be brought to light, what started as a routine check quickly revealed some highly unconventional behaviors for a standard desktop app. It utilizes on-the-fly C# compilation, pulling its core logic straight from the internet.

Here is a step-by-step breakdown of how this application was unpacked to expose its fileless execution flow.

### Step 1: False Trust and the Digital Signature

Before the executable is even dropped into a decompiler, its static properties reveal why security tools might initially trust it.

The binary is fully digitally signed. The Authenticode signature traces back to a valid GlobalSign EV CodeSigning CA, issued to an organization named “THE-SHOP STOP LLC”. With a certificate valid from July 2024 through July 2025, the executable presents a seemingly legitimate face to the operating system. If the certificate expired in July 2025, why does the operating system still read the signature as completely valid in 2026? This is due to a cryptographic mechanism called timestamping. When the file was originally signed, the file’s hash was sent to a Time Stamping Authority (TSA), which cryptographically proved the exact date and time the signature was applied. Because the timestamp proves the code was signed while the certificate was still active and in good standing, Windows permanently freezes the validity of that signature. This timestamping mechanism provides a massive advantage for malware authors. By holding a stolen or fraudulently purchased GlobalSign certificate just long enough to sign and timestamp their droppers, the resulting malware will continue to bypass SmartScreen and appear trustworthy to victims for years after the certificate naturally expires.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*52DNaiE2OlQtN6FereC52A.png)

### Step 2: Bypassing the Smokescreen

Loading the initial binary into dnSpy immediately highlights the Costura namespace.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*noK11IyvjTvIEx4c1lDemw.png)

Costura.Fody is a legitimate, open-source tool used by developers to embed DLL dependencies directly into a single standalone executable. While excellent for software portability, it also effectively obscures an application’s internal modules from basic static analysis by packing core files inside compressed resources.

Pivoting past the Costura loader and examining the custom classes reveals the true structure of the application, taking shape within the Installer.Modules namespace.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Qa7i3eu0DlXxADhVLw8-jQ.png)

### Step 3: The “Wait, What?” Moment

The most unusual architectural choice of this executable hides inside a module appropriately named ShortCode.cs .

![Article Figure](https://cdn-images-1.medium.com/max/800/1*bY7Kh89WtOc_auUpBNUgxA.png)

Instead of executing pre-compiled DLLs, the application does something entirely unexpected. It leverages the legitimate .NET Roslyn compiler APIs to build its code at runtime. Specifically, it uses CSharpSyntaxTree.ParseText() to read raw C# text strings, and CSharpCompilation.Create() to compile that text into a binary assembly directly in the system's RAM.

The newly compiled assembly is then loaded right into memory via Assembly.Load(memoryStream.ToArray()) and executed. Not a single new executable file is written to the disk during this entire process.

### Step 4: Interrogating the Server

Where is this raw C# text coming from? Tracing the execution flow using dnSpy’s Analyzer leads back to a method called getModules() .

![Article Figure](https://cdn-images-1.medium.com/max/800/1*TXwmm8aUVMS0QznWhX0Mrw.png)

The binary acts as a stager. It makes HTTP POST requests to an external server ( api.mikonfig.com ), explicitly asking for specific modules by name: "shortcut" , "report" , "terms" , and "privacy" .

Observing the network function handling these requests ( GetCode.Post ) shows exactly how the application formats its JSON payload and forces TLS 1.2 for the connection.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*WtYkZcbbwjqUe-RijtLugA.png)

To see exactly what the application is downloading, the server can be interrogated directly. A curl command can mimic the app's exact network request, asking the server for the "index" module:

```
curl -k -X POST 
"https://api.mikonfig.com/api/v1/call"
 \
     --tlsv1.2 \
     -H 
"Content-Type: application/json"
 \
     -d 
'{"product_id": "s12345", "module_id": "index"}'
 | jq -r .data > /tmp/BasicIndexCode.cs
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*BeXaAn7T8pkMTpEt1ivHSw.png)

### Step 5: The Payloads Exposed

Pulling down the snippets directly from the remote server exposes the application’s modular architecture completely. Looking at the actual fetched files, the operations are divided into distinct tasks:

- The Shortcut ( shortcut ): Drops an .ico file into the local AppData folder and creates a desktop shortcut. Interestingly, clicking this shortcut merely opens the default web browser to a PDF-related webpage ( https://home.[ProductName].com/#pdf ), rather than launching a local program.
- Host Reconnaissance ( det ): Silently fingerprints the system. It gathers the OS version (checking for Windows 7, 8, 10, and 11) and counts specific open windows to package into an info object.
- The Loading Screen ( index ): Provides a fake HTML/JS loading screen with a progress bar that eventually displays "Please wait...", keeping the user distracted while the rest of the operations run in the background.
- Process & Database Execution ( fto ): This module executes a multi-stage attack focusing on browser hijacking and persistence.

Process Termination: Because SQLite databases are locked while in use, the module first uses silent command prompt instructions ( taskkill /F /IM ) to forcefully terminate any running instances of chrome.exe or msedge.exe .

Browser Hijacking: Once the browsers are closed, it directly accesses the Web Data SQLite databases for Google Chrome and Microsoft Edge across multiple user profiles. It executes hardcoded SQL UPDATE commands on the Keywords table to overwrite default search engine configurations, forcefully setting new search URLs and modifying the new tab page behavior.

Secondary Payloads and Persistence: The module acts as a downloader, reaching out to an external domain ( installmyapps.com ) to fetch an encrypted archive ( PDFriend2.cab ). It decrypts this payload in memory using AES, drops a secondary executable into the hidden C:\ProgramData\PDFriend directory, and silently utilizes schtasks.exe to create a scheduled task ( PDfriendTask ). This task ensures the new payload runs with elevated privileges daily and upon user logon, establishing long-term persistence without ever displaying a window to the user.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Ovv0RXGR5akU7-sWcWbVRg.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*15VWo1SNctF5MIM6WSTYKg.png)

- Data Exfiltration ( report ): Contains a part of C2 protocol implementation used to push collected data to a remote side. One theory I have after trying to get that malware run in a cloud sandbox is that there could be sandbox detection logic involved. When I run the app in a sandbox, it fails to contact the C2 server. Surprisingly, I couldn’t find any detection logic in the malware code itself, though. The theory is that, given how extensive the initial fingerprinting is and the fact that this fingerprinting data is sent to another C2 endpoint, C2 must be reacting to different fingerprints differently. For instance, if the run is performed from a sandbox, the C2 could just choose to completely ignore traffic from the endpoint. However, it’s just a theory.

### The Takeaway

If there is one interesting architectural choice to take away from this analysis, it is this: A supposedly legitimate, digitally signed PDF application is downloading its own raw source code from the internet, assembling it directly in memory at runtime, querying background databases, and tracking running windows. The reader can decide whether this is simply an eccentric software engineering choice or a highly effective evasion strategy designed to bypass static, file-based detection systems.
