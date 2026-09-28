# SFTP data delivery — Linux setup guide

This repository explains how to send your ERP data to our server securely, using SFTP with an SSH key.

The setup has **two parts**. Please complete them **in order**:

**Part 1** is about setting your SSH keys on your Linux server, establishing first connection and check folder structure where the data is going to be stored. 

**Part 2** (*is not mandatory follow this one, in any case ignore these*): is about the scripting behind the necessary login for the daily file posting to the sftp server.

> 🔒 **Your connection details** (server address, username) are **not** in this repository. We send them to you by email and confirm the fingerprint by phone.


# Part 1 — Access setup and first delivery

**Estimated time:** 20-30 minutes, plus waiting for confirmation.
**You need:** the Linux machine that will send the data every day, and a normal (non-root-necessary) account on it.

---

## Before you start: two important rules

1. **Use the production machine.** Do everything on the machine that will send the files every day, **not** on a test or personal computer. Our server only accepts connections from that machine's IP address.
2. **Use the same account for everything.** The account you use in this guide must be the same one that will run the daily job (Part 2). The key and the server's identity are stored in that account's home folder (`~/.ssh`).

---

## Step 1 — Check the SSH client

```bash
ssh -V
command -v sftp
```

✅ **Expected:** a version such as `OpenSSH_8.x` or `OpenSSH_9.x`, and a path such as `/usr/bin/sftp`.

If `sftp` is missing, install it:

```bash
sudo apt install openssh-client      # Debian / Ubuntu
sudo dnf install openssh-clients     # RHEL / Alma / Rocky
```

---

## Step 2 — Create your SSH key (without passphrase)

The job should run automatically and unattended, so the key **must not have a passphrase**.

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -N "" -f ~/.ssh/{{erp}}_delivery -C "{{erp}}-delivery-<COMPANY>"
```

> *"-N" parameters is necessary to avoid passphrase. It is important not to indicate a passprhase in order to simplify the daily job script, whatever you choose to code.* 

This creates two files:

| File                      | What it is      | Sharing                                       |
| ------------------------- | --------------- | --------------------------------------------- |
| `~/.ssh/{{erp}}_delivery`     | **Private key** | ❌ **Never.** It must never leave this machine |
| `~/.ssh/{{erp}}_delivery.pub` | Public key      | ✅ Yes, send it to us                          |

**Check the key:**

```bash
ls -l ~/.ssh/{{erp}}_delivery ~/.ssh/{{erp}}_delivery.pub
ssh-keygen -y -P "" -f ~/.ssh/{{erp}}_delivery > /dev/null && echo "OK: the key has no passphrase"
```

✅ **Expected:**
- `erp_delivery` shows `-rw-------` (only you can read it).
- The message `OK: the key has no passphrase`.

---

## Step 3 — Collect the information we need

Run all three commands **on the production machine**.

**a) Your public key:**

```bash
cat ~/.ssh/{{erp}}_delivery.pub
```

It is one single line that starts with `ssh-ed25519 AAAA...`.

**b) Your key fingerprint:**

```bash
ssh-keygen -lf ~/.ssh/{{erp}}_delivery.pub
```

It looks like `256 SHA256:AbC123... {{erp}}-delivery-<COMPANY> (ED25519)`.

**c) Your public outbound IP address:**

```bash
curl -s https://api.ipify.org; echo
```

> ⚠️ This IP must be **fixed** (it never changes) and **exclusive** to your company. It must **not** start with `10.`, `192.168.`, `172.16.` to `172.31.`, or `100.64.` to `100.127.`: those are private or shared addresses and will not work.

---

## Step 4 — Send us your details

Send us an email with the **public key file** (e.g. `{{erp}}_delivery.pub`) configured in step 3, attached and this text:

```
Subject: SFTP access – <COMPANY>

1. Public key: attached ({{erp}}_delivery.pub)
2. The last 4 characters of your key fingerprint.
3. Public outbound IP of the production machine: <YOUR_PUBLIC_IP>
```
---

## Step 5 — Wait for our confirmation

We will authorize your key and your IP, and reply by email with:

- **Server address** (`<server_host>`)
- **Username** (`<sftp_user>`) (the one you'll need to connect to the server by SSH)
- **Our server fingerprints** (three lines: ED25519, ECDSA and RSA)

Check and compare the fingerprints sent by mail with the following (**should be exactly the same ones**):
```
SHA256:oiApj1ti+FjzqiAtm7q/uR2NQBsXbq5RJ7HZav+l0/s (ED25519)
SHA256:kxsoIUT7bvyTHRxaqyLOHIGumtfWI2CFDFTlaG0oioo (ECDSA)
SHA256:hyffQYR0OVBSopcwQeqM6JNsvUPFUJ8NJHQwUu4xpyE (RSA)
```

> Until you receive this email, connection attempts will fail with `Connection timed out`. This is normal.

---

## Step 6 — First connection and server verification

This step confirms that you are connecting to **our** server and not to an impostor. It also saves our server's identity on your machine. **The daily job will not work without this step.**

> !!! Have our fingerprint ready **before** connecting. The server waits about 2 minutes for your answer and then closes the connection.

```bash
sftp -o IdentitiesOnly=yes -i ~/.ssh/{{erp}}_delivery <sftp_user>@<server_host>
```

The first time, you will see:

```
The authenticity of host '<Sserver_host> (x.x.x.x)' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

