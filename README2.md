<div id="top"></div>

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

<details open>
<summary><strong>Prerequisites</strong></summary>

## Prerequisites

**macOS (Homebrew)**

```bash
brew install cosign
brew install step          # optional: handy for X.509 introspection
brew install rekor-cli
```

> **Note on identity & privacy**
> With **keyless** signing, your **OIDC identity** (email, GitHub repo/ref, etc.) is embedded in a short-lived certificate and recorded in public logs. Prefer CI/workload identities (e.g., GitHub OIDC) if you don’t want a personal email appearing in logs.

</details>

---

<details open>
<summary><strong>Quickstart (Keyless)</strong></summary>

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

</details>

---

<details open>
<summary><strong>Alternative: Keyful (long-lived keypair)</strong></summary>

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

</details>

---

<details open>
<summary><strong>How it works (under the hood)</strong></summary>

## How it works (under the hood)

* **Ephemeral keys**: cosign creates a short-lived keypair for each signing.
* **Fulcio**: issues a **short-lived X.509** cert binding that public key to your **OIDC identity** (email, repo/ref, service account).
* **TUF**: cosign securely bootstraps & refreshes Sigstore’s trusted roots via The Update Framework.
* **CT (SCT)**: the certificate is logged to Certificate Transparency; the SCT in your cert proves that.
* **Rekor**: stores an **append-only** record (hash of your file, signature, and cert) so anyone can verify what was signed, by whom, and when.

</details>

---

<details open>
<summary><strong>Troubleshooting & Tips</strong></summary>

## Troubleshooting & Tips

* **Headless/CI logins**: Use non-interactive OIDC (e.g., GitHub Actions OIDC) or pass `--identity-token`.
* **Identity mismatch**: Ensure `--certificate-identity` and `--certificate-oidc-issuer` exactly match the cert’s SAN & issuer.
* **Proxies/firewalls**: Allow outbound HTTPS to your OIDC provider and `*.sigstore.dev` (Fulcio, Rekor).
* **Reproducibility**: Sign the **exact bytes**. Changing line endings/whitespace changes the hash/signature.
* **Privacy**: Keyless flows publish identity in public logs. Prefer workload identities if needed.

</details>

---

<details open>
<summary><strong>Cleanup</strong></summary>

## Cleanup

```bash
rm -f fulcio.sig fulcio.crt fulcio.crt.base64 test-file.txt keyful.sig
# Keep cosign.key / cosign.pub only if you intend to use keyful signing later.
```

</details>

---

<details open>
<summary><strong>Appendix: Glossary</strong></summary>

## Appendix: Glossary

* **cosign** — CLI for signing & verifying containers and blobs.
* **Fulcio** — Sigstore CA issuing short-lived certs bound to OIDC identities.
* **Rekor** — Public, append-only transparency log of signing events (stores hashes, not your file).
* **OIDC** — OpenID Connect; identity provider (Google, GitHub, etc.).
* **CT/SCT** — Certificate Transparency / Signed Certificate Timestamp; public logging of X.509 certs.
* **TUF** — The Update Framework; secures trust root distribution/updates.

</details>

---

<details open>
<summary><strong>Copy/Paste Cheatsheet</strong></summary>

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

</details>

---

<details open>
<summary><strong>Publishing, Metadata & Verification Extras</strong></summary>

### What to Keep, publish and store as metadata

## Quick inventory — what each file is (and whether to publish it)

| File                               | What it is                                                            | Keep/Publish?                   | Why                                                                                                                          |
| ---------------------------------- | --------------------------------------------------------------------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `test-file.txt`                    | The artifact (payload)                                                | **Publish**                     | The thing users actually want.                                                                                               |
| `fulcio.sig`                       | Base64 **signature** over the exact bytes of `test-file.txt`          | **Publish**                     | Needed to verify.                                                                                                            |
| `fulcio.crt` / `fulcio.crt.base64` | **Fulcio X.509 cert** for the *ephemeral* key used in keyless signing | **Publish** (pick one)          | Verifiers need the signer’s cert. PEM (`.crt`) is friendliest.                                                               |
| `cosign.key`                       | Your **encrypted private key** (from `cosign generate-key-pair`)      | **Do NOT publish**              | Secret. Not used in your keyless signature.                                                                                  |
| `cosign.pub`                       | Public key that pairs with `cosign.key`                               | **Don’t publish (for keyless)** | Not used for keyless verification; including it just confuses people. Publish only if you also offer a **keyful** signature. |
| `fulcio.sig.der`                   | Binary form of the signature (for OpenSSL demo)                       | **Don’t publish**               | Debug-only; `fulcio.sig` is enough.                                                                                          |

