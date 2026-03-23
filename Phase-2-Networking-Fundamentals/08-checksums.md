# Checksums

**Phase 2 — Networking Fundamentals | Topic 8 of 8**

---

## What is a Checksum?

A checksum is a small fixed-size value computed from data, used to detect errors or corruption in that data.

```
Original data:  "Transfer $1000 to account 12345"
Checksum:       a3f8c2d1

Data arrives:   "Transfer $1000 to account 12346"  ← one digit changed
Checksum:       b7e9a4f2  ← completely different

Receiver computes checksum of arrived data
Compares with original checksum
They don't match → data was corrupted → reject it
```

Any change to the data, no matter how small, produces a different checksum. This makes corruption **detectable** without transmitting the full data twice.

---

## Why Checksums Matter in Distributed Systems

Silent corruption sources:
- **Network:** cosmic rays flipping bits, electrical interference
- **Disk:** bit rot, bad sectors, firmware bugs
- **Memory:** RAM errors, hardware faults
- **Software:** bugs causing partial writes, buffer overflows

```
Without checksums:
  Corrupted data looks valid → processed as if correct
  Wrong results, data loss, financial errors
  No indication anything went wrong

With checksums:
  Corruption detected immediately
  System rejects corrupted data
  Requests retransmission or fails loudly
```

**Silent data corruption is one of the hardest bugs to catch. Checksums turn silent failures into loud, detectable ones.**

---

## Types of Checksums

### CRC (Cyclic Redundancy Check)
- CRC-32 produces 32-bit checksum; CRC-64 produces 64-bit
- Very fast in hardware and software, excellent at detecting burst errors
- Not cryptographically secure — determined attacker can craft matching CRC
- **Use:** Ethernet frames, ZIP files, disk storage, **Kafka messages**

### MD5
- Produces 128-bit hash (32 hex chars)
- Cryptographically broken — collision attacks possible
- **Use:** File integrity checks (non-security), legacy systems
- **DO NOT use** for passwords or security purposes

### SHA Family (SHA-256, SHA-512)
- SHA-256 produces 256-bit hash (64 hex chars)
- Cryptographically secure, extremely collision resistant
- **Use:** Digital signatures, certificates, Git commit hashes, blockchain, password hashing

### MurmurHash / xxHash
- Extremely fast, not cryptographic
- **Use:** Hash tables, consistent hashing, Bloom filters, Kafka partition assignment

---

## Checksums in Systems You Use Daily

### Kafka
```
Every Kafka message has a CRC32 checksum

Producer computes CRC32 of message → appends to record
Broker verifies CRC32 on receipt → rejects corrupted messages
Consumer verifies CRC32 on receipt → detects in-flight corruption
```

### TCP
TCP header contains a 16-bit checksum. Corrupted segment → dropped, retransmitted automatically.

### Git
```
Every Git commit, tree, blob identified by SHA-1 hash

git commit → SHA-1 of (content + parent + metadata)
"a3b4c5d..." is not just an ID — it's a checksum

If any bit in a commit changes → its hash changes
→ Detects repository corruption
→ Makes Git a content-addressed storage system
```

### Databases
```
PostgreSQL: CRC32 on every data page (8KB block)
MySQL InnoDB: Checksum on every page

On every page read:
  Recompute checksum
  Compare with stored checksum
  Mismatch → page corrupted → error rather than wrong query result
```

### AWS S3
```
S3 computes MD5 of every uploaded object
Returns ETag header = MD5 hash
Client can verify upload integrity by comparing ETags
```

---

## Checksums vs Hashing vs Encryption

| | Purpose | Reversible? | Secure? |
|--|---------|------------|---------|
| **Checksum** | Detect accidental corruption | No | No |
| **Hash** | Fixed-size fingerprint, deduplication | No | Depends (SHA-256 yes, MD5 no) |
| **Encryption** | Hide content | Yes (with key) | Yes |

Note: checksums and hashes are often used interchangeably in practice. When someone says "Kafka checksum" they mean CRC32 hash.

---

## HMAC — Hash-based Message Authentication Code

```
HMAC = Hash(data + secret_key)

Proves both integrity AND authenticity
→ Only someone with the secret key can produce valid HMAC
→ Used in JWT signatures, API request signing, Kafka message authentication
```

---

## Checksum Limitations

Checksums **detect** corruption — they don't prevent or correct it.

For **correction**, use ECC (Error Correcting Codes):
- Reed-Solomon codes → used in QR codes, CDs, DVDs, RAID 6
- Can reconstruct original data from corrupted + parity data

---

## Key Takeaways

```
Checksum: Fixed-size value computed from data
          Any change to data → completely different checksum
          Used to detect corruption, not prevent it

Types:
  CRC32      → fast, hardware-accelerated, used in Kafka/TCP/disks
  MD5        → broken cryptographically, fine for non-security integrity
  SHA-256    → cryptographically secure, use for security purposes
  MurmurHash → extremely fast, used for partitioning/hashing
  HMAC       → integrity + authenticity using a secret key

Where used:
  Kafka      → CRC32 on every message
  TCP        → 16-bit checksum on every segment
  Git        → SHA-1/SHA-256 on every object
  Databases  → CRC32 on every page
  S3         → MD5 ETag on every object

Checksums detect corruption silently → loud failure
ECC corrects corruption (RAID, QR codes, storage)
HMAC adds security → integrity + authenticity
```
