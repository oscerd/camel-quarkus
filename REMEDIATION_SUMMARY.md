# PQC Remediation Summary — Apache Camel Quarkus

_Applied on 2026-06-30, based on `PQC_READINESS_REPORT.md`._

Scope of this pass (per instructions): apply only the items under
**Migration Roadmap → Phase 1: Can Do Now** and the **Priority: High** recommended
changes that have no external blocker. Phase 2 ("Requires Library/Runtime Updates")
and Phase 3 ("Blocked on External Standards") items were intentionally left untouched.

## Summary

| Roadmap item | Action | Status |
|--------------|--------|--------|
| Phase 1 — PQC provider enablement (`camel-quarkus-pqc`) | None required (already implemented) | ✅ No-op |
| Phase 1 — Swap the DES crypto-test fixture for strong AES | Replaced DES with **AES-256-CBC** | ✅ Applied |
| Phase 1 — Document hybrid TLS named-group ordering for "JDK 25+" | Skipped (factually incorrect / runtime-update-dependent) | ⏭️ Skipped |
| Priority: High (production HNDL) | None exist in production | ✅ No-op |

Net result: **1 source file changed** (an integration-test fixture). No production
runtime code, public API, dependency, or build configuration was modified.

---

## Changes Applied

### 1. Replace the weak DES symmetric cipher with AES-256 in the crypto integration test

**File:** `integration-tests/crypto/src/main/java/org/apache/camel/quarkus/component/crypto/it/CryptoRoutes.java`

