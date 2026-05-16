# Bug Bounty Toolkit — Termux Installation Guide

Tested on Android with Termux. Run all commands in Termux.

---

## Initial Setup

```bash
pkg update && pkg upgrade -y
pkg install git curl wget python python-pip golang -y
```

Add Go binaries to PATH permanently:

```bash
echo 'export PATH=$PATH:/data/data/com.termux/files/home/go/bin' >> ~/.bashrc
source ~/.bashrc
```

---

## Subfinder
Subdomain enumeration tool. Discovers subdomains of a target domain using passive sources.

**Install:**
```bash
go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
```

**Verify:**
```bash
subfinder -version
```

**Basic usage:**
```bash
subfinder -d target.com -o subdomains.txt
```

---

## httpx
HTTP probe and fingerprinting tool. Identifies live hosts, status codes, technologies, and titles from a list of URLs or subdomains.

**Install:**
```bash
go install github.com/projectdiscovery/httpx/cmd/httpx@latest
```

**Verify:**
```bash
httpx -version
```

**Basic usage:**
```bash
cat subdomains.txt | httpx -status-code -title -tech-detect -o live.txt
```

---

## Nuclei
Template-based vulnerability scanner. Runs a library of community-maintained templates against targets to detect common vulnerabilities and misconfigurations.

**Install:**
```bash
go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
```

**Update templates after install (required):**
```bash
nuclei -update-templates
```

**Verify:**
```bash
nuclei -version
```

**Basic usage:**
```bash
# Unauthenticated scan
nuclei -u https://target.com -t xss,ssrf,misconfigurations

# Authenticated scan with cookie
nuclei -u https://target.com \
  -H "Cookie: session=YOUR_TOKEN" \
  -t xss,ssrf,misconfigurations

# Scan a list of URLs
nuclei -l live.txt -t exposures,misconfigurations
```

---

## Dalfox
XSS scanner and parameter analyzer. Detects reflected, stored, and DOM-based XSS vulnerabilities in web applications.

**Install:**
```bash
go install github.com/hahwul/dalfox/v2@latest
```

**Verify:**
```bash
dalfox version
```

**Basic usage:**
```bash
# Scan a single URL with a parameter
dalfox url "https://target.com/page?param=value"

# Authenticated scan
dalfox url "https://target.com/page?param=value" \
  --header "Cookie: session=YOUR_TOKEN"

# Scan from a list of URLs
dalfox file urls.txt --header "Cookie: session=YOUR_TOKEN"
```

---

## ffuf
Fast web fuzzer for endpoint, directory, and parameter discovery.

**Dependencies:**
```bash
pkg install unzip -y
```

**Install:**
```bash
go install github.com/ffuf/ffuf/v2@latest
```

**Verify:**
```bash
ffuf -V
```

**Get a wordlist:**
```bash
mkdir -p ~/wordlists
wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/api/api-endpoints.txt \
  -O ~/wordlists/api-endpoints.txt
```

**Basic usage:**
```bash
# Fuzz endpoints
ffuf -u https://target.com/FUZZ \
  -w ~/wordlists/api-endpoints.txt \
  -mc 200,301,302,403

# Authenticated fuzz
ffuf -u https://api.target.com/FUZZ \
  -H "Cookie: session=YOUR_TOKEN" \
  -w ~/wordlists/api-endpoints.txt \
  -mc 200,301,302,403

# Fuzz parameters
ffuf -u https://target.com/page?FUZZ=value \
  -w ~/wordlists/params.txt
```

---

## Reqable
Standalone HTTPS traffic interceptor for Android. No desktop required. Automatically configures the WiFi proxy — you only need to install the certificate and configure Firefox Nightly.

> ⚠️ **WARNING: Connection Loss** If apps or your browser lose internet connection after enabling Reqable, it is because they do not trust the Reqable self-signed CA certificate and are refusing the intercepted connection. This is normal behavior for most Android apps and non-configured browsers. To restore internet access immediately without stopping Reqable, **disable your VPN**. With no VPN active, traffic bypasses Reqable's proxy and connects directly. Use Firefox Nightly as configured below — it is the only browser that will work correctly through Reqable without losing connection.

