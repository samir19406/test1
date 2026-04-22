# SSL Certificate Renewal — pmuedge.abn.green.sophos

**Server:** pmuedge.abn.green.sophos  
**Web Server:** Apache (httpd)  
**Apache Config:** `/etc/httpd/conf.d/10-pmx-static2.sophos.com.conf`  
**Date of Renewal:** April 22, 2026  
**Performed by:** pavanbhatt  
**Jira Ticket:** https://sophos.atlassian.net/browse/SIM-92666

---

## Certificate Details

### Old Certificate (Replaced)
- **CN:** pmx-static-uk1.sophos.com
- **SANs:** pmx-static-uk1.sophos.com, sasi-uk.sophos.com, pmx-dynamic.sophos.com
- **Issuer:** GlobalSign RSA OV SSL CA 2018
- **Valid:** Apr 3, 2025 → May 5, 2026

### New Certificate
- **CN:** pmx-static-uk1.sophos.com
- **SANs:** pmx-static-uk1.sophos.com, sasi-uk.sophos.com, pmx-dynamic.sophos.com
- **Issuer:** GlobalSign RSA OV SSL CA 2018
- **Valid:** Apr 21, 2026 → Nov 6, 2026
- **Type:** OV SSL (RSA 2048, SHA-256)
- **Organization:** SOPHOS LIMITED
- **GlobalSign Cert File:** CEPO260421004935.cer

### Intermediate Chain (Unchanged)
- **Subject:** GlobalSign RSA OV SSL CA 2018
- **Issuer:** GlobalSign Root CA - R3
- **Valid:** Nov 21, 2018 → Nov 21, 2028
- **Note:** Same intermediate was reused — still valid until 2028

---

## File Locations on Server

| Purpose | Path |
|---|---|
| Certificate | `/etc/pki/tls/certs/sslcert_sophos-com.cer` |
| Private Key | `/etc/pki/tls/private/private_sophos-com.key` |
| Chain (intermediate) | `/etc/pki/tls/certs/cert_chain_edge.cer` |
| CSR | `/etc/pki/tls/certs/edge-certs.csr` |
| Apache Config | `/etc/httpd/conf.d/10-pmx-static2.sophos.com.conf` |

### 2026 Renewal Files (New Key + CSR)
| Purpose | Path |
|---|---|
| New Private Key | `/etc/pki/tls/private/private_sophos-com_2026.key` |
| New CSR | `/etc/pki/tls/certs/edge-certs_2026.csr` |

### Backup Location
All previous cert files backed up to `/etc/pki/tls/backup_25-26/`

---

## Technical Steps Performed

### Step 0: Investigation & Discovery

#### 0.1 — Found all SSL cert files on the server
```bash
sudo find / -type f \( -name "*.pem" -o -name "*.crt" -o -name "*.cert" -o -name "*.cer" \) -exec sh -c \
  'openssl x509 -in "$1" -noout -subject 2>/dev/null | grep -qi "pmx-static\|GlobalSign" && echo "$1"' _ {} \; 2>/dev/null
```
Result:
- `/home/jayantchakraborty/CEPO240306818694.cer`
- `/etc/pki/tls/certs/cert_chain_edge.cer`
- `/etc/pki/tls/certs/sslcert_sophos-com.cer`
- `/tmp/CEPO250403964077.cer`

#### 0.2 — Checked the current live certificate
```bash
sudo openssl x509 -in /etc/pki/tls/certs/sslcert_sophos-com.cer -noout -subject -dates -issuer
```
Result:
```
subject= /C=GB/ST=Oxfordshire/L=Abingdon/O=SOPHOS LIMITED/CN=pmx-static-uk1.sophos.com
notBefore=Apr  3 10:26:22 2025 GMT
notAfter=May  5 10:26:21 2026 GMT
issuer= /C=BE/O=GlobalSign nv-sa/CN=GlobalSign RSA OV SSL CA 2018
```

#### 0.3 — Checked the intermediate chain certificate
```bash
sudo openssl x509 -in /etc/pki/tls/certs/cert_chain_edge.cer -noout -subject -issuer -dates
```
Result:
```
subject= /C=BE/O=GlobalSign nv-sa/CN=GlobalSign RSA OV SSL CA 2018
issuer= /OU=GlobalSign Root CA - R3/O=GlobalSign/CN=GlobalSign
notBefore=Nov 21 00:00:00 2018 GMT
notAfter=Nov 21 00:00:00 2028 GMT
```

#### 0.4 — Checked full cert details including SANs
```bash
sudo openssl x509 -in /etc/pki/tls/certs/sslcert_sophos-com.cer -noout -text | head -60
```
Result confirmed SANs: `pmx-static-uk1.sophos.com`, `sasi-uk.sophos.com`, `pmx-dynamic.sophos.com`

