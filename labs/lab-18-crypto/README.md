# Lab 18 — Cryptography: GPG, LUKS, OpenSSL, WireGuard

This lab maps to Linux+ XK0-006 Objective 3.5. You will generate a GPG key non-interactively, encrypt and sign files, build a LUKS2 encrypted volume on a loopback device, create an OpenSSL self-signed certificate, compute hashes and HMACs, generate a WireGuard key pair with a sample config, and probe TLS versions on a remote service.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — GPG key generation in batch mode

```bash
apt update && apt install -y gnupg
mkdir -p /root/.gnupg && chmod 700 /root/.gnupg
cat >/tmp/gpgparams <<'EOF'
%no-protection
Key-Type: RSA
Key-Length: 4096
Name-Real: Lab
Name-Email: lab@example.com
Expire-Date: 0
%commit
EOF
gpg --batch --generate-key /tmp/gpgparams
gpg --list-keys
```

`%no-protection` skips passphrase prompts so the lab is unattended. `Expire-Date: 0` makes the key non-expiring.

---

## Step 2 — Encrypt, decrypt, and detached-sign

```bash
echo "top secret payload" > /tmp/msg.txt
gpg --yes --trust-model always -r lab@example.com -e /tmp/msg.txt
gpg --yes -d /tmp/msg.txt.gpg
gpg --yes --detach-sign /tmp/msg.txt
gpg --verify /tmp/msg.txt.sig /tmp/msg.txt
```

The `.gpg` ciphertext is decrypted with the matching private key. A detached signature (`.sig`) lets recipients verify integrity without altering the original file.

---

## Step 3 — LUKS2 on a loopback device

```bash
apt install -y cryptsetup
dd if=/dev/zero of=/tmp/luks.img bs=1M count=64
LOOP=$(losetup -f --show /tmp/luks.img)
echo -n "Passw0rd!" | cryptsetup luksFormat --type luks2 "$LOOP" -
echo -n "Passw0rd!" | cryptsetup open "$LOOP" labvol -
mkfs.ext4 /dev/mapper/labvol
mkdir -p /mnt/lab && mount /dev/mapper/labvol /mnt/lab
echo "data at rest" > /mnt/lab/file
umount /mnt/lab && cryptsetup close labvol && losetup -d "$LOOP"
```

LUKS2 is the modern on-disk format; the header carries key slots and Argon2 parameters. Closing the mapping and detaching the loop returns the file to opaque ciphertext.

---

## Step 4 — OpenSSL keys and self-signed certificate

```bash
apt install -y openssl
openssl genrsa -out /tmp/key.pem 2048
openssl req -x509 -newkey rsa:2048 -nodes -keyout /tmp/srv.key \
  -out /tmp/cert.pem -days 365 -subj "/CN=lab.example.com"
openssl x509 -in /tmp/cert.pem -noout -text | head -n 25
```

`req -x509` produces a certificate in one step. `x509 -text` decodes the cert so you can inspect issuer, subject, validity, and SAN extensions.

---

## Step 5 — Hashing and HMAC

```bash
sha256sum /tmp/msg.txt
openssl dgst -sha256 /tmp/msg.txt
openssl dgst -sha256 -hmac "secret-key" /tmp/msg.txt
```

A hash provides integrity only; an HMAC binds the hash to a shared secret, providing integrity plus authentication.

---

## Step 6 — WireGuard key pair and sample config

```bash
apt install -y wireguard-tools
umask 077
wg genkey | tee /etc/wireguard/priv | wg pubkey > /etc/wireguard/pub
cat >/etc/wireguard/wg0.conf <<EOF
[Interface]
PrivateKey = $(cat /etc/wireguard/priv)
Address = 10.20.0.1/24
ListenPort = 51820

[Peer]
PublicKey = REPLACE_WITH_PEER_PUBKEY
AllowedIPs = 10.20.0.2/32
EOF
cat /etc/wireguard/wg0.conf
```

WireGuard uses Curve25519 keys. Bringing the tunnel up would be `wg-quick up wg0`, but we stop at the config since Killercoda has no peer.

---

## Step 7 — TLS version and cipher inspection

```bash
echo | openssl s_client -connect example.com:443 -tls1_2 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates
echo "TLS 1.0 and 1.1 are deprecated; prefer TLS 1.2 and 1.3."
echo "Avoid RC4, 3DES, MD5, and any 'EXPORT' cipher suites."
```

`s_client` prints the negotiated cert chain. Server policies should disable legacy versions and weak ciphers like RC4 or 3DES.

---

## Step 8 — Cleanup

```bash
gpg --batch --yes --delete-secret-keys lab@example.com 2>/dev/null
gpg --batch --yes --delete-keys lab@example.com 2>/dev/null
rm -f /tmp/msg.txt* /tmp/gpgparams /tmp/key.pem /tmp/cert.pem /tmp/srv.key /tmp/luks.img
rm -rf /etc/wireguard
apt purge -y wireguard-tools cryptsetup gnupg
```

All keys, certificates, and the loopback image are destroyed.

---

## What you learned
- GPG batch key creation, encryption, and detached signatures
- LUKS2 disk encryption end-to-end on a loopback device
- The difference between hashes and HMACs, and how to spot weak TLS configurations

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- GnuPG — https://gnupg.org/
- cryptsetup / LUKS — https://gitlab.com/cryptsetup/cryptsetup
- OpenSSL — https://www.openssl.org/
- WireGuard — https://www.wireguard.com/
