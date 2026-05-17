# Bug Bounty Toolkit - Traffic Interception & Token Refresh

Covers HTTPS traffic interception on Android using Reqable and automated token refresh for all common API authentication patterns.

---

## Reqable

Standalone HTTPS traffic interceptor for Android. No desktop required. Automatically configures the WiFi proxy - you only need to install the certificate and configure Firefox Nightly.

> ⚠️ **WARNING: Connection Loss** If apps or your browser lose internet connection after enabling Reqable, it is because they do not trust the Reqable self-signed CA certificate and are refusing the intercepted connection. This is normal behavior for most Android apps and non-configured browsers. To restore internet access immediately without stopping Reqable, **disable your VPN**. With no VPN active, traffic bypasses Reqable's proxy and connects directly. Use Firefox Nightly as configured below - it is the only browser that will work correctly through Reqable without losing connection.

**Install:** Download from Play Store - search **Reqable**

### Step 1 - Install CA Certificate

1. Open Reqable
2. Tap Certificate Setup
3. Tap the **Developer** tab
4. Download `reqable-ca.crt`
5. Go to: Settings > Security > Encryption & Credentials > Install a Certificate > CA Certificate
6. Select `reqable-ca.crt` and install it
7. Verify: Settings > Security > Encryption & Credentials > Trusted credentials > User tab - Reqable should appear in the list

### Step 2 - Configure Firefox Nightly

> ⚠️ **Must use Firefox Nightly specifically.** Standard Firefox on Android does not expose the configuration required to trust user-installed CA certificates. Firefox Nightly does. Do not use Chrome, Samsung Browser, or standard Firefox - they will refuse the connection.

**Install Firefox Nightly:** Download from Play Store - search **Firefox Nightly**

**Enable user certificate trust:**
1. Open Firefox Nightly
2. In the address bar type: `about:config`
3. Accept the warning
4. Search for: `security.enterprise_roots.enabled`
5. Tap the setting to toggle it to **true**

Firefox Nightly will now trust the Reqable CA certificate and all HTTPS traffic browsed through it will be intercepted and decrypted correctly.

### Step 3 - Start Intercepting

1. Open Reqable and tap the play button to start the proxy
2. Open **Firefox Nightly** and browse your target
3. Traffic appears in Reqable with full decrypted headers and body

### Step 4 - Extract Auth Tokens

1. In Reqable tap any request to your target
2. Tap the **Headers** tab
3. Look for authentication headers such as `Authorization: Bearer ...` or session cookies
4. Copy the full token value for use in Termux tools

### Step 5 - Stop Intercepting

Tap the stop button in Reqable when done. WiFi proxy is automatically restored.

---

## Token Refresh Script

Many APIs issue short-lived access tokens that expire within minutes. This section covers the most common token refresh patterns. Identify which pattern your target uses by intercepting traffic in Reqable, then adapt the corresponding script.

---

### How to Identify Your Target's Pattern

Open Reqable and intercept traffic while logging in or using the app. Look for:

| What you see in Reqable | Pattern |
|--------------------------|---------|
| POST with JSON body containing `refresh_token` field | Pattern 1 |
| POST or GET with a `refresh_token=...` cookie | Pattern 2 |
| POST with `Authorization: Bearer <token>` header | Pattern 3 |
| POST with `Content-Type: application/x-www-form-urlencoded` and `grant_type=refresh_token` | Pattern 4 |
| Same token header on every request, never changes | Pattern 5 - static key |

---

### Configuration Variables

Every script in this section uses the same set of variables at the top. Edit these before running:

| Variable | What to set |
|----------|-------------|
| `REFRESH_TOKEN` | The long-lived token from Reqable |
| `REFRESH_URL` | The endpoint that issues new tokens |
| `TOKEN_FILE` | Where to save the token (default is fine) |
| `REFRESH_INTERVAL` | Seconds between refreshes - set to slightly less than the token lifetime |

**How to find the token lifetime:** Intercept a login response in Reqable and look for `expires_in` (seconds), `exp` (Unix timestamp), or `expiry` in the response. If you see `expires_in: 300` the token lasts 5 minutes - set `REFRESH_INTERVAL = 240` to refresh 1 minute early. If no expiry is shown, start with 240 and adjust if you get auth errors during testing.

---

### Pattern 1: JSON Body Refresh (Most Common)

The refresh token is sent as a JSON field in the request body. The new access token is returned in the JSON response.

**Identify in Reqable:** POST request with `Content-Type: application/json` and a body containing `refresh_token` or similar. Response contains `access_token` or `token` as a JSON field.

