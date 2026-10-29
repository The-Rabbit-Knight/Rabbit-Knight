---
title: "Hunting HTML Smuggling with Velociraptor, Part 2: The Hunt, Forensic Investigation with…"
date: 2026-06-27
medium_url: https://medium.com/@rabbit_knight/hunting-html-smuggling-with-velociraptor-part-2-the-hunt-forensic-investigation-with-56aed2399392
---

> **Original Published Article on Medium:** [Hunting HTML Smuggling with Velociraptor, Part 2: The Hunt, Forensic Investigation with…](https://medium.com/@rabbit_knight/hunting-html-smuggling-with-velociraptor-part-2-the-hunt-forensic-investigation-with-56aed2399392)

### Hunting HTML Smuggling with Velociraptor, Part 2: The Hunt, Forensic Investigation with Velociraptor VQL

![Article Figure](https://cdn-images-1.medium.com/max/800/1*5BO6NxUiSHITSYJiV9ZIQQ.png)

In Part 1: Hunting HTML Smuggling with Velociraptor: Setting the Stage , I built the lab, wrote an HTML smuggling attack that delivers an SVG phishing payload, and detonated it against a Windows workstation exactly the way a real victim would experience it. I closed by answering the most basic question there is, in two different ways: is there an SVG sitting in a Downloads folder that nobody downloaded? Both a raw Generic.Client.VQL query and the prebuilt Windows.Search.FileFinder artifact returned the same file, cat_document.svg, created inside the simulation window.

Finding that file was the taste. This is the main course. Confirming that something landed on disk is not the same as proving what happened, and that gap is what this part closes. Across the hunt steps below I pull on that single thread through the rest of the forensic trail, the Mark of the Web tag, the browser’s own databases, the NTFS journal, and the Windows event log, until I can reconstruct the whole attack from the moment the victim opened the HTML to the final click.

If you are arriving here without reading Part 1, here is the short version of what I am hunting. HTML smuggling (MITRE T1027.006) delivers a harmless-looking HTML file by email. When the victim opens it, JavaScript inside the page assembles a malicious payload in browser memory and forces it to download, so nothing malicious ever crosses the network as a file. In this case the payload is an SVG that renders as a blank page but is one giant invisible clickable link, and clicking it redirects the victim to a credential-harvesting page.

#### Mapping the Forensic Chain Before I Touch a Query

Every stage of the attack writes a different record in a different place. Knowing that map in advance tells me exactly where to look, and just as usefully, where not to waste time.

```
| Stage | What happens                     | Forensic artifact                   | Where it lives                  |
|-------|----------------------------------|-------------------------------------|---------------------------------|
| 1     | Victim opens the HTML file       | URL visit record                    | Browser history SQLite          |              
| 2     | JavaScript fires, Blob assembled | Nothing on disk                     | Browser memory only             |
| 3     | SVG written to disk              | File plus metadata                  | Downloads folder                |
| 3     | SVG download recorded            | Download row with `file:///` source | Browser downloads SQLite        |
| 3     | Filesystem records the write     | USN journal FILE_CREATE             | NTFS change journal             |
| 4     | Mark of the Web written          | Zone.Identifier stream              | Alternate data stream on the SVG|
| 5     | Victim opens the SVG             | Process or navigation record        | Sysmon and browser history      |
| 6     | Victim clicks the invisible link | Navigation to phishing URL          | Browser history SQLite          |
```

The single most important line in that table is stage 2, because it has no disk artifact at all. The payload assembly happens entirely in browser memory. That is not an oversight on my part, it is the entire reason attackers prefer this technique over simply attaching the SVG to the email. The first moment anything is written to disk is stage 3, when the synthetic download fires. That is why the browser download database and the USN journal are my earliest reliable evidence, they both capture the instant the payload first touched the disk.

Ways to Run a Query

Before the hunt steps, a practical note that saves a lot of confusion, because Velociraptor gives you more than one place to run VQL and they do not all do the same thing.

Named artifacts are the structured path. Click the client, open the Collected Artifacts tab, start a New Collection, search for a built-in artifact by name such as Windows.Forensics.Usn, fill in its parameters, and launch. The results come back in the collection’s Results tab. This is how I run the packaged forensic artifacts.

Generic.Client.VQL is the ad-hoc path. When I want to run raw VQL that is not wrapped in a named artifact, I create a New Collection, search for Generic.Client.VQL, paste my query into the parameter box, and launch. This runs my query live against the endpoint. Most of the custom queries below go through this.

The trap is the Notebook. Velociraptor has a Notebook tab, and it looks like a query console, so the instinct is to run live queries there. It does not query the live endpoint. The Notebook queries data that has already been collected and stored on the server. Run a glob() against the filesystem in a Notebook expecting live results and you get nothing, with no error explaining why.

#### Hunt Step 1: Find the Payload on Disk

The first question is the simplest. Is there an SVG in any user’s Downloads folder, and what is it?

I can answer this two ways. The point-and-click way is the built-in Windows.Search.FileFinder artifact: New Collection, search for it, set SearchFilesGlob to C:\Users\*\Downloads\*.svg, leave the upload option unchecked because I only want metadata, and launch. No VQL required, which is genuinely useful when you want a fast answer or you are handing the process to someone less comfortable with the query language.

You will notice the backslashes there, while the VQL below uses forward slashes. Both are deliberate. The GUI parameter box takes the path literally, so native Windows backslashes are fine and you can paste a path straight out of Explorer. VQL source is different: inside a quoted string a backslash is an escape character, so paths there use forward slashes, which Velociraptor accepts on Windows. I unpacked the reasoning in Part 1.

When I want hashes alongside the metadata, I drop to Generic.Client.VQL:

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

![Article Figure](https://cdn-images-1.medium.com/max/800/1*EKEP1kLKye_qFJTHO4gddA.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*P9IayfgUBvRakUs1U9vvgA.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*wX79FKvuUCXuWp0JRZDqPA.png)

glob() walks the filesystem matching the path pattern, and the * in the user position means it checks every profile, not just the one I happen to suspect. hash() returns MD5, SHA1, and SHA256 in one shot, which become the indicators of compromise I would put in a report.

The result is one row: cat_document.svg, 219 bytes, with a creation timestamp inside my simulation window. I want to be careful about what that does and does not tell me. The size on its own is not suspicious. Plenty of legitimate SVGs, icons, logos, UI glyphs, are only a few hundred bytes or less.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*VLGSmQtWBTK9D9uv1jF1pA.png)

#### Hunt Step 2: Read the Mark of the Web

Now the question that reveals how the attack beat the defenses. When Windows downloads a file from the internet, NTFS attaches a hidden Zone.Identifier alternate data stream recording where it came from. That stream is what drives the “are you sure you want to open this?” warnings. Reading it tells me whether the smuggling evaded that whole mechanism.

In this case, it is raw VQL run exactly the way I ran the query in Part 1: New Collection, search for Generic.Client.VQL, drop the query into its parameter box, review and launch.

```
LET svgs 
=
 
SELECT
 FullPath
           
FROM
 glob(globs
=
"C:/Users/*/Downloads/*.svg")
SELECT
 FullPath,
       read_file(filename
=
FullPath 
+
 ":Zone.Identifier", length
=
500
) 
AS
 ZoneIdentifier
FROM
 svgs
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*KtoeGieEV2zo9au2nNfVlw.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*nrc4qsFCUFTw0BqS3NmawA.png)

The Zone.Identifier suffix tells read_file() to read the named alternate data stream attached to the file rather than the file’s own contents. A normal download fetched directly from a website would produce something like this:

```
[ZoneTransfer]
ZoneId=3
ReferrerUrl=https://some-website.com/page
HostUrl=https://some-website.com/file.svg
```

What my lab actually returned was different, and the difference is the whole point:

```
[ZoneTransfer]
ZoneId=4
HostUrl=file:///
```

ZoneId=4 is Restricted Sites, the most restrictive zone Windows has. It assigned that because the payload was generated by JavaScript in a local-file context rather than fetched from a web server. `HostUrl=file:///` confirms it: there was no remote host, the file was assembled locally on the victim’s own machine.

I want to be precise about which part of this is the reliable signature, because it is not the ZoneId number. If the attacker had hosted the HTML on a real web server and the victim had browsed to it, the ZoneId could easily have been 3, or missing entirely on a blob-sourced download. The ZoneId varies with the delivery method. What does not vary in the same way is the HostUrl. A genuine internet download normally shows an https:// host. Seeing HostUrl=file:/// or HostUrl=blob:null/… on a file in the Downloads folder is the immediate tell that something generated this file locally instead of downloading it across the network. That is the forensic fingerprint of HTML smuggling, independent of which zone number shows up.

It also explains why the victim got no warning. The Mark of the Web machinery that throws security prompts keys off the zone. Content that looks locally created, with a file:/// host, is treated as local and trusted, so no prompt appears. The victim opened the SVG with nothing standing in their way, precisely as the attacker intended.

#### Hunt Step 3: The Browser’s Own Download Record

The browser keeps its own record of every download it has ever performed, in a SQLite database inside the user profile, and that record includes downloads triggered by JavaScript rather than a human click. This evidence exists completely independently of the Zone.Identifier. Even if the Mark of the Web stream had never been written, this row would still be here.

This is raw VQL again, so it goes through Generic.Client.VQL like the last one: New Collection, Generic.Client.VQL, paste, review, launch.

```
SELECT
 
*
 
FROM
 foreach(
  
row
=
{
    
SELECT
 FullPath
    
FROM
 glob(globs
=
[
      "C:/Users/*/AppData/Local/Google/Chrome/User Data/*/History",
      "C:/Users/*/AppData/Local/Microsoft/Edge/User Data/*/History"])
  },
  query
=
{
    
SELECT
 FullPath 
AS
 DatabasePath,
           
current_path
 
AS
 SavedTo,
           target_path 
AS
 IntendedPath,
           referrer 
AS
 ReferrerURL,
           tab_url 
AS
 TabURL
    
FROM
 sqlite(
      file
=
FullPath,
      query
=
"SELECT current_path, target_path, referrer, tab_url
             FROM downloads
             WHERE target_path LIKE '%.svg'
             ORDER BY rowid DESC
             LIMIT 20")
  })
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*GDUfMxDBwyiaqDoOkTZIEQ.png)

foreach() is doing the heavy lifting. The outer row query finds every browser history database across every user profile, and for each one the inner query runs standard SQLite against the browser’s actual downloads table. This pattern, iterating an inner query over the results of an outer one, is something I lean on constantly in VQL, because it lets one query cover an unknown number of profiles without me hardcoding paths.

What came back was the connective tissue of the whole case:

![Article Figure](https://cdn-images-1.medium.com/max/800/1*65J7nxa235i6gnhxmDAHfA.png)

The TabURL is the field that matters. It records the page that was open in the tab when the download fired, and here it is a file:/// path pointing straight back at cat_document.html, the smuggler. The browser is telling me, in its own bookkeeping, that the SVG was created by a local HTML file. The empty referrer reinforces it, there was no web server in the chain to act as a referrer. This is textbook T1027.006.

A note for production use. Chrome and Edge store their timestamps as microseconds since January 1, 1601, which is not a Unix timestamp and not human-readable on its own. The Generic.Forensic.SQLiteHunter artifact handles that conversion for you, and it also copies locked databases before reading them. I show the raw query here because it is worth understanding what the convenient artifact is doing underneath, but in a real engagement SQLiteHunter is the faster path.

Two things worth knowing before you lean on it in an engagement. First, SQLiteHunter is not part of the default artifact set, so you import it once through the Server.Import.Extras server artifact (or upload its pack manually under View Artifacts), after which it behaves like any other artifact. Second, it is designed to triage every SQLite store on the host in a single pass, so instead of returning one table it locates each database, fingerprints it, parses it with a purpose-built rule, and writes the output into its own source per application. That is why the view you land on first is just the file inventory, the list of databases it identified. The parsed records live in those per-application sources: open the collection’s Notebook, scroll to the source you need, the browser download history sits under Chromium Browser History_Downloads for example, select it and Recalculate to render the table with its timestamps already converted to readable UTC. The same collection also gives you visit history, cookies, autofill, and a long list of other stores, each in its own source, which is what makes it the efficient choice once you understand its layout.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*XJMnZbNrPIUaIjdnRnZXvw.png)

#### Hunt Step 4: The Filesystem Never Lies

The USN journal is my favorite source in this entire hunt. It is a change log that NTFS maintains at the filesystem level, below any application. Every time a file is created, written, renamed, or deleted, the filesystem writes a record: the filename, the type of change, and the timestamp. It records the SVG’s creation independently of the browser, independently of the Zone.Identifier, and independently of any log an attacker might think to clear. A record of a file’s creation survives in the journal even after the file itself is deleted. That is what makes it one of the most tamper-resistant sources available to me, with one honest caveat: the journal is a rolling buffer, typically around 30 MB, so older records age out as new filesystem activity overwrites them. On a busy machine it may only reach back hours or days, and an attacker with the right privileges can delete it outright with fsutil usn deletejournal. It is resilient, not permanent. For a recent event like this one, it is exactly the corroboration I want.

I run it through the built-in artifact: New Collection, Windows.Forensics.Usn, set FileNameRegex to cat_document\.svg, and launch.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*otrGOKE6fKz_HU4ckZIYYw.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*8i46xEJ_07HygKCdV6PdKg.png)

One thing about that path column. Most filesystem sources in Velociraptor return the path in a column called FullPath, but Windows.Forensics.Usn is the exception, it returns OSPath instead. It is the kind of detail that costs you ten minutes the first time you hit it, when a query that should obviously work comes back with a blank path column. FileNameRegex is a genuine parameter of the artifact, and the \\. escapes the dot so it matches a literal period rather than any character. I also deliberately leave the Reason field unfiltered, because I want the complete sequence rather than just the creation event.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*dVMF_YsUwNK-hEIkMHzKaQ.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*fcs9hqdz8T29ke3TFWOBug.png)

What comes back is a run of records for a single small file, and the reason codes are where the story lives. The first thing to notice is what is absent: there is no FILE_CREATE. That absence is itself the finding.

The earliest record for cat_document.svg is RENAME_NEW_NAME, not a create. That is the fingerprint of a browser download. The browser streams the incoming bytes into a temporary working file, a .crdownload or .tmp name, and only renames it to the final name once the transfer completes. The create and the bulk content writes therefore belong to that temporary name; under the final name cat_document.svg, the file’s life begins with the rename. If I specifically wanted that original FILE_CREATE, I would widen FileNameRegex to catch the temporary name. The rename is still a perfectly good anchor.

The next cluster is the one I care about most: STREAM_CHANGE followed by NAMED_DATA_EXTEND, which means bytes were written to a named alternate data stream rather than to the file’s main content, and STREAM_CHANGE marks that stream being added. This is the Zone.Identifier, the Mark of the Web I read by hand in Step 2, being created and populated, recorded here at the filesystem level. The USN journal is independently witnessing the exact instant the MotW tag was attached, the same evidence from a second, lower source.

OBJECT_ID_CHANGE then records NTFS assigning the file an object ID, which Windows uses for distributed link tracking once a file is interacted with. The CLOSE flags threaded through the sequence each mark a file handle being released after an operation finishes.

Every one of these records exists independently of the browser and of the Zone.Identifier stream itself. Even if the file were deleted and the browser history wiped, this lifecycle, the rename into place, the Mark of the Web write, the object-id assignment would still be sitting in the journal.

#### Hunt Step 5: Sysmon, and the Value of an Unexpected Answer

Every time a user double-clicks a file, Windows Explorer launches the registered handler and Sysmon records it as Event ID 1, capturing the command line, the parent process, the user, the binary hash, and the timestamp. I went looking for the moment the victim opened the SVG.

This query reaches into the Sysmon log through the EvtxHunter sub-artifact, which is why it has to run via Generic.Client.VQL rather than being pasted into EvtxHunter’s own GUI fields:

```
SELECT
 EventTime,
       EventData.Image 
AS
 ExecutableLaunched,
       EventData.CommandLine 
AS
 CommandLine,
       EventData.ParentImage 
AS
 LaunchedBy,
       EventData.ParentCommandLine 
AS
 ParentCommandLine,
       EventData.User 
AS
 
User
,
       EventData.Hashes 
AS
 BinaryHashes
FROM
 Artifact.Windows.EventLogs.EvtxHunter(
  EvtxGlob
=
"C:/Windows/System32/winevt/Logs/Microsoft-Windows-Sysmon%4Operational.evtx",
  IdRegex
=
"^1$")
WHERE
 CommandLine 
=
~
 "(?i)(\\.svg|\\.html)"
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*ZP1j5UFLWvIR8HuN2T5V-A.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*5pioi25qt5oOyanaUdU5Zw.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*VxHBsDxk-0DQJkT04OAlKg.png)

