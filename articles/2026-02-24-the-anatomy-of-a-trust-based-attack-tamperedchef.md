---
title: "The Anatomy of a Trust-Based Attack (TamperedChef)"
date: 2026-02-24
medium_url: https://medium.com/@rabbit_knight/the-anatomy-of-a-trust-based-attack-tamperedchef-fa1811c70601
---

> **Original Published Article on Medium:** [The Anatomy of a Trust-Based Attack (TamperedChef)](https://medium.com/@rabbit_knight/the-anatomy-of-a-trust-based-attack-tamperedchef-fa1811c70601)

### The Anatomy of a Trust-Based Attack (TamperedChef)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*_i87UWulfDhTfBROY-clhw.png)

The “TamperedChef” campaign has redefined modern malvertising. It is not just a simple malware drop; it is a self-contained ecosystem that exploits the inherent trust placed in signed software and professional-looking utilities. Dissecting the artifacts left on infected endpoints reveals a modular attack chain that carefully balances legitimacy with theft.

The core of TamperedChef’s success relies on a technique known as “Living off the Land.” By utilizing node.exe, a standard, digitally signed tool used globally by software developers, the attackers ensure their presence appears normal to basic security monitors.

The infection typically begins with a signed installer that is bundled with other software or directly downloaded by a user. These utilities are usually named like OpenMyManual, ManualReaderPro, PDFEditor (PDF fake tools). The loaders are often signed with valid Extended Validation (EV) certificates acquired through shell companies. Inside the installation folder, the presence of WebView2Loader.dll allows the application to display a functional, embedded browser window.

One of the most calculated aspects of TamperedChef is its patience. Security researchers from Sophos and Acronis have noted a consistent 56-day dormancy period. The malware stays functional but benign for nearly two months. This period strategically aligns with the typical 60-day lifecycle of a paid search engine advertising campaign. By remaining quiet, the attackers allow their ads to reach the maximum number of victims before security teams flag the malicious behavior.

The following two examples detail how these deceptive tools operate once they establish a presence in the environment.

### The PDFInstaller Variant (Data Harvesting)

The first execution chain centers around an installer masquerading as a PDF utility. Persistence is established via a Windows Scheduled Task named \PDF. The execution originates from the Windows Task Scheduler service (svchost.exe -k netsvcs -p -s Schedule), which triggers a hidden PowerShell instance.

The explicit command line executed is:

```
powershell.exe -NoProfile -WindowStyle Minimized -Command \
  "Start-Process 'C:\Users\user\AppData\Local\PDFInstaller\node.exe' \
     -ArgumentList '.' \
     -WorkingDirectory 'C:\Users\user\AppData\Local\PDFInstaller\' \
     -WindowStyle Hidden"
```

This command ensures the Node.js binary runs entirely out of sight, spawning supporting processes like conhost.exe . Once active, the node.exe process establishes a DNS connection to docu-flex[.]com . Its primary objective is clipboard hijacking. The Node.js engine spawns subsequent cmd.exe processes to interact with a bundled tool named clipboard.exe . Telemetry captures the execution of clipboard.exe -paste to read stored clipboard data, followed by clipboard.exe -copy to manipulate or inject new content.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*fjXNF61NAnxPwP9Kr7ZZ2w.png)

### The OpenMyManual Variant or ManualReaderPro (System Fingerprinting)

The second variant, disguised as an application named OpenMyManual. The initial execution chain begins with a minimized command prompt:

```
cmd.exe /C start "" /min \
  "C:\Users\user\AppData\Local\Programs\OpenMyManual\node\node.exe" \
  "C:\Users\user\AppData\Local\Programs\OpenMyManual\helper.js"
```

This launches the Node.js engine and passes the malicious helper.js script as an argument. The execution leaves forensic artifacts in the Background Activity Moderator (BAM) registry keys, confirming the exact path and user profile associated with the binary. Immediately upon execution, the script reaches out to the domain api.pyej17uw09d1bqlndg[.]com ( or api.cjby76nlcynrc4jvrb[.]com ) . It then spawns a command shell to fingerprint the host by executing

```
reg.exe QUERY "HKLM\Software\Microsoft\Cryptography" /v MachineGuid
```

Attackers ensure their persistence through scheduled tasks. The Windows Task Scheduler (svchost.exe) executes a specifically named task (\ ffe2391a-7849–449f-bb31–6df6edbb9053 or \697ea700-bd22–4670–9bf0–638b328e68dc ) ( \sys_component_health_66f458a6–1ea0-fe24–4355–736ad9f3a9d6bc65 — ManualFinder — C:\Users\user\AppData\Local\TEMP\66f458a6–1ea0-fe24–4355–736ad9f3a9d6bc65or[.]js) utilizing a Trigger Type 107. The source of this task directly targets the system registry to continuously query the MachineGuid , ensuring the host remains uniquely identified during every execution cycle.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*7qfdWlx8j9M7Y3hVxZFODQ.png)

### Deobfuscating the Core Payload