```python
import urllib.request, json, time

# --- CONFIGURE THESE ---
REFRESH_TOKEN    = "YOUR_REFRESH_TOKEN_HERE"
REFRESH_URL      = "https://api.target.com/auth/refresh"
TOKEN_FILE       = "/data/data/com.termux/files/home/token.txt"
REQUEST_FIELD    = "refresh_token"   # field name in the request body
RESPONSE_FIELD   = "access_token"   # field name in the response JSON
REFRESH_INTERVAL = 240              # seconds between refreshes
# -----------------------

def refresh():
    data = json.dumps({REQUEST_FIELD: REFRESH_TOKEN}).encode()
    req = urllib.request.Request(
        REFRESH_URL,
        data=data,
        headers={"Content-Type": "application/json", "User-Agent": "Mozilla/5.0"},
        method="POST"
    )
    try:
        resp = urllib.request.urlopen(req)
        token = json.loads(resp.read())[RESPONSE_FIELD]
        open(TOKEN_FILE, "w").write(token)
        print("[+] Token refreshed")
    except Exception as e:
        print(f"[-] Failed: {e}")

while True:
    refresh()
    time.sleep(REFRESH_INTERVAL)
```

---

### Pattern 2: Cookie-Based Refresh

The refresh token is sent as a cookie. The new access token is returned in a `Set-Cookie` response header.

**Identify in Reqable:** POST or GET request with a cookie like `refresh_token=...`. Response has a `Set-Cookie` header containing the new access token.

```python
import urllib.request, time, re

# --- CONFIGURE THESE ---
REFRESH_TOKEN       = "YOUR_REFRESH_TOKEN_HERE"
REFRESH_URL         = "https://target.com/auth/refresh"
TOKEN_FILE          = "/data/data/com.termux/files/home/token.txt"
REFRESH_COOKIE_NAME = "refresh_token"  # cookie name sent in the request
ACCESS_COOKIE_NAME  = "access_token"   # cookie name in the Set-Cookie response
REFRESH_INTERVAL    = 240             # seconds between refreshes
# -----------------------

def refresh():
    req = urllib.request.Request(
        REFRESH_URL,
        headers={
            "Cookie": f"{REFRESH_COOKIE_NAME}={REFRESH_TOKEN}",
            "User-Agent": "Mozilla/5.0"
        },
        method="POST"
    )
    try:
        resp = urllib.request.urlopen(req)
        cookies = resp.getheader("Set-Cookie") or ""
        match = re.search(rf"{ACCESS_COOKIE_NAME}=([^;]+)", cookies)
        if match:
            token = match.group(1)
            open(TOKEN_FILE, "w").write(token)
            print("[+] Token refreshed from Set-Cookie")
        else:
            print("[-] Token not found in Set-Cookie header")
    except Exception as e:
        print(f"[-] Failed: {e}")

while True:
    refresh()
    time.sleep(REFRESH_INTERVAL)
```

---

### Pattern 3: Bearer Token in Authorization Header

The refresh token is sent in the `Authorization` header. The new access token is returned in the JSON response.

**Identify in Reqable:** POST request with `Authorization: Bearer <refresh_token>` header. Response contains `access_token` in JSON.

```python
import urllib.request, json, time

# --- CONFIGURE THESE ---
REFRESH_TOKEN    = "YOUR_REFRESH_TOKEN_HERE"
REFRESH_URL      = "https://api.target.com/auth/token"
TOKEN_FILE       = "/data/data/com.termux/files/home/token.txt"
RESPONSE_FIELD   = "access_token"   # field name in the response JSON
REFRESH_INTERVAL = 240              # seconds between refreshes
# -----------------------

def refresh():
    req = urllib.request.Request(
        REFRESH_URL,
        headers={
            "Authorization": f"Bearer {REFRESH_TOKEN}",
            "Content-Type": "application/json",
            "User-Agent": "Mozilla/5.0"
        },
        method="POST"
    )
    try:
        resp = urllib.request.urlopen(req)
        token = json.loads(resp.read())[RESPONSE_FIELD]
        open(TOKEN_FILE, "w").write(token)
        print("[+] Token refreshed")
    except Exception as e:
        print(f"[-] Failed: {e}")

while True:
    refresh()
    time.sleep(REFRESH_INTERVAL)
```

---

### Pattern 4: OAuth2 grant_type=refresh_token

Standard OAuth2 flow. Refresh token and grant type are sent as form-encoded body fields. Access token returned in JSON response.

**Identify in Reqable:** POST request with `Content-Type: application/x-www-form-urlencoded` and body containing `grant_type=refresh_token`.

