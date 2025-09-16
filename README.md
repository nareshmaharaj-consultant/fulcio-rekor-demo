# Fulcio + Rekor “Sign a Blob” with Cosign (Keyless & Keyful)

This README shows how to sign an **arbitrary file** (“blob”) with \[cosign], using Sigstore’s **keyless** flow backed by **Fulcio** (short-lived certs) and **Rekor** (transparency log). You’ll also see how to verify the signature, inspect the certificate, look up the Rekor entry, and—optionally—do the same with a traditional (“keyful”) keypair.

> **TL;DR**
> Keyless = ephemeral keys + short-lived Fulcio cert bound to your OIDC identity + Rekor log entry.
> You don’t manage long-lived private keys, and anyone can audit what was signed, by whom, and when.

---

## Table of Contents

* [Prerequisites](#prerequisites)
* [Quickstart (Keyless)](#quickstart-keyless)

  * [1) Create a file](#1-create-a-file)
  * [2) Sign the blob (Fulcio keyless)](#2-sign-the-blob-fulcio-keyless)
  * [3) Inspect the certificate (optional)](#3-inspect-the-certificate-optional)
  * [4) Verify the signature with cosign](#4-verify-the-signature-with-cosign)
  * [5) Verify with OpenSSL (optional)](#5-verify-with-openssl-optional)
  * [6) Inspect the Rekor entry](#6-inspect-the-rekor-entry)
* [Alternative: Keyful (long-lived keypair)](#alternative-keyful-longlived-keypair)
* [How it works (under the hood)](#how-it-works-under-the-hood)
* [Troubleshooting & Tips](#troubleshooting--tips)
* [Cleanup](#cleanup)
* [Appendix: Glossary](#appendix-glossary)

---

## Prerequisites

**macOS (Homebrew)**

```bash
brew install cosign
brew install step          # optional: handy for X.509 introspection
brew install rekor-cli
```

> **Note on identity & privacy**
> With **keyless** signing, your **OIDC identity** (email, GitHub repo/ref, etc.) is embedded in a short-lived certificate and recorded in public logs. Prefer CI/workload identities (e.g., GitHub OIDC) if you don’t want a personal email appearing in logs.

---

## Quickstart (Keyless)

### 1) Create a file

```bash
echo "test file contents" > test-file.txt
cat test-file.txt
# test file contents
```

### 2) Sign the blob (Fulcio keyless)

This generates an **ephemeral** keypair, obtains a **short-lived** Fulcio cert bound to your OIDC identity, and writes a signature + certificate. You’ll be prompted to authenticate in your browser.

```bash
cosign sign-blob test-file.txt \
  --output-certificate fulcio.crt.base64 \
  --output-signature fulcio.sig
```

**Outputs** (files created):

* `fulcio.sig` — base64-encoded signature over `test-file.txt`
* `fulcio.crt.base64` — base64 of your Fulcio X.509 certificate (PEM)

Convert the cert to PEM (optional, for inspection):

```bash
# Linux:
base64 -d fulcio.crt.base64 > fulcio.crt
# macOS alternative:
base64 -D -i fulcio.crt.base64 -o fulcio.crt
```

> You’ll also see a line like:
>
> ```
> tlog entry created with index: 522663709
> ```
>
> Save that **log index** for Rekor lookups later.

### 3) Inspect the certificate (optional)

```bash
step certificate inspect fulcio.crt
```

You should see (values vary):

* **Issuer**: `O=sigstore.dev, CN=sigstore-intermediate`
* **Validity**: \~10 minutes (short-lived)
* **EKU**: `Code Signing`
* **SAN**: `email:<YOUR_EMAIL>` (or a workload identity)
* **Sigstore OIDC Issuer**: e.g. `https://accounts.google.com`
* **SCT**: Signed Certificate Timestamp (CT log proof)

### 4) Verify the signature with cosign

Verify the blob with the signature **and** the Fulcio certificate. Pin the expected identity and issuer:

```bash
cosign verify-blob test-file.txt \
  --signature fulcio.sig \
  --cert fulcio.crt.base64 \
  --certificate-identity <YOUR_EMAIL_OR_SUBJECT> \
  --certificate-oidc-issuer <YOUR_OIDC_ISSUER>
```

Examples:

* `--certificate-identity you@example.com`
* `--certificate-oidc-issuer https://accounts.google.com`
* For GitHub OIDC in CI, identity looks like `https://github.com/<org>/<repo>/...`

Expected output:

```
Verified OK
```

What’s checked:

* Signature over the **exact bytes** of `test-file.txt`
* Cert chain to a trusted Sigstore root (cosign manages these roots via TUF)
* SAN identity & OIDC issuer match what you specified

### 5) Verify with OpenSSL (optional)

This shows it’s “just” X.509 + ECDSA under the hood.

Decode the signature to binary:

```bash
# Linux:
base64 -d fulcio.sig > fulcio.sig.der
# macOS alternative:
base64 -D -i fulcio.sig -o fulcio.sig.der
```

Verify:

```bash
openssl dgst -sha256 \
  -verify <(openssl x509 -in fulcio.crt -pubkey -noout) \
  -signature fulcio.sig.der \
  test-file.txt
# Verified OK
```

### 6) Inspect the Rekor entry

If you captured the **log index** during signing:

```bash
rekor-cli get --rekor_server https://rekor.sigstore.dev --log-index <LOG_INDEX>
```

You’ll see a JSON body like:

```json
{
  "HashedRekordObj": {
    "data": {
      "hash": {"algorithm":"sha256","value":"<SHA256_OF_FILE>"}
    },
    "signature": {
      "content":"<BASE64_SIGNATURE>",
      "publicKey":{"content":"<BASE64_PEM_CERT>"}
    }
  }
}
```

Confirm the file hash matches Rekor:

```bash
# Linux:
sha256sum test-file.txt
# macOS:
shasum -a 256 test-file.txt
```

The printed hash must equal `data.hash.value` in the Rekor record. ✅

---

## Alternative: Keyful (long-lived keypair)

If you need a traditional keypair workflow:

**Generate keys (encrypted private key)**

```bash
cosign generate-key-pair
# creates cosign.key (ENCRYPTED) and cosign.pub
```

**Sign**

```bash
cosign sign-blob test-file.txt --key cosign.key \
  --output-signature keyful.sig
```

**Verify**

```bash
cosign verify-blob test-file.txt \
  --signature keyful.sig \
  --key cosign.pub
```

**Keyful vs Keyless**

* **Keyless**: no long-lived private key to manage; identity bound via Fulcio cert; public Rekor receipt.
* **Keyful**: you manage the private key lifecycle & distribution of the public key; identity binding is not automatic.

---

## How it works (under the hood)

* **Ephemeral keys**: cosign creates a short-lived keypair for each signing.
* **Fulcio**: issues a **short-lived X.509** cert binding that public key to your **OIDC identity** (email, repo/ref, service account).
* **TUF**: cosign securely bootstraps & refreshes Sigstore’s trusted roots via The Update Framework.
* **CT (SCT)**: the certificate is logged to Certificate Transparency; the SCT in your cert proves that.
* **Rekor**: stores an **append-only** record (hash of your file, signature, and cert) so anyone can verify what was signed, by whom, and when.

---

## Troubleshooting & Tips

* **Headless/CI logins**: Use non-interactive OIDC (e.g., GitHub Actions OIDC) or pass `--identity-token`.
* **Identity mismatch**: Ensure `--certificate-identity` and `--certificate-oidc-issuer` exactly match the cert’s SAN & issuer.
* **Proxies/firewalls**: Allow outbound HTTPS to your OIDC provider and `*.sigstore.dev` (Fulcio, Rekor).
* **Reproducibility**: Sign the **exact bytes**. Changing line endings/whitespace changes the hash/signature.
* **Privacy**: Keyless flows publish identity in public logs. Prefer workload identities if needed.

---

## Cleanup

```bash
rm -f fulcio.sig fulcio.crt fulcio.crt.base64 test-file.txt keyful.sig
# Keep cosign.key / cosign.pub only if you intend to use keyful signing later.
```

---

## Appendix: Glossary

* **cosign** — CLI for signing & verifying containers and blobs.
* **Fulcio** — Sigstore CA issuing short-lived certs bound to OIDC identities.
* **Rekor** — Public, append-only transparency log of signing events (stores hashes, not your file).
* **OIDC** — OpenID Connect; identity provider (Google, GitHub, etc.).
* **CT/SCT** — Certificate Transparency / Signed Certificate Timestamp; public logging of X.509 certs.
* **TUF** — The Update Framework; secures trust root distribution/updates.

---

### Copy/Paste Cheatsheet

```bash
# 1) Sign (keyless)
cosign sign-blob test-file.txt \
  --output-certificate fulcio.crt.base64 \
  --output-signature fulcio.sig

# 2) Verify (keyless)
cosign verify-blob test-file.txt \
  --signature fulcio.sig \
  --cert fulcio.crt.base64 \
  --certificate-identity <YOUR_EMAIL_OR_SUBJECT> \
  --certificate-oidc-issuer <YOUR_OIDC_ISSUER>

# 3) Rekor lookup
rekor-cli get --rekor_server https://rekor.sigstore.dev --log-index <LOG_INDEX>

# 4) Hash check
sha256sum test-file.txt     # Linux
shasum -a 256 test-file.txt # macOS

# 5) OpenSSL verify (optional)
base64 -d fulcio.sig > fulcio.sig.der        # Linux
# or: base64 -D -i fulcio.sig -o fulcio.sig.der  # macOS
openssl dgst -sha256 \
  -verify <(openssl x509 -in fulcio.crt -pubkey -noout) \
  -signature fulcio.sig.der \
  test-file.txt
```

---

> **Pro tip for CI**: In GitHub Actions, use OIDC to get a token and run `cosign sign-blob` without a browser. Pin identities with `--certificate-identity-regexp` to match your repo/ref pattern for safer verification.

