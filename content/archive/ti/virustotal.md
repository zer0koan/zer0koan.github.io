+++
title = "Virus Total"
date = 2026-04-16
description = "Notes on integrating VT data into TI pipeline"

[taxonomies]
categories = ["Threat Intelligence", "Red Team"]
+++
## VirusTotal
[API documentation](https://docs.virustotal.com/docs/api-overview)

Official API v3 client libraries:
- [Go](https://github.com/VirusTotal/vt-go)
- [Python](https://github.com/VirusTotal/vt-py)

Unofficial API v2 client libraries:
- [Go](https://github.com/dutchcoders/go-virustotal)

### Public API
**base url:** `http://www.virustotal.com/vtapi/v2/`

**base url:** `http://www.virustotal.com/api/v3/`

500 requests/day at 4 requests/minute

Must not be used for commercial products and services, or business workflows which do not contribute new files

#### IPs
```
https://www.virustotal.com/api/v3/ip_addresses/{ip}
https://www.virustotal.com/api/v3/ip_addresses/{ip}/comments
https://www.virustotal.com/api/v3/ip_addresses/{ip}/{relationship}
```
Example:
```
curl --request GET \
     --url https://www.virustotal.com/api/v3/ip_addresses/23.1.52.26 \
     --header 'accept: application/json' \
     --header 'x-apikey: [api_key]'
```

#### Domains
```
https://www.virustotal.com/api/v3/domains/{domain}
https://www.virustotal.com/api/v3/domains/{domain}/comments
https://www.virustotal.com/api/v3/domains/{domain}/{relationship}
https://www.virustotal.com/api/v3/domains/{domain}/relationships/{relationship}
https://www.virustotal.com/api/v3/resolutions/{id}
```

### Premium API
#### VT Hunting
- Uses YARA to search VT's dataset using three components:
    1. Livehunt
    2. Retrohunt
    3. VTDIFF

##### Livehunt
Compares files submitted to VT with YARA rules in real time
- Stream of malware files classified by family
- Discover new malware
- Filter by given language, specific run-time packer
- Heuristic rules to detect suspicious files
- Track threat actors

##### Retrohunt
Compare historical files with YARA rules, which can take up to 4 hours

##### VTDIFF
Provide a collection of hashes to track and avoid, to create YARA rules with common binary subsequences