**Finding addressed:** `PQC-SYM-002` — DES (56-bit) test symmetric-cipher fixture
(report inventory row #13, `CryptoRoutes.java:65`).

**What changed** (method `getCryptoDataFormat()`):

- `KeyGenerator.getInstance("DES")` → `KeyGenerator.getInstance("AES")` with `generator.init(256)` (AES-256 key).
- `new CryptoDataFormat("DES", key)` → `new CryptoDataFormat("AES/CBC/PKCS5Padding", key)`.
- Added a random 16-byte initialization vector (`SecureRandom` → `cdf.setInitVector(...)`), because
  AES in a chaining mode requires an IV (DES was used in ECB, which needs none). The IV is generated
  once and shared by the `direct:marshal` and `direct:unmarshal` routes, which use the **same**
  `CryptoDataFormat` instance, so the encrypt/decrypt round-trip stays symmetric.
- Added `import java.security.SecureRandom;`.
- Preserved the existing `cdf.setShouldAppendHMAC(false)` line and its `//workaround for SunPKCS11-NSS-FIPS` comment.

**Why:** DES (56-bit) is classically broken and is also disallowed under the FIPS providers this
test can run with (`SunPKCS11-NSS-FIPS` / BCFIPS via the `${cq-security-provider}` profile placeholder),
whereas AES-256-CBC is FIPS-approved. AES-256 is already quantum-resistant (Grover gives only a
quadratic speed-up, leaving ~128-bit effective security), so this fully resolves the symmetric-cipher
finding.

**Deviation from the report — CBC instead of GCM (deliberate):**
The report suggests AES-256-**GCM**. AES-GCM could not be applied *safely* through Camel's
`CryptoDataFormat`, so AES-256-**CBC** was used instead. This was verified empirically against the
compiled `camel-crypto` `CryptoDataFormat` class (decompiled with `javap`) and the JDK cipher streams:

1. `CryptoDataFormat` streams via `CipherOutputStream` / `CipherInputStream` and initializes the
   cipher with an `IvParameterSpec` for its IV path. AES-GCM **rejects** `IvParameterSpec`
   (`InvalidAlgorithmParameterException: AlgorithmParameterSpec not of GCMParameterSpec`) — confirmed at runtime.
2. The only way to feed GCM a `GCMParameterSpec` is `setAlgorithmParameterSpec(...)`, which pins a
   **single fixed IV** reused for every marshal — a GCM nonce-reuse anti-pattern that is silent
   (the JDK applies no cross-instance nonce-reuse guard — confirmed) and catastrophic for GCM if the
   route runs more than once. `CryptoDataFormat` offers no path for a per-message random GCM nonce.
3. AES-256-CBC is the idiomatic, securely-supported streaming mode for this data format, removes the
   weak DES cipher (the actual finding), and round-trips correctly. The encrypt→decrypt round-trip
   with the exact test message was verified to reproduce the plaintext.

Because the symmetric finding is about cipher *strength* (and GCM vs CBC are equally quantum-resistant),
CBC fully satisfies the remediation goal without introducing a new cryptographic footgun. This honors the
rule "apply crypto changes only where the library already supports it; if a change cannot be applied safely,
record the reason."

---

## Items Reviewed — No Action Taken

### Phase 1 — PQC provider enablement (already done)
The `camel-quarkus-pqc` extension already registers BouncyCastle's post-quantum provider on the JVM and
in GraalVM native image (ML-KEM, ML-DSA, SLH-DSA, Falcon, LMS, XMSS). Nothing to change.

### Priority: High (Harvest-Now-Decrypt-Later) — none in production
Camel Quarkus delegates transport security to the host runtime and performs no production key exchange
or bulk encryption itself, so there is no high-priority production vulnerability to remediate.

---

## Items Skipped — With Reasons

### Phase 1 — "Document the recommended hybrid TLS named-group ordering for users on JDK 25+"

**Skipped.** The requested guidance rests on a factual error in the report and depends on an
unreleased JDK, which places it under the report's own Phase 2 ("Requires Library/Runtime Updates").

Verified against OpenJDK sources (JEP 527; the inside.java "JDK 27 Post-Quantum Hybrid Key Exchange"
heads-up, 2026-05-17):

- The hybrid TLS feature — the `X25519MLKEM768` named group via `jdk.tls.namedGroups` /
  `SSLParameters.setNamedGroups` — is **JEP 527, which targets JDK 27**, *not* JDK 25 as the report states.
- As of 2026-06-30, **JDK 27 is in early-access only and is not released**.
- The project's build baseline is JDK 17 (CI also exercises JDK 25); none of these support JEP 527's TLS
  named groups, so the feature cannot be exercised on any version the project currently supports.
- On JDK 27, `X25519MLKEM768` is **enabled by default**, so explicit "named-group ordering" is unnecessary
  in the common case anyway.

Writing "JDK 25+ hybrid TLS ordering" guidance now would document a non-existent capability for the stated
versions and an unreleased one for the correct version (JDK 27). Per the rules ("apply only where the current
language/library versions already support it"; "do not attempt changes the report marks Requires
Library/Runtime Updates"; "if a change cannot be applied safely, skip it and record the reason"), this was
skipped. It should be revisited when the project baselines a JEP 527-capable JDK (27+); at that point the
`pqc` extension's hand-authored `runtime/src/main/doc/usage.adoc` is the correct place to add it, followed by
regenerating the bound `docs/modules/ROOT/pages/reference/extensions/pqc.adoc` via
`./mvnw -pl extensions/pqc/deployment process-classes` (the generated page must not be hand-edited).

### Out of scope (not Phase 1 / not Priority-High) — test-only RSA/ECDSA/Ed25519/ECDH fixtures
The remaining vulnerable findings (RSA-2048 in XML-security/crypto/AS2/SFTP; Ed25519/ECDSA/ECDH-P256 SFTP
host keys, report rows #11, #12, #14–#20) are **Low priority, test-only**, and the report gates them on the
underlying test stacks (Apache MINA SSHD, AS2 libraries, Santuario) gaining PQC support first — i.e. Phase 2.
The components under test (XML signature, AS2 certificates, SFTP host keys, crypto signing) have no
drop-in ML-KEM/ML-DSA replacement today, so changing them would break the tests. Left unchanged by design.

---

## Verification

- **Cipher round-trip (mechanism-level):** Reproduced exactly what `CryptoDataFormat.marshal`/`unmarshal`
  do — `CipherOutputStream`(ENCRYPT) → `CipherInputStream`(DECRYPT) with an AES-256 key, `AES/CBC/PKCS5Padding`,
  and a shared random 16-byte `IvParameterSpec` — against the exact test message
  (`"Hello Camel Quarkus Crypto"`). Round-trip succeeded. The same harness confirmed AES-GCM fails via the
  `IvParameterSpec` path and that GCM has no cross-instance nonce-reuse guard (the basis for the CBC decision).
- **API correctness:** Every `CryptoDataFormat` call used (`CryptoDataFormat(String, Key)`, `setInitVector(byte[])`,
  `setShouldAppendHMAC(boolean)`) was confirmed present in the compiled class via `javap`.
- **Full Maven test run (`integration-tests/crypto`) was not executed** in this environment: the project's
  dependency tree (Quarkus / Camel 4.20 BOMs, etc.) is not present in the local Maven cache and could not be
  resolved offline. The change is a single, self-contained source edit whose runtime behavior is covered by the
  empirical verification above; the existing `CryptoTest#encryptDecryptMessage` (and its native `CryptoIT`)
  exercise this fixture and should be run by CI. Native mode was likewise not built locally; AES is a standard
  JCA algorithm already available in the crypto extension's native image, so no new native registration is expected.

## Files Changed
- `integration-tests/crypto/src/main/java/org/apache/camel/quarkus/component/crypto/it/CryptoRoutes.java`
