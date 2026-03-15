# AI Medical Scribe — System Design
**Independence OS | Clinical Documentation from Audio**
*Author: System Design Document | Date: 2026-03-14*

---

## Table of Contents

1. [Problem Framing](#1-problem-framing)
2. [System Overview](#2-system-overview)
3. [Component Deep-Dive](#3-component-deep-dive)
4. [Data Models](#4-data-models)
5. [Pipeline Flow with Latency Budget](#5-pipeline-flow-with-latency-budget)
6. [Medical Coding Engine](#6-medical-coding-engine)
7. [Doctor Review Interface](#7-doctor-review-interface)
8. [HIPAA & Privacy Architecture](#8-hipaa--privacy-architecture)
9. [EHR Integration](#9-ehr-integration)
10. [Key Design Decisions & Trade-offs](#10-key-design-decisions--trade-offs)
11. [Failure Modes & Mitigations](#11-failure-modes--mitigations)
12. [Scaling Considerations](#12-scaling-considerations)

---

## 1. Problem Framing

### What We're Actually Solving

The core challenge is not just transcription — it is **structured semantic extraction under clinical constraints**.

| Dimension | Why It's Hard |
|---|---|
| Audio quality | Clinic environments have background noise, overlapping speech, whispering |
| Multi-speaker | Doctor voice ≠ patient voice; sometimes nurse or MA also speaks |
| Medical jargon | "atrioventricular nodal reentrant tachycardia" is one concept |
| Ambiguity | "She had an MI last year" → could be patient's PMH or family history |
| Section overlap | Symptom onset belongs in HPI *and* ROS |
| Code specificity | ICD-10 needs laterality, severity, encounter type; CPT needs time, complexity |
| Legal weight | Every word is a legal medical record and billing document |
| Speed | Doctor review must be under 3 minutes, not creation |

### Target Outcome

- **Doctor time**: 15 min → under 3 min (note review + corrections, not generation)
- **Note quality**: Passes billing audit; no upcoding or downcoding
- **Latency**: Draft note ready within 2 minutes of audio upload/end of visit
- **Accuracy bar**: < 5% section error rate; flagged items always reviewed

---

## 2. System Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                        CLINIC ENVIRONMENT                          │
│                                                                    │
│   [Microphone / Recording Device]  ──►  [Local Edge Agent]        │
│        (ambient or placed)              (noise filter, VAD)        │
└────────────────────────────┬───────────────────────────────────────┘
                             │  Encrypted audio stream / file
                             ▼
┌────────────────────────────────────────────────────────────────────┐
│                     INGESTION & TRANSCRIPTION                      │
│                                                                    │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐ │
│  │ Audio Intake │───►│ Speaker          │───►│ ASR Engine       │ │
│  │ & Validation │    │ Diarization      │    │ (Medical ASR)    │ │
│  └──────────────┘    └──────────────────┘    └────────┬─────────┘ │
│                                                        │           │
│                                              Timestamped transcript│
│                                              with speaker labels   │
└────────────────────────────────────────────────┬───────────────────┘
                                                 │
                             ┌───────────────────▼──────────────────┐
                             │      TRANSCRIPT POST-PROCESSING       │
                             │  (medical spell-check, filler removal,│
                             │   sentence segmentation, de-ID check) │
                             └───────────────────┬──────────────────┘
                                                 │
                             ┌───────────────────▼──────────────────┐
                             │     CLINICAL EXTRACTION ENGINE        │
                             │                                       │
                             │  ┌─────────────┐  ┌───────────────┐  │
                             │  │ Segment     │  │ Section       │  │
                             │  │ Classifier  │  │ Populator     │  │
                             │  └──────┬──────┘  └───────┬───────┘  │
                             │         │                  │          │
                             │  ┌──────▼──────────────────▼───────┐ │
                             │  │     LLM Orchestrator (Claude)    │ │
                             │  │  - entity extraction             │ │
                             │  │  - section mapping               │ │
                             │  │  - ambiguity resolution          │ │
                             │  │  - confidence scoring            │ │
                             │  └──────────────────────────────────┘ │
                             └───────────────────┬──────────────────┘
                                                 │
                             ┌───────────────────▼──────────────────┐
                             │         CODING ENGINE                 │
                             │                                       │
                             │  ┌─────────────┐  ┌───────────────┐  │
                             │  │ ICD-10      │  │ CPT           │  │
                             │  │ Suggester   │  │ Suggester     │  │
                             │  └──────┬──────┘  └───────┬───────┘  │
                             │         │                  │          │
                             │  ┌──────▼──────────────────▼───────┐ │
                             │  │  E&M Level Determinator          │ │
                             │  │  (MDM complexity scoring)        │ │
                             │  └──────────────────────────────────┘ │
                             └───────────────────┬──────────────────┘
                                                 │
                             ┌───────────────────▼──────────────────┐
                             │     PROGRESS NOTE ASSEMBLER           │
                             │  (fills 13+ sections, formats output) │
                             └───────────────────┬──────────────────┘
                                                 │
                             ┌───────────────────▼──────────────────┐
                             │     DOCTOR REVIEW INTERFACE           │
                             │  - section editing                    │
                             │  - code approval/modification         │
                             │  - confidence flag review             │
                             │  - one-click sign & lock              │
                             └───────────────────┬──────────────────┘
                                                 │
                             ┌───────────────────▼──────────────────┐
                             │      EHR INTEGRATION LAYER            │
                             │   (FHIR R4, HL7, CDA export)          │
                             │   Epic / Athena / Cerner connectors   │
                             └──────────────────────────────────────┘
```

---

## 3. Component Deep-Dive

### 3.1 Local Edge Agent

**Purpose**: Runs in-clinic on a small device (Mac mini, tablet). Handles capture before any data leaves the building.

**Responsibilities**:
- Voice Activity Detection (VAD) — discard silence, reduce upload size by ~60%
- Noise suppression (RNNoise or WebRTC audio processing)
- Audio chunking for streaming (4-second chunks with 1-second overlap to prevent word-cutting)
- AES-256 encryption before upload
- Offline buffering if network drops — queues locally, flushes when back online
- Consent gate — recording does not start until both parties verbally acknowledge (or button press)

**Key decision**: Keep this lightweight. No ML inference on the edge. The edge agent is a capture + security layer only.

---

### 3.2 Audio Intake & Validation

**Inputs accepted**: WAV, MP3, M4A, FLAC, OGG, WebM, raw PCM stream
**Validation checks**:
- Minimum audio duration (reject < 60s — not a real encounter)
- Clipping detection (warn if audio is saturated)
- Language detection (flag non-English for human review)
- Encoding normalization → 16kHz mono PCM for ASR

---

### 3.3 Speaker Diarization

**Goal**: Label each audio segment as DOCTOR, PATIENT, or OTHER (nurse, MA, phone speaker).

**Approach**:
- Primary: [pyannote.audio](https://github.com/pyannote/pyannote-audio) speaker diarization model
- We pre-enroll doctor voice prints per provider at setup (30-second sample sufficient)
- Patient is always the "other" enrolled speaker
- Output: `[(t_start, t_end, speaker_label), ...]`

**Why this matters for the note**: Chief Complaint comes from what the *patient* says. Physical Exam findings come from what the *doctor* says. Without diarization, a model cannot reliably distinguish them.

**Edge cases**:
- Phone-in patient: third "channel" detected — flag for manual label
- Multiple doctors (resident + attending): both labeled DOCTOR, merged
- Translator present: flag utterances in non-English for exclusion from clinical content

---

### 3.4 ASR Engine (Medical Speech Recognition)

**Primary option**: OpenAI Whisper `large-v3` fine-tuned on medical speech
- Handles accents well
- Strong on medical terminology out-of-the-box
- Self-hosted to keep PHI on-premises or in HIPAA BAA cloud

**Alternative**: AWS Transcribe Medical (HIPAA-eligible, specialties: General Medicine, Cardiology, Neurology, etc.)

**Output format**:
```json
{
  "segments": [
    {
      "start": 0.42,
      "end": 3.18,
      "speaker": "DOCTOR",
      "text": "So what brings you in today?",
      "confidence": 0.97
    },
    {
      "start": 3.5,
      "end": 9.2,
      "speaker": "PATIENT",
      "text": "I've had this chest pain for about three days now, it's worse when I breathe in",
      "confidence": 0.94
    }
  ]
}
```

**Medical vocabulary boost**:
- Custom vocabulary list of 50,000+ medical terms injected into Whisper decoder beam search
- Drug name dictionary (RxNorm) → improves medication recognition from ~71% to ~94%
- Procedure/anatomy terms from SNOMED CT

---

### 3.5 Transcript Post-Processing

**Steps in order**:

1. **Medical spell correction**: Contextual correction using a Levenshtein + medical ontology lookup. "Metophrolol" → "Metoprolol". Uses Peter Norvig's approach with a medical corpus.

2. **Filler word stripping**: Remove "um", "uh", "like", "you know" from patient speech; retain doctor speech intact (may contain clinical phrasing).

3. **Sentence boundary detection**: ASR outputs word-level timestamps but not sentences. Use a fine-tuned BERT model for punctuation restoration.

4. **De-identification check**: Run a Named Entity Recognition pass to flag any PII/PHI that leaked into the transcript (patient name, DOB, SSN). This is a safety rail — actual PHI handling is governed separately.

5. **Conversation segmentation**: Group continuous turns into semantic blocks (a block = one coherent topic). Helps the LLM process context chunks rather than a raw flat transcript.

---

### 3.6 Clinical Extraction Engine

This is the intelligence core of the system.

**Architecture**: LLM Orchestrator (Claude claude-sonnet-4-6 or equivalent) + structured extraction pipeline

#### 3.6.1 Segment Classifier

Before full extraction, classify each conversation block into a rough note section:

```
Block → [CC, HPI, ROS, PMH, PSH, FH, SH, ALLERGIES, MEDS, EXAM, ASSESSMENT, PLAN, OTHER]
```

This is a lightweight classifier (fine-tuned BERT or few-shot Claude) that runs fast and narrows the LLM's extraction window.

#### 3.6.2 LLM Extraction — Section by Section

Rather than one massive prompt, we use a **multi-pass extraction strategy**:

**Pass 1 — Entity Extraction** (single pass over full transcript):
Extract atomic clinical facts:
```json
{
  "symptoms": [
    {"text": "chest pain", "duration": "3 days", "character": "pleuritic", "severity": "7/10"}
  ],
  "medications": [
    {"name": "lisinopril", "dose": "10mg", "frequency": "daily"}
  ],
  "diagnoses_mentioned": ["hypertension", "pleurisy", "rule out PE"],
  "procedures_performed": ["auscultation", "EKG ordered"],
  "vitals": {"BP": "142/88", "HR": "96", "RR": "18", "SpO2": "97%"},
  "allergies": [{"allergen": "penicillin", "reaction": "rash"}]
}
```

**Pass 2 — Section Population** (section-by-section, with entities as context):
For each of the 13 sections, prompt the LLM with:
- The relevant extracted entities
- The raw transcript segments classified to that section
- The section's documentation requirements
- A few-shot example of a well-written section

This produces section drafts with inline confidence markers.

**Pass 3 — Cross-section Consistency Check**:
Verify: "Is the Assessment consistent with HPI? Does the Plan address everything in Assessment? Are billed diagnoses supported by documentation?"

#### 3.6.3 Confidence Scoring

Every extracted claim gets a confidence score:

| Score | Meaning | UI treatment |
|---|---|---|
| 0.9+ | Explicitly stated, clear | No flag |
| 0.7–0.9 | Inferred, high confidence | Subtle highlight |
| 0.5–0.7 | Ambiguous, needs review | Yellow flag |
| < 0.5 | Low confidence, possibly hallucinated | Red flag + mandatory review |

Low-confidence items are never silently included. The doctor must take an explicit action on them.

#### 3.6.4 Hallucination Guards

This is critical for a medico-legal document.

- **Grounding constraint**: Every sentence in the generated note must trace back to a timestamped segment in the transcript. If it can't be traced, it's flagged.
- **No inference beyond evidence**: The model is instructed never to infer diagnoses not mentioned or implied in the audio.
- **Negation preservation**: "No chest pain" must not become "chest pain". Negation is explicitly tracked.
- **Audit trail**: The final document stores source segment IDs for every claim. Doctor can click any sentence and hear the relevant audio.

---

### 3.7 Progress Note Sections

The system must produce all 13+ standard sections of a SOAP/APSO note:

| Section | Source in conversation | Special requirements |
|---|---|---|
| **Chief Complaint (CC)** | Patient's opening statement | Verbatim or near-verbatim, 1–2 sentences |
| **History of Present Illness (HPI)** | Patient narrative + doctor questions | Must include: onset, duration, character, severity, location, radiation, modifying factors, associated symptoms |
| **Review of Systems (ROS)** | Doctor asking systems review questions | List per system: positive and pertinent negatives |
| **Past Medical History (PMH)** | Doctor/patient discussing existing conditions | Conditions with approximate dates if mentioned |
| **Past Surgical History (PSH)** | Surgeries mentioned | Procedure + year if known |
| **Family History (FH)** | Family conditions discussed | Relationship + condition |
| **Social History (SH)** | Smoking, alcohol, occupation, living situation | Quantify: "1 PPD × 20 years" |
| **Allergies** | Any allergy mentioned | Allergen + reaction type |
| **Current Medications** | Medication list reviewed | Name + dose + frequency + indication |
| **Physical Examination** | Doctor performing/describing exam | Systems examined + findings |
| **Assessment** | Doctor's diagnostic conclusions | Problem list, numbered |
| **Plan** | Treatment plan discussed | Per-problem plan, including: Rx, labs, imaging, referrals, follow-up |
| **Billing** | Derived from above | ICD-10 + CPT codes |

---

## 4. Data Models

### 4.1 Encounter

```python
class Encounter:
    encounter_id: UUID
    clinic_id: UUID
    provider_id: UUID
    patient_id: UUID          # links to EHR patient record
    encounter_date: datetime
    encounter_type: str       # "office_visit", "telehealth", "follow_up"
    audio_file_path: str      # encrypted S3/local path
    audio_duration_seconds: int
    status: EncounterStatus   # RECORDING, PROCESSING, DRAFT, IN_REVIEW, SIGNED, LOCKED
    created_at: datetime
    signed_at: datetime | None
    signed_by: UUID | None
```

### 4.2 Transcript

```python
class TranscriptSegment:
    segment_id: UUID
    encounter_id: UUID
    start_ms: int
    end_ms: int
    speaker: SpeakerRole       # DOCTOR, PATIENT, OTHER
    raw_text: str              # direct ASR output
    cleaned_text: str          # post-processed
    asr_confidence: float
    classified_section: NoteSection | None

class Transcript:
    transcript_id: UUID
    encounter_id: UUID
    segments: list[TranscriptSegment]
    processing_model: str      # "whisper-large-v3"
    processing_duration_ms: int
```

### 4.3 Progress Note

```python
class NoteSection:
    section_id: UUID
    note_id: UUID
    section_type: SectionType  # CC, HPI, ROS, PMH, ...
    content: str               # generated markdown/text
    confidence: float          # aggregate section confidence
    source_segment_ids: list[UUID]  # grounding references
    flagged_items: list[FlaggedItem]
    doctor_edited: bool
    final_content: str | None  # content after doctor edits

class ProgressNote:
    note_id: UUID
    encounter_id: UUID
    sections: list[NoteSection]
    icd10_codes: list[CodeSuggestion]
    cpt_codes: list[CodeSuggestion]
    em_level: str              # "99213", "99214", etc.
    generation_model: str
    generation_duration_ms: int
    status: NoteStatus         # DRAFT, IN_REVIEW, SIGNED, LOCKED

class CodeSuggestion:
    code: str                  # "I21.9", "99213"
    description: str
    confidence: float
    supporting_text: str       # excerpt from note that supports this code
    code_type: str             # "ICD10", "CPT", "EM"
    accepted: bool | None      # None = pending doctor review
```

### 4.4 Audit Log

```python
class AuditEvent:
    event_id: UUID
    encounter_id: UUID
    actor_id: UUID
    actor_role: str            # "SYSTEM", "DOCTOR", "MA"
    action: str                # "SECTION_EDITED", "CODE_ACCEPTED", "NOTE_SIGNED"
    before_value: str | None
    after_value: str | None
    timestamp: datetime
```

Every change to the note — by system or human — is logged. This is a legal requirement.

---

## 5. Pipeline Flow with Latency Budget

The key insight: **pipeline stages can be parallelized and pipelined** so the doctor sees the note within 2 minutes, not 4.

```
Timeline after visit ends (or audio upload):

T+0s    ─── Audio upload begins ──────────────────────────────────►

T+0s    ─── VAD + chunking (streaming, overlaps upload) ─────────►

T+15s   ─── Speaker diarization starts (on first 60s chunk) ─────►
        ─── ASR starts (parallel with diarization) ──────────────►

T+60s   ─── First transcript segments available ─────────────────►
        ─── Post-processing starts (rolling) ──────────────────────►

T+90s   ─── LLM Pass 1 (entity extraction) starts ───────────────►

T+120s  ─── Transcript fully available ──────────────────────────►
        ─── LLM Pass 2 (section population, parallel for each section)►

T+150s  ─── Coding engine starts (runs on Pass 2 output) ─────────►

T+170s  ─── Draft note ready for doctor review ◄──────────────────

[Doctor reviews for ~2–3 minutes]

T+350s  ─── Note signed and locked ◄──────────────────────────────
        ─── Pushed to EHR ◄───────────────────────────────────────
```

**Latency budget breakdown**:

| Stage | Duration | Optimization |
|---|---|---|
| Upload (15-min audio at 128kbps) | ~25s | Compress to OPUS, stream upload |
| VAD + chunking | ~5s | Runs during upload |
| Speaker diarization | ~45s | Parallelized with ASR |
| ASR transcription (15 min audio) | ~90s | Whisper large-v3, 2× GPU |
| Post-processing | ~10s | CPU-bound, fast |
| LLM Pass 1 (entity extraction) | ~20s | Claude API, ~3k tokens |
| LLM Pass 2 (13 sections, parallel) | ~30s | 13 parallel Claude calls |
| LLM Pass 3 (consistency check) | ~10s | Short prompt |
| Coding engine | ~15s | Lookup-heavy, fast |
| Note assembly | ~2s | Template fill |
| **Total** | **~170s** | **< 3 min** |

**Why < 3 min is achievable**: The pipeline is streaming and parallel. The doctor doesn't wait for the entire audio to be transcribed — the note sections are populated progressively as transcript chunks become available.

---

## 6. Medical Coding Engine

This is the highest-stakes component from a revenue and compliance perspective.

### 6.1 ICD-10 Suggestion (Diagnosis Codes)

**Input**: Assessment section text + extracted entities

**Approach: Two-stage**

**Stage 1 — Semantic retrieval**:
- Embed the diagnosis text using a medical embeddings model (BioBERT or text-embedding-3-large)
- Retrieve top-50 candidate ICD-10 codes from a vector index of all 70,000+ codes + their descriptions
- Pre-filter by body system (cardiology, pulmonology, etc.) based on HPI context

**Stage 2 — LLM re-ranking**:
- Feed top-50 candidates + assessment text to Claude
- Claude selects the most specific applicable code with justification
- Specificity matters: I21.9 (AMI unspecified) → I21.01 (STEMI of LAD, initial) is the difference between correct and incorrect billing

**Hierarchical code selection**:
```
"Chest pain" in HPI → R07.9 (chest pain, unspecified) [if no diagnosis made]
"Pleuritis" in Assessment → R07.81 (pleurodynia)
"Pleurisy confirmed by CT" → J90 (pleural effusion) + R09.1 (pleurisy)
```

**Laterality enforcement**: ICD-10 often requires side specification. If "left knee pain" is mentioned but the code needs laterality and the doctor didn't specify which side, flag for explicit doctor confirmation.

### 6.2 CPT Suggestion (Procedure Codes)

**Input**: Physical Exam section + Plan section + encounter metadata

**E&M Code Determination (the most valuable CPT)**:
E&M codes (99202–99215) determine the bulk of reimbursement. They're based on:

1. **Medical Decision Making (MDM)** — complexity of diagnoses, data reviewed, risk
2. **Total time** — can use time-based billing if documented

The system scores MDM automatically:
```
Number of diagnoses/problems  → Points
Amount of data reviewed       → Points (labs, imaging, records)
Risk level                    → Low / Moderate / High

Score → E&M Level:
  99213 = Low MDM
  99214 = Moderate MDM  [most common office visit]
  99215 = High MDM
```

**Procedure CPT codes** (beyond E&M):
- Pattern match from Plan section: "EKG" → 93000
- "Joint injection" → 20610 (major joint) with appropriate modifier
- Lab orders → do NOT auto-code (labs billed separately by lab)

### 6.3 Compliance Guard

- **Upcoding prevention**: If MDM score suggests 99213 but the note's documentation doesn't support 99214, block the upgrade and explain what documentation is missing
- **Bundling rules**: Check CCI (Correct Coding Initiative) edits — some CPT codes can't be billed together
- **Payer-specific rules**: Some insurers have different coverage rules. Flag accordingly

---

## 7. Doctor Review Interface

This is where the 15-minute time saving is captured or lost. The UI must be near-zero-friction.

### 7.1 Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  Patient: John Smith | DOB: 1965-03-12 | Date: 2026-03-14          │
│  Provider: Dr. Martinez | Encounter: Office Visit (Est. Patient)   │
│                                          [🎤 Play Audio]  [⏱ 2:47] │
├──────────────────────────────────┬──────────────────────────────────┤
│  NOTE SECTIONS                   │  BILLING CODES                   │
│  ──────────────                  │  ─────────────                   │
│  ✅ Chief Complaint              │  ICD-10                         │
│  ✅ HPI                          │  ┌─────────────────────────────┐ │
│  ⚠️  Review of Systems  [3 flags]│  │ R07.81  Pleurodynia   0.91 ✓│ │
│  ✅ Physical Exam                │  │ J18.9   Pneumonia NOS 0.73 ?│ │
│  ✅ Assessment                   │  │ I10     Hypertension  0.98 ✓│ │
│  ⚠️  Plan           [1 flag]    │  └─────────────────────────────┘ │
│  ✅ Allergies                    │                                  │
│  ✅ Medications                  │  CPT                            │
│  ✅ PMH / PSH / FH / SH          │  ┌─────────────────────────────┐ │
│                                  │  │ 99214  Level 4 E&M    0.88 ✓│ │
│  [Select section to edit]        │  │ 93000  EKG            0.95 ✓│ │
│                                  │  └─────────────────────────────┘ │
├──────────────────────────────────┴──────────────────────────────────┤
│  SECTION EDITOR                                                      │
│  ──────────────                                                      │
│  History of Present Illness                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ John Smith is a 61-year-old male with a history of           │   │
│  │ hypertension who presents with a 3-day history of            │   │
│  │ [⚠️ pleuritic chest pain] that is worse with inspiration,   │   │
│  │ rated 7/10 in severity, and associated with mild shortness   │   │
│  │ of breath. He denies fever, cough, or hemoptysis.            │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  [▶ Click any sentence to hear supporting audio]                    │
│                                                                      │
│         [← Prev Section]    [Accept & Next Section →]               │
│                                                                      │
├──────────────────────────────────────────────────────────────────────┤
│  [💾 Save Draft]        [❌ Request Re-generation]    [✅ SIGN NOTE]  │
└──────────────────────────────────────────────────────────────────────┘
```

### 7.2 Key UX Decisions

**1. Section-level progress, not word-level review**
The doctor moves through sections, not individual sentences. Accepting a section is one click if no flags are present.

**2. Confidence flags are always visible, never dismissible without action**
A flagged item requires an explicit "Accept as-is" or "Edit" — not just click-through.

**3. Audio linkage is passive but available**
Every generated sentence can be clicked to hear the supporting audio. The doctor doesn't have to listen — but can verify anything instantly.

**4. Inline diff when editing**
When the doctor edits text, a diff is shown so they can see exactly what changed from the AI draft. This builds trust and allows reversion.

**5. "Add dictation" escape hatch**
Any section can be replaced by the doctor speaking into the microphone. The dictation is transcribed and inserted. Doctor can still use the AI draft as a base.

**6. Keyboard-first navigation**
- `Tab` → next section
- `A` → accept section
- `E` → focus edit
- `S` → sign (requires confirmation modal)

### 7.3 Sign & Lock

When the doctor signs:
1. A snapshot of the final note is created (immutable)
2. All audit events are finalized
3. EHR push is triggered
4. Billing submission is queued
5. The note becomes read-only (amendments require an addendum, not editing)

---

## 8. HIPAA & Privacy Architecture

### 8.1 PHI Data Flow

```
Clinic → [TLS 1.3] → Edge Agent → [AES-256] → Secure Upload → Processing
                                                                     │
                                                    PHI never leaves HIPAA-compliant zone
                                                    (AWS GovCloud or on-prem)
```

**What counts as PHI**: Name, DOB, address, phone, MRN, dates, geographic data, biometric identifiers, voice (audio itself).

**Rule**: Audio and identified transcripts never leave HIPAA-covered infrastructure. All third-party API calls (Claude, etc.) must be under a signed BAA (Business Associate Agreement).

### 8.2 Encryption

| Data at rest | Data in transit | Key management |
|---|---|---|
| AES-256 for audio files | TLS 1.3 minimum | AWS KMS per-clinic key rotation |
| AES-256 for transcript text | mTLS for service-to-service | Key access logged |
| AES-256 for note content | HTTPS for all API calls | Doctor private key signs note |

### 8.3 De-identification for Model Training

To improve the model over time without exposing PHI:
- Notes and transcripts are de-identified using Philter or AWS Comprehend Medical de-identification before any use in model training
- De-identification removes: names, dates (shifted ±30 days), locations, phone numbers, MRNs
- Only de-identified data leaves the HIPAA zone

### 8.4 Access Control

| Role | Can see audio | Can see transcript | Can edit note | Can sign |
|---|---|---|---|---|
| Doctor (owner) | ✅ | ✅ | ✅ | ✅ |
| Other doctor | ❌ | ❌ | ❌ | ❌ |
| Medical Assistant | ❌ | ❌ | ✅ (limited sections) | ❌ |
| Billing staff | ❌ | ❌ | Codes only | ❌ |
| System/AI | Processing only | Processing only | Generates draft | ❌ |

### 8.5 Audit Requirements

HIPAA requires audit logs for all PHI access. Every audio play, transcript view, note edit, and code modification is logged with actor, timestamp, and IP address.

---

## 9. EHR Integration

### 9.1 Standards

**FHIR R4** is the output format for all EHR integrations:
- `DocumentReference` resource for the note text
- `Condition` resources for ICD-10 diagnoses
- `Procedure` resources for CPT codes
- `Encounter` resource links everything together

**HL7 v2** for legacy EHR systems that don't support FHIR (still ~40% of practices).

### 9.2 EHR Connectors

| EHR | Integration method | Notes |
|---|---|---|
| Epic | SMART on FHIR + Epic APIs | Industry standard, well-documented |
| Athenahealth | REST API + webhooks | Common in independent practices |
| Cerner (Oracle Health) | FHIR R4 | Requires Cerner-approved app |
| eClinicalWorks | HL7 + their API | Complex, requires certification |
| DrChrono | REST API | Simpler, common in small practices |
| Manual export | PDF + CDA XML | Fallback for any EHR |

### 9.3 Push Strategy

```
Note signed
    │
    ├─► FHIR R4 bundle assembled
    │
    ├─► Check EHR connector type
    │       ├─ Epic: POST /DocumentReference + POST /Condition[]
    │       ├─ Athena: POST /chart/encounter/{id}/clinicalnote
    │       └─ Fallback: Generate CDA XML + PDF, email/sftp
    │
    ├─► Confirm receipt (200 OK + EHR record ID)
    │
    └─► Update encounter status → LOCKED, store EHR record ID
```

**Failure handling**: If EHR push fails, note is still signed and stored locally. Retry with exponential backoff. After 3 failures, alert admin and queue for manual export.

---

## 10. Key Design Decisions & Trade-offs

### Decision 1: Single LLM vs. Ensemble of Specialized Models

**Option A — Single Claude call** for everything:
- Pros: Simple, coherent output, easy to maintain
- Cons: If one section is wrong, whole generation fails; harder to debug

**Option B — Per-section specialized models**:
- Pros: Each model optimized for its section; easier to swap
- Cons: Stitching outputs is complex; inconsistency between sections

**Decision: Hybrid**. Single entity extraction pass (Pass 1) for consistency, then parallel per-section calls (Pass 2) for thoroughness. Claude claude-sonnet-4-6 as the base model with section-specific system prompts.

---

### Decision 2: Streaming vs. Batch Processing

**Option A — Batch**: Wait for full audio, then process
- Pros: Simpler, can use full-context models
- Cons: Doctor waits 3+ minutes; perception of lag

**Option B — Streaming**: Process audio chunks in real-time during the visit
- Pros: Note begins forming during visit; sub-2-min wait time
- Cons: More complex; early context might be wrong and need revision

**Decision: Streaming for transcription + diarization, batch for LLM extraction**. The transcript is built in real-time, but the LLM extraction waits for the complete transcript to avoid section confusion from partial context.

---

### Decision 3: On-Premises vs. Cloud Processing

**Option A — On-premises**: Run all models in-clinic
- Pros: PHI never leaves the building; no cloud dependency
- Cons: Hardware cost per clinic ($15k+ GPU server); maintenance burden

**Option B — Cloud (HIPAA-compliant)**: Process in AWS/Azure with BAA
- Pros: Cheaper per-clinic; scale easily; always updated models
- Cons: Network dependency; latency for rural clinics; BAA required with every vendor

**Option C — Hybrid**: Transcription on-premises (audio is most sensitive), LLM in HIPAA cloud
- Pros: Best privacy for audio; leverage cloud LLM capabilities
- Cons: Must trust cloud LLM provider with transcript text

**Decision: Hybrid (Option C)** with a "fully on-prem" mode for high-security requirements. Transcription stays local because audio is biometric PHI. Transcript (text) can go to HIPAA-compliant cloud.

---

### Decision 4: How Aggressive to Be with Automation

**Conservative**: Flag everything with confidence < 0.99, doctor reviews every sentence
**Aggressive**: Auto-accept everything above 0.7, doctor only sees flagged items

**Decision: Adaptive automation** based on track record:
- New section type or new provider → conservative (more flags)
- Section type with 100+ successful edits with low correction rate → liberal (fewer flags)
- Billing codes: always require explicit acceptance regardless of confidence (too high stakes)

---

### Decision 5: Real-time Feedback to the Doctor During Visit

**Option A**: Doctor gets the note only after the visit ends
**Option B**: Show live transcript on a tablet during visit; note sections auto-populate

**Decision: Post-visit only** for Phase 1. Showing live transcript during the visit changes clinical behavior and introduces distraction. In Phase 2, consider a "heads-up" ambient mode showing a live note on a second screen.

---

## 11. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Audio recording fails mid-visit | No note | Edge agent writes to local buffer; alert doctor immediately; offer dictation fallback |
| ASR produces garbage (heavy accent, loud background) | Unusable transcript | Confidence thresholds: if < 70% of segments are high confidence, alert doctor and offer manual dictation |
| LLM hallucinates clinical content | Incorrect legal record | Grounding constraint + mandatory review for any sentence not traceable to transcript |
| ICD-10 code is wrong | Billing rejection or fraud risk | Codes always require explicit doctor acceptance; confidence threshold displayed |
| EHR push fails | Note not in patient record | Note stored locally with signed status; retry with backoff; admin alert after 3 failures |
| Doctor signs note with low-confidence items | Incorrect record | Warning modal: "X items were flagged and accepted without editing. Continue?" |
| Network outage during processing | Doctor waiting indefinitely | Show progress bar; after 5 min offer "partial note" with transcript only |
| Diarization fails (single channel audio) | Can't separate doctor/patient speech | Fall back to combined transcript; flag all section attributions as unverified |
| HIPAA breach (audio accessed unauthorized) | Legal/regulatory | Audit logs + real-time anomaly detection; per-record encryption keys |

---

## 12. Scaling Considerations

### Per-Clinic Model

Each clinic gets:
- A provider voice profile (enrolled once at setup)
- A clinic-specific vocabulary booster (specialty-specific terminology)
- A quality feedback loop — corrections made by providers in that clinic fine-tune the section population prompts over time

### Multi-Clinic Scale

| Component | Scaling approach |
|---|---|
| Audio ingestion | S3 + Lambda fan-out, elastic |
| ASR | GPU cluster auto-scales with queue depth |
| LLM extraction | Claude API (rate limit managed per clinic) |
| Note storage | PostgreSQL per clinic, sharded by clinic_id |
| Review interface | CDN-served SPA, WebSocket for live progress |
| EHR push | Per-EHR worker queues with dead-letter |

### Quality Metrics to Track

| Metric | Target | Alert threshold |
|---|---|---|
| Note generation time (P95) | < 180s | > 240s |
| Doctor correction rate per section | < 15% | > 30% |
| Code rejection rate | < 10% | > 20% |
| Note sign time (doctor review duration) | < 3 min | > 5 min |
| PHI breach incidents | 0 | Any |
| ASR confidence (mean) | > 0.92 | < 0.85 |

---

## Appendix A: Progress Note Section Format Examples

### HPI Example (system-generated)

> John Smith is a 61-year-old male with a history of hypertension who presents with a 3-day history of sharp, left-sided chest pain rated 7/10, worsening with inspiration and movement. He denies fever, productive cough, or hemoptysis. He endorses mild shortness of breath on exertion. Symptoms began acutely without a precipitating event. He has not taken anything for the pain.

### Assessment Example

> 1. Left-sided pleuritic chest pain — most likely pleuritis vs. early pneumonia; PE less likely given absence of risk factors but cannot be excluded.
> 2. Hypertension — blood pressure elevated at 142/88 today; currently on lisinopril 10mg daily.

### Plan Example

> 1. **Pleuritic chest pain**: D-dimer ordered; chest X-ray in radiology today; empiric NSAIDs (ibuprofen 600mg TID with food × 5 days); return if worsens or develops fever > 101.5°F.
> 2. **Hypertension**: Increase lisinopril to 20mg daily; recheck BP in 4 weeks.

---

## Appendix B: ICD-10 / CPT Code Examples

| Clinical finding | ICD-10 code | Code description |
|---|---|---|
| Pleurisy | R09.1 | Pleurisy |
| Pleuritic chest pain (undiagnosed) | R07.81 | Pleurodynia |
| Essential hypertension | I10 | Essential (primary) hypertension |
| New patient office visit, high complexity | 99205 | Office or other outpatient visit, new patient |
| Established patient, moderate complexity | 99214 | Office or other outpatient visit, established patient |
| EKG with interpretation | 93000 | Electrocardiogram routine ECG |

---

*Document version: 1.0 | Independence OS — Internal Design Document*

---

## Appendix C: Eraser.io Diagram Code

> **How to use**: In your Eraser.io document, click `+` → **Diagram** → switch to **Code** tab → select all existing text → paste the block below exactly as-is.

```
title AI Medical Scribe System Flowchart

direction right

// ── CAPTURE (Blue) ────────────────────────────────────────────────
Microphone [shape: oval, color: blue, icon: mic]
Consent Gate [color: blue, icon: shield]
Local Edge Agent [color: blue, icon: cpu] {
  Noise Suppression [color: blue, icon: volume-2]
  Voice Activity Detection [color: blue, icon: activity]
  "AES-256 Encryption" [color: blue, icon: lock]
  Offline Buffer [color: blue, icon: hard-drive]
}
Audio Intake and Validation [color: blue, icon: check-circle]

// ── AUDIO ERROR PATH (Red) ────────────────────────────────────────
// Isolated — only re-enters at Transcript Post-Processing
// Never connects to coding, review, or output nodes
Invalid Audio [color: red, icon: alert-triangle]
Alert Doctor [color: red, icon: alert-octagon]
Dictation Fallback [color: red, icon: edit-3]

// ── TRANSCRIPTION (Green) ─────────────────────────────────────────
Transcription [color: green, icon: file-text] {
  Speaker Diarization [color: green, icon: users]
  Medical ASR Engine [color: green, icon: cpu]
  "Transcript Post-Processing" [color: green, icon: filter]
  Clean Labelled Transcript [color: green, icon: file-check]
}

// ── CLINICAL EXTRACTION (Purple) ──────────────────────────────────
Clinical Extraction [color: purple, icon: activity] {
  Entity Extraction [color: purple, icon: search]
  Section Population [color: purple, icon: layout]
  Consistency Check [color: purple, icon: check-square]
}

// ── CODING ENGINE (Orange) ────────────────────────────────────────
Coding Engine [color: orange, icon: codesandbox] {
  "ICD-10 Suggester" [color: orange, icon: file-plus]
  CPT Suggester [color: orange, icon: file-plus]
  Progress Note Assembler [color: orange, icon: file-text]
}

// ── DOCTOR REVIEW (Yellow) ────────────────────────────────────────
Draft Note [color: yellow, icon: file-text]
Doctor Review Interface [color: yellow, icon: user-check]
Flagged Items Unresolved [color: yellow, icon: alert-circle]
Doctor Signs [shape: oval, color: yellow, icon: check-circle]

// ── OUTPUT — only reachable after Doctor Signs (Cyan) ─────────────
Signed Note [color: cyan, icon: file-check]
EHR Integration [color: cyan, icon: server]
Epic [color: cyan, icon: file-text]
Athena [color: cyan, icon: file-text]
Cerner [color: cyan, icon: file-text]
Billing Submission [color: cyan, icon: dollar-sign]
Insurance Claims [color: cyan, icon: credit-card]

// ── OUTPUT FAILURE QUEUES (Red) ───────────────────────────────────
// Completely separate from the audio-error Alert Doctor node above
EHR Dead Letter Queue [color: red, icon: inbox]
Billing Dead Letter Queue [color: red, icon: inbox]

// ══ RELATIONSHIPS ═════════════════════════════════════════════════

// CAPTURE
Microphone > Consent Gate
Consent Gate > Local Edge Agent
Local Edge Agent > Audio Intake and Validation

// AUDIO ERROR PATH
// Dead end — loops back into Transcript Post-Processing only
// No path from here can reach output nodes
Audio Intake and Validation > Invalid Audio: invalid [color: red, style: dashed]
Invalid Audio > Alert Doctor: [color: red, style: dashed]
Alert Doctor > Dictation Fallback: [color: red, style: dashed]
Dictation Fallback > "Transcript Post-Processing": skips ASR + Diarization [color: red, style: dashed]

// TRANSCRIPTION — valid audio, parallel diarization + ASR
Audio Intake and Validation > Speaker Diarization: valid audio
Audio Intake and Validation > Medical ASR Engine: valid audio
Speaker Diarization > "Transcript Post-Processing"
Medical ASR Engine > "Transcript Post-Processing"
"Transcript Post-Processing" > Clean Labelled Transcript

// CLINICAL EXTRACTION — sequential passes
Clean Labelled Transcript > Entity Extraction
Entity Extraction > Section Population
Section Population > Consistency Check

// CODING ENGINE
// Consistency Check connects explicitly to both children (parallel)
// so ICD-10 and CPT run simultaneously, both feed assembler
Consistency Check > "ICD-10 Suggester"
Consistency Check > CPT Suggester
"ICD-10 Suggester" > Progress Note Assembler
CPT Suggester > Progress Note Assembler
Progress Note Assembler > Draft Note

// DOCTOR REVIEW
// Loopback uses > so arrow goes FROM Flagged Items BACK TO Doctor Review
Draft Note > Doctor Review Interface
Doctor Review Interface > Flagged Items Unresolved: Flags unresolved
Flagged Items Unresolved > Doctor Review Interface: Loop back
Doctor Review Interface > Doctor Signs: All flags resolved

// OUTPUT — sole entry point is Doctor Signs
// No upstream node bypasses this gate
Doctor Signs > Signed Note
Signed Note > EHR Integration
Signed Note > Billing Submission
EHR Integration > Epic
EHR Integration > Athena
EHR Integration > Cerner
Billing Submission > Insurance Claims

// OUTPUT FAILURE PATHS
// Use their own dead-letter nodes — zero connection to Alert Doctor
EHR Integration > EHR Dead Letter Queue: Delivery failed after 3 retries [color: red, style: dashed]
Billing Submission > Billing Dead Letter Queue: Submission failed after 3 retries [color: red, style: dashed]

legend {
  [connection: ">", color: black, label: "Primary process flow"]
  [connection: ">", color: red, label: "Error or escalation path"]
  [connection: ">", color: cyan, label: "Output and integration flow"]
  [connection: ">", color: green, label: "Transcription parallel flow"]
  [connection: ">", color: orange, label: "Coding parallel flow"]
  [connection: ">", color: purple, label: "Clinical extraction flow"]
  [connection: ">", color: yellow, label: "Review process flow"]
  [connection: ">", style: dashed, color: red, label: "Error fallback or escalation"]
  [shape: oval, label: "Start / End state"]
  [shape: rectangle, label: "Process or system component"]
}
```