This is not what I went looking for. I wanted the SVG being opened, and the query returned the HTML file instead. For a second that looks like the query missed. It did not, and understanding why is more valuable than the answer I originally wanted.

What I actually caught is Patient Zero, the exact moment the initial phishing file was executed. Read the fields. LaunchedBy: explorer.exe tells me a human being double-clicked this. If the parent had been cmd.exe, powershell.exe, or wscript.exe, I would be looking at scripted or automated execution, a far more serious finding. Explorer as the parent means a real user performed the action. The “single-argument” flag is how Edge opens a local file handed to it by Explorer, which pins down the exact file that was the initial vector. And the timestamp, 13:43:51, is the true start of the human side of this incident.

So why did the SVG itself not generate its own Event ID 1? Because when Edge opens a local SVG by following a file:// link inside a page it is already rendering, it does that as an in-process navigation. No new process is spawned, so Sysmon has nothing to log. The evidence of the SVG being opened lives in the browser history, not in Sysmon.

#### Hunt Step 6: Browser History and the Victim’s Timeline

The browser history database records every URL navigated to, and crucially that includes file:// URLs for local files opened in the browser. This is the single best source for reconstructing what the victim did, in order, after opening the phishing file. Same locking caveat as Step 3, so I close the browser first, and the same delivery, Generic.Client.VQL with the query pasted into its parameter box.