#### 0.5 — Found existing CSR
```bash
sudo find / -type f -name "*.csr" 2>/dev/null
```
Result: `/etc/pki/tls/certs/edge-certs.csr`

#### 0.6 — Checked existing CSR details
```bash
sudo openssl req -in /etc/pki/tls/certs/edge-certs.csr -noout -subject -text | head -30
```
Result:
```
Subject: C=GB, ST=Oxfordshire, L=Abingdon, O=Sophos CIS, CN=pmx-static-uk1.sophos.com/emailAddress=nsg-cis@sophos.com
```

#### 0.7 — Identified web server
```bash
sudo systemctl list-units --type=service --state=running | grep -iE "nginx|apache|httpd|haproxy|tomcat"
```
Result: `httpd.service — The Apache HTTP Server`

#### 0.8 — Checked Apache SSL configuration
```bash
sudo grep -rn "SSLCertificate" /etc/httpd/
```
Result:
```
/etc/httpd/conf.d/10-pmx-static2.sophos.com.conf:31:  SSLCertificateFile      "/etc/pki/tls/certs/sslcert_sophos-com.cer"
/etc/httpd/conf.d/10-pmx-static2.sophos.com.conf:32:  SSLCertificateKeyFile   "/etc/pki/tls/private/private_sophos-com.key"
/etc/httpd/conf.d/10-pmx-static2.sophos.com.conf:33:  SSLCertificateChainFile "/etc/pki/tls/certs/cert_chain_edge.cer"
/etc/httpd/conf.d/10-pmx-static2.sophos.com.conf:34:  SSLCACertificatePath    "/etc/pki/tls/certs"
```

#### 0.9 — Checked Puppet status
```bash
sudo systemctl status puppet
```
Result: `inactive (dead)` — Puppet agent is disabled, manual changes are safe.

#### 0.10 — Listed all existing cert/key files with dates
```bash
sudo ls -la /etc/pki/tls/private/ /etc/pki/tls/certs/ | grep -iE "sophos|edge|private_|\.csr|\.cer|\.crt|\.key|\.pem"
```
Result showed previous backups: `sslcert_sophos-com.cer.back24-25`, `sslcert_sophos-com.cer_backup`

---

### Step 1: Generated New Private Key + CSR

```bash
# Created OpenSSL config file with SANs (needed because server has older OpenSSL without -addext support)
sudo bash -c 'cat > /tmp/csr_2026.cnf << EOF
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = v3_req

[dn]
C = GB
ST = Oxfordshire
L = Abingdon
O = SOPHOS LIMITED
CN = pmx-static-uk1.sophos.com

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = pmx-static-uk1.sophos.com
DNS.2 = sasi-uk.sophos.com
DNS.3 = pmx-dynamic.sophos.com
EOF'

# Generated new key + CSR with different filenames to avoid disrupting live cert
sudo openssl req -new -newkey rsa:2048 -nodes \
  -keyout /etc/pki/tls/private/private_sophos-com_2026.key \
  -out /etc/pki/tls/certs/edge-certs_2026.csr \
  -config /tmp/csr_2026.cnf

# Set permissions to match existing key (400 = read-only by root)
sudo chmod 400 /etc/pki/tls/private/private_sophos-com_2026.key

# Verified CSR content
sudo openssl req -in /etc/pki/tls/certs/edge-certs_2026.csr -noout -subject -text | grep -A1 "Subject:\|DNS:"
```
Result:
```
Subject: C=GB, ST=Oxfordshire, L=Abingdon, O=SOPHOS LIMITED, CN=pmx-static-uk1.sophos.com
DNS:pmx-static-uk1.sophos.com, DNS:sasi-uk.sophos.com, DNS:pmx-dynamic.sophos.com
```

---

### Step 2: Sent CSR to GlobalSign

- Contacted GlobalSign account manager
- Provided CSR content:
```bash
sudo cat /etc/pki/tls/certs/edge-certs_2026.csr
```

---

### Step 3: Received & Verified New Certificate

```bash
# Verified new cert details locally
openssl x509 -in /Users/pavan.bhatt/Downloads/CEPO260421004935.cer -noout -subject -issuer -dates -ext subjectAltName
```
Result:
```
subject=C=GB, ST=Oxfordshire, L=Abingdon, O=SOPHOS LIMITED, CN=pmx-static-uk1.sophos.com
issuer=C=BE, O=GlobalSign nv-sa, CN=GlobalSign RSA OV SSL CA 2018
notBefore=Apr 21 18:06:18 2026 GMT
notAfter=Nov  6 18:06:18 2026 GMT
X509v3 Subject Alternative Name:
    DNS:pmx-static-uk1.sophos.com, DNS:sasi-uk.sophos.com, DNS:pmx-dynamic.sophos.com
```

