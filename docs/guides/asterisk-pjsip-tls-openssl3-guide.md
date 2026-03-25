# Asterisk PJSIP TLS Broken After OpenSSL 3 Upgrade? Here's the Fix for 'Wrong Curve' and Every Other Handshake Failure

## The Problem Nobody Warned You About

You upgraded your server — maybe a routine `apt upgrade`, maybe a Docker image rebuild, maybe a distro migration. Asterisk was humming along, PJSIP endpoints connecting over TLS on port 5061 like they have for years. Then suddenly:

```
WARNING[12345]: pjproject: SSL SSL_ERROR_SSL (Handshake):
  err: <error:0A00017A:SSL routines::wrong curve>
```

Your phones stop registering. Your ATAs go offline. Your SIP trunks drop. Disabling TLS brings everything back, but that's not a solution — it's a surrender.

This is the OpenSSL 3.x ECDH curve compatibility problem, and it's hit a *lot* of Asterisk administrators who didn't see it coming. The error message is cryptic, the Asterisk logs don't tell you what curve is "wrong," and the fix requires understanding a change that happened deep inside OpenSSL's TLS negotiation logic.

This guide covers exactly what changed, why it breaks specific devices, and the concrete steps to fix it — whether you're running Asterisk 18, 20, 21, or 22.

## What Changed in OpenSSL 3.x

OpenSSL 1.1.x was relatively forgiving about elliptic curve negotiation during TLS handshakes. If a client offered curves that didn't perfectly match the server's certificate key type, OpenSSL would often find a workable combination and proceed.

OpenSSL 3.0 (released September 2021) and its successors introduced **stricter curve validation**. Specifically:

1. **Certificate key curves must match negotiated ECDH curves.** If your TLS certificate uses a P-384 (secp384r1) key but the client only offers P-256 (secp256r1) in its supported groups, OpenSSL 3.x rejects the handshake with "wrong curve." OpenSSL 1.1.x would have silently worked around this in many cases.

2. **Default curve lists changed.** OpenSSL 3.x defaults to `X25519:P-256:P-384:P-521` for key exchange groups. Older devices — especially embedded SIP hardware — may only support `P-256` (secp256r1) or use non-standard curve names.

3. **ECDH auto-selection became stricter.** OpenSSL 3.x validates that the server's ECDSA certificate curve is in the client's supported groups list. If the client doesn't advertise the curve used by the certificate, the handshake fails immediately.

The practical impact: a certificate generated with `openssl ecparam -name secp384r1` will fail to connect with any client that only supports P-256 — and many SIP phones and ATAs fall into this category.

## Why SIP Devices Are Especially Affected

Grandstream, Yealink, Polycom, and other SIP hardware vendors often ship firmware with limited TLS curve support. A Grandstream HT801 or HT802, for example, may only advertise support for:

- `secp256r1` (P-256)
- `secp384r1` (P-384) — sometimes, depending on firmware

It will **not** support X25519 or X448 — the modern curves that OpenSSL 3.x prefers.

The failure sequence looks like this:

1. Your Asterisk server has a TLS certificate with a P-384 ECDSA key (or sometimes RSA + ECDH negotiation triggers the issue)
2. The SIP phone connects to port 5061 and sends its ClientHello with supported curves: `secp256r1`
3. OpenSSL 3.x on Asterisk sees the certificate uses P-384 but the client only supports P-256
4. OpenSSL raises `SSL_ERROR_SSL: wrong curve`
5. pjproject logs the error, the handshake fails, the phone can't register

The cruel part: this often breaks **without any intentional change** on your end. A base OS update, a Docker image rebuild, or even an unattended security patch can swap OpenSSL 1.1.x for 3.x underneath Asterisk.

## Diagnosing the Problem

Before jumping to fixes, confirm you're actually hitting the curve mismatch issue.

### Step 1: Check Your OpenSSL Version

```bash
openssl version
# OpenSSL 3.0.x or 3.1.x or 3.2.x = you are in the danger zone
```

If you're running Asterisk in Docker, check inside the container:

```bash
docker exec -it asterisk openssl version
```

### Step 2: Check Your Certificate Key Type and Curve

```bash
openssl x509 -in /etc/asterisk/keys/asterisk.pem -text -noout | grep -A 2 "Public Key"
```

You'll see something like:

```
Public Key Algorithm: id-ecPublicKey
  Public-Key: (384 bit)
  ASN1 OID: secp384r1
```

or:

```
Public Key Algorithm: rsaEncryption
  Public-Key: (2048 bit)
```

If it's ECDSA with `secp384r1` or `secp521r1`, and your phones only support `secp256r1`, that's your problem.

### Step 3: Capture the TLS Handshake

Use `openssl s_client` to simulate what your phone is doing:

```bash
openssl s_client -connect your-asterisk-ip:5061 \
  -curves secp256r1 \
  -servername your-asterisk-ip \
  -tls1_2 2>&1 | head -30
```