**Install:** Download from Play Store — search **Reqable**

### Step 1 — Install CA Certificate

1. Open Reqable
2. Tap Certificate Setup
3. Tap the **Developer** tab
4. Download `reqable-ca.crt`
5. Go to: Settings → Security → Encryption & Credentials → Install a Certificate → CA Certificate
6. Select `reqable-ca.crt` and install it
7. Verify: Settings → Security → Encryption & Credentials → Trusted credentials → User tab — Reqable should appear in the list

### Step 2 — Configure Firefox Nightly

> ⚠️ **Must use Firefox Nightly specifically.** Standard Firefox on Android does not expose the configuration required to trust user-installed CA certificates. Firefox Nightly does. Do not use Chrome, Samsung Browser, or standard Firefox — they will refuse the connection.

**Install Firefox Nightly:** Download from Play Store — search **Firefox Nightly**

**Enable user certificate trust:**
1. Open Firefox Nightly
2. In the address bar type: `about:config`
3. Accept the warning
4. Search for: `security.enterprise_roots.enabled`
5. Tap the setting to toggle it to **true**

Firefox Nightly will now trust the Reqable CA certificate and all HTTPS traffic browsed through it will be intercepted and decrypted correctly.

### Step 3 — Start Intercepting

1. Open Reqable and tap the play button to start the proxy
2. Open **Firefox Nightly** and browse your target
3. Traffic appears in Reqable with full decrypted headers and body

### Step 4 — Extract Auth Tokens

1. In Reqable tap any request to your target
2. Tap the **Headers** tab
3. Look for authentication headers such as `Authorization: Bearer ...` or session cookies
4. Copy the full token value for use in Termux tools

### Step 5 — Stop Intercepting

Tap the stop button in Reqable when done. WiFi proxy is automatically restored.

---

## Token Refresh Script
Many APIs issue short-lived access tokens that expire within minutes. This script automatically refreshes an access token using a longer-lived refresh token and writes the current token to a file every 4 minutes. Other scripts read from this file to always use a valid token.

**Adapt to your target's auth endpoint and token field names.**

**Save as `~/refresh.py`:**

```python
import urllib.request, json, time

REFRESH_TOKEN = "YOUR_REFRESH_TOKEN_HERE"
REFRESH_URL = "https://api.target.com/auth/refresh"
TOKEN_FIELD = "access_token"  # adjust to match your target's response field

def get_access_token():
    data = json.dumps({"refresh_token": REFRESH_TOKEN}).encode()
    req = urllib.request.Request(
        REFRESH_URL,
        data=data,
        headers={
            "Content-Type": "application/json",
            "User-Agent": "Mozilla/5.0"
        },
        method="POST"
    )
    try:
        resp = urllib.request.urlopen(req)
        token = json.loads(resp.read())[TOKEN_FIELD]
        with open("/data/data/com.termux/files/home/token.txt", "w") as f:
            f.write(token)
        print("[+] Token refreshed")
        return token
    except Exception as e:
        print(f"[-] Refresh failed: {e}")

while True:
    get_access_token()
    time.sleep(240)
```

**Run in background:**
```bash
python3 ~/refresh.py &
```

**Read current token in other scripts:**
```bash
TOKEN=$(cat ~/token.txt)
```

---

## PATH Reference

After installing any Go tool, if command not found run:

```bash
export PATH=$PATH:/data/data/com.termux/files/home/go/bin
source ~/.bashrc
```

To make permanent (already done in Initial Setup):

```bash
echo 'export PATH=$PATH:/data/data/com.termux/files/home/go/bin' >> ~/.bashrc
```

---

## Quick Verification — All Tools

```bash
subfinder -version
httpx -version
nuclei -version
dalfox version
ffuf -V
python3 --version
```

All should return version numbers without errors.