# Bug Bounty Toolkit - Termux Installation Guide

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

## Quick Verification - All Tools

```bash
subfinder -version
httpx -version
nuclei -version
dalfox version
ffuf -V
python3 --version
```

All should return version numbers without errors.
