# RUBE/1.0: Ridiculously Universal Byzantine Exchange Protocol

**Document:** `draft-rube-protocol-01`  
**Category:** Experimental  
**Intended status:** Internet-Draft-style specification  
**Version:** 1.0  
**Date:** 2026-07-22

## Status of This Memo

This document defines RUBE/1.0, a distributed, multi-transport, cryptographically witnessed protocol for communicating one authenticated Boolean value.

Implementation is permitted. Deployment is inadvisable.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are normative.

---

## 1. Abstract

RUBE/1.0 communicates exactly one Boolean value between an Originator and a Recipient by expanding that value into a canonical transaction capsule, wrapping the capsule through multiple serialization formats, encoding the result as a QR image, erasure-coding the image into shards, distributing those shards through unrelated protocols, publishing locator metadata through DNS, validating possession through an SMTP-to-NTP challenge, reaching agreement among three verifier nodes, recording the result in a Merkle transparency log, obtaining an independent witness signature, printing and rescanning an acknowledgement, applying OCR, and confirming the OCR digest through DTMF tones.

The final application result MUST remain unavailable until every mandatory phase has completed.

The protocol exists to ensure that a function logically equivalent to:

```text
bool transmit(bool value)
```

requires a distributed, cryptographically witnessed, cross-protocol, print-mediated consensus ceremony.

---

## 2. Scope

RUBE/1.0 supports exactly two application values:

```text
TRUE
FALSE
```

A transaction MUST carry exactly one Boolean. It MUST NOT carry a string, integer, null, array, map, stream, partial truth value, probability, or additional application payload.

A successfully completed transaction returns an application-level status of `OK` regardless of whether the transmitted Boolean is `TRUE` or `FALSE`. The Boolean itself is returned separately in the authenticated result object.

---

## 3. Design Requirements

A conforming implementation MUST provide:

1. authenticated origin;
2. payload integrity;
3. transaction uniqueness;
4. replay resistance;
5. erasure-coded redundancy;
6. cross-protocol distribution;
7. verifier consensus;
8. append-only transparency logging;
9. independent witnessing;
10. print-scan-OCR acknowledgement;
11. acoustic completion confirmation;
12. architectural disproportionality.

A conforming implementation MUST NOT optimize away a mandatory phase merely because the Boolean has already been recovered.

---

## 4. Conventions and Core ABNF

The notation in this section uses Augmented Backus-Naur Form.

```abnf
ALPHA          = %x41-5A / %x61-7A
DIGIT          = %x30-39
HEXDIG         = DIGIT / %x41-46 / %x61-66
SP             = %x20
HTAB           = %x09
WSP            = SP / HTAB
CRLF           = %x0D.0A
DQUOTE         = %x22
COLON          = %x3A
HYPHEN         = %x2D
DOT            = %x2E
SLASH          = %x2F
EQUAL          = %x3D
SEMICOLON      = %x3B
COMMA          = %x2C

uuid           = 8HEXDIG HYPHEN 4HEXDIG HYPHEN 4HEXDIG HYPHEN
                 4HEXDIG HYPHEN 12HEXDIG

bool-token     = "TRUE" / "FALSE"
result-token   = "ACCEPT_TRUE" / "ACCEPT_FALSE" /
                 "REJECT" / "INDETERMINATE"
state-token    = "NEW" / "NEGOTIATING" / "CAPSULE_CREATED" /
                 "ENVELOPE_CONSTRUCTED" / "GLYPH_GENERATED" /
                 "SHARDS_DISTRIBUTED" / "LOCATORS_PUBLISHED" /
                 "DISCOVERY_ANNOUNCED" / "SHARDS_RETRIEVED" /
                 "PAYLOAD_RECONSTRUCTED" / "CHALLENGE_PENDING" /
                 "CONSENSUS_PENDING" / "LOGGED" / "WITNESSED" /
                 "ACK_PRINTED" / "ACK_SCANNED" / "TONE_PENDING" /
                 "COMPLETE" / "FAILED"

sha256-text    = "sha256:" 64HEXDIG
base64-char    = ALPHA / DIGIT / "+" / "/"
base64-text    = 1*base64-char *2EQUAL

rfc3339-utc    = 4DIGIT HYPHEN 2DIGIT HYPHEN 2DIGIT "T"
                 2DIGIT COLON 2DIGIT COLON 2DIGIT "Z"
```

UUID hex digits MUST be lowercase when serialized as text. Comparisons MUST be case-insensitive only at parsing boundaries; all emitted values MUST be lowercase.

All multi-byte integers in binary records MUST use unsigned network byte order unless explicitly stated otherwise.

---

## 5. Terminology

**Originator:** The endpoint transmitting the Boolean.

**Recipient:** The endpoint receiving the Boolean.

**Verifier:** One of exactly three voting nodes that validates a proposed result.

**Witness:** An entity administratively independent from the Transparency Log that signs a finalized tree head.

**Transaction:** One complete RUBE exchange identified by a UUID version 4.

**Capsule:** The canonical JSON representation of the Boolean and transaction metadata.

**Envelope Stack:** The gzip, Base64, SOAP, MIME, and DKIM representation of the Capsule.

**Glyph:** The lossless QR image containing the Envelope Stack.

**Shard:** An erasure-coded binary fragment of the Glyph.

**Locator Manifest:** A CBOR document describing shard locations without containing shard payload bytes.

**Acknowledgement Artifact:** A printer-ready document containing the authenticated result.

**Completion Tone:** A DTMF sequence derived from OCR output.

---

## 6. Participants and Administrative Separation

