# LifeLedger — A Patient-Owned, Lifelong Health Record on Web3

> **Your Health. Your Keys. Your Life.**
> A decentralized EHR system where custody of a medical record evolves with the patient — from guardian-managed at birth, to co-managed as a teen, to fully sovereign at 18 — enforced by smart contracts, not paperwork.

This document is the single source of truth for the project: what we're building, why it's built this way, exactly how data moves through the system, the tech stack, and how the two of us split the work so we can build in parallel without blocking each other.

---

## 1. Elevator Pitch

Every time a patient switches doctors, moves cities, or changes insurance, their medical history fractures into another disconnected silo. Hospitals own the copy of your data — you don't. That's the root cause of ransomware exposure, lost vaccination records, and the "please fax my old provider" problem that still defines healthcare IT.

LifeLedger flips this: the patient (or their guardian, while they're a minor) holds the only keys that matter. Records are encrypted client-side, pinned to IPFS, and only a cryptographic pointer (hash) ever touches the blockchain. Access is short-lived and re-negotiated per visit, not standing forever. And — the centerpiece — a smart contract, not a company or a bureaucrat, automatically hands the patient full ownership of their own eighteen years of history the moment they turn 18.

---

## 2. Where This Project Comes From (and What We're Changing)

We're synthesizing two things:

**From the Healthchain paper (academic base):**
- The core privacy pattern: never put PII or medical content on-chain — only a hash/pointer. Actual data lives encrypted, off-chain, on IPFS.
- Fine-grained, **rule-based access control** — a doctor/pharmacist/receptionist only sees what their role and the patient's grant permit, not the whole record by default.
- The idea of a **composite view** — a subset of the full record generated per-request, not the entire history handed over wholesale.
- Provenance/audit trail — every read/write is a traceable, hashed transaction.

**From our own LifeLedger blueprint (the hackathon-ready design):**
- **ERC-4337 Account Abstraction** — no seed phrases. A parent logs in with email/passkey and a smart contract wallet is created for the child behind the scenes.
- **Progressive custody lifecycle** — Custodial (0–13) → Co-Managed (13–18) → Independent (18+), encoded directly into contract logic, not app-level policy.
- **Vaccination Soulbound Tokens (SBTs)** — non-transferable, unforgeable, un-losable proof of immunization minted straight to the patient's wallet.
- **Gasless UX** via a sponsoring paymaster, so parents/doctors never touch a gas token.

**Our "tadka" — what we're adding on top of both:**

1. **Scoped access grants, not all-or-nothing.** Healthchain's "composite view" idea gets implemented on-chain as a *category bitmask*. When a patient grants a doctor access, they can restrict it to specific record types (e.g., "Vaccination + Allergy only," not the full 18-year history). This is strictly more granular than the base LifeLedger blueprint, which grants full access per visit.
2. **Break-glass emergency access.** Healthchain's related-work section explicitly flags that MedRec-style systems don't handle emergencies well ("does not allow any family member to allow data access in emergency situations"). We add a time-locked, mandatory-audit-logged emergency override so a verified ER doctor can access critical fields (allergies, blood type, current meds) without a live patient grant — with the access event permanently and visibly flagged on the patient's own activity feed.
3. **A lighter-weight chain choice.** Healthchain used permissioned Hyperledger Fabric; we're deliberately going with a public L2 testnet (Polygon Amoy or Base Sepolia) instead — because SBTs, ERC-4337 smart accounts, and free public paymasters are EVM/L2-native and dramatically cheaper to prototype with than standing up a permissioned Fabric network from scratch in hackathon time.

Everything below assumes these three additions are "should-have," not mandatory for a first working demo — flagged as such in the feature list so we can decide together how much of the tadka makes it into the first working version.

---

## 3. Complete Feature List

Priority tags: **[CORE]** = the demo doesn't work without it · **[SHOULD-HAVE]** = strong differentiator, build if time allows · **[STRETCH]** = nice pitch line, cut first if squeezed.

### 3.1 Identity & Custody Lifecycle
- [CORE] Guardian creates a child's identity with just email/passkey login — smart contract wallet deployed behind the scenes via Account Abstraction, no seed phrase ever shown.
- [CORE] Guardian has full signing authority while `custodyStatus == Custodial`.
- [CORE] At age 13 (or demo-toggled), `enableCoManagement()` registers the teen's own login as a second signer; status moves to `CoManaged`.
- [SHOULD-HAVE] Dual-control (2-of-2) requirement for *sensitive* teen-phase actions (granting a new doctor long-term access, exporting data) while routine actions (viewing records, accepting a vaccination SBT) need only the teen's signature.
- [CORE] At 18, `claimIndependence()` is **permissionlessly callable by anyone** (the app, a keeper bot, or the user) — it verifies the age threshold on-chain, revokes the guardian's signer key, and re-points the decryption key chain so only the now-adult patient can derive access going forward. Nobody — not us, not the original guardian, not the original hospital — can reverse it.

### 3.2 Vaccination Soulbound Tokens (SBTs)
- [CORE] Verified clinicians (whitelisted in a `DoctorRegistry`) mint a non-transferable ERC-5192-style token directly to the patient's wallet on every vaccination.
- [CORE] Each SBT encodes vaccine type, batch number (for recall tracing), administering clinician, and timestamp.
- [SHOULD-HAVE] QR-based instant verification of a vaccination SBT by any third party (school, airline) without needing full record access.

### 3.3 The Doctor Visit Flow (the demo's centerpiece)
- [CORE] Patient/guardian generates a single-use, time-limited QR code encoding a signed access grant.
- [CORE] Doctor's clinic portal scans it, submits the signature, decrypts only what's authorized, and can add a new encrypted record anchored back on-chain.
- [CORE] Access **automatically expires** (default 24h) — the doctor retains zero standing access after the window closes.
- [SHOULD-HAVE] Scoped grants: the QR/grant transaction includes a record-category bitmask so the doctor only decrypts, e.g., allergy + past-diagnosis records, not everything.
- [STRETCH] Break-glass emergency access path for unscheduled ER visits.

### 3.4 Encryption & Storage
- [CORE] Every record is encrypted **client-side**, before it ever leaves the browser — nobody (including us) can read plaintext in transit or at rest.
- [CORE] Encrypted blobs are pinned to IPFS; only the resulting content hash (CID) — plus timestamp, issuer address, record-type tag — is written on-chain.
- [SHOULD-HAVE] Envelope encryption: each record gets its own random AES-256-GCM key, which is then individually wrapped (ECIES) for each authorized party's public key — so granting a new doctor access is a cheap key-wrap operation, not a full re-upload.

### 3.5 Access Control & Consent
- [CORE] Patient (or guardian, during custodial phase) is the sole grantor/revoker of access — `grantAccess()`, `revokeAccess()` gated by an `onlyController` modifier that respects the current custody phase.
- [SHOULD-HAVE] Role-based *and* scope-based rules (doctor vs. pharmacist vs. receptionist see different slices), inspired directly by Healthchain's XML-style ACL model, implemented here as an on-chain enum + bitmask instead of an off-chain XML ruleset.

### 3.6 GDPR — "Crypto-Shredding" Right to Erasure
- [SHOULD-HAVE] "Deletion" = destroying the per-record symmetric key everywhere we control it + issuing an IPFS unpin request. The on-chain CID becomes a pointer to content that is cryptographically unrecoverable and, over time, physically gone from the network — satisfying erasure without needing (or being able) to rewrite the chain.

### 3.7 Provenance / Audit Trail
- [CORE] Every access grant, revocation, and record addition emits an on-chain event, giving the patient a human-readable activity feed of *exactly* who touched their record and when.

### 3.8 Adult Onboarding & Legacy Record Migration
Covers the case of an adult who has, up to now, only ever had physical/paper records and wants to switch over.
- [SHOULD-HAVE] `createIndependentIdentity()` — a dedicated factory entry point for adults signing up directly: `custodyStatus = Independent` from creation, no guardian phase, no `claimIndependence()` detour needed.
- [SHOULD-HAVE] `addSelfAttestedRecord(cid, recordType)` — the patient adds their own historical records (old discharge summaries, allergy lists, prescriptions) without a doctor present. Same client-side-encrypt → IPFS → anchor pipeline as every other record.
- [SHOULD-HAVE] Provenance tagging on every `RecordRef`: `SelfReported` vs `ClinicianVerified`. Self-reported entries are visibly badged as such in the UI — they carry less trust weight than a record a doctor wrote live.
- [STRETCH] `attestRecord(recordId)` — callable only by a registered doctor, upgrades a `SelfReported` record to `ClinicianVerified` once they've reviewed the original paper document at a real visit. The original event date is preserved separately from the attestation date.
- [STRETCH] Retroactive vaccination attestation: a doctor who reviews a physical vaccination card can mint an SBT carrying `attestedRetroactively = true` and the true historical date — distinct from a live-administered SBT.
- [STRETCH] Client-side OCR (e.g. Tesseract.js, free/OSS, runs entirely in-browser) to help transcribe a photographed paper record into structured fields before encryption — keeps "nothing plaintext ever leaves the device" intact even during bulk migration.

### 3.9 Prescriptions & Pharmacist Flow
- [SHOULD-HAVE] `PharmacistRegistry.sol` — whitelist of verified pharmacists, mirrors `DoctorRegistry`.
- [SHOULD-HAVE] Prescription records carry a `PrescriptionStatus { Issued, PartiallyFilled, Fulfilled, Expired, Cancelled }` and a `refillsRemaining` counter — genuinely protects against a script being filled twice, a real gap in paper prescriptions today.
- [SHOULD-HAVE] `dispensePrescription(prescriptionId)` — callable only by a whitelisted pharmacist holding a valid, scoped (`Prescription`-only) access grant. Decrements `refillsRemaining`, flips status at zero.
- [SHOULD-HAVE] Same QR-based, time-boxed grant pattern as the doctor visit flow — patient shows a QR at the pharmacy counter, access lapses right after.

### 3.10 E-Referrals & Out-of-Network Sharing
- [SHOULD-HAVE] Doctor-to-doctor e-referrals: a referral record type linking `referringDoctorId → referredDoctorId`, which also auto-extends a scoped access grant to the receiving doctor for the referenced record categories.
- [SHOULD-HAVE] `grantReadAccess(anyPublicKey, scope, expiry)` — read-only sharing with a party *outside* any registry (a specialist abroad for a second opinion, a family member) via pure envelope-encryption re-wrap. No registry whitelisting, no write privilege.

### 3.11 Additional Record Granularity (Stretch)
- [STRETCH] Split `LabReport` into distinct **lab order** and **lab result** record types instead of conflating the two.
- [STRETCH] Appointment booking / receptionist role, modeled on Healthchain's receptionist ACL — flagged as scope creep for a hackathon demo since it's closer to hospital-operations software than a health record, but easy to bolt on if time allows.

### 3.12 Insurance Claim Verification — The Last Hook
- [LAST HOOK] Deliberately positioned as the final feature to build, after the rest of the pipeline is solid. It's a strong closing pitch line ("we thought about the business model"), not something the demo depends on.

**Methodology:**
1. `InsurerRegistry.sol` — a third whitelist, mirroring `DoctorRegistry`/`PharmacistRegistry`, for verified insurer addresses.
2. Patient grants an insurer a scoped access grant restricted to the *specific* claim-relevant record ID(s) — not a category, not the full history. A claim for a surgery shouldn't unlock mental health notes.
3. A read-only `verifyClaim(patientAddress, recordId)` view function lets the insurer's off-chain claims system confirm three things, without seeing anything beyond that one record: (a) the record's CID and issuer signature are genuinely anchored on-chain and untampered; (b) the insurer currently holds a valid, unexpired grant covering that exact record; (c) — optionally — that the record is `ClinicianVerified`, not `SelfReported`, since a claim probably shouldn't be payable off a patient-typed entry.
4. This is **verification, not payout**. The contract confirms a claimed diagnosis/procedure genuinely exists in the patient-controlled record and hasn't been altered — it does not move money or adjudicate the claim. Automated conditional payout stays a true future-scope item (§12), not something to design into the prototype.
5. Reuses the existing provenance pattern (§3.7): `ClaimVerificationRequested` / `ClaimVerified` events mean the patient sees "Insurer X verified claim for record Y on date Z" in their own activity feed — same audit trail as every other access, just with an insurer as the actor.

---

## 4. Full System Architecture

Four layers, and — deliberately — **no traditional backend server**. This keeps the whole thing on free-tier infrastructure and removes an entire class of "who's paying for hosting" problems.

```
┌──────────────────────────────────────────────────────────────────┐
│  FRONTEND — Next.js (App Router) + Tailwind + shadcn/ui           │
│  Role-aware dashboards: Guardian / Teen / Adult Patient / Doctor  │
│  Deployed as a static/serverless app — no server we manage        │
└───────────────────────────────┬──────────────────────────────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│  IDENTITY & AUTH — ERC-4337 Account Abstraction                     │
│  Privy (email/passkey login) → counterfactual Smart Contract Wallet │
│  Paymaster sponsors gas on testnet — patients/doctors never hold a  │
│  gas token                                                           │
└────────────────────────────────┬────────────────────────────────────┘
                                  │
                ┌─────────────────┼──────────────────┐
                ▼                                     ▼
┌───────────────────────────┐         ┌───────────────────────────────┐
│  ENCRYPTION + STORAGE      │         │  BLOCKCHAIN LAYER              │
│  eth-crypto (ECIES) client-│  CID    │  Polygon Amoy / Base Sepolia   │
│  side encryption →         │◄───────►│  Solidity contracts:           │
│  IPFS via web3.storage     │ anchored│  IdentityFactory, Identity,    │
│  (encrypted blobs only —   │ on-chain│  VaccinationSBT, Doctor/       │
│  we never see plaintext)   │         │  Pharmacist/InsurerRegistry    │
└───────────────────────────┘         └───────────────────────────────┘
```

### 4.1 Exactly How a Record Moves Through the System

This is the part that's easiest to get fuzzy on, so let's be precise about it — this is the sequence for a doctor adding a new diagnosis during a visit:

1. **Grant.** Patient's app generates a QR encoding a signed `grantAccess(doctorAddress, expiry, scopeBitmask)` call. This is a gasless UserOperation — the paymaster covers it.
2. **Scan & verify.** The clinic portal scans the QR, submits it, and the contract checks the signature and custody-phase authority (`onlyController`).
3. **Doctor writes.** Doctor fills a structured form (diagnosis notes, prescription, etc.) in the portal.
4. **Client-side encryption.** The instant "Save" is pressed, the browser generates a fresh AES-256-GCM key, encrypts the record with it, then wraps that AES key with the *patient's* public key via ECIES (`eth-crypto`). Nothing plaintext ever leaves the browser.
5. **Upload.** The encrypted blob goes to IPFS via `web3.storage`, returning a content-addressed CID.
6. **Anchor.** Doctor's (gas-sponsored) wallet calls `addRecord(cid, recordType)` — only valid because the access grant from step 1 is still active (`onlyAuthorizedIssuer` modifier).
7. **Expiry.** When `expiryTimestamp` passes (or the patient manually revokes), the doctor's decryption capability lapses. Every future visit needs a fresh, explicit grant — there is no standing access, ever.
8. **Patient reads.** Later, the patient's app fetches the CID from the on-chain record index, downloads the encrypted blob from IPFS, and decrypts it locally using their own private key (held inside their AA smart account's associated keypair, never exported to us).