```
SELECT
 
*
 
FROM
 foreach(
  
row
=
{
    
SELECT
 FullPath
    
FROM
 glob(globs
=
[
      "C:/Users/*/AppData/Local/Google/Chrome/User Data/*/History",
      "C:/Users/*/AppData/Local/Microsoft/Edge/User Data/*/History"])
  },
  query
=
{
    
SELECT
 
timestamp
(winfiletime
=
visit_time 
*
 
10
) 
AS
 VisitTime,
           url 
AS
 URL,
           title 
AS
 PageTitle,
           visit_count 
AS
 TimesVisited
    
FROM
 sqlite(
      file
=
FullPath,
      query
=
"SELECT u.url, u.title, u.visit_count, v.visit_time
             FROM urls u
             JOIN visits v ON u.id = v.url
             ORDER BY v.visit_time DESC
             LIMIT 100")
    
WHERE
 URL 
=
~
 "(?i)cat_document"
  })
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*738FXXHoy3oxTGGAkfJMrw.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*fWw--4wVft76ceO8nSr94w.png)

Read from the bottom up, this is not one attack but two passes through the same chain, and the first pass is the one that ties the whole case together.

At 13:43:51 the victim opened cat_document.html. That timestamp is not new to me: Sysmon recorded Explorer launching Edge against that exact file at 13:43:51 in Step 5, and the USN journal in Step 4 shows cat_document.svg being renamed into place at 13:43:51.887 with the Zone.Identifier written milliseconds later. Three independent sources, the event log, the NTFS journal, and the browser’s own history, agree on the same second. That is the kind of corroboration that turns a plausible story into an evidenced one. At 13:44:22 the victim opened the dropped SVG for the first time.

Then, at 13:49:40, the chain runs again: cat_document.html is opened a second time. The USN journal already told me this would be here, that was the second instance of the file, the one with the fresh MFT record. Those repeat opens you see are not noise, they are the documented behaviour of a confused victim.

This is where the earlier timestamp question resolves itself. When Step 5 showed Sysmon at 13:43:51 and an analyst might have expected the browser history to start later, the answer was not a clock skew or a margin of error, it was simply that the simulation was run twice. The browser history captures both passes, the USN journal captures both file instances, and Sysmon captured the first launch. Reconciling forensic sources is rarely about forcing clocks to agree; it is about finding the sequence of events that explains every record at once.

#### Packaging It Into a Reusable Hunt

Running six steps by hand is how you investigate one machine. The next move, the one that separates a one-off investigation from an operational capability, is to package the whole thing into a single artifact I can deploy across every endpoint at once. The first run is exploratory and manual. Every run after that should be automated.

The artifact below is a deliberately streamlined, triage-focused version. It runs four high-fidelity queries in parallel, each returning as its own result tab, and skips the slow full-disk sweeps so it returns answers in seconds without hammering the endpoint. To install it, go to View Artifacts in the GUI, click Add an Artifact, paste the YAML, and save.

```
name:
 