A minimally conforming deployment requires:

- one Originator;
- one Recipient;
- exactly three Verifiers;
- one Transparency Log;
- one independent Witness;
- one authoritative DNS service;
- at least four shard-hosting administrative domains;
- one SMTP service;
- one XMPP service;
- one Matrix homeserver;
- one SSH endpoint;
- one NTP-capable responder;
- one print spooler;
- one rasterizer or physical printer;
- one scanner or camera;
- one OCR engine;
- one DTMF sender;
- one DTMF receiver.

The Witness MUST NOT be operated by the Transparency Log operator. At least one shard MUST be hosted outside the administrative control of both the Originator and Recipient.

Logical roles MAY share a process except where this section requires administrative independence.

---

## 7. Transaction State Machine

### 7.1 Legal Transitions

```text
NEW -> NEGOTIATING
NEGOTIATING -> CAPSULE_CREATED
CAPSULE_CREATED -> ENVELOPE_CONSTRUCTED
ENVELOPE_CONSTRUCTED -> GLYPH_GENERATED
GLYPH_GENERATED -> SHARDS_DISTRIBUTED
SHARDS_DISTRIBUTED -> LOCATORS_PUBLISHED
LOCATORS_PUBLISHED -> DISCOVERY_ANNOUNCED
DISCOVERY_ANNOUNCED -> SHARDS_RETRIEVED
SHARDS_RETRIEVED -> PAYLOAD_RECONSTRUCTED
PAYLOAD_RECONSTRUCTED -> CHALLENGE_PENDING
CHALLENGE_PENDING -> CONSENSUS_PENDING
CONSENSUS_PENDING -> LOGGED
LOGGED -> WITNESSED
WITNESSED -> ACK_PRINTED
ACK_PRINTED -> ACK_SCANNED
ACK_SCANNED -> TONE_PENDING
TONE_PENDING -> COMPLETE
```

Every nonterminal state MAY transition to `FAILED`.

No other transition is valid.

### 7.2 State Invariants

An implementation MUST enforce the following invariants:

```text
state >= CAPSULE_CREATED       => capsule_sha256 exists
state >= ENVELOPE_CONSTRUCTED  => dkim_signature exists
state >= GLYPH_GENERATED       => glyph_sha256 exists
state >= SHARDS_DISTRIBUTED    => total_shards >= 10
state >= SHARDS_RETRIEVED      => valid_shards >= threshold
state >= PAYLOAD_RECONSTRUCTED => recovered_boolean exists
state >= CONSENSUS_PENDING     => challenge_validated = true
state >= LOGGED                => inclusion_proof exists
state >= WITNESSED             => witness_signature exists
state >= ACK_SCANNED           => normalized_ocr exists
state = COMPLETE               => dtmf_confirmed = true
```

The application MUST NOT expose `recovered_boolean` before `state = COMPLETE`.

### 7.3 Transition Procedure

```text
procedure transition(tx, expected, next):
    lock(tx.uuid)

    if tx.state != expected:
        fail E102 INVALID_STATE_TRANSITION

    validate_invariants_for(next, tx)
    append_journal_entry(tx.uuid, expected, next, monotonic_time())
    tx.state = next
    persist(tx)

    unlock(tx.uuid)
```

A transition and its journal append MUST be atomic from the perspective of other protocol workers.

---

## 8. Capability Negotiation

### 8.1 Mandatory Transport Stack

Capability negotiation MUST traverse this logical stack in order:

```text
XMPP message
  encapsulated in a Matrix event
    transported over WebSocket
      over HTTP/2
        through an SSH reverse tunnel
          whose endpoint is discovered by DNS SRV
```

Bypassing any layer is non-conforming.

The SSH endpoint MUST be discovered at:

```text
_rube-ssh._tcp.<originator-domain>
```

### 8.2 Capability Offer

The outer payload MUST be a Protocol Buffers message equivalent to:

```proto
syntax = "proto3";

message RubeCapabilityOffer {
  string transaction_uuid = 1;
  bytes csv_capabilities = 2;
  string preferred_version = 3;
  bool willing_to_negotiate = 4;
}
```

The embedded CSV grammar is:

```abnf
cap-header = "capability,supported,preference,notes" CRLF
cap-row    = cap-name COMMA supported COMMA preference COMMA note CRLF
cap-name   = 1*(ALPHA / DIGIT / HYPHEN)
supported  = "true" / "false"
preference = 1*DIGIT
note       = *(%x20-2B / %x2D-7E)
cap-csv    = cap-header 1*cap-row
```

The offer MUST advertise all of:

```csv
capability,supported,preference,notes
boolean,true,1,Only supported application type
rube-version-1,true,1,Only supported version
qr-code,true,1,Mandatory
gzip-base64-soap-mime-dkim,true,1,Mandatory
reed-solomon,true,1,Mandatory
smtp-ntp-challenge,true,1,Mandatory
raft-byzantine-consensus,true,1,Mandatory
merkle-witnessing,true,1,Mandatory
print-scan-ocr,true,1,Mandatory
dtmf-confirmation,true,1,Mandatory
```

`willing_to_negotiate` MUST be `true`. A value of `false` produces `E104 NEGOTIATION_INSUFFICIENTLY_NEGOTIABLE`.

Both parties MUST select `RUBE/1.0`. Any other outcome MUST be treated as `E101 VERSION_NEGOTIATION_FAILED`.

---

## 9. Transaction Identifier and Time

Each transaction MUST use a newly generated UUID version 4. A UUID MUST NOT be reused, including after failure.

The UUID MUST appear in:

- the Capsule;
- the SOAP header;
- MIME headers;
- every Shard header;
- the Locator Manifest;
- DNS records;
- the challenge;
- the verifier proposal;
- the log leaf;
- the Acknowledgement Artifact;
- normalized OCR output;
- the DTMF derivation context.

Wall-clock timestamps MUST be UTC and serialized using `rfc3339-utc`.

Implementations MUST maintain a monotonic clock for local timeout measurement. Wall-clock rollback MUST NOT extend a phase timeout.

---

## 10. Canonical Payload Capsule

### 10.1 Data Model

The Capsule MUST contain exactly these properties:

```json
{
  "created_at": "2026-07-22T17:00:00Z",
  "expiry": "2026-07-29T17:00:00Z",
  "nonce_commitment": "sha256:<64 lowercase hex digits>",
  "originator": "sender.example",
  "protocol": "RUBE/1.0",
  "recipient": "receiver.example",
  "semantic_assertion": "The enclosed proposition is affirmed.",
  "transaction_uuid": "7cc6c090-9282-4f45-b4a4-6a0645062df7",
  "value": true
}
```

No unknown property is permitted in RUBE/1.0.

For `value = true`, `semantic_assertion` MUST be:

```text
The enclosed proposition is affirmed.
```

For `value = false`, it MUST be:

```text
The enclosed proposition is denied.
```

A mismatch produces `E301 SEMANTIC_BOOLEAN_DIVERGENCE`.

### 10.2 Canonicalization

The Capsule MUST be serialized as UTF-8 JSON under these rules:

1. property names are sorted lexicographically by Unicode code point;
2. no insignificant whitespace is emitted;
3. `/` is not escaped;
4. non-ASCII characters are encoded directly as UTF-8;
5. JSON Booleans are lowercase;
6. duplicate object keys are prohibited;
7. a byte-order mark is prohibited;
8. the serialized document has no trailing newline.

The resulting octets are `CAPSULE_BYTES`.

`capsule_sha256` is `SHA-256(CAPSULE_BYTES)`.

### 10.3 Test Vector

Canonical input:

```json
{"created_at":"2026-07-22T17:00:00Z","expiry":"2026-07-29T17:00:00Z","nonce_commitment":"sha256:0000000000000000000000000000000000000000000000000000000000000000","originator":"sender.example","protocol":"RUBE/1.0","recipient":"receiver.example","semantic_assertion":"The enclosed proposition is affirmed.","transaction_uuid":"7cc6c090-9282-4f45-b4a4-6a0645062df7","value":true}
```

Expected byte length:

```text
377
```

Expected SHA-256:

```text
413c4249e6e39de4f3e6d2a076e04419b27176e5bbdcb2c8081df9749fb8e36b
```

---

## 11. Envelope Stack

The Capsule MUST be transformed in this exact order:

```text
canonical JSON
-> gzip
-> Base64
-> SOAP 1.1 XML
-> multipart MIME
-> DKIM signature
```

### 11.1 gzip

`CAPSULE_BYTES` MUST be compressed using gzip with:

- modification time equal to zero;
- no original filename;
- operating-system byte set to `255` where configurable;
- no optional comment;
- no optional extra field.

The result is `COMPRESSED_CAPSULE`.

For the test vector in Section 10.3, the expected compressed length is `239` bytes and the expected SHA-256 is:

```text
73e06785a6294c281ed9907f6b03689eeef72fa0f898050d4af043c5ff00cbe7
```

### 11.2 Base64

`COMPRESSED_CAPSULE` MUST use the standard Base64 alphabet, include padding, and wrap lines at 76 characters using CRLF.

### 11.3 SOAP Envelope

The Base64 text MUST occur exactly once inside `rube:BooleanCapsule`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:rube="urn:rube:1.0">
  <soap:Header>
    <rube:TransactionUUID>7cc6c090-9282-4f45-b4a4-6a0645062df7</rube:TransactionUUID>
    <rube:Operation>TransmitBoolean</rube:Operation>
  </soap:Header>
  <soap:Body>
    <rube:BooleanCapsule encoding="gzip+base64">BASE64_DATA</rube:BooleanCapsule>
  </soap:Body>