That's the whole loop. Everything else in the app (dashboards, QR UI, the 18th-birthday demo toggle) is UI wrapped around this core sequence plus the custody-transition logic.

### 4.2 Record Types & Provenance

The `recordType` field on every anchored record is deliberately more granular than the original blueprint's four-value enum, to properly support hospital reports, prescriptions, and referrals:

```solidity
enum RecordType {
    Diagnosis, LabOrder, LabResult, Imaging, ClinicalNote,
    HospitalAdmission, DischargeSummary, SurgicalReport,
    Prescription, AllergyRecord, ReferralRecord, InsuranceClaim
}

enum ProvenanceSource { ClinicianVerified, SelfReported }
```

Every `RecordRef` carries both fields. `ClinicianVerified` records come out of the doctor-visit flow (§4.1) or a hospital/pharmacist writing directly during a visit; `SelfReported` records come from an adult migrating in paper history (§3.8) and stay visibly badged as such until a clinician reviews and attests them.

---

## 5. Detailed User Journeys

**Guardian (birth → child turns 13)**
Logs in with email → smart account deployed for the child, counterfactually (no gas spent until first real use) → adds birth data/allergies/family history (encrypted, same pipeline as above) → attends vaccination visits where the pediatrician mints SBTs directly to the child's wallet → has full `onlyController` authority.