Custom.Hunt.HTMLSmugglingSVG.Streamlined
description:
 
|
  Hunts for forensic evidence of HTML Smuggling attacks delivering SVG
  phishing payloads. Streamlined to four high-fidelity, immediate sources:
    
1
.
 
SVGFiles
         
-
 
SVG
 
files
 
in
 
Downloads
 
with
 
Zone.Identifier
 
ADS
    
2
.
 
BrowserDownloads
 
-
 
Browser
 
SQLite
 
records
 
of
 
local
 
SVG
 
creation
    
3
.
 
SysmonHTMLSVG
    
-
 
Sysmon
 
Event
 
ID
 
1
 
for
 
HTML
 
or
 
SVG
 
execution
    
4
.
 
USNJournal
       
-
 
NTFS
 
change
 
journal
 
proving
 
the
 
physical
 
disk
 
write
  
MITRE
 
ATT&CK:
 
T1027.006
 
(HTML
 
Smuggling),
 
T1566.001
 
(Phishing
 
Attachment)
type:
 
CLIENT
parameters:
  
-
 
name:
 
DateAfter
    
description:
 
Only
 
return
 
artifacts
 
created
 
after
 
this
 
time
 
(YYYY-MM-DD
 
HH:MM:SS)
    
default:
 
"2020-01-01 00:00:00"
  
