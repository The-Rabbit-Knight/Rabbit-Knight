---
title: "How to Extract VirusTotal digest for a List of File Hashes Using AI"
date: 2025-11-26
medium_url: https://medium.com/@rabbit_knight/how-to-extract-virustotal-digest-for-a-list-of-file-hashes-using-ai-b9d167a62da8
---

> **Original Published Article on Medium:** [How to Extract VirusTotal digest for a List of File Hashes Using AI](https://medium.com/@rabbit_knight/how-to-extract-virustotal-digest-for-a-list-of-file-hashes-using-ai-b9d167a62da8)

### How to Extract VirusTotal digest for a List of File Hashes Using AI

![Article Figure](https://cdn-images-1.medium.com/max/800/1*iGTcZCrMm3xn55UGGc1qfQ.jpeg)

Today I want to share a very short article about something practical. I was recently tasked with checking a long list of file hashes in VirusTotal to see whether any of them were malicious. Doing this manually would take forever, so it became obvious that the process needed automation. Using AI and the VirusTotal API documentation , I created a script that converts a plain text list of hashes into structured JSON or CSV with VirusTotal results. It takes just a few steps.

#### Step 1: You need a VirusTotal account and an API key

Before touching any code, you need:

- A free VirusTotal account (takes 30 seconds to create).
- Your public API key, which you can find under your account settings.
- Keep in mind: Never hardcode your API key; store it in an environment variable instead.

#### Step 2: Start With a Simple Text List of Hashes

Your starting point might look something like this:

```
sha#1
sha#2
sha#3
```

It could be 3 hashes or 300 — you don’t need to format anything.

#### Step 3: Tell the AI What Structure You Want

For example, you can say:

Write me a Python script using only standard HTTP libraries and the VirusTotal v3 API that does the following:

- Reads a newline-separated list of hashes from STDIN
- Uses an API key stored in an environment variable
- Calls https://www.virustotal.com/api/v3/files/{id} for each hash
- Extracts: last_analysis_stats.malicious, last_analysis_stats.suspicious
- A list of providers that flagged the file as suspicious or malicious
- Uses the original hash as the dictionary key
- Serializes everything into JSON and pretty-prints it to STDOUT.

#### Step 3: Validate the Results

Never trust AI output blindly, especially when dealing with APIs or security data.

#### Results:

Python Script (Standard Library Only)

```
#!/usr/bin/env python3
import
 sys
import
 os
import
 json
import
 urllib.request
API_KEY = os.getenv(
"VT_API_KEY"
)
if
 
not
 API_KEY:
    
print
(
"Error: VT_API_KEY environment variable not set"
, file=sys.stderr)
    sys.exit(
1
)
BASE_URL = 
"https://www.virustotal.com/api/v3/files/"
def
 
vt_request
(
file_hash
):
    url = BASE_URL + file_hash.strip()
    req = urllib.request.Request(url)
    req.add_header(
"x-apikey"
, API_KEY)
    
try
:
        
with
 urllib.request.urlopen(req) 
as
 resp:
            data = resp.read().decode(
"utf-8"
)
            
return
 json.loads(data)
    
except
 urllib.error.HTTPError 
as
 e:
        
return
 {
"error"
: 
f"HTTP 
{e.code}
: 
{e.reason}
"
}
    
except
 Exception 
as
 e:
        
return
 {
"error"
: 
str
(e)}
def
 
extract_info
(
json_obj
):
    
if
 
"error"
 
in
 json_obj:
        
return
 {
"malicious"
: 
None
, 
"suspicious"
: 
None
, 
"providers"
: [], 
"error"
: json_obj[
"error"
]}
    stats = json_obj[
"data"
][
"attributes"
].get(
"last_analysis_stats"
, {})
    results = json_obj[
"data"
][
"attributes"
].get(
"last_analysis_results"
, {})
    providers = []
    
for
 provider, analysis 
in
 results.items():
        category = analysis.get(
"category"
)
        
if
 category 
in
 (
"malicious"
, 
"suspicious"
):
            providers.append(provider)
    
return
 {
        
"malicious"
: stats.get(
"malicious"
, 
0
),
        
"suspicious"
: stats.get(
"suspicious"
, 
0
),
        
"providers"
: providers
    }
def
 
main
():
    hashes = [line.strip() 
for
 line 
in
 sys.stdin 
if
 line.strip()]
    output = {}
    
for
 h 
in
 hashes:
        vt_json = vt_request(h)
        output[h] = extract_info(vt_json)
    
print
(json.dumps(output, indent=
4
))
if
 __name__ == 
"__main__"
:
    main()
```

How to Use

1. Set your API key:

```
export
 VT_API_KEY=
"your_api_key_here"
```

2. Run the script with a list of hashes:

```
chmod
 ugo+x ~/scriptname.py
```

```
cat
 hashes.txt | ~/scriptname.py > result.json
```

Python Script With CSV Output (Standard Library Only)

```
#!/usr/bin/env python3
import
 sys
import
 os
import
 json
import
 csv
import
 urllib.request
API_KEY = os.getenv(
"VT_API_KEY"
)
if
 
not
 API_KEY:
    
print
(
"Error: VT_API_KEY environment variable not set"
, file=sys.stderr)
    sys.exit(
1
)
BASE_URL = 
"https://www.virustotal.com/api/v3/files/"
def
 
vt_request
(
file_hash
):
    url = BASE_URL + file_hash.strip()
    req = urllib.request.Request(url)
    req.add_header(
"x-apikey"
, API_KEY)
    
try
:
        
with
 urllib.request.urlopen(req) 
as
 resp:
            
return
 json.loads(resp.read().decode(
"utf-8"
))
    
except
 urllib.error.HTTPError 
as
 e:
        
return
 {
"error"
: 
f"HTTP 
{e.code}
: 
{e.reason}
"
}
    
except
 Exception 
as
 e:
        
return
 {
"error"
: 
str
(e)}
def
 
extract_info
(
json_obj
):
    
if
 
"error"
 
in
 json_obj:
        
return
 {
"malicious"
: 
""
, 
"suspicious"
: 
""
, 
"providers"
: 
""
, 
"error"
: json_obj[
"error"
]}
    stats = json_obj[
"data"
][
"attributes"
].get(
"last_analysis_stats"
, {})
    results = json_obj[
"data"
][
"attributes"
].get(
"last_analysis_results"
, {})
    providers = [
        provider
        
for
 provider, analysis 
in
 results.items()
        
if
 analysis.get(
"category"
) 
in
 (
"malicious"
, 
"suspicious"
)
    ]
    
return
 {
        
"malicious"
: stats.get(
"malicious"
, 
0
),
        
"suspicious"
: stats.get(
"suspicious"
, 
0
),
        
"providers"
: 
";"
.join(providers),
        
"error"
: 
""
    }
def
 
main
():
    hashes = [line.strip() 
for
 line 
in
 sys.stdin 
if
 line.strip()]
    
with
 
open
(
"results.csv"
, 
"w"
, newline=
""
, encoding=
"utf-8"
) 
as
 csvfile:
        writer = csv.writer(csvfile)
        writer.writerow([
"hash"
, 
"malicious"
, 
"suspicious"
, 
"providers"
, 
"error"
])
        
for
 h 
in
 hashes:
            data = vt_request(h)
            info = extract_info(data)
            writer.writerow([h, info[
"malicious"
], info[
"suspicious"
], info[
"providers"
], info[
"error"
]])
    
print
(
"Results saved to results.csv"
)
if
 __name__ == 
"__main__"
:
    main()
```

Enjoy the automation!