**Teen (13 → 18)**
Registers their own login as a co-signer on the *same* smart account (not a new wallet) → can independently approve routine, low-risk actions → sensitive actions require the guardian's co-signature too → guardian retains read visibility throughout, nothing is hidden, but control is structurally shared.

**Adult Patient (18+)**
`claimIndependence()` fires (permissionless, automatic once the age threshold passes) → guardian's signer key is revoked on-chain → patient is now sole controller → can grant/revoke doctor access, view their full 18-year provenance trail, and exercise GDPR erasure on any record.

**Adult Migrating from Paper Records (never used LifeLedger before)**
Signs up directly via `createIndependentIdentity()` — no guardian phase, `custodyStatus = Independent` from day one → uses `addSelfAttestedRecord()` to enter historical vaccination cards, discharge summaries, allergy lists — photographed and optionally OCR-assisted, but still encrypted client-side before anything uploads → those entries are visibly badged `SelfReported` → at any real visit afterward, the attending doctor can review the physical original and call `attestRecord()` to upgrade it to `ClinicianVerified`.

**Doctor**
Registered by a whitelisting authority in `DoctorRegistry` → scans a patient's access QR at check-in → reads only the scoped slice of history they were granted → writes new records, encrypted to the patient's key → access automatically lapses at visit end.