-
 
name:
 
UserGlob
    
description:
 
Glob
 
pattern
 
covering
 
user
 
home
 
directories
    
default:
 
"C:/Users/*"
sources:
  
-
 
name:
 
SVGFiles
    
description:
 
SVG
 
files
 
in
 
Downloads
 
with
 
hashes
 
and
 
Zone.Identifier
 
ADS
    
query:
 
|
      SELECT FullPath,
             Size,
             Ctime AS Created,
             Mtime AS Modified,
             hash(path=FullPath) AS Hashes,
             read_file(filename=FullPath + ":Zone.Identifier",
                       length=500) AS ZoneIdentifier
      FROM glob(globs=UserGlob + "/Downloads/*.svg")
      WHERE Ctime > timestamp(string=DateAfter)
  
-
 
name:
 
BrowserDownloads
    
description:
 
Chrome
 
and
 
Edge
 
download
 
records
 
for
 
SVG
 
files
 
(close
 
browser
 
first)
    
query:
 
|
      SELECT FullPath AS DatabasePath,
             current_path AS SavedTo,
             target_path AS IntendedPath,
             referrer AS ReferrerURL,
             tab_url AS TabURL
      FROM foreach(
        row={
          SELECT FullPath FROM glob(globs=[
            UserGlob + "/AppData/Local/Google/Chrome/User Data/*/History",
            UserGlob + "/AppData/Local/Microsoft/Edge/User Data/*/History"])
        },
        query={
          SELECT FullPath, current_path, target_path, referrer, tab_url
          FROM sqlite(
            file=FullPath,
            query="SELECT current_path, target_path, referrer, tab_url
                   FROM downloads
                   WHERE target_path LIKE '%.svg'
                   ORDER BY rowid DESC
                   LIMIT 50")
        })
  
-
 
name:
 
SysmonHTMLSVG
    
description:
 
|
      Sysmon Event ID 1 for any process launch involving an HTML or SVG file.
      Captures both Patient Zero (the initial phishing file) and any later
      SVG handler launches. Requires Sysmon on the endpoint.
    
query:
 
|
      SELECT EventTime,
             EventData.Image AS ExecutableLaunched,
             EventData.CommandLine AS CommandLine,
             EventData.ParentImage AS LaunchedBy,
             EventData.ParentCommandLine AS ParentCommandLine,
             EventData.User AS User,
             EventData.Hashes AS BinaryHashes
      FROM Artifact.Windows.EventLogs.EvtxHunter(
        EvtxGlob="C:/Windows/System32/winevt/Logs/Microsoft-Windows-Sysmon%4Operational.evtx",
        IdRegex="^1$",
        DateAfter=DateAfter)
      WHERE CommandLine =~ "(?i)(\\.svg|\\.html)"
  
-
 
name:
 
USNJournal
    
description:
 
NTFS
 
USN
 
Journal
 
entries
 
for
 
SVG
 
files
 
(all
 
reasons,
 
full
 
lifecycle)
    
query:
 
|
      SELECT TimeStamp,
             FileName,
             OSPath,
             Reason
      FROM Artifact.Windows.Forensics.Usn(FileNameRegex="\\.svg$")
      WHERE TimeStamp > timestamp(string=DateAfter)
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*CRDtUJUOA6wwnPOTsHiKtg.png)

