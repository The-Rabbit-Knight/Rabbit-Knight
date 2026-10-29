---
title: "When Your Calendar Wants to Steal Your Tokens: A Look at “Calendaromatic”"
date: 2025-09-25
medium_url: https://medium.com/@rabbit_knight/when-your-calendar-wants-to-steal-your-tokens-a-look-at-calendaromatic-81c15bb7ed9e
---

> **Original Published Article on Medium:** [When Your Calendar Wants to Steal Your Tokens: A Look at “Calendaromatic”](https://medium.com/@rabbit_knight/when-your-calendar-wants-to-steal-your-tokens-a-look-at-calendaromatic-81c15bb7ed9e)

### When Your Calendar Wants to Steal Your Tokens: A Look at “Calendaromatic”

![Article Figure](https://cdn-images-1.medium.com/max/800/1*1_9zwx1e6zAyLnVehFJWsQ.png)

Every once in a while, malware shows up wearing a costume so absurd you almost have to respect the creativity. Calendaromatic.exe is a piece of malicious software that pretends to be nothing more than a humble calendar app. Spoiler: it’s not here to help you remember your colleagues’ birthdays.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*KFSClzcClvErXE82k7Zk6Q.png)

#### What Made It Interesting

On the surface, Calendaromatic looked harmless. But hidden inside, it was pulling tricks straight out of a magician’s handbook. One of the more clever moves? Calendaromatic had a hidden Command & Control (C2) channel, which was obfuscated as a normal API interaction over HTTP. Even if traffic is captured and read by a Security Analyst, it still looks like a normal interaction between a calendar application and its backend server. It’s just normal JSON with (almost) normal data. But inside that data, some normal symbols would be replaced by their homoglyphs. Basically, the C2 server was encoding hidden instructions to the application with a particular sequence that replaced ordinary-looking characters (like a dash) with nearly identical Unicode twins, making hidden instructions stay hidden from the human eye in trivial calendar data.

And, of course, the whole thing was signed with a valid digital certificate by CROWN SKY LLC. Fancy. Except the certificate had already been revoked.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*RbqAKBkHpaWySzB4JPE6Hw.png)

#### The Power of Homoglyphs in Malware

One of the trickiest parts of the Calendaromatic attack was its use of Unicode homoglyphs : characters that look the same to us, but are actually different to a computer. For example, what looks like a normal hyphen (-) could secretly be an en dash (–) or an em dash ( — ). To the eye, they’re nearly identical, but under the hood, they’re treated differently.

The app’s clean() function was supposed to tidy up holiday data by checking for these look-alike characters. But instead of just cleaning the text, it used them to also read and decode secret instructions. When decoded, those instructions could be turned into commands executed on the host where Calendaromatic is installed. This gave the attackers a way to make the “harmless” calendar app run malicious code.

#### Leveraging Modern Application Frameworks

Calendaromatic was built using NeutralinoJS , a lightweight framework that enables developers to create cross-platform desktop applications using web technologies. NeutralinoJS apps use WebView2 , they create folders like ‘EBWebView’ to store things like cookies, cache, and site settings.

This design choice meant that the application could interact directly with the host operating system, executing native commands and spawning processes. In Calendaromatic’s case, it launched cmd.exe to execute a command that opened a new Chrome window pointing to a malicious URL ( httpx[:]//lovetravellinga[.]com). This technique exemplifies how modern application frameworks can be exploited to execute arbitrary code, bypassing traditional security measures.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Z1lxBdBcNi6hgN8zme4Saw.png)

#### The Process Trail

The executable, calendaromatic.exe, was a 7-Zip self-extracting archive (SFX). The primary payload, calendaromatic-win_x64.exe, was a thin wrapper around the actual malicious logic contained within a file named resources.neu. This file housed the application’s web code, including HTML, CSS, and minified JavaScript, which was responsible for executing the attack.

Digging into execution shows how messy the chain really was:

- chrome.exe kicked things off, launching Calendaromatic.exe from the user’s Downloads folder. That unpacked into a temp directory: C:\Users\<UserName>\AppData\Local\Temp\7ZipSfx.000\calendaromatic-win_x64.exe
- Next, cmd.exe popped up, running a command string that reopened Chrome in a hidden window and quietly sent it to a shady domain: httpx[:]//lovetravellinga[.]com/ (don’t go there)
- Then came the kicker: msedgewebview2.exe was launched with a long list of flags to embed Calendaromatic as a fake webview application, complete with its own “userdata” directory: C:\Users\<UserName>\AppData\Roaming\calendaromatic-win_x64.exe\EBWebView

That’s a whole lot of browser and system processes being hijacked just to get a “calendar app” running.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*1tYVggDxpZC8K_9TUr8jiw.png)

#### Behavior in the Wild

The malware didn’t just sit there. Once active, it could reach out to a ( C2) server, ready to fetch and execute commands, read files and execute applications in the host system. This combination of tricks together with steganography through homoglyphs makes the intrusion hard to catch unless you’re piecing the whole chain of events together.

#### Why It’s Tricky

Calendaromatic raised an interesting point about detection. Some attacks don’t fire off a single big red alert. Instead, they spread tiny suspicious actions across the environment: a weird download here, a strange process there. On their own, nothing screams “breach.” But when you stitch the pieces together, you see the bigger picture, a calendar app that’s not really about calendars at all.

#### What We Can Learn

- Malware loves disguises. Sometimes it’s a PDF, sometimes it’s a fake installer, sometimes it’s a calendar app with a suspiciously dramatic name.
- Signatures aren’t enough. A revoked certificate can still fool users and even some security tools.
- Events matter in context. A single cmd.exe pop-up isn’t scary. But if it’s spawned by a “calendar” app that just hit a shady domain registered in Iceland? Yeah, that’s not normal.

#### Final Thoughts

Calendaromatic isn’t world-ending ransomware, but it’s a great reminder of how attackers blend creativity with persistence. It’s also a fun case study in why security folks can’t just rely on signatures or simple red-flag alerts. The next time you download a “handy little app” from a random website, remember: it might not just be tracking your deadlines, it might be stealing your tokens, too. For more information, check this link Artificial Intelligence Exposes the Homoglyph Hustle
