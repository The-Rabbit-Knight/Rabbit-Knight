---
title: "The Update That Wasn’t: A Deep Dive into Easy2ConvertUpdater.exe"
date: 2025-10-27
medium_url: https://medium.com/@rabbit_knight/the-update-that-wasnt-a-deep-dive-into-easy2convertupdater-exe-5d9f0407d870
---

> **Original Published Article on Medium:** [The Update That Wasn’t: A Deep Dive into Easy2ConvertUpdater.exe](https://medium.com/@rabbit_knight/the-update-that-wasnt-a-deep-dive-into-easy2convertupdater-exe-5d9f0407d870)

### The Update That Wasn’t: A Deep Dive into Easy2ConvertUpdater.exe

![Article Figure](https://cdn-images-1.medium.com/max/800/1*ZxgGybigNaKi4hKMJZyzsg.png)

In this article, I’ll dive into the analysis of Easy2ConvertUpdater.exe, a file that appeared after the process Easy2ConvertApp.exe created a scheduled task to launch it. You can find a full analysis of Easy2ConvertApp.exe here

In that article, my focus is solely on the Updater, trying to understand what it really does, where it came from, and whether it’s truly an update utility or something more sinister.

#### Sample Details

- Sample name: Easy2ConvertUpdater.exe
- SHA256: 27262f4bf8096f04e53309d4ce603cfbeb27ed10abdf1c461d3ccb14e012f61e

#### Static Analysis (PEStudio)

I began with PEStudio, starting by inspecting the PE header to confirm the binary type and gather metadata.

The summary confirms two key findings: first, that the file is a standard Windows executable, identified by the file signature (the ‘MZ’ magic bytes, 4D 5A in hex); and second, that this application is built on the .NET framework, confirmed by the successful location of the CLR Header inside the file’s structure.

The first odd thing here is the compiler timestamp: April 2050. Unless this updater came from the future, that’s a clear sign of timestomping, a common evasion technique used to mislead timeline analysis.

Another red flag appeared in the certificate section: the binary was signed by bluetakin, an issuer linked to suspicious domains and phishing activity, according to VirusTotal. Here more: the hardcoded links ( hiko[.]lakienti[.]com/...) found in this updater is also reported for malicious activity by VirusTotal . This is getting even more interesting. An interesting theory to entertain could be that it’s a link to a specific, unchangeable Command and Control server. It can be a remote endpoint for data exfiltration or subsequent payload delivery.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*TIcfzIpQuYz-gRsmLAENOA.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*WB148zDKNqNfYKkDaFP8cA.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*9chXoY0uYQkb9m5W8Ave0w.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*6IfjiOsi8VKajQq7ZV7PNw.png)

#### Decompiled Code Analysis (dnSpy)

Because the sample is a .NET binary, dnSpy was my next stop. The tool lets you peek under the hood of any .NET assembly, though reading decompiled code that is purposely misleading is never easy. What I found reads like a small, purpose-built communications pipeline rather than a typical updater:

1. The application reads a file named u.txt , including its content and metadata.
2. It wraps that data together with the current timestamp, then serializes everything into a JSON payload.
3. The JSON is sent via HTTP POST to the previously discovered endpoint ( hiko[.]lakienti[.]com ).
4. If it returns non-empty data, the data is used as a part of next HTTP request to the same server.
5. This second response delivers binary data, which is encrypted.
6. The data is apparently encrypted and decrypted upon retrieval using AES (symmetric) algorithm using a key and IV generated based on the initial request data and the token returned by the server before.
7. Finally, the decrypted payload is loaded into memory and executed dynamically using the AppDomain class.

Suspicious Traits

Several characteristics point toward malicious intent:

- Legitimate applications rarely use custom encryption and key generation for server communication; this suggests obfuscation.
- The in-memory execution of downloaded code is a hallmark of a dropper or loader.
- Combined with the timestomped metadata and suspicious domains, it’s reasonable to assume this binary was designed to bypass detection and deliver secondary payloads.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*WJaelsz-dDEFmOkSHx0qRA.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*iXbryCfMUah9Wa9lJQ8XCQ.png)

#### Capability Analysis (CAPA)

Next, I ran the sample through CAPA to get a high-level picture of its behavior and confirm my findings from reading the decompiled code. One finding immediately stood out: reflective code loading, the more formal name of the technique I discovered in the code above. In simple terms, this means the program can load and run code directly in memory instead of saving it to disk first. That’s a clever way to stay under the radar, since many antivirus tools only scan files written to disk.

This matches MITRE ATT&CK technique T1620: Reflective Code Loading — a method often used by attackers to execute hidden payloads and avoid detection by security software.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Qqot7R_H0y2jvOZtztr45A.png)

#### Dynamic Analysis (Procmon & Wireshark)

For runtime testing, I recreated the environment:

- Added a u.txt file containing data observed from a previous sample (Easy2ConvertApp.exe created this one prior).
- Reconstructed the original folder structure ( C:\Users\user\AppData\Local\Easy2Convert\Easy2Convert.exe)
- Launched Wireshark to capture network traffic and Procmon for syscall monitoring.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*KiyRyr6Bw5KRhUufg5iSNA.png)

Upon execution, the binary showed no visible activity or file creation. Wireshark logs contained no DNS requests to the flagged domains.

A closer look revealed that the application crashed shortly after launch, invoking the Windows crashdump reporter (WerFault.exe), likely due to missing dependencies.

Overall, dynamic analysis didn’t reveal additional behavior beyond what was already uncovered in static analysis.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*9LnMVNIgkpKsoQUJbbyB4Q.png)

#### Conclusion

The file masquerades as an updater but functions as a .NET dropper/loader. It sends system identifiers to hiko[.]lakienti[.]com and dynamically loads encrypted assemblies into memory for execution. With its manipulated timestamp (April 2050), suspicious certificate, and reflective code loading, this sample demonstrates multiple anti-forensic and evasion techniques.

In short: it’s not an updater — it’s a delivery mechanism.