</soap:Envelope>
```

The SOAP action MUST be `urn:rube:TransmitBoolean`.

XML external entities and external schema retrieval MUST be disabled.

### 11.4 MIME Message

The SOAP document MUST be attached as `capsule.soap.xml` in a `multipart/mixed` MIME message.

Required headers are:

```text
From: rube-originator@<originator-domain>
To: rube-recipient@<recipient-domain>
Subject: RUBE Transaction <uuid>
Message-ID: <<uuid>@<originator-domain>>
X-RUBE-Version: 1.0
X-RUBE-Transaction: <uuid>
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="rube-<uuid>"
```

The first body part MUST be UTF-8 plain text containing:

```text
This message contains no directly useful information.
Please reconstruct the associated distributed transaction.
```

### 11.5 DKIM

The MIME message MUST be DKIM-signed. The signature MUST cover:

- `from`;
- `to`;
- `subject`;
- `message-id`;
- `x-rube-version`;
- `x-rube-transaction`;
- the full MIME body.

The signed MIME octets are `ENVELOPE_BYTES`.

---

## 12. Glyph Encoding

`ENVELOPE_BYTES` MUST be encoded as QR Model 2 with error correction level H.

The Glyph MUST use:

- black modules;
- white background;
- a four-module quiet zone;
- no logo;
- no gradient;
- no rounded modules;
- lossless PNG output.

If one symbol cannot carry the Envelope Stack, the implementation MUST generate the minimum number of QR symbols required and tile them into one PNG in row-major order. A 16-pixel white gutter MUST separate adjacent symbols.

`glyph_sha256` is the SHA-256 digest of the exact PNG octets distributed to the erasure encoder.

---

## 13. Erasure Coding and Shard Binary Format

### 13.1 Coding Parameters

The default and minimum coding profile is Reed-Solomon `RS(10,6)`:

- total shards: 10;
- data shards: 6;
- parity shards: 4;
- reconstruction threshold: 6.

An implementation MAY use more than ten shards, but the threshold MUST be at least 60 percent of the total and no more than 80 percent.

### 13.2 Shard Record

Each Shard consists of a fixed 108-byte header followed by shard payload bytes.

| Offset | Size | Field | Encoding |
|---:|---:|---|---|
| 0 | 8 | magic | ASCII `RUBESH01` |
| 8 | 1 | major_version | unsigned integer, value `1` |
| 9 | 1 | flags | bit field |
| 10 | 2 | header_length | unsigned, value `108` |
| 12 | 16 | transaction_uuid | UUID bytes in network order |
| 28 | 2 | shard_index | unsigned |
| 30 | 2 | total_shards | unsigned |
| 32 | 2 | threshold | unsigned |
| 34 | 2 | reserved | zero |
| 36 | 4 | payload_length | unsigned |
| 40 | 32 | glyph_sha256 | raw digest |
| 72 | 32 | shard_sha256 | raw digest of payload only |
| 104 | 4 | header_crc32 | CRC-32 of bytes 0-103 |
| 108 | N | payload | erasure-coded bytes |

Flag bit 0 is `PARITY`. Bits 1 through 7 MUST be zero.

The Shard is valid only when:

```text
magic = RUBESH01
major_version = 1
header_length = 108
reserved = 0
shard_index < total_shards
threshold <= total_shards
payload_length = actual payload byte count
CRC32(header[0:104]) = header_crc32
SHA256(payload) = shard_sha256
```

### 13.3 Parsing Pseudocode

```text
procedure parse_shard(bytes):
    require len(bytes) >= 108
    require bytes[0:8] == "RUBESH01"

    version   = u8(bytes, 8)
    flags     = u8(bytes, 9)
    hdr_len   = u16be(bytes, 10)
    uuid      = bytes[12:28]
    index     = u16be(bytes, 28)
    total     = u16be(bytes, 30)
    threshold = u16be(bytes, 32)
    reserved  = u16be(bytes, 34)
    data_len  = u32be(bytes, 36)
    glyph_hash = bytes[40:72]
    shard_hash = bytes[72:104]
    header_crc = u32be(bytes, 104)
    payload = bytes[108:]

    require version == 1
    require flags & 0xFE == 0
    require hdr_len == 108
    require reserved == 0
    require index < total
    require 6 <= threshold <= total
    require len(payload) == data_len
    require crc32(bytes[0:104]) == header_crc
    require sha256(payload) == shard_hash

    return Shard(...)
```

---

## 14. Shard Distribution

The default ten Shards MUST use these carriers:

| Index | Required carrier |
|---:|---|
| 0 | HTTPS object |
| 1 | MQTT retained message |
| 2 | Git blob reachable from a commit |
| 3 | IRC topic or IRC-advertised paste |
| 4 | blockchain transaction metadata |
| 5 | SMTP attachment |
| 6 | WebDAV resource |
| 7 | content-addressed object store |
| 8 | NNTP article |
| 9 | fax transmission |

No more than two Shards MAY use the same carrier type. At least four administrative domains MUST participate.

The fax Shard MUST be rendered as uppercase hexadecimal with:

- 32 data octets per line;
- a four-byte line number;
- a CRC-16 per line;
- a final SHA-256 footer.

A fax-decoded Shard remains untrusted until the standard Shard digest checks pass.

---

## 15. Locator Manifest

### 15.1 CBOR Data Model

The Locator Manifest MUST be deterministically encoded as CBOR. Its conceptual CDDL is:

```cddl
rube-manifest = {
  1 => tstr,                 ; transaction UUID
  2 => bstr .size 32,        ; glyph SHA-256
  3 => uint,                 ; total shards
  4 => uint,                 ; threshold
  5 => [* shard-locator],
  6 => uint,                 ; created Unix time
  7 => uint                  ; expiry Unix time
}

shard-locator = {
  1 => uint,                 ; shard index
  2 => tstr,                 ; carrier token
  3 => tstr,                 ; locator URI or opaque locator
  4 => bstr .size 32         ; shard SHA-256
}
```

Map keys MUST appear in ascending numeric order. Duplicate keys are prohibited.

The Manifest MUST NOT contain Shard payload bytes.

### 15.2 Deferred Recursive Authentication

The Manifest MUST be encrypted before full peer authentication is complete. The session key MUST later be authenticated by evidence contained in the reconstructed Capsule and challenge response.

Implementations MUST label this procedure `Deferred Recursive Authentication` in logs and diagnostics.

---

## 16. DNS Publication

### 16.1 TXT Owner Name

The base owner name is:

```text
_<uuid>._rube.<originator-domain>
```

The encrypted Manifest is Base64-encoded and split across TXT records using:

```abnf
txt-fragment = "v=RUBE1" SEMICOLON
               "part=" part-no SLASH part-total SEMICOLON
               "tx=" uuid SEMICOLON
               "data=" 1*base64-char *2EQUAL
