# DKIM

## Summary

DomainKeys Identified Mail (DKIM, RFC 6376) lets a sending domain sign outgoing messages with a private key and publish the matching public key in DNS, so that receivers can verify that the message was signed by that domain and that the signed headers and body were not changed in transit. The signature travels in a `DKIM-Signature` header that names the signing domain (`d=`), the key selector (`s=`) and the signed headers; the receiver fetches `selector._domainkey.domain` and checks the signature. Unlike SPF, DKIM is independent of the sending IP, so it survives forwarding as long as the message is not modified. DMARC builds on it by requiring the signing domain to align with the visible From domain.

## Prerequisites

- [Email overview](email-overview.md), [SPF](spf.md) and basic public-key cryptography ([SSL and TLS](../../network-security/ssl-and-tls.md)).

## Core concepts

### Definition and importance

DKIM is an email authentication method that lets a receiving server verify a message with a cryptographic signature. It ensures that:

- the message has not been tampered with in transit (for the signed parts), and
- the message was signed by the claimed domain (the `d=` domain).

Unlike SPF, which verifies sending IP addresses, DKIM uses public-key cryptography to sign and validate messages. It helps prevent spoofing (together with DMARC), improves deliverability, and works with SPF and DMARC for strong email security.

*Analogy.* A wax seal with the company's crest on a letter. Anyone can check the seal against the crest published in the official register (DNS) and see that the letter was sealed by that company and not opened since. The analogy breaks on what is sealed: DKIM seals only the listed headers and the body, and a forger can seal a letter with their own crest, which is why DMARC checks that the crest matches the letterhead.

### How it works

1. **Signing**: the sender's mail server computes a hash of the body and of selected headers, signs it with the domain's private key and adds a `DKIM-Signature` header.
2. **Key lookup**: the receiving server reads `d=` and `s=` from the signature and queries DNS for `s._domainkey.d` (for example `default._domainkey.example.com`).
3. **Validation**: it recomputes the hashes and verifies the signature with the public key:
   - **pass**: the signature matches, so the message is authenticated for the `d=` domain,
   - **fail** or **none**: missing or invalid signature; the message may be rejected or treated as spam (usually decided by DMARC and spam filters).
4. **Processing** continues according to the result.

```mermaid
sequenceDiagram
  participant S as Sending MTA (has private key)
  participant R as Receiving MTA
  participant D as DNS
  S->>S: hash body (bh=), sign headers (b=) with selector "default"
  S->>R: message with DKIM-Signature: d=example.com; s=default
  R->>D: TXT default._domainkey.example.com
  D-->>R: v=DKIM1; k=rsa; p=MIIB...
  R->>R: verify signature and body hash
  R-->>R: dkim=pass header.d=example.com
```

Example header:

```text
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=example.com; s=default;
  h=From:To:Subject:Date:Message-ID;
  bh=WL5zmbD5i6cGd3PZOCS9YYFGg2bfrg5G7dYAlnE4eEU=;
  b=SIJ45Q...==
```

| Tag | Meaning |
| --- | --- |
| `v=1` | Signature version |
| `a=rsa-sha256` | Algorithm (`ed25519-sha256` is also defined, RFC 8463) |
| `c=relaxed/relaxed` | Canonicalization for headers and body: how much whitespace change is tolerated |
| `d=example.com` | Signing domain |
| `s=default` | Selector: locates the public key in DNS and allows several keys per domain |
| `h=` | Signed header fields (From must always be signed) |
| `bh=` | Hash of the body |
| `b=` | The signature itself |

### DNS record structure

```text
default._domainkey.example.com.  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBgkqh..."
```

| Tag | Meaning |
| --- | --- |
| `v=DKIM1` | Record version |
| `k=rsa` | Key type (RSA; `ed25519` also possible) |
| `p=MIIB...` | The public key in Base64; an empty `p=` revokes the key |
| `s=email` | Optional service type: restricts the key to email (`*` means all) |
| `t=y` | Optional flag: the domain is testing DKIM, receivers should not treat failures differently from unsigned mail |
| `t=s` | Optional flag: the `d=` domain must match exactly, no subdomains |

Use 2048-bit RSA keys (1024-bit keys are considered weak; 4096-bit keys exceed what many DNS providers and libraries handle comfortably).

**Signing policy tags.** Some older documentation shows `o=~` (some mail is signed) and `o=-` (all mail is signed) in the key record. These tags come from DomainKeys, the predecessor of DKIM, and are not part of DKIM (RFC 6376); the later ADSP standard for signing policy was declared historic in 2013. Today, a domain states that all its mail must authenticate through DMARC.

### Implementation

**Step 1: generate a key pair** (2048-bit RSA):

```bash
openssl genpkey -algorithm RSA -out dkim_private.key -pkeyopt rsa_keygen_bits:2048
openssl rsa -in dkim_private.key -pubout -out dkim_public.key
```