I left the USNJournal source unfiltered by Reason on purpose, so it returns the full FILE_CREATE through DATA_EXTEND through CLOSE lifecycle described in Step 4 rather than just the creation event.

To run it against one machine, open the client, start a New Collection, search for Custom.Hunt.HTMLSmugglingSVG.Streamlined, set DateAfter to just before the suspected window, and launch. All four sources run in parallel and each lands in its own results tab.

To run it across the whole environment, go to Hunt Manager, create a New Hunt, select the artifact, tag it with something like html-smuggling, choose which clients or labels to target, and launch. Velociraptor pushes the collection to every matching endpoint at once and aggregates the results. The same six-step investigation that took me a focused session by hand now runs against an entire fleet in a single operation.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Lh3PFmzz27T_xhvt5YhQbw.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*v-tqPN_jnKgTwEZ9WvLO6w.png)

#### Pulling the sample for analysis

Everything to this point has been metadata. When you want the file itself, to hash it against threat intel, detonate it, or read the link embedded in it, Velociraptor retrieves it for you. Browse the client’s Virtual File System to the user’s Downloads folder, select cat_document.svg, and download it; Velociraptor collects the bytes off the endpoint to the server, where you can pull them down. The same is available non-interactively through Windows.Search.FileFinder with Upload_Files enabled, which returns the file inside the collection.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*ArWp0gsyf0aaSmLBNs5OSw.png)

#### What This Attack Beats, and Where It Still Bleeds Evidence

The evasion was covered as the attack unfolded, so the short version: nothing malicious crosses the wire, the Mark of the Web that SmartScreen and many EDRs depend on is never set to the internet zone, and an SVG walks past file-type blocks built for executables. Quiet on the network by design.

It is loud on the endpoint, though, and that is the whole point of the hunt. The browser download record, the USN journal, Sysmon, and the browser history each captured a piece independently, and several outlive file deletion or a cleared history. For detection, the highest-fidelity signal is the pairing this hunt surfaced: a small SVG in a Downloads folder whose Zone.Identifier is missing or carries a file:/// or blob: host, alongside a matching browser download record. That combination is rare in normal use. Velociraptor’s real-time watches can flag it the moment the file lands instead of waiting for a scan, and on endpoints that have no reason to open SVGs in a browser, blocking that file association kills the second stage outright.

If you have not read it, Part 1 covers the lab build, the attack itself, and the simulation that produced every artifact I hunted here. And if the SIEM side interests you, my Wazuh deployment and detection-engineering write-up covers the other half of this lab.