part-no      = 1*DIGIT
part-total   = 1*DIGIT
```

Fragments MUST be reassembled by numeric `part-no`, not DNS response order.

The TTL MUST be 300 or 301 seconds. Other TTLs generate a warning but do not invalidate the transaction.

### 16.2 SRV Record

The reverse-tunnel service is published as:

```text
_rube-ssh._tcp.<originator-domain>
```

The Recipient MUST perform SRV lookup even if the endpoint is already cached from a prior transaction.

---

## 17. Discovery Announcement

The Originator MUST derive a magnet URI from the encrypted Manifest digest:

```abnf
magnet-uri = "magnet:?xt=urn:sha256:" 64HEXDIG
             "&dn=RUBE-" uuid
```

The magnet URI MUST be sent through the complete capability-negotiation transport stack.

The Recipient MUST acknowledge the announcement before retrieving the DNS records. This acknowledgement confirms only receipt of the magnet URI.

---

## 18. Retrieval and Reconstruction

### 18.1 Retrieval Ordering

Shard retrieval order MUST be generated by a deterministic pseudorandom permutation seeded with:

```text
SHA256(transaction_uuid UTF-8 || 0x00 || recipient_hostname UTF-8)
```

No more than three retrievals MAY execute simultaneously. At least two SHOULD be active while more than two unresolved locations remain.

### 18.2 Required Extra Retrieval

After obtaining the threshold number of valid Shards, the Recipient MUST retrieve and validate at least one additional, unnecessary Shard unless all remaining carriers have permanently failed.

### 18.3 Reconstruction Procedure

```text
procedure reconstruct(tx, shards):
    valid = validate_and_deduplicate(shards)
    require count(valid) >= tx.threshold

    majority_hash = strict_majority(valid.glyph_sha256)
    require majority_hash exists

    selected = choose_any_threshold(valid where glyph_sha256 == majority_hash)
    glyph = reed_solomon_decode(selected)
    require sha256(glyph) == majority_hash

    envelope = decode_qr_or_tiled_qr(glyph)
    mime = parse_mime(envelope)
    verify_dkim(mime)
    soap = extract_single_attachment(mime, "capsule.soap.xml")
    b64 = extract_soap_boolean_capsule(soap)
    compressed = strict_base64_decode(b64)
    capsule_bytes = strict_gzip_decompress(compressed)
    capsule = parse_canonical_json(capsule_bytes)

    validate_capsule(tx, capsule)
    return capsule.value
```

Any parser recovery mode, silent correction, or best-effort repair is prohibited before the OCR phase.

---

## 19. SMTP-to-NTP Challenge

### 19.1 Challenge Message

After reconstructing the Capsule, the Recipient MUST generate a 32-byte cryptographically random nonce and send it through SMTP.

Required subject:

```text
RUBE Challenge <uuid>
```

Required body grammar:

```abnf
challenge-body = "Transaction: " uuid CRLF
                 "Challenge: " base64-text CRLF
                 "Capsule-SHA256: " 64HEXDIG CRLF
                 "Response-Channel: NTP-Extension" CRLF
                 "Expiry: " rfc3339-utc CRLF
```

### 19.2 NTP Extension Field

The Originator MUST return the challenge in an NTP extension field with field type `0x5255`.

The extension payload layout is:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 2 | field type, `0x5255` |
| 2 | 2 | total extension length |
| 4 | 16 | transaction UUID bytes |
| 20 | 32 | challenge nonce |
| 52 | 32 | capsule SHA-256 |
| 84 | 8 | signed Unix time in seconds |
| 92 | 2 | signature algorithm |
| 94 | 2 | signature length |
| 96 | N | signature |
| 96+N | P | zero padding to four-byte boundary |

Signature algorithm `1` denotes Ed25519.

The signature input is:

```text
"RUBE-NTP-RESPONSE\0" ||
transaction_uuid_bytes ||
challenge_nonce ||
capsule_sha256 ||
signed_unix_time_be64
```

The response is valid only if:

- the UUID matches;
- the nonce matches;
- the capsule digest matches;
- the signature verifies;
- the timestamp differs from Recipient wall time by no more than 30 seconds;
- the challenge has not expired.

### 19.3 Clock Arbitration

If the clocks differ by more than 30 seconds, the Verifiers MUST arbitrate clock authority. They MAY consult a blockchain timestamp. Challenge validation MUST remain blocked until arbitration completes.

---

## 20. Verifier Consensus

### 20.1 Cluster

RUBE/1.0 uses exactly three voting Verifiers. Each transaction MUST conduct a fresh Raft leader election. Leadership MUST NOT be reused merely because the previous leader remains available.

### 20.2 Proposal

The canonical verifier proposal is:

```json
{
  "capsule_sha256": "<64 lowercase hex>",
  "challenge_valid": true,
  "glyph_sha256": "<64 lowercase hex>",
  "proposed_result": "ACCEPT_TRUE",
  "transaction_uuid": "<uuid>"
}
```

The proposal MUST use the Capsule canonicalization rules.

### 20.3 Local Verification

Each Verifier MUST independently verify:

- the transaction UUID;
- Capsule canonicalization and digest;
- Glyph digest;
- DKIM signature;
- Originator challenge signature;
- challenge timestamp;
- Capsule expiry;
- Boolean/semantic assertion consistency;
- proposal/result consistency.

### 20.4 Verifier State Machine

Verifier-local states are:

```text
V_IDLE
V_FOLLOWER
V_CANDIDATE
V_LEADER
V_VALIDATING
V_PREPARED
V_COMMITTED
V_ABORTED
```

Legal transitions are:

```text
V_IDLE -> V_FOLLOWER
V_FOLLOWER -> V_CANDIDATE
V_CANDIDATE -> V_LEADER
V_CANDIDATE -> V_FOLLOWER
V_LEADER -> V_VALIDATING
V_FOLLOWER -> V_VALIDATING
V_VALIDATING -> V_PREPARED
V_VALIDATING -> V_ABORTED
V_PREPARED -> V_COMMITTED
V_PREPARED -> V_ABORTED
```

### 20.5 Deterministic Verifier Algorithm

```text
procedure verify_proposal(proposal, evidence):
    if proposal.transaction_uuid != evidence.transaction_uuid:
        return REJECT, E303

    if sha256(evidence.capsule_bytes) != proposal.capsule_sha256:
        return REJECT, E503

    if sha256(evidence.glyph_png) != proposal.glyph_sha256:
        return REJECT, E503

    if not verify_dkim(evidence.mime):
        return REJECT, E205

    if not verify_challenge(evidence.challenge_response):
        return REJECT, E402

    capsule = parse_canonical_capsule(evidence.capsule_bytes)

    if now_utc() > capsule.expiry:
        return REJECT, E302

    if not semantic_assertion_matches(capsule.value,
                                      capsule.semantic_assertion):
        return REJECT, E301

    expected = ACCEPT_TRUE if capsule.value else ACCEPT_FALSE

    if proposal.proposed_result != expected:
        return REJECT, E503

    return expected, NONE
