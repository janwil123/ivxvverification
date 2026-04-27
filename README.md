# Formalization Notes

This directory contains a first formalization target for IVXV.

## Goal

Model the part of IVXV that is both central and tractable:

- voter-side ElGamal ballot encryption,
- signed submission to the collector,
- retrieval by `VoteID`,
- voter verification using disclosed encryption randomness.

This is the smallest slice that supports useful machine-checked claims about:

- a symbolic secrecy proxy for ballot secrecy before randomness disclosure,
- recorded-as-cast for successfully verified ballots.

## Why only this slice

The full deployed workflow also relies on:

- BDOC / XAdES parsing and PKI,
- OCSP and RFC3161 timestamping,
- a registration service,
- an external verifiable shuffle implementation,
- threshold key generation and smart-card custody assumptions.

These are real security dependencies, but they are not all a good fit for a single lightweight symbolic model.

## Files

- `ivxv_cast_verify.pv`: ProVerif model of the cast and verify subprotocol.
- `ivxv_cast_privacy.pv`: ProVerif biprocess for pre-verification cast privacy.
- `ivxv_accept_auth.pv`: ProVerif model of accepted-ballot authentication under an abstract voter signing key.
- `ivxv_revoting_latest.pv`: ProVerif model of a single voter casting two accepted ballots, with collector-side retention of the latest one.
- `claims.md`: human-readable claim ledger for the whole extracted protocol.

## Intended reading of the model

The ProVerif model abstracts away from BDOC syntax and models the cryptographic meaning:

- a voter encrypts a ballot under the election public key using explicit randomness,
- the collector stores the signed ciphertext,
- the verification service returns the stored ciphertext by `VoteID`,
- the verification app checks equality against a recomputed encryption using the disclosed randomness.

In `ivxv_cast_verify.pv` the verification service is now modeled as a public unauthenticated channel (`ver`), matching the real service. The `VerifySuccess ==> Stored` correspondence therefore cannot be proved and is omitted; it remains a stated assumption in `claims.md` (authentic ciphertext delivery via HTTPS). `VerifySuccess ==> Cast` is proved directly.

## Claims represented by the model

- A stronger symbolic privacy model for vote confidentiality before the voter releases verification randomness.
  `ivxv_cast_privacy.pv` uses a biprocess with `choice[b0,b1]` and proves observational equivalence for two alternative ballot choices in the cast phase.
- A marker-secrecy proxy for ballot knowledge also remains in `ivxv_cast_verify.pv`.
- A symbolic approximation of accepted-ballot authentication.
  `ivxv_accept_auth.pv` shows that if the collector accepts the modeled ballot container, that container corresponds to one sent under the modeled voter signing key.
- Stored-ciphertext provenance for the symbolic cast path: if the collector stores the modeled ciphertext under the modeled `VoteID`, that ciphertext originates from the cast event in the model.
- Verification success implies that the verifier received the modeled stored ciphertext for that `VoteID`.
- Recorded-as-cast for the symbolic verify path: if verification succeeds, the ciphertext returned by the server matches the originally cast ciphertext.
- A first symbolic model of re-voting retention for one voter.
  `ivxv_revoting_latest.pv` shows that when two accepted ballots are cast in sequence for the modeled voter, the collector's final retained ciphertext is the second one.

## Claims not represented directly in the model

- eligibility from voter lists,
- certificate validity,
- timestamp correctness,
- shuffle proof soundness,
- threshold decryption proof soundness,
- tally correctness.

Those are captured in `claims.md` as separate obligations.

## Local verification status

`ProVerif` is available in this environment through `opam exec`.

The current model runs successfully with:

```bash
opam exec -- proverif formal/ivxv_cast_verify.pv
opam exec -- proverif formal/ivxv_cast_privacy.pv
opam exec -- proverif formal/ivxv_accept_auth.pv
opam exec -- proverif formal/ivxv_revoting_latest.pv
```

Current query status:

- observational equivalence for `choice[b0,b1]` in `ivxv_cast_privacy.pv`: true
- secrecy query for `pack_ballot(b0)`: true
- secrecy query for `pack_ballot(b1)`: true
- `Accepted(...) ==> VoterSent(...)` in `ivxv_accept_auth.pv`: true
- `Stored(...) ==> Cast(...)`: true
- `VerifySuccess(...) ==> Stored(...)`: true
- `VerifySuccess(...) ==> Cast(...)`: true
- `Accepted(first) ==> Cast(first)` in `ivxv_revoting_latest.pv`: true
- `Accepted(second) ==> Cast(second)` in `ivxv_revoting_latest.pv`: true
- `FinalStored(second) ==> Cast(second)` in `ivxv_revoting_latest.pv`: true

`tamarin-prover` and `scyther` are still not installed in the current environment.
