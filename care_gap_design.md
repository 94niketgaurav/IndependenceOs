# Care Gap Detection & Closure — System Design
**Independence OS | Population Health & Quality Measure Engine**
*Author: System Design Document | Date: 2026-03-15*

---

## Table of Contents

1. [Problem Framing](#1-problem-framing)
2. [System Overview](#2-system-overview)
3. [Data Ingestion & Patient Identity](#3-data-ingestion--patient-identity)
4. [Measure Evaluation Engine](#4-measure-evaluation-engine)
5. [Gap Prioritization](#5-gap-prioritization)
6. [Outreach Engine](#6-outreach-engine)
7. [Closure Confirmation](#7-closure-confirmation)
8. [Quality Reporting](#8-quality-reporting)
9. [Platform Integration](#9-platform-integration)
10. [Data Models](#10-data-models)
11. [Edge Cases & Mitigations](#11-edge-cases--mitigations)
12. [Key Design Decisions & Trade-offs](#12-key-design-decisions--trade-offs)
13. [Failure Modes & Mitigations](#13-failure-modes--mitigations)
14. [Scaling Considerations](#14-scaling-considerations)
15. [Appendix A: Key HEDIS Measures Reference](#appendix-a-key-hedis-measures-reference)
16. [Appendix B: Gap State Machine](#appendix-b-gap-state-machine)
17. [Appendix C: Eraser.io Diagram Code](#appendix-c-eraserio-diagram-code)

---

## 1. Problem Framing

### What We're Actually Solving

A **care gap** is the space between what a patient *should* have received and what they *actually* received. A diabetic patient who hasn't had an A1C test in 14 months has a care gap. A 52-year-old woman who has never had a mammogram has a care gap. A child missing two immunizations has a care gap.

These gaps are measured against **HEDIS quality measures** — ~90 standardized rules published annually by NCQA (National Committee for Quality Assurance) that define exactly who is eligible for each measure, what counts as satisfying it, and who is excluded.

| Dimension | Why It's Hard |
|---|---|
| Data fragmentation | Patient had a mammogram at a different clinic — we don't know |
| Measure complexity | HEDIS has eligibility + exclusions + numerator criteria + code sets, updated annually |
| 1,000+ payers | Each payer weights measures differently; some have custom contracts |
| Identification is not closure | Finding a gap is easy; getting the patient to act on it is hard |
| Closure confirmation | We need to know it actually happened, not just that we tried |
| Outreach compliance | TCPA for SMS/calls, patient consent, language barriers |
| Annual resets | Gaps close and re-open on a 12-month cycle |

### Why Care Gaps Matter

**Patient health**: Preventive care catches conditions early. An undiagnosed A1C of 9.2% means uncontrolled diabetes causing silent organ damage.

**Clinic revenue**: Quality measures directly drive revenue via:
- **Value-Based Care (VBC) contracts**: Payers pay bonuses for hitting quality thresholds
- **Star Ratings**: Medicare Advantage Star ratings affect premium rates
- **Shared savings**: ACO contracts distribute shared savings based partly on quality scores

A single percentage point improvement in HEDIS measure rates can be worth $50,000–$200,000 annually to a 10-provider independent practice.

### Target Outcome

- **Gap identification rate**: ≥ 98% of eligible patients identified within 24 hours of data availability
- **Outreach-to-closure conversion**: ≥ 35% of outreached patients close their gap within 90 days
- **False positive rate**: < 2% of "open gaps" that are actually already closed
- **Provider trust**: Zero cases of incorrect outreach to excluded patients

---

## 2. System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  EXTERNAL DATA SOURCES                                              │
│  EHR ─── Payer Claims ─── State IIS ─── HIE ─── Lab Gateway       │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
               ┌───────────────────▼──────────────────────┐
               │   PATIENT IDENTITY RESOLUTION (EMPI)      │
               │   Tiered matching: deterministic →        │
               │   high-confidence → staged pending        │
               └───────────────────┬──────────────────────┘
                                   │
               ┌───────────────────▼──────────────────────┐
               │   LONGITUDINAL PATIENT PROFILE            │
               │   Unified evidence: EHR + claims +        │
               │   IIS + HIE + lab + pharmacy              │
               └───────────────────┬──────────────────────┘
                                   │
               ┌───────────────────▼──────────────────────┐
               │   MEASURE EVALUATION ENGINE               │
               │   Denominator → Exclusion → Numerator    │
               │   Versioned HEDIS specs + payer overlays │
               └───────────────────┬──────────────────────┘
                                   │
               ┌───────────────────▼──────────────────────┐
               │   GAP RECORD STORE                        │
               │   OPEN / IN-PROGRESS / CLOSED / EXCLUDED  │
               └──────┬─────────────────────┬─────────────┘
                      │                     │
          ┌───────────▼──────┐  ┌───────────▼──────────────┐
          │  PRIORITY SCORER  │  │  PROVIDER GAP SUMMARY    │
          │  Clinical urgency │  │  → Medical Scribe UI     │
          │  + VBC financial  │  │    at encounter start    │
          └───────────┬───────┘  └──────────────────────────┘
                      │
          ┌───────────▼───────────────────────────────────┐
          │        OUTREACH ENGINE                         │
          │  Campaign Orchestrator → Channel Dispatcher   │
          │  SMS / Phone / Portal / Letter                 │
          │  Response Handler → Appointment Scheduler     │
          └───────────┬───────────────────────────────────┘
                      │
          ┌───────────▼───────────────────────────────────┐
          │        CLOSURE CONFIRMATION                    │
          │  Lab result / Note signed / Claim paid /       │
          │  Provider attestation / Manual exclusion       │
          └───────────┬───────────────────────────────────┘
                      │
          ┌───────────▼───────────────────────────────────┐
          │        QUALITY REPORTING                       │
          │  HEDIS submission / Payer contracts /          │
          │  Population dashboard / VBC financial          │
          └───────────────────────────────────────────────┘

PLATFORM INTEGRATION:
  ←── NoteSignedEvent    (from Medical Scribe)
  ←── ClaimPaidEvent     (from Claim Scrubbing)
  ──► GapSummaryAPI      (to Medical Scribe provider UI)
  ──► EvidenceAnnotation (to Claim Scrubbing Layer 4)
```

---

## 3. Data Ingestion & Patient Identity

### 3.1 Data Sources

The core problem: no single system has a complete view of a patient's preventive care history. A patient may have received services at pharmacies, specialist offices, urgent care centers, and hospitals — all of which may bill claims to the same payer but have no direct data connection to the primary care practice.

| Source | What it provides | Mechanism | Latency |
|---|---|---|---|
| **EHR** | Encounters, diagnoses, procedures, orders, results, meds | FHIR R4 bidirectional sync | Near-real-time |
| **Payer Claims** | Services billed to insurance (CPT + ICD-10) | 837/835 EDI + payer API | 14-90 day lag |
| **State IIS** | Immunization records from any provider | SOAP/REST + batch export | Hours–weekly |
| **HIE** | Cross-facility clinical data (labs, encounters, procedures) | FHIR, IHE XDS, HL7 ADT/ORU | Variable |
| **Retail Pharmacy** | Immunizations administered at pharmacy | Claims feed or direct API | 24-48 hours |
| **Lab Gateway** | Direct lab interfaces (Quest, LabCorp, in-house) | HL7 ORU v2 messages | Hours |

### 3.2 Patient Identity Resolution (EMPI)

This is the most critical infrastructure component. Every piece of external clinical evidence must be attached to the correct patient. A false match — attaching another patient's A1C result to the wrong record — closes a gap that should remain open, and could constitute a HIPAA breach.

**Tiered Trust Model**:

```
Level 1 — DETERMINISTIC (auto-link, no review needed):
  EHR MRN + payer member ID cross-reference established by practice
  at onboarding. These are explicitly mapped.

Level 2 — HIGH CONFIDENCE (auto-link, logged):
  DOB exact + sex exact + last name exact match (Jaro-Winkler ≥ 0.95)
  Automatically linked. Logged for audit.

Level 3 — PROBABLE (staged, human review required):
  Fuzzy demographic match: 3 of 5 fields (DOB, sex, first name, last name,
  zip code) match above threshold.
  → Created as CANDIDATE_LINK in staging queue.
  → Evidence held in staging — NOT applied to gap profile until human
    reviews and approves the link.

Level 4 — UNLINKED:
  No match found. Record staged indefinitely. Searchable by clinicians.
```

**Why Level 3 requires human review**: A false positive at Level 3 applies another patient's clinical data to this patient. If that data closes a gap, the patient receives no follow-up for a screening they never actually had. This is the "silent false closure" problem — the most dangerous failure mode in the entire system.

**FHIR standard**: Implemented as `Patient.$match` operation, allowing any future system to use the same identity layer via a standard API contract.

### 3.3 Recalculation Trigger Model

Rather than running a full gap recalculation on all patients every night (too slow at scale) or in real-time on every event (too expensive), we use a **dirty-flag priority queue**:

```
Data event arrives (lab result, HIE update, claim paid, note signed)
    ↓
Set needs_recalculation = true on patient gap profile
Tag event with priority:
  Tier 1 (HIGH): likely closure event (lab result, completed procedure)
                 → process within 15 minutes
  Tier 2 (STD):  new data arrived (claims update, IIS refresh)
                 → process within 24 hours

Nightly batch:   patients with no recent event → rolling 30-day refresh
January 1:       full population recalculation for annual measure reset
Spec update:     targeted recalculation for patients in affected measures
```

---

## 4. Measure Evaluation Engine

### 4.1 HEDIS Measure Specification Store (Versioned)

HEDIS specifications update annually (October release for next measurement year). Between releases, NCQA publishes errata. Payer contracts overlay custom modifications.

Each measure is stored as a versioned specification with `effective_date`:

```python
MeasureSpec(
    measure_id    = "BCS-E",
    measure_name  = "Breast Cancer Screening",
    hedis_year    = 2026,
    effective_date = date(2026, 1, 1),
    denominator   = {
        "age_min": 50, "age_max": 74, "sex": "F",
        "enrollment_months_required": 11,
        "continuous_enrollment": True
    },
    exclusions    = [
        {"type": "diagnosis",  "code_set": "bilateral_mastectomy_codes"},
        {"type": "diagnosis",  "code_set": "history_mastectomy_codes"},
        {"type": "enrollment", "condition": "hospice_any_time_in_year"}
    ],
    numerator     = {
        "lookback_years": 2,
        "service_code_sets": ["mammography_cpt_codes", "mammography_hcpcs_codes"],
        "data_sources": ["claims", "ehr_procedure", "hie_procedure"],
        "exclude_diagnostic": True,     # diagnostic mammo ≠ screening mammo
        "result_required": False        # presence of service = closure
    },
    not_due_until_months = 18,          # don't outreach in first 6 months
                                        # if closed in back half of prior year
    telehealth_eligible = False,
    payer_overrides = []
)
```

**Payer contract overlays** inherit from the base spec and override specific fields:

```python
PayerMeasureOverlay(
    payer_id       = "AETNA_MEDICARE_ADV",
    base_measure_id = "BCS-E",
    hedis_year     = 2026,
    overrides      = {
        "numerator.lookback_years": 1,  # Aetna requires annual (not biennial)
        "not_due_until_months": 11
    }
)
```

### 4.2 Three-Phase Evaluation

For each patient × measure combination, evaluation runs in three sequential phases:

**Phase 1 — Denominator Evaluation**:
Is this patient eligible for this measure?

```
Is patient enrolled for required months? (continuous enrollment check)
Is patient within age/sex criteria?
Does patient have qualifying diagnoses (for condition-based measures)?
Is patient attributed to this practice?
    → YES to all: patient is in denominator
    → NO to any: patient is NOT in this measure (skip phases 2-3)
```

**Phase 2 — Exclusion Evaluation**:
Is this patient excluded from this measure?

Exclusions are evaluated INDEPENDENTLY per measure (not as a global patient flag).
A patient in hospice is excluded from ALL measures. A patient with bilateral
mastectomy is excluded from breast cancer screening only, not colorectal screening.

```
Does patient have an exclusion diagnosis code? (from code set registry)
Does patient have an exclusion procedure code?
Does patient have a documented medical contraindication?
Does patient have a manually entered exclusion (provider attested)?
    → YES to any: patient is EXCLUDED from this measure
    → EXCLUDED gaps do NOT trigger outreach
```

**Phase 3 — Numerator Evaluation**:
Has this patient met the measure criteria?

```
For each data source (EHR, claims, HIE, IIS):
    Has a qualifying service code been received?
    Is the service within the lookback window?
    Was it a qualifying service type (not diagnostic if measure requires screening)?
    Is a result required, and if so, was it received and was it valid?
        → YES to all: gap is CLOSED (or never opened)
        → NO: gap is OPEN
```

### 4.3 Gap State Machine

```
OPEN ──────────────────────────────────────────────► CLOSED
  │                                                    ▲
  │ outreach sent                                      │
  ▼                                                    │
IN_PROGRESS ──► appointment_made ──► service_completed─┘
  │
  │ patient declined    │ result pending timeout
  ▼                     ▼
PATIENT_DECLINED       OPEN (re-opens)
  │
  ▼
EXCLUDED (if clinical contraindication confirmed)
```

Terminal states: `CLOSED`, `EXCLUDED`
Re-openable states: `CLOSED` (on annual measure reset for recurring measures)

---

## 5. Gap Prioritization

Not all open gaps deserve equal urgency. A diabetic patient whose A1C has been untested for 18 months is more clinically urgent than a 51-year-old due for their first mammogram. A patient on a VBC contract where an A1C result contributes to a shared savings bonus is more financially urgent than a fee-for-service patient.

### 5.1 Priority Score Components

```
Priority Score = (
    clinical_urgency_score      × 0.40
  + vbc_financial_impact_score  × 0.35
  + outreach_receptiveness_score × 0.15
  + practice_contract_weight    × 0.10
)
```

**Clinical Urgency Score** (0–100):
- Measure category weight: diabetes care > preventive screening > immunization
- Time overdue: days past due × measure urgency multiplier
- Clinical context: patient has related active diagnoses

**VBC Financial Impact Score** (0–100):
- Patient is on a capitated/shared-savings contract: high weight
- Practice is near a quality tier threshold: higher urgency to close gaps in the measure determining tier
- Patient is on fee-for-service only: lower financial weight

**Outreach Receptiveness Score** (0–100):
- Has valid contact info on file
- Has responded to prior outreach
- Has no prior "patient declined" on this measure type
- Preferred language available in template library

**Practice Contract Weight** (0–100):
- Is this measure in an active payer contract?
- Is this measure weighted highly in the contract?
- Is the practice's current performance near a bonus threshold for this measure?

### 5.2 Provider-Facing Gap Summary

At encounter start (when Medical Scribe creates an encounter), the Priority Queue generates a real-time gap summary pushed to the provider's UI:

```
OPEN CARE GAPS — JOHN SMITH (DOB: 1965-03-12)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔴 OVERDUE 14 months:  HbA1c Test (last: Jan 2025)
                       [Order A1C now →]
🟡 OVERDUE 2 years:    Colorectal Screening (never done)
                       [Order FIT test →] [Refer colonoscopy →]
🟢 DUE THIS YEAR:      Diabetic Eye Exam (last: Mar 2025)
                       Ophthalmology referral in place ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Address all gaps in this visit →]
```

This is the highest-ROI intervention in the system. In-visit gap closure rates are 3–5× higher than outreach-driven closure. The action buttons ("Order A1C now") create the order directly in the EHR, which the Medical Scribe system captures in the transcript and includes in the Plan section.

---

## 6. Outreach Engine

### 6.1 Campaign State Machine

Each patient-gap pair gets an outreach campaign. The campaign has its own state machine:

```
PENDING
    ↓ (priority scorer assigns)
OUTREACH_SCHEDULED
    ↓ (send time reached)
CONTACTED
    ↓ (response received)
    ├─► APPOINTMENT_MADE ──► SERVICE_COMPLETED ──► [triggers gap closure]
    ├─► PATIENT_DECLINED ──► [exclusion workflow]
    └─► NO_RESPONSE ──────── [next channel attempt, up to N times]
                              │
                              ↓ (N attempts exhausted)
                         OUTREACH_EXHAUSTED ──► [manual review queue]
```

### 6.2 Channel Selection Logic

Channel selection follows a **preference-first, hard-constraint model**:

```
1. Check patient opt-outs (TCPA): if SMS opted-out → skip SMS entirely
2. Check patient preferences: if portal preferred → start with portal
3. Check contact data quality: if phone number confidence < 0.7 → skip phone
4. Check language: if preferred_language ≠ "en" → verify template exists in that language
5. Apply clinical urgency: if urgency HIGH and portal not opened in 30 days → escalate to phone
6. Check concurrent outreach lock: if payer program also outreaching → apply hold period
```

**Default campaign sequence** (no strong preference on file):
```
Day 1:  SMS (if consent) or Portal message
Day 7:  Portal message (or SMS if portal not opened)
Day 21: Automated voice call (if phone on file)
Day 42: Mailed letter
Day 90: Manual review queue (outreach exhausted)
```

**LLM-generated personalized messages**: Claude generates the outreach message body using the patient's name, the specific measure, the clinical reason it matters, and a clear call to action. Messages are literacy-adapted (target 6th-grade reading level for general outreach) and translated to the patient's preferred language.

Example generated SMS:
> "Hi John — Dr. Martinez's office here. We noticed you're due for a diabetes blood test (A1C). This test shows how your blood sugar has been over 3 months. You can get it done at any Quest or LabCorp lab — no appointment needed. Want us to send the order? Reply YES or call us at (312) 555-0100. Reply STOP to opt out."

### 6.3 Response Handling

**SMS responses parsed automatically**:
- "YES", "yes", "y", "ok", "sure" → trigger appointment scheduling or lab order
- "STOP", "stop", "unsubscribe" → immediately opt out, apply to all active campaigns
- "NO", "no", "not interested" → patient declined, enter exclusion workflow
- Anything else → route to front desk for human follow-up

**Appointment No-Show Handling**:
A no-show does NOT close the campaign. The campaign re-activates after 7 days with a gentler follow-up message. After 2 no-shows on the same gap, the campaign escalates to a personal phone call from a care coordinator.

---

## 7. Closure Confirmation

### 7.1 Evidence Sources (in priority order)

| Source | Reliability | Latency | Notes |
|---|---|---|---|
| EHR result + NoteSignedEvent | Highest | Minutes | Doctor performed and documented the service |
| Direct lab HL7 result | High | Hours | Lab result received directly |
| HIE clinical data | High | Hours–days | Service at another facility confirmed |
| Payer claim paid (ClaimPaidEvent) | High | 14-90 days | Insurance accepted the service was performed |
| State IIS immunization record | High | Hours–weekly | Vaccination confirmed |
| Provider attestation | High | Manual | Service happened outside system's visibility |
| Patient self-report | Low | Immediate | Unverified; triggers hold period |

### 7.2 Pending Result Tracker

When a service is ordered but the result is not yet received:

```
Service ordered (EHR order event received)
    ↓
Gap status → IN_PROGRESS (suppress outreach)
    ↓
Wait for result (with configurable TAT per test type):
    A1C:         48 hours
    Colonoscopy: 14 days (path report)
    Mammogram:   5 business days
    ↓
Result received → Numerator Evaluator checks it → Gap CLOSED or still OPEN
    ↓ (if no result by TAT + buffer)
Pending result timeout
    → Gap status → OPEN (re-opens)
    → Provider notified: "Expected result not received for [test]"
    → Outreach NOT immediately re-triggered (provider review first)
```

### 7.3 Provider Attestation

When a service was performed outside the system's visibility (community clinic, paper records, foreign country):

```
Provider or staff opens attestation form:
    - Select patient + measure
    - Enter date of service
    - Select evidence type: verbal report / scanned document / external note
    - Upload supporting document (optional)
    - Enter clinical note (required)
    ↓
Attestation stored in GapEvidenceStore (append-only)
    ↓
Numerator Evaluator applies the attested service date
    ↓
Gap status → CLOSED (attestation satisfies HEDIS hybrid method)
    ↓
Audit event logged with attestor ID, timestamp, evidence reference
```

### 7.4 Annual Measure Reset

On January 1 of each measurement year:
- All annual measures recalculate for all patients
- A patient who completed their A1C in December gets:
  - `not_due_until` = September (9 months into the new year)
  - No outreach generated until September
- A patient who completed their A1C in January of last year gets immediate outreach

This prevents the "January blizzard" of re-outreach to patients who just did their screening last month.

---

## 8. Quality Reporting

### 8.1 HEDIS Administrative Submission

At year-end, the system generates the NCQA-required HEDIS reporting files:
- **Administrative method**: Uses claims and EHR data only
- **Hybrid method**: Supplements administrative data with medical record attestations
- Output: NCQA-format XML/CSV submission per payer

### 8.2 Payer Quality Reports

Each payer has its own quality reporting format and schedule (monthly, quarterly, annual).

Medicare Advantage: Star Ratings (annual, CMS-specified format)
Commercial payers: HEDIS MY submission to NCQA
VBC contracts: Custom P4P (Pay for Performance) scorecards per contract

### 8.3 VBC Financial Impact Report

Translates gap closure performance into dollar projections:

```
Current A1C testing rate: 78% of eligible patients
Bonus threshold:          80% for Tier 2, 90% for Tier 1
Patients needed to close to hit Tier 2:  12 patients
Estimated bonus at Tier 2:              $47,500
Revenue per gap closure:                ~$3,958

PRIORITY TARGETS (top 12 patients closest to VBC contract):
  1. John Smith    — A1C overdue 14 months  | Contract: BCBS Medicare Adv
  2. Maria Garcia  — A1C overdue 8 months   | Contract: Aetna VBC
  ...
```

---

## 9. Platform Integration

### 9.1 Connection to Medical Scribe

**Before the visit** (Medical Scribe → pulls from Care Gap):
When an encounter is created in the Medical Scribe system, it calls the
`GapSummaryAPI` to retrieve the patient's open gaps. These are shown as a
pre-encounter banner so the provider can address them during the visit.

**During the visit** (Medical Scribe → pushes to Care Gap):
When the Medical Scribe transcription detects a procedure or order ("I'm
going to order an A1C today"), it fires a provisional gap event. The Care Gap
system moves the gap to `IN_PROGRESS` and suppresses outreach.

**After the visit** (Medical Scribe → Care Gap via event):
`NoteSignedEvent` carries the signed CPT codes and ICD-10 codes. Care Gap
uses this as primary closure evidence — faster and more reliable than waiting
for claims data (which can take 14-90 days).

**Documentation quality feedback** (Care Gap → Medical Scribe):
For measures requiring specific documentation (e.g., PHQ-9 score for depression
screening), Care Gap informs the Medical Scribe system what to include in the
note. This appears as a flagged item in the doctor review interface:
"To close DEP measure for this patient, document PHQ-9 score in Assessment."

### 9.2 Connection to Claim Scrubbing

**Claims as closure evidence** (Claim Scrubbing → Care Gap):
`ClaimPaidEvent` is the authoritative insurance confirmation that a service
was performed and accepted. Care Gap subscribes to this event for secondary
closure confirmation, especially for services at other facilities.

**Denial does not reopen gaps** (important rule):
A `ClaimDeniedEvent` means a billing/administrative error — not that the
service didn't happen. If a note has already confirmed the service, a claim
denial does NOT reopen the gap. HEDIS hybrid method allows medical record
evidence to stand independent of claim status.

**VBC financial context** (Care Gap → Claim Scrubbing):
Care Gap provides payer contract context to Claim Scrubbing's Layer 4
documentation audit. When a gap-closure service is being scrubbed: "This
A1C test satisfies the HEDIS CDC measure — document medical necessity in
terms of HbA1c monitoring for uncontrolled diabetes." This annotation
improves the LLM's documentation audit accuracy.

### 9.3 Event Bus Contracts

```python
# Events PUBLISHED by Medical Scribe (Care Gap subscribes):
NoteSignedEvent(
    encounter_id: UUID,
    patient_id: UUID,
    provider_id: UUID,
    date_of_service: date,
    icd10_codes: list[str],
    cpt_codes: list[str],
    signed_at: datetime
)

# Events PUBLISHED by Claim Scrubbing (Care Gap subscribes):
ClaimPaidEvent(
    claim_id: UUID,
    patient_id: UUID,
    payer_id: str,
    date_of_service: date,
    icd10_codes: list[str],
    cpt_codes: list[str],
    paid_at: datetime
)

# Events PUBLISHED by Care Gap (other systems can subscribe):
GapClosedEvent(
    gap_id: UUID,
    patient_id: UUID,
    measure_id: str,
    closed_at: datetime,
    closure_source: str  # "NOTE_SIGNED", "CLAIM_PAID", "HIE", "ATTESTATION"
)

GapOpenedEvent(
    gap_id: UUID,
    patient_id: UUID,
    measure_id: str,
    opened_at: datetime,
    priority_score: float
)
```

---

## 10. Data Models

### 10.1 Patient Gap Profile

```python
class PatientGapProfile:
    profile_id: UUID
    patient_id: UUID
    clinic_id: UUID
    gaps: list[CareGap]
    last_recalculated_at: datetime
    needs_recalculation: bool
    recalculation_priority: int    # 1 = high, 2 = standard
    linked_identity_sources: list[PatientIdentityLink]
```

### 10.2 Care Gap

```python
class CareGap:
    gap_id: UUID
    patient_id: UUID
    clinic_id: UUID
    measure_id: str                # "BCS-E", "CDC-HbA1c", etc.
    hedis_year: int
    payer_id: str | None           # None = HEDIS standard, str = payer-specific
    status: GapStatus              # OPEN, IN_PROGRESS, CLOSED, EXCLUDED, PATIENT_DECLINED
    opened_at: datetime
    closed_at: datetime | None
    not_due_until: date | None     # suppresses outreach until this date
    priority_score: float
    evidence: list[GapEvidence]    # append-only
    outreach_campaign: OutreachCampaign | None
    exclusion_reason: str | None
```

### 10.3 Gap Evidence

```python
class GapEvidence:
    evidence_id: UUID
    gap_id: UUID
    evidence_type: EvidenceType    # LAB_RESULT, NOTE_SIGNED, CLAIM_PAID,
                                   # HIE_PROCEDURE, IIS_IMMUNIZATION,
                                   # PROVIDER_ATTESTATION, PATIENT_SELF_REPORT
    source_id: str                 # encounter_id, claim_id, hl7_message_id, etc.
    service_date: date
    cpt_code: str | None
    loinc_code: str | None
    result_value: str | None       # for result-required measures
    confidence: float              # 1.0 for deterministic, lower for probabilistic
    received_at: datetime
    applied_to_numerator: bool
```

### 10.4 Outreach Campaign

```python
class OutreachCampaign:
    campaign_id: UUID
    gap_id: UUID
    patient_id: UUID
    status: CampaignStatus         # PENDING, SCHEDULED, CONTACTED, APPOINTMENT_MADE,
                                   # SERVICE_COMPLETED, EXHAUSTED, DECLINED
    channel_sequence: list[OutreachStep]
    current_step: int
    language: str                  # "en", "es", "vi", etc.
    created_at: datetime
    last_contact_at: datetime | None
    next_contact_at: datetime | None
    appointment_id: UUID | None

class OutreachStep:
    step_id: UUID
    campaign_id: UUID
    step_number: int
    channel: Channel               # SMS, PHONE, PORTAL, LETTER
    scheduled_at: datetime
    sent_at: datetime | None
    response: str | None
    outcome: StepOutcome | None    # RESPONDED, NO_RESPONSE, DECLINED, OPTED_OUT
```

### 10.5 Patient Identity Link

```python
class PatientIdentityLink:
    link_id: UUID
    patient_id: UUID               # internal ID
    source_system: str             # "EHR", "BCBS_IL", "IL_IIS", "COMMONWELL_HIE"
    source_patient_id: str         # ID in that system
    trust_level: int               # 1=deterministic, 2=high-conf, 3=probable
    created_at: datetime
    approved_by: UUID | None       # required for trust_level=3
    revocable: bool                # False for level 1-2, True for level 3
```

### 10.6 Measure Spec (simplified)

```python
class MeasureSpec:
    spec_id: UUID
    measure_id: str
    measure_name: str
    hedis_year: int
    effective_date: date
    denominator: dict              # age range, sex, enrollment criteria
    exclusions: list[dict]         # code sets, enrollment conditions
    numerator: dict                # code sets, lookback window, result requirements
    not_due_until_months: int      # months before re-outreach after recent closure
    telehealth_eligible: bool
    payer_overrides: list[PayerMeasureOverlay]
```

---

## 11. Edge Cases & Mitigations

| # | Edge Case | Impact | Mitigation |
|---|---|---|---|
| EC-1 | Conflicting results from two sources for same test | Wrong gap status | Store all results; flag discrepancy to provider; use most recent draw date, not receipt date |
| EC-2 | Duplicate patient identity across sources | Gap evidence misattributed | EMPI tiered trust model; Level 3 requires human review before evidence applied |
| EC-3 | Lab result arrives before order in data pipeline | Result unattached, gap stays open | Staging queue holds unmatched results; attempts match for 48 hours; then processed as unsolicited with provider flag |
| EC-4 | Diagnostic mammogram billed under screening CPT | Gap falsely closed | Measure spec carries `exclude_diagnostic: true`; context-aware numerator evaluator checks claim context flags |
| EC-5 | State IIS batch latency — patient already vaccinated | Unnecessary outreach sent | Patient self-report triggers hold period; IIS batch confirms or contradicts within weekly cycle |
| EC-6 | EHR migration imports historical data with wrong timestamps | Mass false-positive gap identification | `encounter_date` validation: flag records where `encounter_date ≈ created_at` within 24 hours for batch import review |
| EC-7 | Patient turns 50 mid-measurement year | Patient missed for first half of year | Denominator evaluator re-runs on birthday. Measurement year evaluation considers the full eligible period, not just year-start eligibility |
| EC-8 | Exclusion documented in external record not in EHR | Incorrect outreach to excluded patient | Manual exclusion registry; provider can enter exclusions with document upload; HIE-sourced exclusions accepted |
| EC-9 | Telehealth visit counts for some measures, not others | Wrong gap status | Per-measure `telehealth_eligible` flag in spec. Never apply blanket telehealth rule |
| EC-10 | Colorectal screening requires result, not just order | Gap closed prematurely | Pending result tracker; gap stays IN_PROGRESS until result confirmed received and valid |
| EC-11 | Hospice exclusion applied globally across all measures | May miss measure-specific exclusion logic | Exclusions evaluated independently per measure, not as patient-level flag |
| EC-12 | HEDIS hybrid method — service not in claims but in records | Gap stays open despite real closure | Provider attestation workflow; attested evidence satisfies hybrid method; audit trail required |
| EC-13 | VBC contract measure differs from standard HEDIS | Wrong gap status per contract | Gap records maintained per measure-set (standard HEDIS vs. each payer contract) |
| EC-14 | Clinically contraindicated measure | Incorrect standard outreach | Flagged as "requires provider review before outreach"; not auto-excluded without clinical confirmation |
| EC-15 | Concurrent outreach from payer, ACO, employer simultaneously | Patient frustration, trust erosion | Outreach coordination layer; outreach lock check; configurable hold period |
| EC-16 | Patient texts STOP mid-campaign | TCPA violation if outreach continues | STOP processed immediately; all active campaigns for that patient on SMS channel halted; already-queued letters can still be sent (different channel) |
| EC-17 | Contact info conflicts across sources | Outreach reaches wrong person | Contact confidence scoring; fallback sequence: verified > EHR primary > EHR secondary > payer-sourced; contradictory data → human review flag |
| EC-18 | Appointment made but patient no-shows | Campaign falsely marked success | Appointment completion required for closure credit. No-show → campaign reactivates after 7 days. 2nd no-show → escalate to personal call |
| EC-19 | Patient's preferred language not in template library | Message ignored or not understood | Pre-flight check: if `preferred_language` not in template library → default to phone call in patient's language via interpreter; log as "language gap" for template roadmap |
| EC-20 | Lab result never arrives (lost specimen) | Gap stays incorrectly IN_PROGRESS indefinitely | Pending result timeout per test type. On timeout: gap → OPEN; provider notified; new outreach NOT auto-triggered |
| EC-21 | Annual measure reset triggers early re-outreach | Patient who closed gap 6 weeks ago receives outreach | `not_due_until` date calculated based on last closure date; outreach suppressed until date passes |
| EC-22 | Service performed at non-integrated facility | Gap stays open indefinitely | Provider attestation workflow satisfies HEDIS hybrid method; audit trail required |

---

## 12. Key Design Decisions & Trade-offs

### Decision 1: EMPI Tiered Trust Model

**Why not a single probabilistic threshold?**
A single threshold (e.g., "match score ≥ 0.85 = auto-link") either produces false positives (misattributing evidence) or false negatives (missing real matches). The tiered model separates the decision: deterministic matches auto-link always; probabilistic matches require human review. The cost of false positives (incorrect gap closure) exceeds the cost of false negatives (delayed closure), so the system errs toward requiring review.

### Decision 2: Versioned Measure Specifications as Data

HEDIS measures update annually. If measure logic is hardcoded, every annual update requires a code deployment. With versioned specs, an annual HEDIS update is a database insert of new `MeasureSpec` rows with `effective_date = next year`. Historical gap evaluations use the spec that was current at the measurement period. Payer overlays are child rows that inherit and override. The annual update pipeline: ingest NCQA release → generate spec diff → human review → activate on `effective_date`.

### Decision 3: Pre-Encounter Provider Prompt (the Highest-ROI Feature)

The provider-facing gap summary at encounter start is architecturally the most important integration in the platform. In-visit gap closure rates are 3–5× higher than outreach-driven closure because: the provider can order the service immediately, the patient is present and committed, and the service is performed in the same encounter. Every design decision about data freshness and recalculation latency must be judged against the question: "Can this gap summary be shown accurately at encounter start?"

### Decision 4: Auto-Outreach vs. Provider-Approved Outreach

**Option A — Auto-outreach**: The system sends messages without any provider review.
**Option B — Provider must approve each outreach batch**.

**Decision: Tiered automation**.
- HIGH confidence gaps (patient in denominator, no exclusions, clearly overdue) → auto-outreach allowed after 24-hour hold for provider veto.
- UNCERTAIN gaps (new patient, limited data, complex exclusion logic) → goes to provider review queue before outreach.
- Contraindicated measures (EC-14) → always requires provider review before outreach.

The 24-hour hold gives the provider a window to veto incorrect outreach without requiring active approval of every message.

### Decision 5: Event Bus vs. Direct API Integration with Medical Scribe and Claim Scrubbing

**Decision: Start with direct API calls (Phase 1), migrate to event bus (Phase 2)**.
While the three systems are being co-developed, direct API calls have lower overhead and clearer debugging. Once the event schemas are stable (after 6+ months in production), migrate to an event bus for loose coupling and replay capability. The `NoteSignedEvent` and `ClaimPaidEvent` schemas defined in Section 9.3 are designed to be the stable Phase 2 contracts — start building toward them even in Phase 1.

---

## 13. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| EMPI false positive match | Wrong patient's evidence closes a gap | Tiered trust model; Level 3 requires human review; all links logged for audit |
| State IIS batch job fails | Immunization gaps stay open | Retry with exponential backoff; alert after 3 failures; patient self-report accepted as provisional |
| HIE connection down | External encounter data not available | Queue events for retry; gaps stay open; no false closures |
| LLM outreach message generation fails | No personalized message generated | Fall back to pre-written template; log failure for monitoring |
| Annual HEDIS spec update not applied | Wrong measure logic used all year | Spec update pipeline has human review gate; versioned specs allow rollback |
| Patient sent outreach for excluded condition | Patient distress; trust erosion | Exclusion check runs as Phase 2 of measure evaluation; outreach is only generated for OPEN gaps after exclusion check passes |
| Pending result timeout not fired | Gap stays IN_PROGRESS indefinitely | Timeout job runs every 6 hours; missed timeouts caught by nightly audit |
| Appointment scheduling integration failure | Gap campaign stuck in APPOINTMENT_MADE | Bidirectional appointment status sync; timeout if appointment status not updated within 24 hours |
| NoteSignedEvent not received | Gap closure delayed by 14-90 days until claim | Duplicate closure sources (HIE, direct lab) also watch for the service; redundant, not dependent on any single source |
| Outreach sent in wrong language | Patient ignores or is distressed | Pre-flight language check; missing language → phone call with interpreter flag |
| Measure spec has error (published NCQA errata) | Incorrect gap population | Spec versioning allows fast correction and re-evaluation; errata pipeline monitored |
| January 1 annual reset job fails | All annual gaps appear newly opened | Measurement year reset is an idempotent job; safe to re-run; alert on failure; manual trigger available |

---

## 14. Scaling Considerations

### Per-Practice Model

- Patient population: 2,000–15,000 patients per 10-provider independent practice
- HEDIS measures: ~90 measures × patient population = gap evaluation budget
- At 10,000 patients × 90 measures = 900,000 evaluations per nightly batch
- Target: < 30 minutes per full population evaluation (achievable at 500 evals/second)

### Multi-Practice Scale

| Component | Scaling approach |
|---|---|
| EMPI matching | Per-clinic patient namespaces; shared matching algorithm; clinic_id enforced on all data |
| Measure evaluation | Stateless workers; patient-gap evaluations are independent; parallelizable |
| Gap record store | PostgreSQL partitioned by clinic_id + measurement_year |
| Outreach engine | Message queue per channel; per-clinic rate limiting for SMS/voice |
| LLM message generation | Claude API with per-clinic rate limiting |
| Quality reports | Generated on-demand per clinic; cached for 24 hours |

### Quality Metrics

| Metric | Target | Alert threshold |
|---|---|---|
| Gap identification latency (P95) | < 24 hours | > 48 hours |
| EMPI false positive rate | < 0.1% | > 0.5% |
| Outreach-to-closure conversion | > 35% | < 20% |
| Pre-encounter gap summary latency | < 2 minutes | > 5 minutes |
| Annual HEDIS spec update lag | < 30 days from NCQA release | > 60 days |
| Patient declined / opt-out rate | < 15% per measure | > 30% |

---

## Appendix A: Key HEDIS Measures Reference

| Measure ID | Name | Eligible Population | What closes it |
|---|---|---|---|
| BCS-E | Breast Cancer Screening | Women 50-74 | Mammogram in last 2 years |
| CDC-HbA1c | HbA1c Testing | Diabetic patients 18-75 | A1C test in last 12 months |
| CDC-Eye | Diabetic Eye Exam | Diabetic patients 18-75 | Retinal exam in last 12 months |
| COL-E | Colorectal Cancer Screening | Adults 45-75 | Colonoscopy 10yr / FIT 1yr / Stool DNA 3yr |
| CCS | Cervical Cancer Screening | Women 21-64 | Pap smear (per age-based schedule) |
| DEP | Depression Screening | Adults 12+ | PHQ-9 administered + score documented |
| FLU-AD | Flu Vaccination (Adults) | Adults 18+ | Influenza vaccine each flu season |
| PCV | Pneumococcal Vaccination | Adults 65+ | PCV15 or PCV20 vaccination |
| CBP | Blood Pressure Control | Hypertension 18-85 | BP reading < 140/90 |
| MED-ACE | ACE/ARB Adherence | Heart failure patients | Medication refill rate > 80% |

---

## Appendix B: Gap State Machine

```
                    ┌─────────────────────────────┐
                    │                             │
    data available  │         OPEN                │◄── annual measure reset
    patient eligible│   (priority scored,         │
    not excluded    │    outreach triggered)       │
                    │                             │
                    └──────────┬──────────────────┘
                               │ outreach sent /
                               │ service ordered
                    ┌──────────▼──────────────────┐
                    │                             │
                    │       IN_PROGRESS           │
                    │   (service ordered or       │◄── result timeout (re-opens)
                    │    appointment made)         │
                    │                             │
                    └───┬──────────┬──────────────┘
                        │          │
               result   │          │ patient
               confirmed│          │ declined
                        │          │
           ┌────────────▼──┐  ┌────▼────────────────┐
           │               │  │                     │
           │    CLOSED     │  │  PATIENT_DECLINED   │
           │  (terminal    │  │  (enter exclusion   │
           │   for year)   │  │   review workflow)  │
           │               │  │                     │
           └───────────────┘  └──────┬──────────────┘
                                     │ clinical
                                     │ contraindication confirmed
                              ┌──────▼──────────────┐
                              │                     │
                              │     EXCLUDED        │
                              │  (terminal,         │
                              │   never outreach)   │
                              │                     │
                              └─────────────────────┘
```

---

## Appendix C: Eraser.io Diagram Code

> **How to use**: In your Eraser.io document, click `+` → **Diagram** → switch to **Code** tab → select all → paste the block below.

```
title Care Gap Detection and Closure System

direction right

// ── DATA SOURCES (Blue) ───────────────────────────────────────────
EHR Data Source [color: blue, icon: database]
Payer Claims Feed [color: blue, icon: file-text]
State IIS Registry [color: blue, icon: shield]
HIE Connector [color: blue, icon: share-2]
Lab Results Gateway [color: blue, icon: activity]

// ── PATIENT IDENTITY (Green) ──────────────────────────────────────
Patient Identity Resolution [color: green, icon: users]
Data Staging Queue [color: red, icon: inbox]
Longitudinal Patient Profile [color: green, icon: user]

// ── MEASURE ENGINE (Purple) ───────────────────────────────────────
Measure Evaluation Engine [color: purple, icon: sliders] {
  Denominator Evaluator [color: purple, icon: filter]
  Exclusion Evaluator [color: purple, icon: x-circle]
  Numerator Evaluator [color: purple, icon: check-circle]
}
Measure Spec Store [color: purple, icon: book-open]

// ── GAP MANAGEMENT (Orange) ───────────────────────────────────────
Gap Record Store [color: orange, icon: layers]
Priority Scorer [color: orange, icon: trending-up]
Provider Gap Summary [color: orange, icon: monitor]

// ── OUTREACH ENGINE (Yellow) ──────────────────────────────────────
Outreach Campaign Manager [color: yellow, icon: send]
Channel Dispatcher [color: yellow, icon: git-branch]
SMS Gateway [color: yellow, icon: message-square]
Patient Portal Message [color: yellow, icon: mail]
Automated Voice [color: yellow, icon: phone]
Letter Service [color: yellow, icon: file-text]
Outreach Response Handler [color: yellow, icon: inbox]

// ── SCHEDULING (Yellow) ───────────────────────────────────────────
Appointment Request Service [color: yellow, icon: calendar]
No Show Handler [color: yellow, icon: alert-circle]

// ── CLOSURE CONFIRMATION (Cyan) ───────────────────────────────────
Gap Evidence Monitor [color: cyan, icon: eye]
Pending Result Tracker [color: cyan, icon: clock]
Provider Attestation [color: cyan, icon: user-check]
Manual Exclusion Registry [color: cyan, icon: slash]

// ── PLATFORM INPUTS (Teal) ────────────────────────────────────────
Note Signed Event [color: teal, icon: file-check]
Claim Paid Event [color: teal, icon: dollar-sign]

// ── REPORTING (Teal) ──────────────────────────────────────────────
Quality Reporting Engine [color: teal, icon: bar-chart-2]
HEDIS Submission [color: teal, icon: send]
Population Dashboard [color: teal, icon: pie-chart]
VBC Financial Report [color: teal, icon: trending-up]

// ── ERROR AND EDGE CASE PATHS (Red) ───────────────────────────────
Patient Declined [color: red, icon: x]
Outreach Exhausted [color: red, icon: alert-octagon]
Manual Review Queue [color: red, icon: user]
Pending Result Timeout [color: red, icon: alert-triangle]

// ══ RELATIONSHIPS ═════════════════════════════════════════════════

// DATA SOURCES → PATIENT IDENTITY
EHR Data Source > Patient Identity Resolution
Payer Claims Feed > Patient Identity Resolution
State IIS Registry > Patient Identity Resolution
HIE Connector > Patient Identity Resolution
Lab Results Gateway > Patient Identity Resolution

// PATIENT IDENTITY → PROFILE (matched) or STAGING (unmatched)
Patient Identity Resolution > Longitudinal Patient Profile: matched
Patient Identity Resolution > Data Staging Queue: unmatched pending review [color: red, style: dashed]
Data Staging Queue > Patient Identity Resolution: retry after human review [color: red, style: dashed]

// PROFILE → MEASURE ENGINE
Longitudinal Patient Profile > Denominator Evaluator

// MEASURE ENGINE internal flow (sequential phases)
Denominator Evaluator > Exclusion Evaluator: eligible patients only
Exclusion Evaluator > Numerator Evaluator: non-excluded patients

// MEASURE SPEC provides criteria to all three phases
Measure Spec Store > Denominator Evaluator
Measure Spec Store > Exclusion Evaluator
Measure Spec Store > Numerator Evaluator

// NUMERATOR EVALUATOR → GAP RECORD STORE
Numerator Evaluator > Gap Record Store: open or closed gap

// GAP RECORD → PRIORITIZATION AND PROVIDER SUMMARY
Gap Record Store > Priority Scorer
Gap Record Store > Provider Gap Summary: pre-encounter display
VBC Financial Report > Priority Scorer: financial weighting

// PROVIDER SUMMARY → Medical Scribe (external)
Provider Gap Summary > Note Signed Event: provider addresses gap during visit [color: teal, style: dashed]

// PRIORITY SCORER → OUTREACH CAMPAIGN
Priority Scorer > Outreach Campaign Manager

// OUTREACH CAMPAIGN → CHANNELS
Outreach Campaign Manager > Channel Dispatcher
Channel Dispatcher > SMS Gateway
Channel Dispatcher > Patient Portal Message
Channel Dispatcher > Automated Voice
Channel Dispatcher > Letter Service

// ALL CHANNELS → RESPONSE HANDLER
SMS Gateway > Outreach Response Handler
Patient Portal Message > Outreach Response Handler
Automated Voice > Outreach Response Handler
Letter Service > Outreach Response Handler

// RESPONSE HANDLER BRANCHES
Outreach Response Handler > Appointment Request Service: patient responds
Outreach Response Handler > Patient Declined: patient declines [color: red, style: dashed]
Outreach Response Handler > Outreach Campaign Manager: no response — next step [color: yellow, style: dashed]

// OUTREACH EXHAUSTED PATH
Outreach Campaign Manager > Outreach Exhausted: all steps attempted no response [color: red, style: dashed]
Outreach Exhausted > Manual Review Queue: care coordinator follow-up [color: red, style: dashed]

// PATIENT DECLINED PATH
Patient Declined > Manual Exclusion Registry: clinical review [color: red, style: dashed]
Manual Exclusion Registry > Exclusion Evaluator: re-evaluate with new exclusion [color: red, style: dashed]

// SCHEDULING
Appointment Request Service > Gap Evidence Monitor: appointment completed
Appointment Request Service > No Show Handler: patient no-show [color: red, style: dashed]
No Show Handler > Outreach Campaign Manager: reactivate campaign after 7 days [color: yellow, style: dashed]

// PLATFORM EVENTS → CLOSURE
Note Signed Event > Gap Evidence Monitor
Claim Paid Event > Gap Evidence Monitor

// CLOSURE PATHS
Gap Evidence Monitor > Numerator Evaluator: evidence received — re-evaluate
Gap Evidence Monitor > Pending Result Tracker: service ordered result pending
Provider Attestation > Gap Evidence Monitor: manual attestation submitted

// PENDING RESULT TIMEOUT
Pending Result Tracker > Pending Result Timeout: result not received within TAT [color: red, style: dashed]
Pending Result Timeout > Gap Record Store: reopen gap [color: red, style: dashed]
Pending Result Timeout > Manual Review Queue: notify provider [color: red, style: dashed]

// REPORTING
Gap Record Store > Quality Reporting Engine
Quality Reporting Engine > HEDIS Submission
Quality Reporting Engine > Population Dashboard
Quality Reporting Engine > VBC Financial Report

legend {
  [connection: ">", color: black, label: "Primary process flow"]
  [connection: ">", color: red, label: "Error, failure or edge case path"]
  [connection: ">", color: teal, label: "Platform integration (Medical Scribe / Claim Scrubbing)"]
  [connection: ">", color: yellow, label: "Outreach loop or retry"]
  [connection: ">", color: purple, label: "Measure evaluation flow"]
  [connection: ">", color: orange, label: "Gap management and prioritization"]
  [connection: ">", color: cyan, label: "Closure confirmation flow"]
  [connection: ">", style: dashed, color: red, label: "Error fallback or edge case"]
  [connection: ">", style: dashed, color: yellow, label: "Loop or retry within outreach"]
  [connection: ">", style: dashed, color: teal, label: "Cross-system platform event"]
  [shape: oval, label: "Start / End state"]
  [shape: rectangle, label: "Process or system component"]
}
```