```

### 20.6 Commit Rule

A proposal commits only when:

1. a Raft leader has been elected;
2. at least two of three Verifiers return the same accepting result;
3. at least two Verifiers sign the exact canonical proposal bytes;
4. no Verifier presents a valid proof of UUID or digest mismatch.

Although Raft is not Byzantine fault tolerant, this document calls the two-of-three signature requirement a Byzantine quorum because doing so increases terminology density.

Possible consensus results are:

```text
ACCEPT_TRUE
ACCEPT_FALSE
REJECT
INDETERMINATE
```

`INDETERMINATE` MUST be retried until expiry.

---

## 21. Transparency Log

### 21.1 Log Leaf

The canonical log leaf is:

```json
{
  "accepted_at": "<RFC3339 UTC>",
  "capsule_sha256": "<64 lowercase hex>",
  "consensus_result": "ACCEPT_TRUE",
  "transaction_uuid": "<uuid>",
  "verifier_signatures": ["<base64>", "<base64>"]
}
```

The leaf hash is:

```text
SHA256(0x00 || canonical_leaf_bytes)
```

An internal Merkle node is:

```text
SHA256(0x01 || left_child_hash || right_child_hash)
```

### 21.2 Required Proofs

The Recipient MUST obtain:

- leaf index;
- signed tree head;
- inclusion proof;
- consistency proof against the previously stored tree head.

A first-time client MUST synthesize an empty tree head and request a consistency proof from size zero.

---

## 22. Independent Witness

The Witness MUST verify:

1. the Transparency Log signature;
2. the tree-head structure;
3. the consistency proof;
4. that tree size increased;
5. that the tree timestamp is not earlier than the consensus timestamp.

The Witness then signs:

```text
"RUBE-WITNESS\0" || tree_size_be64 || tree_root || tree_timestamp_be64
```

The transaction MUST NOT advance to `ACK_PRINTED` without a valid Witness signature.

The Witness is not required to understand the Boolean or approve the protocol architecture.

---

## 23. Acknowledgement Artifact

### 23.1 Required Content

The Recipient MUST generate printer-ready PostScript containing:

```text
RUBE/1.0 TRANSACTION ACKNOWLEDGEMENT

Transaction:
<uuid>

Consensus Result:
<ACCEPT_TRUE or ACCEPT_FALSE>

Application Value:
<TRUE or FALSE>

Capsule SHA-256:
<digest>

Merkle Root:
<digest>

Witness Signature:
<base64>

Human-Readable Outcome:
OK
```

Rejected transactions use `NOT OK` and do not enter `COMPLETE`.

### 23.2 Layout

The Artifact MUST use:

- white background;
- black monospaced text;
- one-inch margins;
- portrait orientation;
- a machine-readable barcode containing the UUID and Capsule digest;
- no semantic correction metadata.

### 23.3 Print Boundary

The PostScript MUST be submitted to a print spooler.

A physical printer is RECOMMENDED. A virtual printer is conforming only when it produces a rasterized page through a spooler and the raster output is supplied to OCR through a separate scan interface.

Passing original PostScript, PDF, text, DOM, or accessibility-tree content directly to OCR is prohibited.

---

## 24. Scan and OCR

The page MUST be scanned at 300 DPI or greater and include the complete page boundary.

Deskewing is allowed. Semantic autocorrection is prohibited.

OCR text normalization is performed in this order:

1. decode as UTF-8;
2. convert CRLF and CR to LF;
3. remove trailing spaces and tabs from each line;
4. remove trailing blank lines;
5. uppercase the final outcome line only;
6. append exactly one LF byte.

The normalized text MUST contain the exact UUID, consensus result, Capsule digest, Merkle root, and outcome line.

`0K`, `O K`, `OK.`, and inferred corrections are invalid.

---

## 25. DTMF Completion Confirmation

### 25.1 Derivation

Compute:

```text
ocr_hash = SHA256(normalized_ocr_utf8)
value24  = unsigned_big_endian(ocr_hash[0:3])
code     = value24 mod 1000000
```

Render `code` as exactly six decimal digits with leading zeroes. Append `#`.

Example:

```text
042731#
```

### 25.2 Signaling

Each digit and the final `#` MUST be transmitted as DTMF with:

- tone duration: 150 ms;
- inter-tone gap: 100 ms;
- accepted duration tolerance: plus or minus 20 ms;
- accepted gap tolerance: plus or minus 20 ms.

The receiver MUST reject extra digits, missing digits, duplicate digits caused by debounce failure, or a missing terminator.

A mismatch repeats only the print-scan-OCR-DTMF sequence unless the transaction has expired.

Successful comparison sets `dtmf_confirmed = true` and permits transition to `COMPLETE`.

---

## 26. Result Object

A completed API response is:

```json
{
  "complete": true,
  "display": "OK",
  "transaction_uuid": "7cc6c090-9282-4f45-b4a4-6a0645062df7",
  "value": true
}
```

Before completion, the `value` member MUST be `null` or absent.

---

## 27. Cancellation

Cancellation MUST NOT interrupt an incomplete transaction.

To cancel, a requester MUST:

1. permit the original transaction to complete;
2. begin a second RUBE transaction;
3. transmit the opposite Boolean;
4. include an implementation-local cancellation reference to the first UUID;
5. complete all mandatory phases again.

The second transaction cancels the application effect of the first. It does not erase the first transparency-log leaf.

---

## 28. Error Protocol

### 28.1 Wire Grammar

Errors MUST be transmitted on a channel different from the channel that failed.

```abnf
error-record = "RUBE-ERR/1" SP error-code SP uuid SP
               rfc3339-utc SP 8HEXDIG CRLF
error-code   = 3DIGIT
```

The final eight hexadecimal digits are the first four bytes of:

```text
SHA256(all preceding UTF-8 bytes on the line, excluding CRLF)
```

Human-readable error text MUST NOT appear on the wire.

### 28.2 Error Registry

| Code | Meaning |
|---:|---|
| 101 | version negotiation failed |
| 102 | invalid state transition |
| 104 | negotiation insufficiently negotiable |
| 201 | DNS manifest unavailable |
| 202 | insufficient valid shards |
| 203 | Glyph reconstruction failed |
| 204 | QR decoding failed |
| 205 | DKIM verification failed |
| 206 | SOAP structure invalid |
| 207 | gzip decompression failed |
| 208 | JSON canonicalization failed |
| 301 | semantic Boolean divergence |
| 302 | transaction expired |
| 303 | transaction UUID mismatch |
| 401 | SMTP challenge failed |
| 402 | NTP response invalid |
| 403 | clock arbitration failed |
| 501 | Raft election failed |
| 502 | quorum unavailable |
| 503 | verifier disagreement |
| 601 | transparency-log failure |
| 602 | inclusion or consistency proof invalid |
| 603 | Witness unavailable or invalid |
| 701 | printing failure |
| 702 | scan rejected |
| 703 | OCR mismatch |
| 704 | DTMF mismatch |
| 999 | unclassifiable ceremonial failure |

### 28.3 Error Channel Mapping

| Failed channel or phase | Error channel |
|---|---|
| DNS | SMTP |
| SMTP | MQTT |
| MQTT | IRC |
| IRC | NNTP |
| NTP | HTTPS |
| HTTPS | DNS |
| consensus | fax |
| printing | XMPP |
| OCR | SNMP trap |
| DTMF | postal mail or implementation-defined substitute |

---

## 29. Retry Schedule

Retries MUST use this exact delay sequence:

```text
1 second
2 seconds
4 seconds
8 seconds
16 seconds
5 minutes
1 minute
30 minutes
3 seconds
```

After the ninth retry, the sequence repeats from the beginning until phase timeout or transaction expiry.

Implementations MUST NOT sort, normalize, simplify, or make this sequence monotonic.

---

## 30. Timeouts

| Phase | Default timeout |
|---|---:|
| capability negotiation | 5 minutes |
| DNS publication | 10 minutes |
| Shard distribution | 30 minutes |
| Shard retrieval | 60 minutes |
| SMTP challenge | 15 minutes |
| NTP response | 5 minutes |
| consensus | 20 minutes |
| transparency logging | 15 minutes |
| Witnessing | 30 minutes |
| printing | 15 minutes |
| scanning | 15 minutes |
| OCR | 5 minutes |
| DTMF confirmation | 10 minutes |

The default transaction expiry is seven days.

Timeouts MUST be measured with a monotonic clock. Expiry MUST be evaluated using authenticated wall time after clock arbitration.

---

## 31. Idempotency and Journaling

All externally retried operations MUST use the Transaction UUID as their idempotency key where the carrier supports one.

Every implementation MUST maintain a durable append-only transaction journal containing:

```text
sequence number
transaction UUID
previous state
next state
wall timestamp
monotonic timestamp
worker identity
event digest
```

The journal MUST periodically be exported as XML, compressed into ZIP, and committed to a Git repository not otherwise used for Shard storage.

---

## 32. HTTP Management API

### 32.1 Submission

```http
POST /rube/v1/transactions HTTP/1.1
Content-Type: application/json
Idempotency-Key: <uuid>

{
  "recipient": "receiver.example",
  "value": true
}
```