**Pharmacist**
Registered in `PharmacistRegistry` → patient shows a QR at the counter, scoped to `Prescription` records only → pharmacist calls `dispensePrescription()`, which checks and decrements `refillsRemaining`, flipping status to `Fulfilled` at zero → access lapses immediately after, no standing view into anything else in the patient's history.

**Receptionist (stretch)**
Read-only, appointment-metadata-scoped access only, if appointment booking (§3.11) gets built — modeled on Healthchain's role-based ACL pattern.

**Insurer (last hook, §3.12)**
Registered in `InsurerRegistry` → holds a per-record (not per-category) scoped grant for exactly the record(s) tied to a specific claim → calls `verifyClaim()` to confirm the record is genuinely anchored, grant-covered, and (optionally) clinician-verified → the verification itself is logged to the patient's own activity feed, same as any other access.

---

## 6. Tech Stack (and Why — With Free-Tier Notes)

| Layer | Choice | Why / Free-tier notes |
|---|---|---|
| Chain | **Polygon Amoy** testnet (fallback: Base Sepolia) | Sub-cent gas, ~2s finality, free faucet. Fine to switch to Base Sepolia if Coinbase Smart Wallet tie-ins matter more for the pitch. |
| Smart contracts | **Solidity ^0.8.24 + Foundry** | Foundry is fully free/open-source, fast fuzz testing, no license cost. |
| RPC provider | **Alchemy** (free tier) | Generous free compute-unit allowance, works for both Amoy/Base Sepolia. |
| Account Abstraction | **Privy** (embedded wallets, free dev tier) | Email/passkey login → ERC-4337 smart account, no seed phrase UX. |
| Gas sponsorship | **Biconomy paymaster** (free tier) or Alchemy Account Kit's bundled paymaster | Sponsors testnet UserOperations so nobody needs a gas token. |
| Client-side encryption | **eth-crypto** (npm, free/OSS) | ECIES envelope encryption, works entirely in-browser. |
| Off-chain storage | **web3.storage** (free tier via w3up) | Free IPFS/Filecoin pinning, generous quota for a prototype's data volume. Pinata is a fine fallback. |
| Frontend | **Next.js 14 (App Router)** | Free to build/run; deploys free on Vercel. |
| Styling/UI | **Tailwind CSS + shadcn/ui** | Free component primitives, fast to theme. |
| Contract interaction | **wagmi + viem** | Typed hooks over the deployed ABI. |
| QR generation/scan | **qrcode.react** + **html5-qrcode** | Both free/OSS. |
| OCR (paper record transcription) | **Tesseract.js** (free/OSS, client-side) | Stretch feature for adult onboarding (§3.8) — transcription assist stays entirely in-browser, nothing plaintext leaves the device even mid-migration. |
| Automation (18th-birthday trigger) | **Chainlink Automation** or **Gelato** (free testnet tiers) | Keeps `claimIndependence()` genuinely automatic instead of relying on a manual click; contract remains permissionlessly callable as a trustless fallback either way. |
| Hosting | **Vercel** (frontend, free tier, custom domain supported) | No backend server needed — everything is client + chain + IPFS. |
| DNS | **Your existing domain**, via Cloudflare (free DNS) | Point a subdomain at Vercel — see §9. |
| Version control | **GitHub** (free, private repo is fine for two collaborators) | + GitHub Actions free minutes for running Foundry tests on push, if we want CI. |