> TL;DR for your repo: publish **`test-file.txt` + `fulcio.sig` + `fulcio.crt`**. Exclude `cosign.key`, `cosign.pub` (unless you do keyful too), and `fulcio.sig.der`.

---

## 1) Suggested repo layout

Two solid ways to organize. Pick **A** (modern & tidy) or **B** (your current files).

### A) Recommended (bundle)

Cosign can write a single **Sigstore bundle** that includes the signature, cert, and Rekor proof:

```bash
cosign sign-blob test-file.txt --bundle test-file.sigstore
```

**Publish:**

```
my-app-repo/
  artifacts/test-file.txt
  signatures/test-file.sigstore
  README.md
```

**Verify (downloaders):**

```bash
cosign verify-blob artifacts/test-file.txt \
  --bundle signatures/test-file.sigstore \
  --certificate-identity nmaharaj@clavium.io \
  --certificate-oidc-issuer https://accounts.google.com
```

### B) Another approach using separate files

**Publish:**

```
my-app-repo/
  artifacts/test-file.txt
  signatures/test-file.sig
  certs/test-file.crt
  checksums/test-file.txt.sha256        # optional but nice
  README.md
```

Create the checksum file:

```bash
shasum -a 256 test-file.txt | awk '{print $1}' > checksums/test-file.txt.sha256
```

**Verify (downloaders):**

```bash
# 1) Check SHA256 (optional, if you publish it)
shasum -a 256 artifacts/test-file.txt
# expect: 8376f260fca20effdf3c7b66f2f56d7f118b532e6712af09a4f8922ccab58c5c

# 2) Verify signature with Fulcio cert and pinned identity/issuer
cosign verify-blob artifacts/test-file.txt \
  --signature signatures/test-file.sig \
  --cert certs/test-file.crt \
  --certificate-identity nmaharaj@clavium.io \
  --certificate-oidc-issuer https://accounts.google.com
# -> "Verified OK"
```

> Note: The cert is **short-lived**, but verification remains valid because the signature was **transparency-logged** at signing time (Rekor). You’re checking *who* signed *what* and *when*, not whether the cert is currently unexpired.

---

## 2) What to store in your database (and why)