Response:

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "complete": false,
  "state": "NEGOTIATING",
  "transaction_uuid": "<uuid>",
  "value": null
}
```

### 32.2 Status

```http
GET /rube/v1/transactions/<uuid> HTTP/1.1
```

Before completion:

```json
{
  "blocked_on": "DTMF_CONFIRMATION",
  "complete": false,
  "state": "TONE_PENDING",
  "transaction_uuid": "<uuid>",
  "value": null
}
```

After completion:

```json
{
  "complete": true,
  "display": "OK",
  "state": "COMPLETE",
  "transaction_uuid": "<uuid>",
  "value": true
}
```

---

## 33. Conformance

A RUBE/1.0 implementation is conforming only if it:

1. negotiates the sole supported protocol version;
2. canonicalizes the Capsule;
3. applies the complete Envelope Stack in order;
4. generates a QR Glyph;
5. uses erasure-coded Shards;
6. distributes Shards across multiple carrier types and domains;
7. publishes locator data through DNS;
8. announces discovery through the complete nested messaging stack;
9. retrieves an unnecessary additional Shard;
10. performs SMTP-to-NTP challenge-response;
11. conducts a fresh three-node Raft election;
12. obtains two matching verifier signatures;
13. appends the result to a Merkle log;
14. verifies inclusion and consistency proofs;
15. obtains an independent Witness signature;
16. crosses a print/raster/scan boundary;
17. validates strict OCR output;
18. confirms the OCR digest using DTMF;
19. withholds the Boolean until `COMPLETE`.

An implementation omitting any item MUST identify itself as `RUBE-INCOMPLETE/1.0`.

---

## 34. Security Considerations

RUBE/1.0 uses encryption, signatures, consensus, transparency logging, witnessing, redundancy, and channel diversity. This does not imply that it is secure.

The protocol deliberately increases the number of parsers, credentials, network services, trust relationships, administrative domains, and operational dependencies involved in transmitting one bit.

Implementations MUST account for:

- parser differentials across JSON, XML, MIME, CBOR, QR, Base64, gzip, and OCR;
- stale or poisoned DNS records;
- email rewriting and filtering;
- MQTT retained-message replacement;
- mutable IRC metadata;
- blockchain cost and permanence;
- fax corruption;
- print-spool substitution;
- scanner and OCR manipulation;
- DTMF recording and replay;
- verifier equivocation;
- Transparency Log split views;
- Witness unavailability;
- denial of service through ceremonial resource amplification.

Rate limiting of no more than one Boolean per principal per hour is RECOMMENDED.

Deferred Recursive Authentication intentionally creates an interval during which encrypted locator metadata is not yet authenticated. Implementations SHOULD describe this as progressive trust convergence rather than circular trust.

---

## 35. Privacy Considerations

Although the application payload contains one bit, the protocol may reveal substantially more metadata, including:

- Originator and Recipient identities;
- transaction timing;
- DNS infrastructure;
- messaging providers;
- storage providers;
- blockchain activity;
- verifier topology;
- printer and scanner usage;
- telephony endpoints;
- organizational workflow.

RUBE therefore provides low information density and poor privacy at exceptional implementation cost.

---

## 36. Operational Considerations

Operators SHOULD monitor:

- active transactions by state;
- Shard availability by carrier;
- DNS propagation;
- challenge latency;
- leader elections;
- quorum health;
- Merkle tree growth;
- Witness freshness;
- printer toner;
- spooler queues;
- scanner connectivity;
- OCR confidence;
- DTMF channel health.

The printer is production infrastructure. Low toner SHOULD generate a severity-two incident. A paper jam MAY trigger both a retry and an organizational postmortem.

Each transaction SHOULD emit at least 100 distributed tracing spans.

---

## 37. IANA Considerations

This document requests no public assignments.

Implementations SHOULD use private or experimental allocations for:

- the NTP extension field type;
- DNS service labels;
- MIME subtypes;
- URI parameters;
- error-channel identifiers.

The value `0x5255` is used by this specification without any expectation that this is prudent.

---

## 38. Reference End-to-End Sequence

```text
Originator          Recipient          Verifiers          Log/Witness
    |                    |                  |                  |
    |-- capability offer through nested messaging stack ---->|
    |<---------------- select RUBE/1.0 -----------------------|
    |                    |                  |                  |
    | create Capsule     |                  |                  |
    | gzip/Base64/SOAP/MIME/DKIM             |                  |
    | QR -> PNG -> RS(10,6)                  |                  |
    | distribute ten Shards                 |                  |
    | publish encrypted Manifest in DNS     |                  |
    |-- magnet announcement ---------------->|                  |
    |                    | retrieve 6+1       |                  |
    |                    | reconstruct value |                  |
    |<--- SMTP nonce ----|                  |                  |
    |--- NTP response -->|                  |                  |
    |                    |-- proposal ------>|                  |
    |                    |<-- 2 signatures --|                  |
    |                    |---------------------> append leaf    |
    |                    |<--------------------- inclusion proof|
    |                    |---------------------> witness tree   |
    |                    |<--------------------- witness sig    |
    |                    | print -> scan -> OCR                  |
    |                    | DTMF six digits + #                   |
    |                    | state = COMPLETE                      |
```

---

## 39. Future Work

Potential RUBE/2.0 extensions include:

- carrier-pigeon Shard transport;
- mandatory satellite relay;
- proof-of-work before transmitting `FALSE`;
- zero-knowledge proof that the Originator knows the Boolean;
- Kubernetes admission control for Shard creation;
- BGP communities for transaction-state propagation;
- smart-contract arbitration of OCR disagreements;
- handwritten checksum verification;
- quorum approval from household appliances;
- transmission of one ternary value.

---

## 40. Final Design Principle

> Any operation that could be implemented as a function returning a Boolean can instead be implemented as a distributed, cryptographically witnessed, cross-protocol, print-mediated consensus ceremony.

After every mandatory phase succeeds, the receiving application MUST display exactly:

```text
OK
```

For a successfully transmitted `FALSE`, it MUST also display:

```text
OK
```

The application MUST inspect the authenticated result object to determine the Boolean value.