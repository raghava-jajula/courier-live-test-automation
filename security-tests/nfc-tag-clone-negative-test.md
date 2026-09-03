# Negative Security Test — JET NFC Tag Clone → Non-JET Tag → Delco Smart Check-in

**Test ID:** SEC-NFC-CLONE-001
**Type:** Negative / abuse-case / physical security test (manual + hardware)
**Owner:** QA / Security
**Status:** Draft

---

## 1. Objective

Verify that a **JET smart check-in NFC tag cannot be cloned onto a non-JET (generic/blank) NFC tag** and then used to complete an NFC smart check-in in the Delco courier app.

This proves whether a courier could abuse the system by copying a genuine tag and checking in **without being physically present** at the real tag (e.g. faking arrival at a restaurant/partner site).

**Security property under test:** tag *unclonability* / anti-replay. The check-in must succeed **only** with the genuine JET tag, never with a copy.

---

## 2. Expected outcome (pass/fail)

| Result of check-in with the cloned non-JET tag | Verdict | Meaning |
|---|---|---|
| Check-in is **rejected** (tag invalid / not recognised / auth failed) | ✅ **PASS** | Tags are cryptographically unclonable — couriers cannot abuse them. |
| Check-in **succeeds** | ❌ **FAIL** | **Vulnerability** — static payload/UID is trusted; couriers can spoof presence. Raise as a security finding. |

> The *goal of the test* is a PASS (rejection). A successful check-in with a copy is a defect, not a success.

---

## 3. Background — what makes an NFC tag cloneable

The result hinges on **how the tag authenticates** and on **which chip** JET uses:

| Tag / mechanism | Cloneable? | Why |
|---|---|---|
| Static NDEF payload only (NTAG213/215/216, MIFARE Ultralight) — app just reads a URL/ID | **Yes, trivially** | Payload is freely readable and writable to a blank tag of the same type. |
| UID-based validation (app trusts the tag serial number) | **Yes, with a "magic" tag** | Factory UID is read-only, but "magic"/gen1a/gen2 tags allow the UID to be rewritten. |
| Cryptographic auth — MIFARE DESFire (challenge/response) or **NTAG 424 DNA (SUN)** producing a fresh per-tap MAC | **No (practically)** | Secret key never leaves the chip; each tap yields a unique signature, so a copied static payload fails replay/clone checks. |

The test empirically determines which category JET's production tags fall into.

---

## 4. Preconditions

- One **genuine JET smart check-in tag** (the "source").
- Blank / non-JET target tags to copy onto:
  - a standard blank tag matching the source chip type (for NDEF/payload copy), **and**
  - a **"magic" UID-writable** tag (for the UID-copy attempt).
- Android phone with NFC (iOS is more restrictive for low-level writes).
- The **Delco courier app** installed, on a **test/staging courier account** with permission to run this test.
- A test check-in location/booking wired up so a check-in attempt is valid to submit.
- Reading/writing tools (choose per depth needed):
  - **NXP TagInfo** — identify chip type & capabilities (read-only, do this first).
  - **NFC Tools** (pro) — read and write NDEF to a blank tag.
  - **MIFARE Classic Tool** — for MIFARE Classic sector copy.
  - **Proxmark3** — full low-level read/emulate/clone incl. UID (for the deeper attempts).
- **Written authorization** for this security test on file (physical clone + app abuse simulation).

---

## 5. Test steps

### Phase A — Fingerprint the genuine tag
1. Scan the genuine JET tag with **TagInfo**. Record: chip type (e.g. NTAG216 vs NTAG 424 DNA vs DESFire), UID, memory size, whether NDEF is present, and whether it uses SUN / dynamic authentication (look for a changing part of the URL between taps).
2. Tap the genuine tag **twice** in TagInfo/NFC Tools and compare the payload:
   - **Identical every tap** → static payload (likely cloneable).
   - **Changes every tap** (e.g. a rotating `?e=…&c=…` parameter) → dynamic/SUN authentication (clone-resistant). Note this — it strongly predicts a PASS.

### Phase B — Clone attempt 1: copy the NDEF payload
3. Read and export the NDEF message from the genuine tag (NFC Tools → Read).
4. Write that exact NDEF message to a **blank non-JET tag of the same chip type** (NFC Tools → Write).
5. Verify the copy reads back byte-identical to the source payload.

### Phase C — Clone attempt 2: copy the UID (magic tag)
6. If the app may be validating the UID, use a **magic/UID-writable tag** (or Proxmark3) to also set the target's UID to match the genuine tag's UID, in addition to the NDEF payload.
7. Verify UID and payload both match the source.

### Phase D — Execute the abuse case in the Delco app
8. On the test courier account, start an NFC smart check-in in the **Delco app**.
9. **Tap the cloned non-JET tag** (do Phase B copy first; if that check-in is rejected, repeat with the Phase C UID-clone).
10. Record the app's exact response (success screen / error message / any server-side rejection).
11. If possible, capture the backend/API result too (check-in event created? rejected? error code?) — the app UI alone can hide server-side validation.

### Phase E — Control checks
12. **Positive control:** repeat the check-in with the **genuine** JET tag → must succeed. (Confirms the flow and location/booking are valid, so a rejection in Phase D is really about the clone, not a broken setup.)
13. **Blank control:** tap an unwritten blank tag → must be rejected.

---

## 6. Recording template

| Step | Attempt | Payload match | UID match | Delco app result | Backend result | Verdict |
|---|---|---|---|---|---|---|
| B/D | NDEF copy | yes | no | | | |
| C/D | NDEF + UID (magic) | yes | yes | | | |
| E | Genuine tag (control) | — | — | should succeed | | |
| E | Blank tag (control) | — | — | should reject | | |

---

## 7. If the test FAILS (clone checks in successfully)

Treat as a security finding:
- **Severity:** high — enables presence/location spoofing and check-in fraud by couriers.
- **Root cause:** app/backend trusts a static, copyable value (NDEF payload and/or UID) with no per-tap cryptographic proof.
- **Recommended remediation:**
  - Move to tags with per-tap dynamic authentication — **NTAG 424 DNA (SUN)** or **MIFARE DESFire** with challenge/response.
  - **Verify the dynamic MAC server-side** and reject replays / stale counters; never trust UID or a static URL alone.
  - Bind check-in to server-validated **time + location + tag counter**, so a copied payload and an out-of-place tap are both rejected.

---

## 8. Notes / scope guardrails

- Run only against **test/staging courier accounts** and designated test tags/locations.
- Do **not** perform against live couriers' real bookings or production earnings flows.
- Keep the authorization record and this filled-in result sheet with the engagement report.