No paid service is required anywhere in this stack for a hackathon-scale prototype.

---

## 7. Repository Structure

A single monorepo keeps the contract ABI and shared types flowing to the frontend without copy-pasting files around.

```
lifeledger/
├── contracts/                  # Foundry project — TRACK A (you)
│   ├── src/
│   │   ├── LifeLedgerIdentity.sol
│   │   ├── IdentityFactory.sol
│   │   ├── VaccinationSBT.sol
│   │   ├── DoctorRegistry.sol
│   │   ├── PharmacistRegistry.sol
│   │   └── InsurerRegistry.sol     # build last — §3.12
│   ├── test/                   # unit + fuzz tests
│   ├── script/                 # deploy scripts
│   └── foundry.toml
│
├── apps/
│   └── web/                    # Next.js frontend — TRACK B (teammate)
│       ├── app/
│       │   ├── guardian/
│       │   ├── patient/
│       │   ├── doctor/
│       │   ├── pharmacist-portal/
│       │   └── clinic-portal/
│       └── components/
│
├── packages/                   # the shared "contract" between tracks
│   ├── contract-abi/           # ABI + deployed addresses.json — generated by Track A, consumed by Track B
│   ├── shared-types/           # TS types: CustodyStatus, RecordType enum, etc. — owned jointly
│   ├── crypto/                 # encryptRecord() / decryptRecord() wrappers — Track A builds, Track B calls
│   ├── ipfs-client/            # uploadEncrypted() / fetchFromIPFS() — Track A builds, Track B calls
│   └── mocks/                  # fake ABI/crypto/IPFS responses — lets Track B build UI before contracts are live
│
├── docs/
│   └── architecture.md         # this doc, or a trimmed version of it
│
├── .github/workflows/          # optional: run `forge test` on every push
├── pnpm-workspace.yaml
├── package.json
└── README.md
```

