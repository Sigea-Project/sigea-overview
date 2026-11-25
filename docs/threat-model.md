# Threat model (Exposure)

A short statement of what is protected, where trust changes, and what is **not** defended. It describes the
design and what its tests exercise; it is not an audit and has had no independent review.

## Assets

- The operator's authorization of *what may be scanned*.
- The integrity of findings and of the evidence behind them.
- Agent credentials and the operator session.
- The catalog (the checks that decide what a finding means).

## Trust boundaries

| Boundary | Who is trusted | Who is not |
|---|---|---|
| Scanner host to the network | The operator's scope policy | Every response from a scanned host |
| Scanner to the server | The agent's signed identity | The network between them; the content of a report |
| Operator to the server | An authenticated session | Anyone else |
| Catalog to the scanner | A file signed by a key the operator approved, by hash | Any other file |

## Threats and what answers them

| Threat | Mitigation | Residual risk |
|---|---|---|
| A scan reaches outside what was authorized | Scope (networks, ports, rate, budget) enforced on every connection; no DNS; the peer address is re-checked before connecting; a request outside the scope is a configuration error, not a skipped item | A wrong scope written by the operator is honored |
| A hostile scanned host feeds the checks crafted data | Checks evaluate bounded facts, never the network; script checks run in a separate namespaced process with resource limits; HTTP bodies are capped (32 KiB each, 512 KiB in total per endpoint) | The embedded script engine is not a sandbox by itself; isolation relies on the operating-system layer |
| A replayed or tampered upload | HMAC-SHA256 over method, path, query, content type, body hash, timestamp and a random nonce; nonces are stored durably; a bounded clock window; one signed request cannot be accepted twice | A compromised agent host can send false reports under its own identity |
| Theft of the database | Agent keys are stored encrypted (AES-256-GCM) with a key held outside the database and the repository; enrollment tokens are one-use and stored as hashes | Loss of the key file together with the database exposes the keys |
| A tampered or substituted catalog | Ed25519 signature checked against a key the operator trusts; the operator approves the catalog by its SHA-256; an altered file or another signer is rejected | A leaked signing key lets its holder sign checks |
| A scan silently shrinks | A spent connection budget marks the report incomplete and the command fails; an unobservable condition is `inconclusive`, never "not affected" | None known |
| A guess presented as proof | Findings carry a state derived from the check's own claim; a version match is `probable`; a fix is shown only when a check refutes the candidate | A version or banner can be wrong (backported fixes, hidden banners) |
| Credential brute force | Login rate limits and a global gate; sessions in memory on the panel | Not tested against a determined attacker |

## Non-goals

- SIGEA does not exploit. A scan is an observation, never a confirmation of exploitability.
- It does not try to hide check content from someone who controls the scanner host: a check that executes is
  in memory there. The design limits *where* content exists instead of promising secrecy.
- The server does not repeat a scan. It stores what an authenticated agent reported and checks its signature
  and size; the panel says so.
- No claim is made about resistance to a compromised operator, a compromised server host or a malicious
  catalog signer.
