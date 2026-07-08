# ADR-005: Benefits Navigator data posture (Privacy Act / VA vs HIPAA)

**Status:** Proposed — needs legal confirmation before "Member" is final
**Date:** 2026-07-08
**Deciders:** Beau Access Solutions LLC
**Related:** [ADR-001](001-platform-architecture-and-identity.md), [ADR-003](003-pairwise-subject-identifiers.md), [ADR-004](004-existing-user-migration.md)

## Context

Benefits Navigator (BN) is the platform's second sensitive-data candidate. Before it
can move from **Candidate** → **Member** in the portfolio ([TRACKER.md §1](../../TRACKER.md)),
its regulatory posture has to be pinned down — the same gate CIT cleared with its HIPAA
determination. This ADR frames the question and records a conservative working
assumption so the engineering work (membership doc, identity integration, deploy) can
proceed without waiting on the final legal answer.

BN handles, for two flows:

- **Path A — Veterans (B2C):** document upload, AI analysis of VA decisions, denial
  decoding, statement generation. Stores `va_file_number` and `date_of_birth`
  (`EncryptedCharField`, `core/encryption.py`), uploaded VA correspondence, and
  AI-derived analysis of a person's claim.
- **Path B — VSOs (B2B):** caseworkers/advocates managing shared documents and veteran
  invitations — i.e. one user acting on another identified person's claim data.

This is veteran benefits-claim data plus PII. The key question: **which regime governs
it, and does that regime impose obligations distinct from the HIPAA-shaped handling the
platform already assumes for CIT?**

## The question, unpacked

Three candidate regimes, none of which map cleanly:

1. **HIPAA.** HIPAA binds *covered entities* (providers, health plans, clearinghouses)
   and their business associates. BN is a consumer-facing benefits tool, **not** a
   covered entity, so HIPAA very likely does **not** apply *directly*. But BN's data is
   health-adjacent (disability claims reference medical conditions), so HIPAA-grade
   handling is the right conservative *engineering* floor even if the statute doesn't bind.

2. **Privacy Act of 1974 (5 U.S.C. § 552a).** Binds *federal agencies* and their
   systems of records. BN is a private LLC, not a federal agency, so the Act does not
   bind BN directly. The live question is **downstream/derivative**: if BN ingests data
   that originates from VA records, or ever integrates with a VA system, contractual or
   flow-down obligations could attach.

3. **VA claimant confidentiality — 38 U.S.C. § 5701 and 38 CFR §§ 1.500–1.527.** This
   is the regime most specific to BN's data. It governs the confidentiality of VA
   claimant records and names/addresses of claimants. Whether and how it reaches a
   third-party tool that *helps a veteran with their own claim* (vs. one that receives
   records *from* VA) is the crux — and is a legal determination, not an engineering one.

## Decision

1. **Working assumption (engineering, effective now):** treat BN as a **sensitive
   resource-server Member at the same handling tier as CIT** — full layered-session
   identity integration ([INVARIANTS.md](../../INVARIANTS.md) #1), decoupled
   deletion/export (#3), contribution-boundary repo isolation + CODEOWNERS (#4). This
   is the most conservative posture and is correct regardless of which regime formally
   applies, so no engineering work is blocked on the legal answer.

2. **Legal confirmation is a gating item for the "Member" label, not for the code.**
   BAS LLC must obtain a determination on: (a) whether 38 U.S.C. § 5701 / 38 CFR
   §§ 1.500–1.527 reach BN's Path A and Path B data; (b) whether any Privacy Act
   flow-down attaches via data provenance or a future VA integration; (c) any
   VSO-specific obligations on Path B (a VSO handling a third party's claim data).
   Until (a)–(c) land, BN stays **Candidate** in the tracker even though it is
   engineered to Member spec.

3. **If the determination adds obligations beyond CIT's tier** (e.g. specific
   retention, disclosure-accounting, or breach-notification duties under the VA
   regime), they are captured as a follow-up ADR and BN-repo TODOs — they extend, not
   replace, the working-assumption handling.

## Consequences

- BN's `docs/platform-membership.md` and identity integration are written to the
  full-member (CIT) shape **now**, referencing this ADR for posture.
- The tracker's data-posture blocker ([TRACKER.md §6](../../TRACKER.md)) is narrowed
  from "undetermined" to "engineered conservatively; awaiting legal confirmation to
  promote Candidate → Member."
- This ADR does **not** itself constitute legal advice or a determination — it records
  the questions, the conservative default, and what a qualified determination must cover.