**Why this shape works for two people:** the `packages/` folder is the contract between you. Track A owns the *implementation* of `crypto/` and `ipfs-client/`, but publishes a small, stable function signature early (`encryptRecord()`, `uploadEncrypted()`, etc.) that Track B can start calling — against a mock — from day one, without waiting for the real contracts to be deployed.

---

## 8. The Parallel Work Split

### Track A — Blockchain, Cryptography & Chain Infra (you)

- [ ] `LifeLedgerIdentity.sol` — custody-phase logic, `onlyController` modifier, `grantAccess`/`revokeAccess`/`addRecord`, `enableCoManagement`, `claimIndependence`
- [ ] `VaccinationSBT.sol` — non-transferable mint, verified-issuer gating
- [ ] `IdentityFactory.sol` — minimal-proxy (EIP-1167) clone deployment per patient
- [ ] `DoctorRegistry.sol` — whitelist/attestation for clinicians
- [ ] Foundry unit + fuzz tests, especially around access-control edge cases (expired grants, custody-phase transitions, replay of an old QR)
- [ ] Deploy scripts to Polygon Amoy (or Base Sepolia); export ABI + addresses into `packages/contract-abi`
- [ ] `packages/crypto`: envelope encryption (`eth-crypto` ECIES + AES-256-GCM) with a clean, documented function signature
- [ ] `packages/ipfs-client`: `web3.storage` upload/fetch wrapper
- [ ] Paymaster/AA backend config (Privy project setup, Biconomy paymaster policy) — publish the resulting provider config as something Track B just imports
- [ ] Keeper setup for `claimIndependence()` (Chainlink Automation / Gelato registration on testnet)
- [ ] If we build the scoped-access "tadka" feature: design the record-category bitmask and expose it as a shared enum in `packages/shared-types`
- [ ] `PharmacistRegistry.sol` + prescription status/refill logic on top of `addRecord` (§3.9)
- [ ] Adult onboarding path: `createIndependentIdentity()`, `addSelfAttestedRecord()`, `attestRecord()`, `ProvenanceSource` tagging (§3.8)
- [ ] `grantReadAccess()` for out-of-network sharing (§3.10)
- [ ] `InsurerRegistry.sol` + `verifyClaim()` — build **last**, only once the rest of the pipeline is solid (§3.12)