**Step 2: publish the public key** in DNS. Remove the PEM header and footer lines and line breaks:

```bash
grep -v -- '-----' dkim_public.key | tr -d '\n'
```

```text
default._domainkey.example.com  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBgkqh..."
```

A 2048-bit key exceeds the 255-character limit of a single TXT string, so the value is split into several quoted strings in one record (`"v=DKIM1; k=rsa; p=MIIBIj..." "...rest..."`), which receivers concatenate; most DNS provider interfaces do this automatically.

**Step 3: configure signing**, for example Postfix with OpenDKIM (`/etc/opendkim.conf`, one option per line):

```text
Domain      example.com
KeyFile     /etc/opendkim/keys/example.com/default.private
Selector    default
Socket      inet:8891@localhost
```

```bash
sudo systemctl restart opendkim
```

Hosted providers (Google Workspace, Microsoft 365, SendGrid, Amazon SES) generate keys and show the records to publish; Microsoft and SES use CNAME records pointing to provider-hosted keys, which lets the provider rotate keys.

**Step 4: test** that the record is published and that signatures verify:

```bash
dig TXT default._domainkey.example.com +short
# "v=DKIM1; k=rsa; p=MIIBIjANBgkqh..."
opendkim-testkey -d example.com -s default -vvv
```

Then send a message to a mailbox you control and check `Authentication-Results: ... dkim=pass header.d=example.com`.

### Limitations and solutions

| Issue | Cause | Solution |
| --- | --- | --- |
| Messages fail DKIM | Wrong private key or a selector that does not match the published key | Check the selector and key pair; publish before signing |
| Key record truncated or rejected | A single TXT string is limited to 255 characters | Split the value into several strings in one record |
| DKIM fails after passing through mailing lists or gateways | Intermediaries modify the subject, footer or body, so the signature no longer matches (plain forwarding without changes keeps DKIM valid) | Relaxed canonicalization, ARC on intermediaries, DMARC that also accepts aligned SPF |
| Stale keys | Keys never rotated | Rotate with a new selector, publish, switch signing, then remove the old key |
| DKIM pass but phishing | Attackers sign with their own domain | DMARC alignment between `d=` and the From domain |

### DKIM, SPF and DMARC

| Feature | SPF | DKIM | DMARC |
| --- | --- | --- | --- |
| Purpose | Verifies sending IPs | Verifies message integrity and signing domain | Defines handling policies and reports |
| Protects against | Envelope spoofing | Tampering, forged signatures | Spoofing of the visible From domain, phishing |
| Works with | SMTP envelope | Message content | SPF and DKIM |
| DNS location | TXT at the domain | TXT at `selector._domainkey.domain` | TXT at `_dmarc.domain` |

## Worked example

Key rotation without downtime. Untested; OpenDKIM and dig, checked 2026-11.

| Day | Action | Effect |
| --- | --- | --- |
| 0 | Generate a new key with selector `2026q4`, publish `2026q4._domainkey.example.com` | Old selector still signs |
| 2 | Verify the new record from several resolvers; switch OpenDKIM `Selector` to `2026q4` and restart | New mail signed with the new key |
| 9 | After the longest expected delivery and verification window, revoke the old key with `p=` (empty) | Old key unusable |
| 30 | Delete the old record | Clean zone |

Date-based selectors make the active key obvious and allow overlap.

## Trade offs and when to use it

- **DKIM signing at every sending service** is essential for DMARC; each third-party sender needs its own selector and key.
- **Longer keys** are stronger but produce larger DNS records; 2048-bit RSA is the current baseline.
- **Strict canonicalization** detects more changes but breaks with harmless reformatting; relaxed is the common default.

## Common mistakes

- **Publishing the private key** or keeping it world-readable on the server. Fix: permissions `600`, only the signing service user.
- **Forgetting that `d=` must align with From** for DMARC. Fix: sign with your own domain at third-party senders (custom DKIM).
- **Relying on `o=-` to enforce signing.** It is not a DKIM tag. Fix: DMARC.

## Practice

1. Where does a receiver find the public key for `d=example.com; s=mail2026`?
2. What does the `bh=` tag contain?
3. Does plain forwarding break DKIM?
4. Why can a phishing message pass DKIM?

Answers:

1. In the TXT record at `mail2026._domainkey.example.com`.
2. The hash of the canonicalized message body.
3. Not if the message is unchanged; DKIM does not depend on the sending IP. Modifications by intermediaries break it.
4. The attacker signs it with a domain they control; only DMARC alignment requires the signing domain to match the visible From domain.

## Further reading

- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376) (IETF, 2011).
- [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301) (IETF, 2018).
- [RFC 8463: A New Cryptographic Signature Method for DKIM (Ed25519)](https://www.rfc-editor.org/rfc/rfc8463) (IETF, 2018).
- [OpenDKIM](http://www.opendkim.org/) (OpenDKIM project).