Extracting the payload from the OpenMyManual variant required navigating layers of intentional obfuscation. The raw helper.js (sha256: de101b0a881d69ab314e0863845e5f0e62c749eea87a704ecbb3bccb5c0bb1ac) script was heavily scrambled to deter static analysis and hide its network indicators. Uncovering the core logic required a multi-stage normalization process, first utilizing the Synchrony tool to clean up the control flow, followed immediately by Restringer to resolve obfuscated strings and evaluate safe expressions

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Zp3e88zlF4sopHqTK4T1QA.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Ob497XAOd8UecAFR5tFayQ.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*xElg37sL24xAhU4pAhCvSQ.png)

The resulting deobfuscated code exposes the inner workings of the malware.

### The Core Backdoor: helper.js

The helper.js script serves as the persistent backdoor and dynamic downloader for the OpenMyManual variant. A decrypted analysis reveals a sophisticated, bundle-like architecture that organizes its operations into discrete number-labelled modules.

- Module 693 provides full Create, Read, Update, and Delete (CRUD) operations for the Windows Registry, primarily utilized to retrieve the MachineGuid.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*uWvCfu4Y-xajqE_wvUB4wQ.png)

- Module 194 ensures operational silence by hooking into the environment, redirecting standard console.log and console.error outputs to a local log.txt file.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*V1SLgK_Imyjs6hlMqcGDQg.png)

- Module 242 contains the hardcoded Command and Control (C2) configuration, directing traffic back to api.pyej17uw09d1bqlndg[.]com.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*QIlpICZMYCtFLC1OHCnbfg.png)

### Network Protocol and Cryptography

Communications are handled by Module 159, which implements a custom bi-directional encrypted protocol over HTTPS.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*sCaQoT8r2NIGgXDvpeLSlw.png)

The malware periodically sends a JSON-formatted telemetry payload, or “heartbeat.” This object contains the specific MachineId, a randomly generated SessionId, and the OSVersion, along with any command-line arguments the script received at startup.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*z4-jqrXWOOLYX2wjfCOheA.png)

To evade network-level detection, all traffic utilizes a symmetric XOR cipher. For outgoing data, the malware generates 16 random bytes to act as an Initialization Vector (IV). The JSON payload is serialized, and each byte is XORed against the IV utilizing a modulo operation (IV[index % 16]). The IV is prepended to the encrypted data, Base64-encoded, and transmitted via a POST request.

Incoming C2 instructions reverse this process. The malware decodes the Base64 response, extracts the first 16 bytes as the Response IV, and decrypts the remaining payload. The decrypted string is then passed directly into an eval() function, allowing the C2 server to execute arbitrary JavaScript dynamically on the infected host.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*ap9wcXBQQ0Zq8AA4gtG5XA.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*NfKVwF_x_mI0XL3-EaB3cw.png)

### Anti-Analysis and Evasion Techniques

To hinder forensic analysis and automated sandboxing, helper.js incorporates several defensive mechanisms before initiating any malicious network activity.

- Self-Integrity Checks: The script uses Function.prototype.toString() combined with regular expressions (specifically checking for patterns like (((.+)+)+)+$) to detect if its code has been “beautified” or modified by a security researcher.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*jEesoqGWO2nERHEri5ojPA.png)

- Anti-Debugging Loops: It employs recursive functions that trigger debugger; statements and infinite loops if it detects an active debugging session or code tracing.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*y_ZJOI73U0WyARVVCD9jeg.png)

- Console Hooking: It intercepts and replaces global console methods, effectively silencing the script’s output and redirecting it to hidden files, ensuring no suspicious output appears in standard development consoles.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*l8MaIzzdQguO7sQ0D3Wchw.png)

### Conclusion

The true danger of TamperedChef is not just the code itself, but the patience and deception behind it. By hiding malicious intent inside everyday developer tools, attackers are forcing defenders to question what trusted applications are actually doing behind the scenes. Relying solely on static file signatures is no longer a viable defense. Security teams must pivot entirely to behavioral monitoring because when the software is legitimate, the behavior is the only giveaway.

### IOC

sha256 : ec3487b0c87a072d9a730cb7ad6671fc1c9b983f26ef1f9ce66b9bd718fe92e9 - ManualReaderPro.exe

sha256 : 0abd1e39e17fa99366c8f1cc9171730867b6e86f6362b0492a090170f0305e55 - manualreaderpro.exe

sha256 : 167359b715610003752cbc89b122a6df97e501304cb4a1ee94a6e75ebf51d6d6 –24c92c24–5c4e-451a-8885–9509dc69ab38[.]js

sha256 : de101b0a881d69ab314e0863845e5f0e62c749eea87a704ecbb3bccb5c0bb1ac -helper[.]js

sha256 : a0dae9b551026295575dcf4b1f668069b8fe8119458e792e8293299a74e79436 -OpenMyManual.exe

sha256 : a16cbf9ab535d4ad628b583ec3e026799f38bb50b98c495333302f7b804390ea -OpenMyManual.exe

sha256 : 33a9d45e2d549b81718593134e3ee3cb1cfd3a38a8bbde3c62a7062d2015d6bd — OpenMyManual.exe

Tasks

\697ea700-bd22–4670–9bf0–638b328e68dc

\ffe2391a-7849–449f-bb31–6df6edbb9053

\PDF

\sys_component_health_66f458a6–1ea0-fe24–4355–736ad9f3a9d6bc65

DNS

api.pyej17uw09d1bqlndg[.]com

api.cjby76nlcynrc4jvrb[.]com

docu-flex[.]com