* `reponame: my-app-repo`
* `artifact_path: artifact_url`
* `app-file: test-file.txt`
* `cert: fulcio.crt`
* `cert_pem_base64: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5ekNDQWxHZ0F3SUJBZ0lVSkVZZG0yU2JXOUhqaXBtNWxkeENGMHB3U09rd0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpVd09URTJNRFl5T0RFNFdoY05NalV3T1RFMk1EWXpPREU0V2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUVJWUVnWk0zSlBVeStoQU5rT1VSVjZoeS9JOTZjVmJEb1ZJMVQKbVJpckpFSXJ0VE42bFBCYnhBeDl4WXhOWXA3VmxCcE96UHNKTzVDZ0l4SEVnSE51ZnFPQ0FYQXdnZ0ZzTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVaYU43CkRBbmZJdS9HWDNYTlNsc1BhMit1Skdrd0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d0lRWURWUjBSQVFIL0JCY3dGWUVUYm0xaGFHRnlZV3BBWTJ4aGRtbDFiUzVwYnpBcEJnb3JCZ0VFQVlPLwpNQUVCQkJ0b2RIUndjem92TDJGalkyOTFiblJ6TG1kdmIyZHNaUzVqYjIwd0t3WUtLd1lCQkFHRHZ6QUJDQVFkCkRCdG9kSFJ3Y3pvdkwyRmpZMjkxYm5SekxtZHZiMmRzWlM1amIyMHdnWWtHQ2lzR0FRUUIxbmtDQkFJRWV3UjUKQUhjQWRRRGRQVEJxeHNjUk1tTVpIaHlaWnpjQ29rcGV1TjQ4cmYrSGluS0FMeW51amdBQUFabFJOYmRoQUFBRQpBd0JHTUVRQ0lHZmI4bHRNa2luYTNvQnByQWVqQWJva01GamFhZzc1ZHV6ODl4RlFHS1RFQWlBbTFtS2lTa2pHClRMMjFVQU1RbU5UNWFOTTNtYzBwRWxlRW1Qek9EOGg2Z3pBS0JnZ3Foa2pPUFFRREF3Tm9BREJsQWpFQWxhVFIKZWRHZjlRUEpJeWVvWi9FamJ0YmtwNWNKa3lTeXVlR09tZnZHM1BIenZzMUdSZTltdnd3b1AzcFEyRVlrQWpCagp1alRMbWZHbTVxS3FvRGxLNUNPbGxSbDdXa3lLYW1DV2gxSHo4ZU5Rc2JXRm0zUlphUzNKU01jRjFDb01rTmM9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K`
* `signature: fulcio.sig`
* `signature_not before: Sep 16 06:28:18 2025 UTC`
* `signature_not_after: Sep 16 06:38:18 2025 UTC`
* `signature_base64: TUVVQ0lRRHdDVmNLWENTN2tYQ08zdHlIQmF0YzBPekhRNWFaSWxncWYzYURnTDRaSXdJZ1BveEY5NGZFZHYzSHRUc3ZOU0M0TjhJRDYyVXZpaG0welhTbzNhbnZmbWM9`
* `log-index: 522663709`
* `sha256: 8376f260fca20effdf3c7b66f2f56d7f118b532e6712af09a4f8922ccab58c5c`
* `certificate_pem` — store the PEM bytes (or a blob reference).
* `certificate_identity: nmaharaj@clavium.io`
* `certificate_oidc_issuer: https://accounts.google.com`
* `rekor_index: 522663709`
* `rekor_uuid: 108e9186e8c5677a32635644ebf547c3d66dbe5a0b89a359c80919eb5be03635e8a5f09ce2daa766` # (more portable than index).
* `rekor_log_id: c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d` # identifies the log instance.
* `rekor_integrated_time: 2025-09-16T06:28:19Z` # when the entry landed in Rekor.

### Example SQL (Postgres-ish)