If you see the "wrong curve" error, you've confirmed the issue.

### Step 4: Check Asterisk PJSIP TLS Transport

```bash
asterisk -rx "pjsip show transport <your-tls-transport>"
```

Look at the `cipher` and `method` fields. Note the certificate paths.

### Step 5: Enable Verbose TLS Logging

In `pjsip.conf` or your realtime config, increase logging:

```ini
[global]
debug = yes
```

And in `logger.conf`:

```ini
[logfiles]
full => notice,warning,error,verbose(5)
```

## The Fixes (Pick What Fits Your Setup)

### Fix 1: Regenerate Your Certificate with P-256 (Recommended)

The most reliable fix. P-256 (secp256r1) is universally supported by every SIP device on the market.

**Generate a new P-256 ECDSA key and self-signed certificate:**

```bash
# Generate P-256 private key
openssl ecparam -genkey -name prime256v1 -out /etc/asterisk/keys/asterisk.key

# Generate self-signed certificate (valid 10 years)
openssl req -new -x509 -key /etc/asterisk/keys/asterisk.key \
  -out /etc/asterisk/keys/asterisk.crt \
  -days 3650 \
  -subj "/CN=asterisk.local/O=Asterisk PBX"

# Combine into PEM (what Asterisk expects)
cat /etc/asterisk/keys/asterisk.key /etc/asterisk/keys/asterisk.crt \
  > /etc/asterisk/keys/asterisk.pem

# Set permissions
chown asterisk:asterisk /etc/asterisk/keys/*
chmod 600 /etc/asterisk/keys/asterisk.key
chmod 644 /etc/asterisk/keys/asterisk.crt /etc/asterisk/keys/asterisk.pem
```

**If you are using RSA instead of ECDSA**, use 2048-bit RSA (avoids curve issues entirely):

```bash
openssl req -new -x509 -nodes \
  -newkey rsa:2048 \
  -keyout /etc/asterisk/keys/asterisk.key \
  -out /etc/asterisk/keys/asterisk.crt \
  -days 3650 \
  -subj "/CN=asterisk.local/O=Asterisk PBX"

cat /etc/asterisk/keys/asterisk.key /etc/asterisk/keys/asterisk.crt \
  > /etc/asterisk/keys/asterisk.pem
```

After generating, reload PJSIP:

```bash
asterisk -rx "module reload res_pjsip.so"
```

### Fix 2: Force Specific Curves in pjsip.conf

If you can't regenerate certificates (e.g., using a CA-signed cert), force Asterisk's TLS transport to offer specific curves:

```ini
[tls-transport]
type = transport
protocol = tls
bind = 0.0.0.0:5061
cert_file = /etc/asterisk/keys/asterisk.crt
priv_key_file = /etc/asterisk/keys/asterisk.key
method = tlsv1_2
cipher = ECDHE-ECDSA-AES128-GCM-SHA256,ECDHE-RSA-AES128-GCM-SHA256
```

**Important:** As of Asterisk 22, the PJSIP transport configuration does not have a direct `curves` or `groups` option — curve selection is controlled by the certificate key type and OpenSSL's defaults. This is why Fix 1 (regenerating the cert with P-256) is the most reliable approach.

### Fix 3: Configure OpenSSL System-Wide (openssl.cnf)

You can modify OpenSSL's default behavior system-wide:

```ini
# /etc/ssl/openssl.cnf (or /etc/pki/tls/openssl.cnf on RHEL)

[system_default_sect]
Groups = P-256:P-384:X25519
# Put P-256 first to maximize device compatibility
```

On Debian/Ubuntu with OpenSSL 3.x, you may also need:

```ini
# /etc/ssl/openssl.cnf — at the top
openssl_conf = openssl_init

[openssl_init]
ssl_conf = ssl_sect

[ssl_sect]
system_default = system_default_sect

[system_default_sect]
Groups = P-256:P-384:X25519
CipherSuites = TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256
```

Restart Asterisk after modifying:

```bash
systemctl restart asterisk
```

### Fix 4: Docker-Specific Fix

If Asterisk runs in Docker, the OpenSSL version is determined by the base image, not the host. Check and pin it:

```dockerfile
# Use a specific base image with known OpenSSL version
FROM debian:bookworm-slim

# Check what you got
RUN openssl version

# If needed, install specific OpenSSL config
COPY openssl-asterisk.cnf /etc/ssl/openssl.cnf
```

Or mount your certs and openssl.cnf from the host:

```yaml
# docker-compose.yml
volumes:
  - ./certs:/etc/asterisk/keys:ro
  - ./openssl-asterisk.cnf:/etc/ssl/openssl.cnf:ro
```

The key insight for Docker: when you rebuild the image or pull a new base, OpenSSL can silently upgrade. Pin your base image digest or at minimum track what OpenSSL version each rebuild introduces.

