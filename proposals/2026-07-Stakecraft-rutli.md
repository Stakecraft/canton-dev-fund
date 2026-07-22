## Development Fund Proposal — Rütli: Open-Source Threshold KMS for Canton Authority Keys

**Author:** Stakecraft ([https://stakecraft.com](https://stakecraft.com)) · repo: [https://github.com/Stakecraft/rutli](https://github.com/Stakecraft/rutli)
**Status:** Submitted
**Created:** 2026-07-22
**Label:** `node-deployment-operations` (secondary: Security SIG review)
**Champion:** Need Champion — preferably from the Node Deployment & Operations SIG and/or Security SIG ([sig-directory](https://github.com/canton-foundation/canton-dev-fund/blob/main/sig-directory.md))

---

## Abstract

Canton participant **namespace / identity keys** are the highest-consequence secrets on a node: they define on-chain identity, authorize topology changes, **cannot be rotated**, and if lost or leaked leave hosted parties and balances permanently compromised. Today, operators who want keys off the participant process choose among **cloud KMS** (AWS/GCP via Canton Enterprise), **managed / proprietary custody products**, or holding keys in software on the node. There is no **open-source, self-hosted, threshold (t-of-n) KMS** that plugs into Canton’s official KMS Driver SPI so an unmodified OSS or Splice node never sees the private key.

**Rütli** fills that category. It is a Canton KMS Driver (Scala/JVM jar) with pluggable backends; the production path for the Ed25519 namespace key is a **FROST MPC cosigner cluster** (Rust) that generates keys via DKG and produces standard Ed25519 signatures indistinguishable to Canton. Operators run the entire stack inside their own trust domains (Docker Compose or Kubernetes). Apache-2.0. No Canton fork.

This proposal does **not** fund sunk cost. The stack is already implemented through v0.2.x (driver + FROST + mTLS + sealed shares + approval quorum + audit journal + Helm/k8s + soak harness) and verified end-to-end on a real Canton node. The grant funds the work that turns a pre-audit OSS project into **ecosystem infrastructure**: independent security review and remediation, a sustained Splice DevNet (then TestNet) validator soak with public evidence, packaging and operator documentation suitable for non-Stakecraft teams, and **adoption milestones** that pay when independent operators run Rütli.

---

## Specification

### 1. Objective

**Full delivery of this proposal will result in:**

1. A **1.0 production-ready release** of Rütli suitable for protecting Canton/Splice validator (and optionally SV participant) namespace keys under an operator-run t-of-n FROST cluster, with digest-pinned images, hardened k8s defaults, and published runbooks (key ceremony, disaster recovery, approval quorum, observability).
2. An **independent third-party security audit** of the FROST core, cosigner daemon, Scala driver policy path, and deploy threat model — with critical/high findings remediated and a public remediation summary.
3. A **public soak report**: sustained Splice DevNet validator operation with Rütli-backed keys, scheduled failure drills (failover, mid-traffic refresh, backup/restore), and exit criteria met per the published soak runbook; followed by at least one TestNet (or equivalent committee-approved) run when onboarding allows.
4. **Measurable external adoption**: independent organizations (not Stakecraft) install and operate Rütli against real Canton/Splice nodes, with adoption-weighted grant tranches.

**Single objective:** make open, self-hosted threshold protection of Canton authority keys a **common good** that any competent operator can run and that auditors and compliance teams can reason about.

**Out of scope for this grant:**

- Building a managed custody / SaaS product, a wallet, or an institutional key-custody marketplace.
- Replacing or competing with **managed / proprietary KMS and custody offerings** (including MPCH Stronghold and similar). Those serve a different buyer: outsourced or productized custody. Rütli serves operators who must keep key material and control planes **inside their own SOC / ISO / DORA boundary** and want an auditable OSS stack. Both categories can (and should) coexist.
- Cloud KMS (AWS/GCP) drivers already shipped with Canton Enterprise — Rütli is the self-hosted alternative category, not a rewrite of those drivers.
- CometBFT/SV consensus-key threshold signing — that category is already served by tools such as Horcrux; Rütli targets **Canton participant keys** reached only via the KMS Driver SPI.
- Threshold ECDSA for protocol/sequencer keys (documented Phase 4 roadmap) — may be a follow-on proposal if demand appears after 1.0 adoption.
- Auditing Canton/Splice themselves.

### 2. Implementation Mechanics

#### Integration seam (no fork)

Canton exposes a public **KMS Driver SPI** (`com.digitalasset.canton.crypto.kms.driver.api.v1`). Rütli ships a `KmsDriverFactory` / `KmsDriver` jar discovered via `ServiceLoader`. With `crypto.provider = kms` and external key storage, private keys are generated and held entirely outside the Canton process. The driver passes upstream conformance tests (`KmsDriverTest` / `KmsDriverFactoryTest`).

#### Component model

```
Canton / Splice participant (unmodified)
        │  KMS Driver SPI (in-JVM)
        ▼
Rütli driver (Scala) — policy · M-of-N approval · hash-chained audit · signing journal
        │  routes by key type
        ├── namespace / identity (Ed25519)  → FROST cosigner cluster (t-of-n)
        ├── protocol / sequencer (ECDSA)   → HSM backend (PKCS#11) in v1
        └── encryption keys                → HSM / software (never threshold)
```

**FROST cluster (Rust):** distributed key generation (no trusted dealer — required because Canton cannot migrate an existing namespace key into KMS mode), two-round FROST-ed25519 signing producing RFC 8032 signatures, crash-safe single-use nonce guard, proactive share refresh, share repair primitive, mTLS mesh, AES-256-GCM sealed shares, driver-side failover across cosigner endpoints.

**Operator surface already in-repo (baseline for this grant):** Docker Compose (dev + mTLS), Kubernetes/Helm, Prometheus metrics + Grafana dashboard, soak compose overlay that injects the driver via Splice’s `EXTRA_CLASSPATH` / `ADDITIONAL_CONFIG_*` (no Splice image fork), ceremony and DR runbooks.

#### What this grant builds on top

| Workstream | Content |
|---|---|
| Harden & release 1.0 | Close remaining production gaps called out in soak/audit-prep docs; digest-pinned GHCR images; release notes; version matrix vs Canton/Splice releases |
| Security audit | Engage independent firm; scope per `docs/audit-prep.md`; remediate; publish summary |
| Soak & evidence | Execute ≥14-day DevNet soak with drill calendar; publish metrics/report; pursue TestNet when eligible |
| Adoption enablement | Operator workshops / office hours; adoption playbook; optional reference k8s overlays for multi-account cosigners; triage external issues |
| Stewardship | Commit to maintain Rütli for ≥12 months post-1.0 against supported Canton/Splice lines |

### 3. Architectural Alignment

- **Uses the official extension point** (KMS Driver SPI) — default approach of extending Canton rather than forking or wrapping the node.
- Aligns with Development Fund priorities for **Security and Resilience** (hardening authority-key custody, fail-closed audit, separation of duties) and **Stability / Node Operations** (runbooks, observability, DR for non-rotatable keys).
- Complementary to ecosystem pieces:
  - **Cloud KMS drivers** — enterprise cloud-backed category.
  - **Managed / proprietary custody (e.g. MPCH and peers)** — productized / outsourced custody category.
  - **Horcrux-class tools** — CometBFT consensus keys for SVs; orthogonal to Canton participant namespace keys.
- Open-source (Apache-2.0) public good: any validator, SV operator, or app provider running a participant can reuse it without vendor lock-in to Stakecraft.

### 4. Backward Compatibility

- **No change to Canton consensus, Daml, or ledger APIs.** Signatures are standard Ed25519; verifiers are unchanged.
- **Onboarding constraint (Canton platform):** `crypto.provider = kms` must be selected at participant birth — existing software-key nodes cannot retrofit namespace keys into KMS mode. Rütli documents this clearly; migration is greenfield or planned re-onboarding only.
- **Driver versioning:** Rütli will publish a compatibility matrix for supported Canton/Splice versions; breaking SPI changes are handled by release branches, not silent breakage.
- If none of the above is considered a compatibility break for existing non-KMS deployments: *No backward compatibility impact on nodes that do not adopt Rütli.*

---

## Milestones and Deliverables

### Milestone 1: Production-Ready 1.0 Release & Public DevNet Soak

- **Estimated Delivery:** 2–3 months from grant approval
- **Focus:** Turn the existing v0.2.x stack into a release operators can trust for testnet authority keys, with public evidence.
- **Deliverables / Value Metrics:**
  - Tagged **v1.0.0** (or v1.0.0-rc → v1.0.0) on GitHub with Apache-2.0 artifacts: `rutli-driver.jar`, digest-pinned `rutli-cosigner` multi-arch images, Helm chart.
  - Compatibility matrix for at least one current Splice validator bundle + matching Canton API jars; CI green including mTLS cluster smoke.
  - **DevNet soak completed** per published runbook (≥14 days, scheduled drills: cosigner failover, mid-traffic refresh, backup/restore of a share), with a **public soak report** (metrics summaries, drill outcomes, known limitations).
  - Operator docs pack linked from README: quickstart, ceremony, DR, approval quorum, observability, wire contract.
  - Stakecraft runs at least one non-production validator path on Rütli continuously through the soak window (design-partner proof).

### Milestone 2: Independent Security Audit & Remediation

- **Estimated Delivery:** 3–5 months from grant approval (overlaps late M1 / early M3)
- **Focus:** Third-party review of the crypto integration and operator threat model before any recommendation of mainnet authority keys.
- **Deliverables / Value Metrics:**
  - Auditor engaged with published scope (FROST core, cosigner, driver policy/audit path, deploy configs) agreed with Security subcommittee input where appropriate.
  - Written report delivered to Stakecraft and summary shared with the Tech & Ops Committee under the Fund’s usual confidentiality practices for security findings.
  - **All critical and high findings remediated** in a tagged release; medium/low either fixed or explicitly deferred with rationale in CHANGELOG.
  - Public remediation blog or technical note (no exploit detail that increases risk).

### Milestone 3: Multi-Operator Adoption

- **Estimated Delivery:** Opens on M1 acceptance; completes within 9 months of grant approval
- **Focus:** Prove Rütli is used beyond Stakecraft.
- **Deliverables / Value Metrics:**
  - **Adoption events** (tranche-paid), each independently evidenced:
    - An **external organization** (not Stakecraft) runs a Canton/Splice participant whose Ed25519 namespace/identity key is generated and signed via Rütli FROST (DevNet, TestNet, or Mainnet), for ≥14 continuous days, with operator attestation and (where possible) public metrics or Scan visibility.
  - Adoption-enablement work: ≥2 public operator sessions (workshop, office hours, or conference talk); issue tracker SLA for external bug reports during the grant window.
  - Maintenance plan published: supported versions, security disclosure process, and ≥12 months stewardship commitment from Stakecraft after M1.

| Adoption deliverable | Acceptance evidence | Tranche |
|---|---|---|
| External operator #1 | Attestation + soak-style checklist (or Mainnet if ready) | See Funding |
| External operator #2 | Same | See Funding |
| External operator #3 | Same | See Funding |
| Adoption completion gate | ≥3 external GitHub issues/PRs from non-Stakecraft accounts triaged to resolution; docs used by adopters without private Stakecraft screenshare as primary onboarding | See Funding |

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- **M1:** Public v1.0 artifacts + public DevNet soak report meeting the drill/exit criteria in the soak runbook; docs pack complete.
- **M2:** Audit completed; critical/high closed in a tagged release; remediation note published; Committee (or Security subcommittee) acknowledges residual risk posture is acceptable for the claimed deployment tier (testnet vs mainnet guidance stated explicitly).
- **M3:** Adoption tranches only when **independent operators** demonstrate real Rütli-backed keys — not Stakecraft self-attestation alone; completion gate shows community engagement beyond the founding team.
- Documentation and knowledge transfer sufficient for a third party to deploy without reverse-engineering the repo.
- Alignment with stated value: **more Canton operators can protect non-rotatable authority keys with open, self-hosted threshold crypto.**

Artifact-only criteria (e.g. “CI green”) are necessary but **not sufficient**.

---

## Funding

**Total Funding Request:** **950,000 CC** base (engineering + adoption), plus **audit pass-through** estimated **120,000–180,000 CC** (vendor quote), paid with Milestone 2.

Weighting intentionally favors **audit readiness and external adoption** over net-new feature build, because the core implementation already exists. This grant buys trust, evidence, and distribution — not a greenfield rewrite.

### Payment Breakdown by Milestone

| Milestone | Payment | % of base |
|---|---|---|
| M1 — 1.0 release & public DevNet soak | **250,000 CC** upon committee acceptance | ~26% |
| M2 — Security audit remediations (engineering) | **150,000 CC** upon committee acceptance | ~16% |
| M2 — Audit vendor (pass-through) | **120,000–180,000 CC** against invoice / quote | outside base |
| M3 — Adoption | **up to 550,000 CC** (see tranches) | ~58% |
| **Base total** | **950,000 CC** | **100%** |

**Milestone 3 tranche detail (up to 550,000 CC):**

| Tranche | Amount |
|---|---|
| External operator adoption #1 | 120,000 CC |
| External operator adoption #2 | 120,000 CC |
| External operator adoption #3 | 120,000 CC |
| Adoption completion gate (docs + community triage + stewardship plan) | 190,000 CC |

Unused adoption tranches are simply not paid. No acceleration bonus requested.

### Volatility Stipulation

Project duration for M1–M2 is targeted **under 6 months**. Milestone 3 adoption may extend to **9 months** from approval. The grant is denominated in fixed Canton Coin. At the standard 6-month review point, remaining milestones may be re-evaluated for USD/CC volatility and scope. Should the Committee request material scope changes that push remaining work beyond 6 months, those milestones are renegotiated per CIP-0100 / Fund norms.

---

## Co-Marketing

Upon milestone releases, Stakecraft will collaborate with the Canton Foundation on:

- Coordinated announcement of the open-source 1.0 and audit completion
- Technical blog / case study on self-hosted threshold protection of Canton namespace keys
- Operator-facing workshop or demo at a Foundation-aligned ecosystem event
- Clear messaging that Rütli is **optional open infrastructure** for self-hosted operators — not a mandate and not a displacement narrative toward other custody categories

---

## Motivation

**Problem.** Namespace keys are single points of catastrophic failure. Software keys on the participant host and single-device HSM setups concentrate risk. Institutional and regulated operators increasingly require **threshold control, separation of duties, and auditability** inside their own perimeter.

**Why a Development Fund grant.** This is shared security infrastructure: every validator and many app providers eventually face the same key-custody question. An OSS reference implementation lowers systemic risk and reduces duplicated private engineering across operators.

**Who benefits (estimate).**

- **Validator / participant operators** — primary adopters (self-hosted key protection).
- **SV operators** — participant-side namespace keys (CometBFT keys remain a separate stack).
- **App providers running their own participants** — same SPI path.
- Rough framing: if even **5–10%** of publicly visible validators adopt self-hosted KMS drivers over a year, the network’s authority-key posture improves materially; Rütli aims to be the default OSS option in that segment.

**Why Stakecraft.** We operate Canton infrastructure, already built and open-sourced the stack, and will remain long-term stewards. We are asking the Fund to finance the **trust and adoption layer** (audit, soak evidence, external operators), not to reimburse past R&D.

---

## Rationale

**Why threshold FROST via the KMS Driver SPI.** Canton has no remote-signer wire protocol analogous to CometBFT’s; the supported extension point for keeping keys out of the node is the KMS Driver. FROST-ed25519 yields standard signatures with no verifier changes. DKG is mandatory because namespace keys cannot be rotated into KMS after the fact.

**Why not only cloud KMS.** Cloud KMS is the right category for many enterprises already standardized on AWS/GCP. It is not available the same way for all OSS deployments, and some operators require on-prem / sovereign control planes. Rütli addresses that **self-hosted** category.

**Why not a managed custody product.** Managed and proprietary custody products (including MPCH Stronghold and peers) are a **different category**: productized custody, often with vendor-operated controls and commercial support models. Rütli is intentionally **Horcrux-shaped open infrastructure** — the operator runs every component. Positioning Rütli “against” those products would be a category error; the Fund should evaluate Rütli as **public-good tooling for self-hosted operators**, the same way Horcrux is evaluated in CometBFT ecosystems alongside commercial custody.

**Why fund now.** The implementation risk is largely retired (e2e on Canton 3.5.x, CI, soak harness). The remaining blockers to ecosystem use are exactly what grants are for: **independent audit, public operational evidence, and multi-party adoption**.

**Alternatives considered.**

| Alternative | Why not sufficient alone |
|---|---|
| Software keys on participant | Key in-process / on-disk; fails institutional threat models |
| Single HSM only | No t-of-n for the non-rotatable namespace key; HSM remains complementary for non-threshold keys (Rütli already has PKCS#11 backend) |
| Cloud KMS only | Different category; not universal for OSS / sovereign operators |
| Managed custody products | Different category; not OSS self-hosted common goods |
| Wait for DA to ship OSS threshold KMS | No public roadmap item replaces community Horcrux-class tooling; SPI exists specifically for external drivers |

**Default approach honored:** extend Canton through the published KMS Driver SPI; reuse upstream FROST constructions; do not fork Splice images (classpath/config overlay only).

---

## Team & Stewardship

- **Organization:** Stakecraft — Canton Network infrastructure operator.
- **Repository:** [https://github.com/Stakecraft/rutli](https://github.com/Stakecraft/rutli) (Apache-2.0). The repo is **private during proposal review** and will be made **public under Apache-2.0 at Milestone 1 acceptance** (or earlier if the Committee prefers visibility during review — read access can be granted to champions / Tech & Ops on request).
- **Maintenance:** Stakecraft commits to maintain Rütli for a minimum of **12 months after M1 acceptance**, including security patches, Canton/Splice version tracking on the declared support matrix, and responsive handling of vulnerability reports via GitHub Security Advisories.
- **Disclosure:** `SECURITY.md` private reporting path (in-repo).

---

## Notes for Reviewers

- Please treat **category positioning** carefully: this proposal requests funding for **open-source self-hosted threshold KMS infrastructure**, not for a custody product shoot-out.
- Suggested champions: Node Deployment & Operations SIG and Security SIG members listed in `sig-directory.md`.
- Audit vendor selection can be coordinated with the Security subcommittee; pass-through cost is capped by quote and invoiced evidence.
- Happy to split Phase-4 threshold ECDSA into a **separate later proposal** if the Committee prefers a narrower v1 surface.
