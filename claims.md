# IVXV Security Claims Ledger

This file states security claims in a form suitable for later proof work.

## Claim C1: Ballot confidentiality before verification

Assumptions:

- ElGamal under the configured IVXV group is IND-CPA secure.
- the voter does not reveal encryption randomness `r`.
- the collector and network observer do not hold the election decryption key.

Claim:

- from the ciphertext contained in a cast vote alone, an attacker cannot learn the voter’s plaintext choice beyond what follows from public metadata.

Status:

- partially modeled in `ivxv_cast_privacy.pv` as a symbolic observational-equivalence argument between two ballot choices before randomness disclosure, and in `ivxv_cast_verify.pv` as a simpler secrecy proxy.

## Claim C2: Recorded-as-cast under successful verification

Assumptions:

- the verification application receives the actual voter-generated randomness `r`,
- the verification application receives, for the queried `VoteID`, the ciphertext currently stored by the collector, rather than an attacker-injected replacement,
- the verification application recomputes ciphertexts correctly.

Claim:

- if the verification application reports success, then the ciphertext accepted by the verifier equals the ciphertext stored by the collector for that `VoteID`, and that ciphertext equals the ciphertext produced on the voter side for that ballot.

Status:

- partially modeled in `ivxv_cast_verify.pv`.
  - `Stored ==> Cast` is proved: the Collector stores only ciphertexts whose BDOC signature verifies under the voter's key.
  - `VerifySuccess ==> Cast` is proved directly: if the verifier's recomputed ciphertext matches the received one, it must be the one the voter computed (since the encryption randomness is private).
  - `VerifySuccess ==> Stored` is **not proved**; it holds only under the second assumption above (authentic ciphertext delivery). The model exposes an attack: an attacker who intercepts the VoteID in the Collector's response can cause the verifier to check a VoteID that was never stored. An authenticated submission channel (HTTPS server certificate) is the real mitigation. This attack trace is documented inline in the model.

## Claim C3: Authentication of accepted ballots

Assumptions:

- BDOC signature verification is sound,
- the authenticated session identity is bound to the same natural person as the BDOC signer certificate,
- the Estonian PKI validation path is correct.

Claim:

- every stored accepted vote is signed by the authenticated eligible voter identity accepted by the collector.

Status:

- modeled in `ivxv_accept_auth.pv` as a symbolic correspondence.
  - The improved model now captures the two-layer identity cross-check from `voting/main.go:160`: the session-authenticated identity (`auther`, carried on a private pre-auth channel) is explicitly compared against the identity extracted from the BDOC signing certificate (`signer`, recovered via the `signer_pk` reduction). The Collector rejects any ballot where these do not match.
  - VoteID is now collector-generated (`new`) and returned in the response, matching the source.
  - Simplification: the real system uses two distinct X.509 certificates (one for session auth, one for BDOC signing) whose Estonian personal ID codes are compared as strings. ProVerif does not support equational axioms over free names, so the model uses one key per voter with key equality as a proxy for identity equality. The cross-check logic is preserved.
  - Remaining outside the model: BDOC/XAdES container structure, PKI certificate-path validation, eligibility check (`VoterChoices`), ciphertext structural validity (`ASN1CiphertextVerify`), and rate limiting.

## Claim C4: Collector-side ciphertext well-formedness

Assumptions:

- ASN.1 parsing and group-element validation code are correct.

Claim:

- the collector rejects malformed ciphertexts that do not parse under the configured election public key and group.

Status:

- extracted from code, not yet machine-modeled.

## Claim C5: Anonymity after verified shuffle

Assumptions:

- the processor has removed voter-linked metadata correctly,
- the Verificatum shuffle proof system is sound and zero-knowledge,
- the auditor verifies the shuffle proof correctly.

Claim:

- the shuffled ciphertext list is a rerandomized permutation of the accepted anonymous ciphertext list, without revealing the linking permutation.

Status:

- supported by the implemented audit path, not modeled in `ivxv_cast_verify.pv`.

## Claim C6: Correctness of decryption proofs

Assumptions:

- the discrete-log hardness assumption holds in the configured group,
- Fiat-Shamir is modeled in the random oracle heuristic,
- proof verification code is correct.

Claim:

- an accepted decryption proof certifies that the published plaintext group element corresponds to the published ciphertext under the election public key.

Status:

- extracted from code, suitable for separate proof in a game-based or algebraic model.

## Claim C7: Tally correctness from verified pipeline

Assumptions:

- Claims C3 through C6 hold,
- invalid plaintext ballots are filtered exactly as specified,
- tally software counts valid decoded ballots correctly.

Claim:

- the final tally equals the count of valid decrypted ballots obtained from the verified shuffled ciphertext set.

Status:

- decomposed claim, not yet machine-modeled.

## Intermediate property R1: Latest accepted ballot wins for one modeled voter

Assumptions:

- the same voter successfully casts two accepted ballots in sequence,
- collector acceptance is abstracted to signature validation on the modeled ciphertext container,
- canonical time (`ctime`) for each vote is provided by an honest OCSP/TSP qualification service and is not attacker-controlled,
- the two votes have distinct canonical times (the equal-timestamp tie-breaking case is not modeled).

Claim:

- after two accepted casts, the collector's final retained ciphertext is the one whose canonical time is strictly later, regardless of submission order.

Status:

- modeled in `ivxv_revoting_latest.pv` for a single-voter, two-cast symbolic abstraction.
- The previous version modeled "latest" by reception order, which does not match the source. The source compares `ctime` values derived from OCSP/TSP qualifying properties (`storage/txn.go:264–274`). The improved model introduces a trusted ordering oracle (`q_oracle`, private channel) and explores both cases — vote 2 wins and vote 1 wins — via an `Oracle` process that non-deterministically provides either outcome. The three proved correspondences hold in both cases: every accepted ciphertext was legitimately cast, and the final stored ciphertext was accepted.

## Recommended proof split

Use different tools for different claims instead of forcing one monolithic model:

- ProVerif or Tamarin:
  - C1, C2, and a symbolic approximation of C3.
- pen-and-paper or EasyCrypt / algebraic proof:
  - C5 and C6.
- TLA+ or Coq / Dafny style trace refinement for implementation pipeline:
  - C7.