### Fix 5: Downgrade to TLS 1.2 Only

If you're hitting TLS 1.3 negotiation issues specifically (TLS 1.3 has different curve handling than 1.2), force TLS 1.2:

```ini
[tls-transport]
type = transport
protocol = tls
bind = 0.0.0.0:5061
method = tlsv1_2
```

Most SIP devices don't support TLS 1.3 anyway, so this doesn't sacrifice real security.

## Complete Working pjsip.conf TLS Configuration

Here's a battle-tested TLS transport configuration that works with OpenSSL 3.x and the widest range of SIP devices:

```ini
;; TLS Transport — OpenSSL 3.x compatible
[tls-transport]
type = transport
protocol = tls
bind = 0.0.0.0:5061
cert_file = /etc/asterisk/keys/asterisk.crt
priv_key_file = /etc/asterisk/keys/asterisk.key
; ca_list_file is optional for self-signed setups
; ca_list_file = /etc/asterisk/keys/ca.crt
method = tlsv1_2
; ECDHE preferred, RSA fallback, no weak ciphers
cipher = ECDHE-RSA-AES128-GCM-SHA256,AES128-GCM-SHA256
require_client_cert = no
verify_client = no
verify_server = no

;; Endpoint using TLS transport
[my-phone]
type = endpoint
transport = tls-transport
context = internal
disallow = all
allow = ulaw,alaw,g722
media_encryption = sdes
media_encryption_optimistic = yes
```

## Testing After the Fix

After applying your fix, verify it works:

```bash
# 1. Reload PJSIP
asterisk -rx "module reload res_pjsip.so"

# 2. Test with openssl s_client
openssl s_client -connect localhost:5061 -curves secp256r1 -tls1_2

# 3. Check the certificate being served
openssl s_client -connect localhost:5061 2>/dev/null | \
  openssl x509 -text -noout | grep -E "Public Key|ASN1 OID"

# 4. Watch for successful registrations
asterisk -rx "pjsip show registrations"

# 5. Check for any remaining TLS errors
grep -i "SSL\|TLS\|wrong curve\|handshake" /var/log/asterisk/full
```

## The ca_list_file Red Herring

One common source of confusion: the `ca_list_file` setting in PJSIP transport configuration. If you see errors like:

```
ERROR: Failed to read CA list file /etc/asterisk/keys/ca.crt
```

This is often a **red herring** — it won't cause the "wrong curve" error. The CA list is only needed if you're verifying client certificates (mutual TLS) or verifying the server's certificate against a CA chain. For most Asterisk SIP setups with self-signed certs:

```ini
; You can safely omit ca_list_file for self-signed setups
; ca_list_file = /etc/asterisk/keys/ca.crt
require_client_cert = no
verify_client = no
verify_server = no
```

Don't chase CA list errors when the real problem is curve negotiation.

## Preventing This From Happening Again

1. **Pin your OpenSSL version** in Docker images or package management. Know when it changes.
2. **Use P-256 (prime256v1) for all new certificates.** It's the safest choice for device compatibility.
3. **Monitor TLS handshake failures.** Add a cron job or monitoring check:

```bash
# Quick health check — test TLS handshake
echo | openssl s_client -connect localhost:5061 -tls1_2 2>&1 | \
  grep -q "Verify return code" && echo "TLS OK" || echo "TLS BROKEN"
```

4. **Test after every upgrade.** Asterisk, OpenSSL, or base OS — always verify TLS connectivity after any package update.
5. **Keep a working certificate backed up.** Store your last-known-good cert/key pair somewhere recoverable.

## Quick Reference: Curve Compatibility Matrix

| Device/Software | P-256 | P-384 | P-521 | X25519 |
|----------------|-------|-------|-------|--------|
| Grandstream HT80x | Yes | Sometimes | No | No |
| Grandstream GXP21xx | Yes | Yes | No | No |
| Yealink T4x/T5x | Yes | Yes | No | Firmware-dependent |
| Polycom VVX | Yes | Yes | No | No |
| Linphone | Yes | Yes | Yes | Yes |
| Obihai/Obi | Yes | No | No | No |
| Asterisk + OpenSSL 3.x | Yes | Yes | Yes | Yes |

The takeaway: **P-256 is the only curve guaranteed to work with everything.**

## Summary

The "wrong curve" error is OpenSSL 3.x being strict about ECDH curve matching — something OpenSSL 1.1.x was lenient about. The fix is almost always one of:

1. Regenerate your certificate with a P-256 key (`openssl ecparam -name prime256v1`)
2. Use RSA 2048+ instead of ECDSA (avoids curve issues entirely)
3. Configure OpenSSL's default groups to prioritize P-256

Don't panic, don't disable TLS, and don't downgrade OpenSSL. The fix is a 30-second certificate regeneration.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/asterisk-pjsip-tls-openssl3-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
