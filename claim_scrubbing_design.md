# Claim Scrubbing System — System Design
**Independence OS | Pre-Submission Denial Prevention**
*Author: System Design Document | Date: 2026-03-15*

---

## Table of Contents

1. [Problem Framing](#1-problem-framing)
2. [System Overview](#2-system-overview)
3. [Scrubbing Engine — Five Layers](#3-scrubbing-engine--five-layers)
4. [Payer Rules Engine](#4-payer-rules-engine)
5. [Documentation Audit (LLM Layer)](#5-documentation-audit-llm-layer)
6. [ML Denial Predictor](#6-ml-denial-predictor)
7. [Data Models](#7-data-models)
8. [Remediation Interface](#8-remediation-interface)
9. [Feedback Loop](#9-feedback-loop)
10. [Key Design Decisions & Trade-offs](#10-key-design-decisions--trade-offs)
11. [Failure Modes & Mitigations](#11-failure-modes--mitigations)
12. [Scaling Considerations](#12-scaling-considerations)
13. [Appendix A: Denial Category Breakdown](#appendix-a-denial-category-breakdown)
14. [Appendix B: CCI / NCCI / LCD Explained](#appendix-b-cci--ncci--lcd-explained)
15. [Appendix C: Eraser.io Diagram Code](#appendix-c-eraserio-diagram-code)

---

## 1. Problem Framing

### What We're Actually Solving

A claim denial is not a billing problem — it is almost always a **documentation or coding problem that was already baked in before the claim left the clinic**. By the time a denial arrives, the damage is done: staff time is wasted, payment is delayed by weeks, and some claims are written off entirely.

The core challenge: **catch every preventable denial before the claim is transmitted**, across 1,000+ payers with different rules that change quarterly.

| Denial Root Cause | Share of Denials | Can be caught pre-submission? |
|---|---|---|
| Missing / invalid demographic info | ~15% | ✅ Yes — structural check |
| Duplicate claim | ~10% | ✅ Yes — deduplication check |
| Patient not eligible / not covered | ~20% | ✅ Yes — eligibility verification |
| Coding errors (CCI, NCCI, bundling) | ~25% | ✅ Yes — rule engine |
| Documentation doesn't support billing level | ~15% | ✅ Yes — LLM audit |
| Medical necessity not established | ~10% | ✅ Yes — LCD/NCD check |
| Payer-specific edge cases | ~5% | ⚠️ Partially — ML predictor |

**Target**: Catch enough errors to move denial rate from 30% → under 5%.

### What a Claim Actually Contains

```
┌─────────────────────────────────────────────────────────┐
│  CLAIM                                                  │
│                                                         │
│  Patient:   John Smith, DOB 1965-03-12, Ins ID XY123    │
│  Provider:  Dr. Martinez, NPI 1234567890                │
│  Payer:     Blue Cross Blue Shield Illinois             │
│  DOS:       2026-03-14                                  │
│  Place:     Office (POS 11)                             │
│                                                         │
│  Diagnoses (ICD-10):                                    │
│    R09.1  Pleurisy                                      │
│    I10    Essential hypertension                        │
│                                                         │
│  Procedures (CPT):                                      │
│    99214  Office visit, established, moderate MDM       │
│    93000  EKG with interpretation                       │
│                                                         │
│  Linked note:  [Progress note from Medical Scribe]      │
└─────────────────────────────────────────────────────────┘
```

The claim is a structured form (CMS-1500 / 837P EDI file), but its validity depends heavily on what is written in the unstructured progress note that backs it up.

---

## 2. System Overview

```
┌───────────────────────────────────────────────────────────────────┐
│  INPUT SOURCES                                                    │
│  EHR (ICD-10 + CPT + Note)  ──►  Claim Builder  ──►  Raw Claim    │
└─────────────────────────────────────────┬─────────────────────────┘
                                          │
                          ┌───────────────▼──────────────────┐
                          │      CLAIM INTAKE & PARSER       │
                          │  Normalize, validate structure,  │
                          │  resolve patient + payer records │
                          └───────────────┬──────────────────┘
                                          │
          ┌───────────────────────────────▼──────────────────────────┐
          │               SCRUBBING ENGINE (5 parallel layers)       │
          │                                                          │
          │  Layer 1          Layer 2          Layer 3               │
          │  Structural       Clinical Rules   Payer-Specific        │
          │  Validation       (CCI/NCCI/LCD)   Rules                 │
          │                                                          │
          │  Layer 4          Layer 5                                │
          │  Documentation    ML Denial                              │
          │  Audit (LLM)      Predictor                              │
          └───────────────────────────────┬──────────────────────────┘
                                          │
                          ┌───────────────▼──────────────────┐
                          │    ISSUE AGGREGATOR & PRIORITIZER│
                          │  BLOCKING → WARNING → INFO       │
                          └───────────────┬──────────────────┘
                                          │
                ┌─────────────────────────▼──────────────────────┐
                │          REMEDIATION INTERFACE                 │
                │  Billing team reviews issues + suggested fixes │
                │  Auto-fix for deterministic errors             │
                │  Manual queue for judgment calls               │
                └─────────────────────────┬──────────────────────┘
                                          │
                          ┌───────────────▼──────────────────┐
                          │       CLEAN CLAIM SUBMISSION     │
                          │  837P EDI → Clearinghouse →      │
                          │  Payer                           │
                          └───────────────┬──────────────────┘
                                          │
                          ┌───────────────▼──────────────────┐
                          │   PAYER RESPONSE & FEEDBACK LOOP │
                          │  Paid / Denied / Pending         │
                          │  Denials → retrain ML model      │
                          │  New patterns → update rules     │
                          └──────────────────────────────────┘
```

---

## 3. Scrubbing Engine — Five Layers

All five layers run **in parallel** on the same claim. Total scrub time target: under 10 seconds.

---

### Layer 1 — Structural Validation

**What it checks**: Every required field is present and syntactically valid. These are deterministic, binary checks — pass or fail.

| Check | Example failure | Severity |
|---|---|---|
| Patient demographics complete | Missing DOB | BLOCKING |
| Insurance ID present and formatted | No member ID | BLOCKING |
| Provider NPI valid (Luhn check) | NPI checksum fails | BLOCKING |
| ICD-10 codes exist in current codebook | Deleted code from 2022 | BLOCKING |
| CPT codes exist in current codebook | Non-existent CPT | BLOCKING |
| Date of service not in future | DOS = tomorrow | BLOCKING |
| Date of service within timely filing limit | Claim filed 18 months late | BLOCKING |
| Place of service code valid | POS 99 doesn't exist | BLOCKING |
| Rendering provider matches billing provider | NPI mismatch | WARNING |
| Diagnosis codes in correct order | Primary diagnosis must be first | WARNING |
| Units reasonable | 99 units of 99214 in one day | WARNING |

**Implementation**: Pure rule validation, no ML. Codebooks (ICD-10, CPT) loaded into a Redis set at startup. Rules run as a validation chain — fail-fast on BLOCKING issues.

**Codebook freshness**: ICD-10 updates annually (October 1). CPT updates annually (January 1). Automated diff job runs on these dates, flags any codes in active claims that are no longer valid.

---

### Layer 2 — Clinical Rule Engine (CCI / NCCI / LCD / NCD)

This is the densest layer. It enforces published government and industry coding rules.

#### 2a. CCI Edits (Correct Coding Initiative)

CMS publishes quarterly tables of CPT code pairs that cannot be billed together on the same date of service **without a specific modifier**.

```
Mutually exclusive pairs (never allowed together):
  99214 + 99213  → can't bill two E&M codes same day, same provider

Comprehensive/component pairs (one bundles the other):
  20610 (joint aspiration) + 99214
  → If both are on the same claim, modifier 25 is REQUIRED on 99214
    to indicate the E&M was separately identifiable
```

**Data source**: CMS NCCI files (quarterly release, downloadable CSV/Excel). ~250,000 code pairs.
**Implementation**: Hash map of `{(cpt_a, cpt_b): modifier_required}` loaded in memory. O(1) lookup per pair.

#### 2b. NCCI Procedure-to-Procedure Edits

Similar to CCI but specifically between procedure codes:

- **Mutually exclusive edits**: Two codes that by definition can't happen in the same session
- **Comprehensive/component edits**: One code is a subset of another

```
Example:
  27447 (total knee replacement) + 27310 (arthrotomy, knee)
  → 27310 is a component of 27447 — can't bill both
```

#### 2c. LCD (Local Coverage Determination) / NCD (National Coverage Determination) (Medical Necessity Coverage)

Every CPT procedure has a list of ICD-10 diagnosis codes that make it **medically necessary**. If the diagnosis on the claim doesn't appear on that list, the payer will deny for lack of medical necessity.

```
CPT 72148 (MRI lumbar spine, without contrast):
  Covered diagnoses: M54.5 (low back pain), M51.16 (disc herniation), ...
  NOT covered for: Z00.00 (routine exam), R53.83 (fatigue)

Check: Do the ICD-10 codes on this claim appear in the LCD for each CPT?
```

**Data source**: CMS (Centres for medical services) LCD database (downloadable per MAC/region). ~2,000 LCDs covering thousands of CPT-ICD pairs.

**Complexity**: LCDs vary by MAC (Medicare Administrative Contractor) — the rules for New England are different from Texas. The clinic's geographic region determines which LCD applies.

#### 2d. Diagnosis Code Specificity

ICD-10 rewards specificity. Billing an unspecified code when a more specific one is available invites audits and some payers outright deny unspecified codes.

```
I21.9   Acute MI, unspecified       → will be flagged
I21.01  STEMI of LAD, initial       → acceptable

S82.001A  Fracture, unspecified     → flagged if laterality known
S82.001D  Fracture of right patella → correct
```

**Check**: For each ICD-10 code ending in digits like `9` (unspecified), check if the progress note contains enough detail to justify a more specific code. Flag for review.

#### 2e. Age / Sex Validity

Some CPT codes and ICD-10 codes are only valid for specific demographics:

```
V27.0   (outcome of delivery — single liveborn) invalid on male patient
99381   (preventive visit, infant < 1 year) invalid if patient DOB > 1 year
```

---

### Layer 3 — Payer-Specific Rules

This is the hardest layer to maintain. Each of the 1,000+ payers has its own coverage policies, prior authorization requirements, and claim submission rules — updated quarterly, often with minimal notice.

#### 3a. Payer Rule Database

Structure of a stored payer rule:

```python
PayerRule(
    payer_id = "BCBS_IL",
    rule_type = "PRIOR_AUTH",
    cpt_codes = ["27447"],          # total knee replacement
    condition = "units >= 1",
    action = "REQUIRE_AUTH",
    auth_portal = "https://provider.bcbsil.com/auth",
    effective_date = "2026-01-01",
    source = "PAYER_PORTAL"
)
```

#### 3b. Rule Sources (Triple-Source Strategy)

**Source 1 — Clearinghouse APIs** (Availity, Change Healthcare):
- Pre-built payer rule feeds from clearinghouses
- Cover ~600 of the top payers
- Updated in near-real-time

**Source 2 — Payer Portal Scraping**:
- Automated scraper monitors payer portals for policy updates
- Runs weekly, diffs against stored rules
- Human review required before activating new rules
- Covers payers not in clearinghouse feeds

**Source 3 — Denial Feedback Loop** (see Section 9):
- When a claim is denied for a payer-specific reason that wasn't in the rule database → the denial reason is parsed → a new candidate rule is created → human reviewer approves → rule is added

#### 3c. Prior Authorization Check

If a CPT on the claim requires prior authorization from this payer:
1. Check if auth number is present on the claim
2. If auth number exists, verify it hasn't expired and covers the procedure
3. If no auth number, **BLOCKING** error with the auth portal URL

#### 3d. Fee Schedule Validation

Each payer has a contracted fee schedule. If the billed amount is outside the expected range (> 2× the contracted rate or < 50%), flag as a likely data entry error.

---

### Layer 4 — Documentation Audit (LLM)

This is the highest-value layer and the one no rule engine can fully replace. It answers: **does what the doctor wrote justify what the doctor billed?**

#### 4a. E&M Level Audit

The most important check. E&M codes (99213, 99214, 99215) are the most frequently billed and most frequently audited.

CMS uses **Medical Decision Making (MDM)** to determine the appropriate E&M level:

```
MDM has 3 components, each scored independently:

1. Number and complexity of problems addressed
   - 1 self-limited problem          → Minimal
   - 1 stable chronic illness        → Low
   - 1 new problem with workup       → Moderate
   - 1 problem with risk to life     → High

2. Amount and/or complexity of data reviewed
   - No external data                → Minimal
   - Labs/imaging reviewed           → Low
   - Independent interpretation      → Moderate
   - Discussion with another provider→ High

3. Risk of complications / morbidity
   - OTC drug management             → Minimal
   - Prescription drug management    → Low
   - Minor surgery decision          → Moderate
   - Drug therapy requiring monitoring → High

E&M Level = highest level where 2 of 3 components are met:
  99213 = Low MDM     (2 of 3 components at Low)
  99214 = Moderate MDM (2 of 3 components at Moderate)
  99215 = High MDM    (2 of 3 components at High)
```

**The LLM prompt**:

```
You are a medical billing auditor. Review the progress note below and
score the Medical Decision Making (MDM) using CMS 2021 E&M guidelines.

Score each of the 3 components (Problems, Data, Risk) as:
Minimal / Low / Moderate / High

Then determine the supported E&M level (99213/99214/99215).

Progress Note:
[full note text]

Billed E&M code: 99214

Output format:
{
  "problems_score": "Moderate",
  "data_score": "Low",
  "risk_score": "Moderate",
  "supported_em_level": "99214",
  "match": true,
  "reasoning": "...",
  "confidence": 0.87
}
```

If `supported_em_level` < billed level → **BLOCKING** flag (potential upcoding = fraud risk)
If `supported_em_level` > billed level → **WARNING** (potential undercoding = lost revenue)

#### 4b. Diagnosis Support Check

Every ICD-10 code on the claim must be **explicitly mentioned or clearly implied** in the progress note.

```
Claim has:  I10 (hypertension)
Note says:  "patient on lisinopril for blood pressure control"
→ Hypertension is supported ✅

Claim has:  J18.9 (pneumonia)
Note says:  no mention of pneumonia anywhere
→ BLOCKING — unsupported diagnosis on claim
```

#### 4c. Procedure Documentation Check

If a procedure CPT (e.g., 93000 for EKG) is billed, the note must document:
- That the procedure was performed
- The findings or interpretation

```
CPT 93000 billed
Note says: "EKG performed — normal sinus rhythm, no ST changes"
→ Supported ✅

CPT 93000 billed
Note says: "EKG ordered"   ← only ordered, not performed
→ BLOCKING — procedure billed but not documented as performed
```

#### 4d. Specificity Upgrade Suggestion

The LLM scans the note for clinical details that would justify upgrading an unspecified ICD-10 code to a more specific one:

```
Billed:  M54.9 (dorsalgia, unspecified)
Note:    "low back pain, lumbar region, bilateral, chronic"
→ SUGGESTION: Use M54.5 (low back pain) — more specific, supported by note
```

---

### Layer 5 — ML Denial Predictor

Layers 1–4 catch known, rule-based errors. Layer 5 catches **unknown patterns** — the "I don't know why this payer keeps denying this but they do."

#### 5a. Model Architecture

**Input features**:
- Payer ID (encoded)
- CPT code(s)
- ICD-10 code(s)
- E&M level
- Place of service
- Provider specialty
- Patient age bucket
- Day of week / month (some payers batch-deny end-of-quarter)
- Prior auth present (Y/N)
- Claim amount

**Output**: `denial_probability` (0.0 → 1.0) + `top_denial_reasons` (ranked list)

**Model type**: Gradient boosting (XGBoost/LightGBM) — fast inference, interpretable, handles sparse categorical features well. No LLM needed here — this is pure pattern matching on historical outcomes.

#### 5b. Training Data

Source: All previously submitted claims + their outcomes (paid/denied/denied-reason).
Label: `denied = 1`, `paid = 0`
Retrain: Monthly, on rolling 12-month window.
Min training size: 500 claims per payer before payer-specific model is used. Below threshold → use specialty-level aggregate model.

#### 5c. Threshold Behavior

| Denial probability | Action |
|---|---|
| < 0.15 | No flag |
| 0.15 – 0.40 | INFO: "This claim type has elevated denial rate with this payer" |
| 0.40 – 0.70 | WARNING: Show top predicted denial reasons |
| > 0.70 | WARNING + escalate to senior biller review |

Note: Layer 5 never produces BLOCKING issues. It's advisory only — we can't block a claim just because the model thinks it *might* get denied.

---

## 4. Payer Rules Engine

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PAYER RULES ENGINE                       │
│                                                             │
│  ┌──────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│  │ Clearinghouse│  │ Payer Portal    │  │ Denial        │  │
│  │ Rule Feed    │  │ Scraper         │  │ Feedback      │  │
│  │ (Availity,   │  │ (weekly diff,   │  │ (new patterns │  │
│  │ Change HC)   │  │ human review)   │  │ → candidate   │  │
│  └──────┬───────┘  └───────┬─────────┘  │ rules)        │  │
│         │                  │            └───────┬───────┘  │
│         └──────────────────┴────────────────────┘          │
│                            │                               │
│                   ┌────────▼────────┐                      │
│                   │  Rule Store     │                      │
│                   │  (PostgreSQL +  │                      │
│                   │   versioned,    │                      │
│                   │   effective     │                      │
│                   │   dates)        │                      │
│                   └────────┬────────┘                      │
│                            │                               │
│                   ┌────────▼────────┐                      │
│                   │  Rule Cache     │                      │
│                   │  (Redis,        │                      │
│                   │   per-payer     │                      │
│                   │   hash map)     │                      │
│                   └─────────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

### Rule Versioning

Rules have effective and expiry dates. A claim for a DOS in the past uses the rules that were in effect on that date — not today's rules. This is critical for handling appeals and late submissions.

```python
# Fetch rules applicable to a specific date of service
rules = RuleStore.get_rules(
    payer_id="BCBS_IL",
    effective_on=date_of_service
)
```

### Rule Conflict Resolution

When two rules conflict (e.g., a national CCI edit says "bundle these codes" but a payer-specific rule says "these codes are separately payable"):

**Priority order**:
1. Payer-specific rules (highest — they override everything)
2. LCD / NCD (CMS coverage determinations)
3. NCCI / CCI edits (CMS coding edits)
4. Internal billing guidelines (lowest)

---

## 5. Documentation Audit (LLM Layer)

### Detailed LLM Prompt Strategy

Rather than one massive prompt, three focused prompts run in parallel:

**Prompt A — MDM Scoring** (described in Layer 4a above)

**Prompt B — Diagnosis Support Verification**:
```
For each ICD-10 code listed below, determine whether the progress note
provides explicit or clear implicit support for that diagnosis.

ICD-10 codes billed: [list]
Progress note: [full text]

For each code, output:
{
  "code": "I10",
  "supported": true,
  "evidence": "patient noted to be on lisinopril, BP 142/88"
}
```

**Prompt C — Procedure Documentation Verification**:
```
For each CPT procedure code below, verify the progress note documents:
(1) that the procedure was performed (not just ordered)
(2) the findings or result of the procedure

CPT codes billed: [list]
Progress note: [full text]

For each code, output:
{
  "code": "93000",
  "performed_documented": true,
  "findings_documented": true,
  "evidence": "EKG performed — normal sinus rhythm"
}
```

### Confidence Thresholds

| Confidence | Action |
|---|---|
| ≥ 0.90 | Auto-apply result (BLOCKING or PASS) |
| 0.70 – 0.89 | Apply result with review flag |
| < 0.70 | Escalate to senior biller — LLM uncertain |

### Hallucination Guard

Same principle as the Medical Scribe system: every LLM finding must cite a specific sentence from the note as evidence. If the LLM returns a finding with no evidence quote, it's discarded and the item is sent to human review.

---

## 6. ML Denial Predictor

### Feature Engineering

Beyond raw code values, engineered features that matter:

| Feature | Rationale |
|---|---|
| `code_pair_hist_denial_rate` | Historical denial rate for this exact CPT+ICD10 pair with this payer |
| `provider_payer_denial_rate` | This provider's denial rate with this payer (some providers have patterns) |
| `days_since_dos` | Claims filed close to timely filing limit are riskier |
| `auth_present_for_high_value_cpt` | High-value CPTs without auth → denied more often |
| `em_level_delta` | Difference between LLM-scored level and billed level |
| `new_diagnosis_for_patient` | First time this ICD-10 appears for this patient → higher scrutiny |
| `claim_amount_payer_percentile` | Claims in top 10% of amount for payer get audited more |

### Model Interpretability

The billing team must understand WHY the model flagged something. Use SHAP values to output top contributing features:

```
Denial probability: 0.73

Top reasons (SHAP):
  +0.31  This CPT+diagnosis pair denied 68% of the time with Aetna
  +0.22  No prior auth on file for CPT 27447
  +0.12  Claim amount in 94th percentile for this payer
  -0.08  Provider has strong payment history with this payer
```

This is shown directly to the biller, not just the probability score.

---

## 7. Data Models

### 7.1 Claim

```python
class Claim:
    claim_id: UUID
    encounter_id: UUID           # links back to Medical Scribe encounter
    patient_id: UUID
    provider_id: UUID
    payer_id: UUID
    date_of_service: date
    place_of_service: str        # POS code e.g. "11" = office
    icd10_codes: list[str]       # ordered, primary first
    cpt_lines: list[CPTLine]
    note_id: UUID                # progress note from Medical Scribe
    total_charge: Decimal
    status: ClaimStatus          # DRAFT, SCRUBBING, CLEAN, SUBMITTED, PAID, DENIED
    created_at: datetime
```

### 7.2 CPT Line

```python
class CPTLine:
    cpt_code: str
    modifiers: list[str]         # up to 4 modifiers
    units: int
    charge: Decimal
    diagnosis_pointers: list[int]  # which ICD-10 codes support this procedure
```

### 7.3 Scrub Result

```python
class ScrubResult:
    scrub_id: UUID
    claim_id: UUID
    issues: list[Issue]
    denial_probability: float    # from ML layer
    denial_probability_reasons: list[SHAPReason]
    overall_status: ScrubStatus  # CLEAN, HAS_WARNINGS, HAS_BLOCKING
    scrub_duration_ms: int
    scrubbed_at: datetime
```

### 7.4 Issue

```python
class Issue:
    issue_id: UUID
    scrub_id: UUID
    severity: Severity           # BLOCKING, WARNING, INFO
    layer: int                   # 1-5, which scrub layer found this
    category: IssueCategory      # STRUCTURAL, CCI, NCCI, LCD, PAYER_RULE,
                                 # EM_MISMATCH, UNSUPPORTED_DX, PROCEDURE_DOC,
                                 # ML_RISK
    affected_codes: list[str]    # which CPT/ICD codes are implicated
    description: str             # human-readable explanation
    suggested_fix: str           # what the biller should do
    auto_fixable: bool           # can system fix it without human input?
    auto_fix_applied: bool
    rule_reference: str | None   # e.g. "CCI Q1-2026 edit 99214/99213"
    evidence: str | None         # for LLM findings — the note quote
    resolved: bool
    resolved_by: UUID | None
    resolved_at: datetime | None
```

### 7.5 Payer Rule

```python
class PayerRule:
    rule_id: UUID
    payer_id: str                # "BCBS_IL", "AETNA", "MEDICARE_PART_B"
    rule_type: RuleType          # PRIOR_AUTH, BUNDLING, COVERAGE, TIMELY_FILING
    cpt_codes: list[str] | None
    icd10_codes: list[str] | None
    condition: dict              # JSON expression tree
    action: RuleAction           # BLOCK, WARN, REQUIRE_AUTH, FLAG
    effective_date: date
    expiry_date: date | None
    source: RuleSource           # CCI, NCCI, LCD, NCD, CLEARINGHOUSE, PAYER_PORTAL, DENIAL_FEEDBACK
    confidence: float            # for denial-feedback-derived rules: how confident we are
    approved_by: UUID | None     # human reviewer for payer-portal and denial-feedback rules
```

### 7.6 Claim Outcome (Feedback Loop)

```python
class ClaimOutcome:
    outcome_id: UUID
    claim_id: UUID
    payer_id: str
    submitted_at: datetime
    response_received_at: datetime
    status: OutcomeStatus        # PAID, DENIED, PARTIAL_PAYMENT, PENDING
    denial_reason_code: str | None   # ERA/835 denial reason code (CO-4, CO-97, PR-96...)
    denial_reason_text: str | None
    payment_amount: Decimal | None
    action_taken: str | None     # APPEAL, CORRECTED_CLAIM, WRITE_OFF
```

---

## 8. Remediation Interface

This is where the billing team lives. The scrub result must be actionable, not just a list of errors.

### 8.1 Interface Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  Claim #CLM-2026-4821 | John Smith | BCBS-IL | DOS: 2026-03-14     │
│  Scrub completed in 4.2s | Denial risk: 73% (HIGH)                 │
├────────────────────────────────────────────────────────────────────┤
│  ❌ BLOCKING ISSUES (2)    ⚠️ WARNINGS (3)    ℹ️ INFO (1)          │
├────────────────────────────────────────────────────────────────────┤
│  ❌ [Layer 2 — CCI]                                                │
│  CPT 20610 + 99214 billed together without modifier 25            │
│  → Add modifier 25 to CPT 99214 to indicate separately            │
│    identifiable E&M service                                        │
│  [Auto-fix: Add modifier 25]    [View rule: CCI Q1-2026]          │
│────────────────────────────────────────────────────────────────────│
│  ❌ [Layer 4 — Documentation]                                       │
│  Progress note does not document EKG was performed (only ordered) │
│  CPT 93000 requires documentation of interpretation               │
│  Evidence needed: "EKG performed — [findings]"                    │
│  → Ask doctor to amend note with EKG findings, then rebill        │
│  [Request note amendment]    [Remove CPT 93000 from claim]        │
│────────────────────────────────────────────────────────────────────│
│  ⚠️ [Layer 3 — Payer Rule]                                         │
│  BCBS-IL requires prior auth for CPT 27447 (added to plan)        │
│  Auth portal: provider.bcbsil.com/auth                            │
│  [Get auth now]    [Defer — auth pending]                         │
│────────────────────────────────────────────────────────────────────│
│  ⚠️ [Layer 5 — ML Risk]                                            │
│  This CPT+ICD combination denied 68% with BCBS-IL in last 90 days│
│  Top factor: No auth on file for 27447                            │
│  [View denial history]                                             │
│────────────────────────────────────────────────────────────────────│
│  [Apply All Auto-fixes]    [Hold for Doctor Amendment]    [Submit Clean Claim] │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 Issue Severity Rules

**BLOCKING**: Claim will definitely be denied if submitted as-is. Must be resolved before submission is enabled.

**WARNING**: Claim has elevated denial risk but might still be paid. Biller must acknowledge each warning before submitting.

**INFO**: Advisory only. Biller can dismiss without action.

### 8.3 Auto-Fix Capabilities

| Issue type | Auto-fixable? | How |
|---|---|---|
| Missing modifier (CCI edit) | ✅ | Add required modifier to CPT line |
| Diagnosis code order wrong | ✅ | Re-order ICD-10 codes per payer preference |
| Unspecified ICD-10 where specific available | ✅ (with confirmation) | Suggest upgrade, biller approves |
| Duplicate claim | ✅ | Flag and suppress duplicate |
| Invalid CPT (deleted code) | ❌ | Needs human — find replacement code |
| EKG not documented | ❌ | Needs doctor — note amendment required |
| Prior auth missing | ❌ | Needs human — get auth from payer portal |

### 8.4 Note Amendment Workflow

When a documentation issue requires the doctor to update the note:

```
Issue detected → billing team clicks "Request note amendment"
    ↓
Specific request sent to doctor's queue:
    "Please add EKG interpretation findings to note for DOS 2026-03-14"
    ↓
Doctor amends note (addendum — original note immutable)
    ↓
Scrubber auto-reruns Layer 4 on updated note
    ↓
Issue resolved → claim re-enters clean queue
```

---

## 9. Feedback Loop

This is what makes the system better over time and handles the "payer rules change quarterly" problem.

### 9.1 Claim Outcome Ingestion

After submission, payer responses arrive as **835 ERA (Electronic Remittance Advice)** files. These are parsed automatically:

```
835 ERA parsing:
  CAS segment → denial reason code
  CLM segment → claim identifier
  ADJ segment → payment or denial amount

Common denial reason codes:
  CO-4   → CPT code is inconsistent with modifier
  CO-97  → Payment included in allowance for another service
  CO-4   → Claim/service lacks information needed for adjudication
  PR-96  → Non-covered charge(s)
  OA-23  → Prior authorization not obtained
```

### 9.2 Denial Analysis Pipeline

For every denied claim:

```
Denial received
    ↓
Parse denial reason code (CO-4, PR-96, etc.)
    ↓
Match against original scrub result:
    Was this issue caught by the scrubber?
        YES → Measure: biller ignored a BLOCKING/WARNING issue
        NO  → New pattern discovered
    ↓
If new pattern:
    Extract features: (payer, cpt, icd10, denial_reason)
    Check if it meets rule-creation threshold (≥ 5 similar denials)
    Create candidate payer rule
    Route to human reviewer (billing manager)
    On approval → rule added to payer rules engine
    ↓
All denials feed ML model retraining (monthly)
```

### 9.3 Scrubber Performance Dashboard

Tracked metrics, visible to billing manager:

| Metric | Target | Alert threshold |
|---|---|---|
| Pre-scrub denial rate | (baseline) | |
| Post-scrub denial rate | < 5% | > 10% |
| BLOCKING issues auto-fixed rate | > 80% | < 60% |
| New rules added per month | — | > 20 (may indicate rule DB is stale) |
| False positive rate (issues flagged but claim paid anyway) | < 10% | > 25% |
| Scrub latency P95 | < 10s | > 30s |
| ML model AUC | > 0.82 | < 0.70 |

---

## 10. Key Design Decisions & Trade-offs

### Decision 1: When to Run the Scrubber

**Option A — At claim creation** (immediately after note is signed):
- Pros: Issues surfaced immediately while context is fresh; doctor still available to amend note
- Cons: Adds latency to the billing workflow; some issues may resolve before submission

**Option B — At submission** (billing team clicks "submit"):
- Pros: Final check with all information present
- Cons: Doctor may be unavailable for note amendments; delays payment

**Option C — Both** (soft check at creation, hard check at submission):
- Pros: Early warning during note writing, mandatory check before transmission
- Cons: Two scrub runs per claim

**Decision: Option C — Dual-stage**.
- Stage 1 (soft): Runs immediately after note is signed. Structural + CCI checks only. Shows warnings to biller. No blocking.
- Stage 2 (hard): Runs when biller clicks "submit". All 5 layers. BLOCKING issues prevent submission.

---

### Decision 2: How to Handle 1,000+ Payer Rules

**Option A — Manual rule database**: Staff manually enters rules from payer websites.
- Pros: Full control, human-verified
- Cons: Impossible to maintain at scale; rules change quarterly; 1,000 payers × 20 rules = 20,000 rules

**Option B — Clearinghouse API only**: Rely entirely on Availity / Change Healthcare feeds.
- Pros: Automated, covers top 600 payers well
- Cons: Doesn't cover all payers; sometimes lags actual payer changes; subscription cost

**Option C — ML from denial patterns only**: Learn rules from outcomes.
- Pros: Payer-agnostic; catches rules that aren't published anywhere
- Cons: Requires training data (can't catch new rules until we've been burned by them); not explainable to billers

**Decision: Triple-source hybrid** (described in Section 4). Clearinghouse for coverage breadth, portal scraping for accuracy, denial feedback for unknown rules. Each source has a confidence level — only high-confidence rules are auto-activated; others require human review.

---

### Decision 3: LLM for Documentation Audit — Risk of Hallucination

The LLM could hallucinate that a note supports a code when it doesn't — or vice versa. Both errors are costly.

**Mitigation**:
- Every LLM finding requires an evidence quote from the note
- Findings with confidence < 0.70 escalate to human review, not auto-applied
- Conservative default: when uncertain, flag for human review (never silently pass)
- Hallucination rate tracking: sample 5% of LLM decisions for human audit quarterly

---

### Decision 4: Where to Draw the Line on Auto-Fix

**Risk**: Auto-fixing a modifier error is safe. Auto-fixing a diagnosis code is not — changing clinical information without doctor involvement is a compliance violation.

**Rule**:
- Billing/coding corrections (modifier, order, format) → auto-fix allowed
- Clinical content changes (diagnosis codes, procedure codes, note content) → always require human approval
- Hard line: the system never changes ICD-10 codes without explicit biller + doctor confirmation

---

### Decision 5: Build vs. Buy the Clearinghouse Connection

**Option A — Direct payer connections**: Build EDI 837P/835 connections to each payer directly.
- Pros: No middleman fees; direct relationship
- Cons: Each payer has a different connection setup; 1,000+ integrations; massive engineering burden

**Option B — Clearinghouse** (Availity, Change Healthcare, Waystar):
- Pros: Single connection gives access to all payers; clearinghouses pre-validate 837P format; faster
- Cons: Per-claim fees (typically $0.10–$0.50/claim); dependency on third party

**Decision: Clearinghouse for submission** (standard industry practice). Build direct connections only for top 5 payers by volume where the per-claim fee savings justify it.

---

## 11. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Payer rule database stale | Miss new denial reason, claims denied | Weekly scraper diffs + denial feedback loop catches new patterns within 5 denials |
| LLM produces wrong MDM score | Claim blocked incorrectly or fraud risk passed | Confidence threshold → human review below 0.70; quarterly human audit of 5% of LLM decisions |
| Clearinghouse API down | Can't submit claims | Store claims in local queue; retry on restore; fallback to paper/portal submission for urgent claims |
| ML model drift (payer changes behavior) | False negatives — model misses new denial pattern | Monthly retraining + AUC monitoring; alert if AUC drops below 0.70 |
| Auto-fix applies wrong modifier | Claim denied for different reason | Auto-fixes logged and audited; biller always sees what was auto-changed before submission |
| Doctor unavailable for note amendment | Claim stuck, payment delayed | Escalation path: billing manager can override and submit with a note flagging the issue for future audit |
| 835 ERA parsing fails | Denial outcomes not captured | Manual ERA upload fallback; denial reason entered by staff; still feeds feedback loop |
| ICD-10 / CPT codebook not updated | Layer 1 flags valid codes as invalid after annual update | Codebook update job runs Oct 1 (ICD-10) and Jan 1 (CPT) with rollback capability |

---

## 12. Scaling Considerations

### Per-Clinic Model

- Payer mix varies by region and specialty — a cardiology practice has different top payers than a primary care clinic
- ML model is trained per specialty + region (enough claims per segment)
- Payer rules are cached per payer, shared across all clinics — no duplication

### Multi-Clinic Scale

| Component | Scaling approach |
|---|---|
| Claim intake | Queue-based (SQS), fan-out to scrub workers |
| Layer 1–3 (rule checks) | Stateless workers, horizontally scalable; rules in Redis |
| Layer 4 (LLM) | Claude API with per-clinic rate limiting |
| Layer 5 (ML) | LightGBM model served via FastAPI, sub-ms inference |
| Rule database | PostgreSQL with read replicas; rules cached in Redis |
| 835 ERA ingestion | SFTP polling + webhook from clearinghouse |

### Volume Estimate

A 10-provider practice sees ~100 patients/day → ~100 claims/day → ~2,500 scrub jobs/month.
At 100 clinics → 250,000 scrub jobs/month → comfortably within API rate limits with per-clinic throttling.

---

## Appendix A: Denial Category Breakdown

| Denial category | ERA code | Typical cause | Preventable? |
|---|---|---|---|
| Code inconsistent with modifier | CO-4 | Missing/wrong modifier | ✅ Layer 2 (CCI) |
| Duplicate claim | CO-18 | Claim submitted twice | ✅ Layer 1 |
| Non-covered service | CO-96 | Diagnosis not covered for this CPT | ✅ Layer 2 (LCD) |
| Prior auth required | OA-23 | No auth on file | ✅ Layer 3 |
| Bundled service | CO-97 | Component code of another billed code | ✅ Layer 2 (NCCI) |
| Medical necessity | CO-50 | Diagnosis doesn't support procedure | ✅ Layer 2 (LCD) |
| Timely filing exceeded | CO-29 | Claim filed after deadline | ✅ Layer 1 |
| Documentation insufficient | CO-4/CO-B7 | Note doesn't support billing level | ✅ Layer 4 (LLM) |
| Eligibility | CO-27 | Patient not covered on DOS | ✅ Layer 1 (eligibility check) |
| Unknown payer-specific | Various | Payer-specific policy | ⚠️ Layer 5 (ML) |

---

## Appendix B: CCI / NCCI / LCD Explained

### CCI (Correct Coding Initiative)
Published by CMS quarterly. ~250,000 CPT code pairs. Two types:
- **Mutually exclusive**: Can never be billed together (same provider, same DOS)
- **Comprehensive/component**: One code includes the other — unless modifier proves they were separate

### NCCI (National Correct Coding Initiative)
Subset of CCI focused specifically on procedure-to-procedure edits. Enforced by Medicare and most commercial payers that follow CMS guidelines.

### LCD (Local Coverage Determination)
CMS contractors (MACs) publish LCDs for procedures that aren't covered nationally. An LCD specifies which ICD-10 diagnosis codes make a procedure medically necessary in that region.

**Example**: LCD for Continuous Glucose Monitor (CPT A9276):
- Covered for: E11.649 (T2DM with hypoglycemia, without coma), E10.649 (T1DM...)
- NOT covered for: Z13.1 (encounter for screening for diabetes)

### NCD (National Coverage Determination)
CMS-wide policy (applies to all MACs). Takes precedence over LCD when both exist.

---

## Appendix C: Eraser.io Diagram Code

> **How to use**: In your Eraser.io document, click `+` → **Diagram** → switch to **Code** tab → select all existing text → paste the block below exactly as-is.

```
title Claim Scrubbing System

direction right

// ── INPUT (Blue) ──────────────────────────────────────────────────
EHR System [color: blue, icon: database]
Claim Builder [color: blue, icon: file-plus]
Raw Claim [color: blue, icon: file-text]

// ── INTAKE (Blue) ─────────────────────────────────────────────────
Claim Intake and Parser [color: blue, icon: check-circle]

// ── SCRUBBING ENGINE (5 parallel layers) ──────────────────────────
Scrubbing Engine [color: purple, icon: activity] {
  Layer 1 Structural Validation [color: purple, icon: shield]
  Layer 2 Clinical Rules [color: purple, icon: book-open]
  Layer 3 Payer Rules [color: purple, icon: users]
  Layer 4 Documentation Audit [color: purple, icon: file-text]
  Layer 5 ML Denial Predictor [color: purple, icon: trending-up]
}

// ── PAYER RULES ENGINE (Orange) ───────────────────────────────────
Payer Rules Engine [color: orange, icon: sliders] {
  Clearinghouse Feed [color: orange, icon: rss]
  Payer Portal Scraper [color: orange, icon: globe]
  Rule Store [color: orange, icon: database]
  Rule Cache [color: orange, icon: zap]
}

// ── ISSUE AGGREGATOR (Yellow) ─────────────────────────────────────
Issue Aggregator and Prioritizer [color: yellow, icon: alert-circle]

// ── REMEDIATION (Yellow) ──────────────────────────────────────────
Remediation Interface [color: yellow, icon: user-check]
Auto Fix Engine [color: yellow, icon: tool]
Manual Review Queue [color: yellow, icon: inbox]
Note Amendment Request [color: yellow, icon: edit-3]

// ── SUBMISSION GATE (Yellow) ──────────────────────────────────────
All Blocking Issues Resolved [shape: oval, color: yellow, icon: check-circle]

// ── CLEAN CLAIM OUTPUT (Cyan) ─────────────────────────────────────
Clean Claim [color: cyan, icon: file-check]
Clearinghouse [color: cyan, icon: server]
Payer [color: cyan, icon: credit-card]

// ── PAYER RESPONSE (Cyan) ─────────────────────────────────────────
ERA 835 Response [color: cyan, icon: inbox]
Paid [color: cyan, icon: check-circle]
Denied [color: cyan, icon: x-circle]

// ── FEEDBACK LOOP (Green) ─────────────────────────────────────────
Denial Analysis Pipeline [color: green, icon: refresh-cw]
Candidate Rule Generator [color: green, icon: plus-circle]
Human Rule Reviewer [color: green, icon: user]
ML Model Retraining [color: green, icon: cpu]

// ── ERROR PATHS (Red) ─────────────────────────────────────────────
Clearinghouse Down [color: red, icon: alert-triangle]
Local Claim Queue [color: red, icon: hard-drive]

// ══ RELATIONSHIPS ══════════════════════════════════════════════════

// INPUT → INTAKE
EHR System > Claim Builder
Claim Builder > Raw Claim
Raw Claim > Claim Intake and Parser

// INTAKE → SCRUBBING ENGINE (all 5 layers in parallel)
Claim Intake and Parser > Layer 1 Structural Validation
Claim Intake and Parser > Layer 2 Clinical Rules
Claim Intake and Parser > Layer 3 Payer Rules
Claim Intake and Parser > Layer 4 Documentation Audit
Claim Intake and Parser > Layer 5 ML Denial Predictor

// PAYER RULES ENGINE feeds Layer 2 and Layer 3
Clearinghouse Feed > Rule Store
Payer Portal Scraper > Rule Store
Rule Store > Rule Cache
Rule Cache > Layer 2 Clinical Rules
Rule Cache > Layer 3 Payer Rules

// ALL LAYERS → AGGREGATOR
Layer 1 Structural Validation > Issue Aggregator and Prioritizer
Layer 2 Clinical Rules > Issue Aggregator and Prioritizer
Layer 3 Payer Rules > Issue Aggregator and Prioritizer
Layer 4 Documentation Audit > Issue Aggregator and Prioritizer
Layer 5 ML Denial Predictor > Issue Aggregator and Prioritizer

// AGGREGATOR → REMEDIATION
Issue Aggregator and Prioritizer > Remediation Interface

// REMEDIATION BRANCHES
Remediation Interface > Auto Fix Engine: auto-fixable issues
Remediation Interface > Manual Review Queue: judgment calls
Remediation Interface > Note Amendment Request: documentation gaps

// NOTE AMENDMENT LOOP — doctor updates note, scrubber reruns Layer 4
Note Amendment Request > Layer 4 Documentation Audit: doctor amends note [color: yellow, style: dashed]

// ALL FIXES → SUBMISSION GATE
// Note Amendment does NOT go directly here — it must re-run Layer 4 first
// Layer 4 → Issue Aggregator → Remediation → here (path already exists above)
Auto Fix Engine > All Blocking Issues Resolved
Manual Review Queue > All Blocking Issues Resolved: biller resolves

// BLOCKING ISSUE LOOP
Remediation Interface > Remediation Interface: unresolved blocking issues [color: red, style: dashed]

// SUBMISSION
All Blocking Issues Resolved > Clean Claim
Clean Claim > Clearinghouse
Clearinghouse > Payer

// CLEARINGHOUSE FAILURE
Clearinghouse > Clearinghouse Down: connection failed [color: red, style: dashed]
Clearinghouse Down > Local Claim Queue: retry on restore [color: red, style: dashed]

// PAYER RESPONSE
Payer > ERA 835 Response
ERA 835 Response > Paid
ERA 835 Response > Denied

// FEEDBACK LOOP — denial outcomes improve rules and ML model
Denied > Denial Analysis Pipeline
Denial Analysis Pipeline > Candidate Rule Generator: new pattern detected
Denial Analysis Pipeline > ML Model Retraining: all denials feed model
Candidate Rule Generator > Human Rule Reviewer
Human Rule Reviewer > Rule Store: approved new rule
ML Model Retraining > Layer 5 ML Denial Predictor: updated model

legend {
  [connection: ">", color: black, label: "Primary process flow"]
  [connection: ">", color: red, label: "Error or failure path"]
  [connection: ">", color: cyan, label: "Submission and payer flow"]
  [connection: ">", color: green, label: "Feedback and learning loop"]
  [connection: ">", color: yellow, label: "Review and remediation flow"]
  [connection: ">", color: orange, label: "Rules engine data flow"]
  [connection: ">", color: purple, label: "Scrubbing engine flow"]
  [connection: ">", style: dashed, color: yellow, label: "Loop or retry path"]
  [connection: ">", style: dashed, color: red, label: "Error fallback"]
  [shape: oval, label: "Gate / decision point"]
  [shape: rectangle, label: "Process or system component"]
}
```