### Track B — Full-Stack / Frontend (teammate)

- [ ] Next.js app scaffold, role-based routing (`/guardian`, `/patient`, `/doctor`, `/clinic-portal`)
- [ ] Wallet connect / login UX using Privy's React components (against Track A's config)
- [ ] wagmi/viem hooks wired to `packages/contract-abi` for every read/write the UI needs
- [ ] Guardian dashboard: create-child-identity flow, add-record form, vaccination history view
- [ ] Doctor/clinic portal: QR scan flow, patient-history view respecting scoped access, new-record form
- [ ] QR generation (patient side) via `qrcode.react`
- [ ] Activity feed UI, driven by on-chain events
- [ ] The 18th-birthday "time-warp" demo toggle UI
- [ ] Tailwind/shadcn styling pass, loading/error/empty states, toasts
- [ ] Early on: build every screen against `packages/mocks` so none of this is blocked on contracts being deployed
- [ ] Pharmacist portal: QR scan, `dispensePrescription()` call, refill-count UI (§3.9)
- [ ] Adult self-onboarding flow: direct signup form, "add historical record" upload UI, `SelfReported`/`ClinicianVerified` badges (§3.8)
- [ ] Optional: Tesseract.js-powered photo-to-form assist for paper record migration
- [ ] Minimal insurer-facing verification UI — build very last, after everything else works (§3.12)