```bash
# Verified cert matches the new private key (modulus MD5 must match)
# Local (cert):
openssl x509 -in /Users/pavan.bhatt/Downloads/CEPO260421004935.cer -noout -modulus | openssl md5
# Result: 9ebf48ffe8fbb78d8fb6d35753a8ed0f

# Remote (key):
sudo openssl rsa -in /etc/pki/tls/private/private_sophos-com_2026.key -noout -modulus | openssl md5
# Result: 9ebf48ffe8fbb78d8fb6d35753a8ed0f
```
✅ Match confirmed.

---

### Step 4: Backed Up Current Files

```bash
# Created dedicated backup folder
sudo mkdir -p /etc/pki/tls/backup_25-26

# Backed up all 4 files
sudo cp /etc/pki/tls/certs/sslcert_sophos-com.cer /etc/pki/tls/backup_25-26/
sudo cp /etc/pki/tls/private/private_sophos-com.key /etc/pki/tls/backup_25-26/
sudo cp /etc/pki/tls/certs/cert_chain_edge.cer /etc/pki/tls/backup_25-26/
sudo cp /etc/pki/tls/certs/edge-certs.csr /etc/pki/tls/backup_25-26/

# Verified backups
sudo ls -la /etc/pki/tls/backup_25-26/
```
Result:
```
-r--------. 1 root root 1554 Apr 22 08:51 cert_chain_edge.cer
-rw-r--r--. 1 root root 1513 Apr 22 08:51 edge-certs.csr
-r--------. 1 root root 1704 Apr 22 08:51 private_sophos-com.key
-rw-r--r--. 1 root root 2415 Apr 22 08:51 sslcert_sophos-com.cer
```

---

### Step 5: Placed New Cert + Key

```bash
# Uploaded new cert from local machine to server
# (via SFTP to /tmp/CEPO260421004935.cer)

# Replaced certificate
sudo cp /tmp/CEPO260421004935.cer /etc/pki/tls/certs/sslcert_sophos-com.cer
sudo chmod 644 /etc/pki/tls/certs/sslcert_sophos-com.cer

# Replaced private key
sudo cp /etc/pki/tls/private/private_sophos-com_2026.key /etc/pki/tls/private/private_sophos-com.key
sudo chmod 400 /etc/pki/tls/private/private_sophos-com.key

# Final key match verification after placement
sudo openssl x509 -in /etc/pki/tls/certs/sslcert_sophos-com.cer -noout -modulus | openssl md5
# Result: 9ebf48ffe8fbb78d8fb6d35753a8ed0f

sudo openssl rsa -in /etc/pki/tls/private/private_sophos-com.key -noout -modulus | openssl md5
# Result: 9ebf48ffe8fbb78d8fb6d35753a8ed0f
```
✅ Match confirmed. Chain file (`cert_chain_edge.cer`) was NOT replaced — same GlobalSign intermediate, valid until 2028.

---

### Step 6: Reloaded Apache

```bash
sudo systemctl reload httpd
```

---

### Step 7: Verified New Cert is Live

```bash
echo | openssl s_client -connect pmuedge.abn.green.sophos:443 -servername pmx-static-uk1.sophos.com 2>/dev/null | openssl x509 -noout -subject -dates -issuer
```
Result:
```
subject= /C=GB/ST=Oxfordshire/L=Abingdon/O=SOPHOS LIMITED/CN=pmx-static-uk1.sophos.com
notBefore=Apr 21 18:06:18 2026 GMT
notAfter=Nov  6 18:06:18 2026 GMT
issuer= /C=BE/O=GlobalSign nv-sa/CN=GlobalSign RSA OV SSL CA 2018
```
✅ New certificate is being served.

---

## Rollback Plan

If anything goes wrong, restore from backup:
```bash
sudo cp /etc/pki/tls/backup_25-26/sslcert_sophos-com.cer /etc/pki/tls/certs/sslcert_sophos-com.cer
sudo cp /etc/pki/tls/backup_25-26/private_sophos-com.key /etc/pki/tls/private/private_sophos-com.key
sudo chmod 644 /etc/pki/tls/certs/sslcert_sophos-com.cer
sudo chmod 400 /etc/pki/tls/private/private_sophos-com.key
sudo systemctl reload httpd
```

---

## Notes
- Apache config is marked "Managed by Puppet" but Puppet agent is **disabled and inactive** on this server — manual changes are safe.
- Next renewal due before **Nov 6, 2026**.
- Previous backups also exist: `sslcert_sophos-com.cer.back24-25` in `/etc/pki/tls/certs/` and `sslcert_sophos-com.cer_backup` from Feb 2023.