```sql
CREATE TABLE signed_blobs (
  id BIGSERIAL PRIMARY KEY,
  repo TEXT NOT NULL,
  artifact_path TEXT NOT NULL,
  sha256 TEXT NOT NULL,
  signature_base64 TEXT NOT NULL,
  certificate_pem TEXT NOT NULL,
  certificate_identity TEXT NOT NULL,
  certificate_oidc_issuer TEXT NOT NULL,
  cert_not_before TIMESTAMPTZ,
  cert_not_after  TIMESTAMPTZ,
  rekor_index BIGINT,
  rekor_uuid TEXT,
  rekor_log_id TEXT,
  rekor_integrated_time TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### Example NoSQL JSON (MongoDb-ish)

```json
{
  "repo": "my-app-repo",
  "artifact_path": "artifact_url",
  "sha256": "8376f260fca20effdf3c7b66f2f56d7f118b532e6712af09a4f8922ccab58c5c",
  "signature_base64": "TUVVQ0lRRHdDVmNLWENTN2tYQ08zdHlIQmF0YzBPekhRNWFaSWxncWYzYURnTDRaSXdJZ1BveEY5NGZFZHYzSHRUc3ZOU0M0TjhJRDYyVXZpaG0welhTbzNhbnZmbWM9",
  "certificate_pem_base64": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5ekNDQWxHZ0F3SUJBZ0lVSkVZZG0yU2JXOUhqaXBtNWxkeENGMHB3U09rd0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpVd09URTJNRFl5T0RFNFdoY05NalV3T1RFMk1EWXpPREU0V2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUVJWUVnWk0zSlBVeStoQU5rT1VSVjZoeS9JOTZjVmJEb1ZJMVQKbVJpckpFSXJ0VE42bFBCYnhBeDl4WXhOWXA3VmxCcE96UHNKTzVDZ0l4SEVnSE51ZnFPQ0FYQXdnZ0ZzTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVaYU43CkRBbmZJdS9HWDNYTlNsc1BhMit1Skdrd0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d0lRWURWUjBSQVFIL0JCY3dGWUVUYm0xaGFHRnlZV3BBWTJ4aGRtbDFiUzVwYnpBcEJnb3JCZ0VFQVlPLwpNQUVCQkJ0b2RIUndjem92TDJGalkyOTFiblJ6TG1kdmIyZHNaUzVqYjIwd0t3WUtLd1lCQkFHRHZ6QUJDQVFkCkRCdG9kSFJ3Y3pvdkwyRmpZMjkxYm5SekxtZHZiMmRzWlM1amIyMHdnWWtHQ2lzR0FRUUIxbmtDQkFJRWV3UjUKQUhjQWRRRGRQVEJxeHNjUk1tTVpIaHlaWnpjQ29rcGV1TjQ4cmYrSGluS0FMeW51amdBQUFabFJOYmRoQUFBRQpBd0JHTUVRQ0lHZmI4bHRNa2luYTNvQnByQWVqQWJva01GamFhZzc1ZHV6ODl4RlFHS1RFQWlBbTFtS2lTa2pHClRMMjFVQU1RbU5UNWFOTTNtYzBwRWxlRW1Qek9EOGg2Z3pBS0JnZ3Foa2pPUFFRREF3Tm9BREJsQWpFQWxhVFIKZWRHZjlRUEpJeWVvWi9FamJ0YmtwNWNKa3lTeXVlR09tZnZHM1BIenZzMUdSZTltdnd3b1AzcFEyRVlrQWpCagp1alRMbWZHbTVxS3FvRGxLNUNPbGxSbDdXa3lLYW1DV2gxSHo4ZU5Rc2JXRm0zUlphUzNKU01jRjFDb01rTmM9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K",
  "certificate_identity": "nmaharaj@clavium.io",
  "certificate_oidc_issuer": "https://accounts.google.com",
  "cert_not_before": "2025-09-16T06:28:18Z",
  "cert_not_after": "2025-09-16T06:38:18Z",
  "rekor_index": 522663709,
  "rekor_uuid": "108e9186e8c5677a32635644ebf547c3d66dbe5a0b89a359c80919eb5be03635e8a5f09ce2daa766",
  "rekor_log_id": "c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d",
  "rekor_integrated_time": "2025-09-16T06:28:19Z"
}
```

---

## 3) What a verifier (or CI) should do with your DB row

Given one DB record, your verification routine is:

1. **Fetch** the artifact and its **signature** and **cert** from the repo.
2. **Hash** the artifact and compare to DB `sha256`.
3. **cosign verify-blob** using the **signature**, **cert**, and DB-pinned **identity** & **issuer**.
4. (Optional but good) **Cross-check Rekor**:

   * `rekor-cli get ...` by `rekor_uuid` or `rekor_index`.
   * Confirm `data.hash.value` equals the DB `sha256`.
   * Confirm the `publicKey.content` (base64-PEM) equals your stored `certificate_pem`.

If all checks pass → accept; else → fail the build/release.

---

## 4) Naming & housekeeping

* Name files consistently so you can scale:

  ```
  artifacts/<name>.txt
  signatures/<name>.sig
  certs/<name>.crt
  checksums/<name>.sha256
  ```
* Add a `README.md` with the **exact verify command** (already shown above) and the expected SHA256 line.
* **Never** commit `cosign.key`. Add to `.gitignore`:

  ```
  cosign.key
  *.sig.der
  ```
* If you *also* want to offer **keyful** verification, publish **another signature** made with `cosign.key` and include `cosign.pub`, clearly labelling it:

  ```
  signatures/<name>.keyful.sig
  keys/cosign.pub
  ```

---

## 5) One-line verify command (copy/paste for consumers)

```bash
cosign verify-blob artifacts/test-file.txt \
  --signature signatures/test-file.sig \
  --cert certs/test-file.crt \
  --certificate-identity nmaharaj@clavium.io \
  --certificate-oidc-issuer https://accounts.google.com
```

(Optional Rekor cross-check)

```bash
rekor-cli get --rekor_server https://rekor.sigstore.dev --log-index 522663709
shasum -a 256 artifacts/test-file.txt
# expect: 8376f260fca20effdf3c7b66f2f56d7f118b532e6712af09a4f8922ccab58c5c
```

---

### Bottom line

* Publish: **artifact + signature + Fulcio cert** (or a single **`.sigstore` bundle**).
* Store in DB: **hash**, **identity**, **issuer**, **cert**, and **Rekor details**.
* Verify by: hashing → `cosign verify-blob` with pinned identity/issuer → Rekor cross-check (optional).

</details>

---