```python
import urllib.request, urllib.parse, json, time

# --- CONFIGURE THESE ---
REFRESH_TOKEN    = "YOUR_REFRESH_TOKEN_HERE"
CLIENT_ID        = "YOUR_CLIENT_ID"    # remove if not required by your target
TOKEN_URL        = "https://auth.target.com/oauth/token"
TOKEN_FILE       = "/data/data/com.termux/files/home/token.txt"
REFRESH_INTERVAL = 240                 # seconds between refreshes
# -----------------------

def refresh():
    data = urllib.parse.urlencode({
        "grant_type": "refresh_token",
        "refresh_token": REFRESH_TOKEN,
        "client_id": CLIENT_ID
    }).encode()
    req = urllib.request.Request(
        TOKEN_URL,
        data=data,
        headers={
            "Content-Type": "application/x-www-form-urlencoded",
            "User-Agent": "Mozilla/5.0"
        },
        method="POST"
    )
    try:
        resp = urllib.request.urlopen(req)
        token = json.loads(resp.read())["access_token"]
        open(TOKEN_FILE, "w").write(token)
        print("[+] Token refreshed via OAuth2")
    except Exception as e:
        print(f"[-] Failed: {e}")

while True:
    refresh()
    time.sleep(REFRESH_INTERVAL)
```

---

### Pattern 5: API Key (No Refresh Required)

Some targets use static API keys that do not expire. No refresh loop needed.

**Identify in Reqable:** The same `X-API-Key`, `api_key`, or `Authorization: ApiKey` header appears on every request and never changes between sessions.

```bash
# Store once
echo "YOUR_API_KEY_HERE" > ~/token.txt

# Read in other scripts (bash)
TOKEN=$(cat ~/token.txt)

# Read in other scripts (fish)
set TOKEN (cat ~/token.txt)
```

---

### Running a Refresh Script in the Background

Termux can run scripts in the background, but Android aggressively kills background processes to save battery. There are three approaches, in order of reliability:

**Option 1: Simple background process**

Works while Termux is open. Killed if Termux is closed or Android reclaims memory.

```bash
python3 ~/refresh.py &
```

Stop it:
```bash
kill %1
# or
ps aux | grep refresh.py
kill <PID>
```

---

**Option 2: nohup**

Survives Termux being closed but may still be killed by Android after a period of inactivity. Output is logged to a file.

```bash
nohup python3 ~/refresh.py > ~/refresh.log 2>&1 &
```

Check if it is still running:
```bash
ps aux | grep refresh.py
```

Check the log:
```bash
cat ~/refresh.log
```

Stop it:
```bash
kill $(pgrep -f refresh.py)
```

---

**Option 3: tmux (Recommended)**

Creates a persistent terminal session that survives Termux being minimized and most Android memory management. You can detach, do other work, and reattach later.

Install tmux if not already installed:
```bash
pkg install tmux -y
```

Start a named session and run the script inside it:
```bash
tmux new -s refresh
python3 ~/refresh.py
```

Detach from the session without stopping it (the script keeps running):
```
Ctrl+B, then D
```

Reattach later to check on it:
```bash
tmux attach -t refresh
```

List all active sessions:
```bash
tmux ls
```

Kill the session when done:
```bash
tmux kill-session -t refresh
```

> **Note on Android battery optimization:** Even with tmux, Android may suspend Termux after extended inactivity. If you find the script has stopped, go to Settings > Battery > App Battery Usage > Termux and set it to Unrestricted. This prevents Android from killing the process.

---

### Reading the Token in Other Tools

Once the refresh script is running and writing to `token.txt`, read it in any other tool:

```bash
# bash
TOKEN=$(cat ~/token.txt)
curl -H "Authorization: Bearer $TOKEN" https://api.target.com/endpoint

# fish
set TOKEN (cat ~/token.txt)
curl -H "Authorization: Bearer $TOKEN" https://api.target.com/endpoint
```

---

### Decoding a JWT Token (Any Pattern)

If the access token is a JWT (three dot-separated base64 segments), decode the payload to inspect claims, expiry, and permissions without any external tools:

```bash
cat ~/token.txt | cut -d'.' -f2 | tr -- '-_' '+/' | \
  awk '{ pad=4-length($0)%4; if(pad<4) for(i=0;i<pad;i++) $0=$0"=" }1' | \
  base64 -d 2>/dev/null | python3 -m json.tool
```

Key fields to look for:

| Field | Meaning |
|-------|---------|
| `exp` | Expiry as Unix timestamp - use this to set `REFRESH_INTERVAL` |
| `sub` | Subject - usually the user ID |
| `aud` | Audience - which service the token is valid for |
| `scope` or `roles` | Permission level granted |
| `iat` | Issued at timestamp |
| Any custom field | Non-standard claims like `isAdmin`, `role`, `plan` indicate privilege levels worth testing |

Convert an `exp` timestamp to a readable date:
```bash
date -d @<timestamp>     # Linux / Termux
date -r <timestamp>      # macOS
```