**Compare** the `SHA256:...` value with the fingerprint of the **same type** in our email (ED25519, ECDSA or RSA): most probably it will compare ED255519.

- ✅ **They match** → paste our fingerprint and press Enter (or type `yes` if your version only offers `yes/no`).
- ❌ **They do not match** → type **`no`** and **reach us**. Do not continue.

✅ **Expected:** `Warning: Permanently added ...` and then the `sftp>` prompt, **without being asked for any password**.

Inside the session, run:

```
sftp> pwd
Remote working directory: /
sftp> ls
incoming
sftp> ls /incoming
amortizacion  maestros  reparto  sat
sftp> bye
```

Check the folder structure in your case.

If you are asked for a password or a passphrase, stop and see [Troubleshooting](#troubleshooting).

---

## Step 7 — Upload the historical file (2022–2025)

After confirming you can connect, this is going to be your first real delivery. It confirms that everything works end to end.

### 7.1 Check the file format

The file must follow these rules, unless we agree otherwise:

| Rule      | Value                                                                                                     |
| --------- | --------------------------------------------------------------------------------------------------------- |
| Format    | **JSON**: one single JSON array, with one object per record (see example below)                           |
| Structure | Every object has the **same field names**. Missing values are `null` (not `""`, `"N/A"` or `0`)           |
| Encoding  | UTF-8 **without BOM**                                                                                     |
| Numbers   | JSON numbers, not text: `1234.56`, not `"1234.56"` or `"1.234,56"`                                        |
| Booleans  | `true` / `false`, not `"yes"`, `"1"` or `"S"`                                                             |
| Dates     | Text in ISO 8601 format: **`"YYYY-MM-DD"`** (e.g. `"2024-03-15"`). Date and time: `"YYYY-MM-DDTHH:MM:SS"` |
| Period    | From 1 January 2022 to 31 December 2025                                                                   |

Example: 

```json
[
  {
    "id": "P20240315001",
    "date": "2024-03-15",
    "customer_code": "C0012",
    "hours": 1.75,
    "total_cost": 145.30,
    "billable": true,
    "notes": null
  },
  {
    "id": "P20240315002",
    "date": "2024-03-15",
    "customer_code": "C0047",
    "hours": 0.5,
    "total_cost": 38.00,
    "billable": false,
    "notes": "Preventive visit"
  }
]
```

### 7.2 Upload it

```bash
# First connect
sftp -i ~/.ssh/erp_delivery <SFTP_USER>@<SERVER_HOST>
```

Then copy to folder on sftp server:

```
sftp> put /path/to/<HISTORY_FILE> /incoming/sat/ 
sftp> ls -l /incoming/sat 
sftp> bye
```

✅ **Expected:** a progress bar reaching `100%`, and the file listed in `/incoming/sat` with the **same size** as on your machine (`ls -l /path/to/<HISTORY_FILE>`).

> If the connection drops during a large upload, connect again and use `reput` instead of `put`, with the same arguments. It resumes where it stopped.

---

## Step 8 — Tell us it is done (Optional step but worth the checking)

Run:

```bash
F=/path/to/<HISTORY_FILE> basename "$F"; stat -c %s "$F"; sha256sum "$F" | cut -d' ' -f1 python3 -c 'import json,sys; print(len(json.load(open(sys.argv[1], encoding="utf-8"))))' "$F" 
```

and send us:

```
Subject: SFTP historical file uploaded – <COMPANY>

File name: __________
Size (bytes): __________
Lines (including header): __________
SHA256: __________
```

We will compare these values with the file we received and confirm. **After our confirmation, continue with [Part 2](2-daily-delivery.md).**

---

## Troubleshooting

| Message | Most likely cause | What to do |
|---|---|---|
| `Connection timed out` | Our authorization is not active yet, or your public IP is not the one you sent us | Check your IP again (Step 3c) and tell us. Also check that your network allows outgoing port 22: `timeout 5 bash -c '</dev/tcp/github.com/22' && echo "port 22 open"` |
| `Connection refused` | Our service is not available | Contact us |
| `Host key verification failed` or `REMOTE HOST IDENTIFICATION HAS CHANGED` | The server identity does not match | **Stop** and call us |
| `Permission denied (publickey)` | Wrong username, wrong key file, or key not yet authorized | Check `<SFTP_USER>` and the `-i ~/.ssh/{{erp}}_delivery` path, then contact us |
| `UNPROTECTED PRIVATE KEY FILE` | Private key permissions are too open | `chmod 600 ~/.ssh/erp_delivery` |
| `Enter passphrase for key` | The key has a passphrase | Repeat Step 2 with a new key and send us the new `.pub` |
| Asked for a **password** | Something is wrong on our side | Do not type anything. Contact us |

If the problem persists, send us the last lines of a detailed connection attempt:

```bash
sftp -v -i ~/.ssh/{{erp}}_delivery <SFTP_USER>@<SERVER_HOST> 2>&1 | tail -n 40
```

This output contains no secrets.

---

## Security rules

- **Never** send, copy or share the private key (`~/.ssh/erp_delivery`).
- Do not copy the key to other machines. If you change machines, create a new key and send us the new `.pub`.
- If you think the key or the machine has been compromised, **tell us immediately** and we will block access.



# Part 2 — Daily automatic delivery

**How it should work:**
```
[ERP] ──exports every day──> /opt/erp-export/field-service.json ──send_sftp.sh (cron)──> our server /incoming/field-service/

[ERP] ──exports every day──> /opt/erp-export/logistics-costs.json ──send_sftp.sh (cron)──> our server /incoming/logistics-costs/

[ERP] ──exports every day──> /opt/erp-export/depreciation.json ──send_sftp.sh (cron)──> our server /incoming/depreciation/
```

1. Your **ERP exports** the file every day to a fixed path, always with the same name.
2. The **script** `send_sftp.sh` uploads that file to our server.
3. **cron** runs the script every day at the agreed time.

Use the **same Linux account** as in Part 1.

---

## Step 1 — Configure the ERP export

This depends on your ERP, so please configure it with your usual tools. The export must meet these requirements:

| Requirement   | Detail                                                                                                            |
| ------------- | ----------------------------------------------------------------------------------------------------------------- |
| Path and name | ==**Always the same**,== e.g. `/opt/erp-export/depreciation.json`. Each export replaces the previous one          |
| Content       | `<AGREED_PERIOD>`. Each upload **replaces** the previous file on our server, so the file must always be complete. |
| Format        | Same as the historical file (Part 1, Step 7.1), with the same columns in the same order                           |
| Timing        | The export must **finish before** the upload time (Step 4)                                                        |
| Permissions   | The account running the script must be able to **read** the file                                                  |

> 💡 **Recommended (Not mandatory):** make the ERP write to a temporary name and rename it at the end, e.g. export to `sat.json.tmp` and then run `mv sat.json.tmp sat.csv`. This way the upload never takes a half-written file.

**Check** after an export:

```bash
ls -l --time-style=long-iso /opt/{{erp}}-export/depreciation.json
```

✅ **Expected:** today's date and time.

---

## Step 2 —  Scripting for daily put (if needed)

Common parameters:

| Variable    | Value                                                       |
| ----------- | ----------------------------------------------------------- |
| `FILE`      | Full path of the file exported by your ERP (Step 1)         |
| `DEST_DIR`  | `/incoming/depreciation` — **do not change**                |
| `SERVER`    | `<SERVER_HOST>` from our email                              |
| `SFTP_USER` | `<SFTP_USER>` from our email                                |
| `KEY`       | `$HOME/.ssh/{{erp}}_delivery` (the private key from Part 1) |

### Send files periodically: send_sftp.sh
```bash
#!/usr/bin/env bash
# send_sftp.sh - Uploads one JSON file to the SFTP server.
# Edit only these five values.
FILE=/opt/erp-export/file.json            # file exported from ERP (always same name)
DEST_DIR=/incoming/<dest_folder>          # folder on the server e.g. /depreciation 
SERVER=<server_host>                      # from our email
SFTP_USER=<sftp_user>                       # from our email
KEY=$HOME/.ssh/{{erp}}_delivery               # PRIVATE key (not the .pub file)

LOG="$(dirname "$(readlink -f "$0")")/send_sftp.log"
log() { echo "$(date '+%F %T') $*" >> "$LOG"; }

# 1. The file exists
if [[ ! -f $FILE ]]; then
  log "ERROR: file not found: $FILE"
  exit 1
fi

# 2. The ERP updated it recently
age=$(( $(date +%s) - $(stat -c %Y "$FILE") ))
if (( age > 86400 )); then
  log "WARNING: $FILE was last modified $(( age / 3600 )) hours ago (did the ERP export run?)"
fi

# 3. It is valid JSON (a broken file is NOT sent: the last good file stays on the server)
if command -v python3 >/dev/null; then
  if ! check=$(python3 -c 'import json,sys; d=json.load(open(sys.argv[1], encoding="utf-8")); assert isinstance(d, list), "not a JSON array"; print(len(d))' "$FILE" 2>&1); then
    log "ERROR: invalid JSON, file NOT sent: $(echo "$check" | tail -n 1)"
    exit 3
  fi
  records="$check records, "
else
  log "WARNING: python3 not found, JSON not validated"
  records=""
fi

# 4. Upload
output=$(echo "put \"$FILE\" \"$DEST_DIR/$(basename "$FILE")\"" |
         sftp -b - -i "$KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes \
              -o IdentitiesOnly=yes -o ConnectTimeout=20 "$SFTP_USER@$SERVER" 2>&1)
rc=$?

if (( rc == 0 )); then
  log "OK: $FILE (${records}$(stat -c %s "$FILE") bytes)"
else
  log "ERROR ($rc) uploading $FILE"
  echo "$output" | sed 's/^/    /' >> "$LOG"
fi
exit $rc
```

### Cron job: setup_cron.sh

```bash
#!/usr/bin/env bash
# setup_cron.sh - Creates (or re-creates) the daily cron job that runs send_sftp.sh.
# Run WITHOUT sudo, with the same account that made the first manual connection.
set -euo pipefail

SCRIPT="$HOME/{{erp}}-sftp/send_sftp.sh"   # path of the upload script
HOUR=2                                 # agreed upload time, 24-hour format
MINUTE=30
TAG="# {{erp}}-sftp-delivery"              # identifies this job in the crontab

if [[ $EUID -eq 0 ]]; then
  echo "WARNING: you are root. The job will use root's key and known_hosts, not your account's."
fi
if [[ ! -f $SCRIPT ]]; then
  echo "ERROR: $SCRIPT not found"; exit 1
fi
if ! [[ $HOUR =~ ^([01]?[0-9]|2[0-3])$ && $MINUTE =~ ^[0-5]?[0-9]$ ]]; then
  echo "ERROR: invalid HOUR or MINUTE"; exit 1
fi
if ! command -v crontab >/dev/null; then
  echo "ERROR: cron is not installed (Debian/Ubuntu: sudo apt install cron | RHEL/Alma/Rocky: sudo dnf install cronie)"
  exit 1
fi
chmod 700 "$SCRIPT"

# Replace any previous job with the same tag, then add the new one
LINE="$MINUTE $HOUR * * * $SCRIPT >/dev/null 2>&1 $TAG"
( crontab -l 2>/dev/null | grep -vF "$TAG" || true; echo "$LINE" ) | crontab -

# Check that the cron service is running
if command -v systemctl >/dev/null; then
  SVC=$(systemctl list-unit-files 2>/dev/null | grep -oE '^(cron|crond)\.service' | head -n1 || true)
  if [[ -n $SVC ]] && ! systemctl is-active --quiet "$SVC"; then
    echo "WARNING: $SVC is not running. Start it with: sudo systemctl enable --now $SVC"
  fi
fi

echo "Cron job created for $(id -un):"
crontab -l | grep -F "$TAG"
echo "System time zone: $(timedatectl show -p Timezone --value 2>/dev/null || cat /etc/timezone 2>/dev/null || echo unknown)"
```

-----
## Ongoing: please tell us in advance if…

- your **public IP** changes (new internet provider, new server, etc.). Otherwise the upload will stop working;
- you move the job to **another machine**. You will need a new key: repeat Part 1, Steps 2 to 6;
- the **file format or columns** change.

If only the **export path** changes, just update `FILE` in `send_sftp.sh` (or in your script of choise). You do not need to tell us.

----
## Troubleshooting

| What you see                                    | Cause                                                              | What to do                                                                                 |
| ----------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| No new line in the log                          | cron did not run the script                                        | `crontab -l` must show the job. Check the service: `systemctl status cron` (Debian/Ubuntu) |
| `ERROR: file not found`                         | The ERP export did not run or used another path                    | Check the ERP export and the `FILE` value                                                  |
| `WARNING: ... last modified N hours ago`        | The ERP did not update the file                                    | Check the ERP export. The old file is sent anyway                                          |
| `Host key verification failed` (only from cron) | cron runs with a **different account** than the one used in Part 1 | Run `setup_cron.sh` with the same account that made the first connection                   |
| `Permission denied (publickey)`                 | Wrong `KEY` path or wrong account                                  | Check `KEY` and that the file exists in that account's `~/.ssh`                            |
| `Connection timed out`                          | Your public IP has changed                                         | Check it (`curl -s https://api.ipify.org`)                                                 |