### Shared / Joint Ownership
- [ ] `packages/shared-types` — enums and interfaces both sides read from
- [ ] Deciding together: Polygon Amoy vs. Base Sepolia; Privy vs. an alternative AA provider; how much of the "tadka" (scoped grants, break-glass) makes the first working cut vs. gets added after the core loop works

### Git Workflow
- `main` — always demoable, protected
- `dev` — integration branch, both merge here first
- Feature branches: `feat/contracts-<name>` for Track A, `feat/frontend-<name>` for Track B
- Small, frequent PRs against `dev`, even solo-reviewed — keeps merge conflicts in `packages/` small
- Conventional commits (`feat:`, `fix:`, `test:`) make it easy to scan history when integrating
- `.env.example` checked in (never the real `.env`) so either of us can spin up the other's half locally

---

## 9. Using Your Domain

Since there's no backend server to host, deployment is just: frontend → Vercel, then point your domain at it.

1. Deploy `apps/web` to Vercel (free tier, connect the GitHub repo, auto-deploys on push to `main`).
2. In Vercel, add your domain (e.g. `app.yourdomain.com`) under Project → Settings → Domains.
3. In your DNS provider (Cloudflare free tier is easiest), add a `CNAME` record: `app` → `cname.vercel-dns.com`.
4. SSL is automatic and free on Vercel's side.
5. Optional: a second subdomain (e.g. `pitch.yourdomain.com` or the bare root) for a static one-page pitch/landing site, deployed the same way.

No server costs, no domain-registrar-specific lock-in beyond DNS records you fully control.

---

## 10. Environment Variables (template)

```bash
# apps/web/.env.example
NEXT_PUBLIC_CHAIN=polygon-amoy
NEXT_PUBLIC_ALCHEMY_RPC_URL=
NEXT_PUBLIC_PRIVY_APP_ID=
NEXT_PUBLIC_LIFELEDGER_IDENTITY_FACTORY_ADDRESS=
NEXT_PUBLIC_VACCINATION_SBT_ADDRESS=
NEXT_PUBLIC_DOCTOR_REGISTRY_ADDRESS=
NEXT_PUBLIC_WEB3_STORAGE_TOKEN=

# contracts/.env.example
ALCHEMY_RPC_URL=
DEPLOYER_PRIVATE_KEY=
BICONOMY_PAYMASTER_API_KEY=
```

---

## 11. Open Design Decisions (decide together before building)

- **Chain:** Polygon Amoy (broader healthcare/RWA precedent) vs. Base Sepolia (Coinbase Smart Wallet ecosystem tie-in for the pitch)?
- **AA provider:** Privy (simplest embedded-wallet DX) vs. Alchemy Account Kit (one vendor for both RPC + AA, less integration surface)?
- **Scope of the "tadka" for v1:** do we build scoped/category access grants for the first demo, or ship full-access-per-grant first and layer scoping in once the core loop (§4.1) works end-to-end?
- **Emergency access:** worth the added contract complexity for a hackathon demo, or a "future scope" pitch line instead?
- **Self-attested record trust in the UI:** should `SelfReported` records be visually de-emphasized in a doctor's view during a visit, or just badge-tagged and otherwise treated equally? Worth mocking up rather than deciding in the abstract.
- **Insurance methodology scope:** confirm we're building *verification only* (§3.12) and treating automated payout as explicitly out of scope for the prototype, so it doesn't quietly balloon into a second contract system.

---

## 12. Future Scope (post-hackathon, not for the prototype)

- AI medical copilot that queries the encrypted record client-side (zero server-side plaintext exposure)
- Wearable/IoT data streams anchored the same way as clinical records
- Automated, conditional insurance payout smart contracts, built on top of the §3.12 verification layer once it exists
- Cross-border portable vaccination passports built on the same SBT mechanism
- Opt-in, compensated, consent-revocable research data pools (DAO-governed)
