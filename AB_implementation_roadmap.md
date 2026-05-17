# A+B Implementation Roadmap
## Address Normalization for the Jakarta Region using Small Language Models

> **Strategy:** Option A (Two-Error-Mode Reframing) + Option B (Per-Field Uncertainty Quantification)  
> **Starting point:** Thesis writing first — lock the academic argument before touching code.  
> **Option C** (constrained generative decoding) is architecturally compatible and can slot in after B without rework.

---

## Overview of All Phases

| Phase | Domain | Key Output | Blocks |
|-------|--------|-----------|--------|
| 1 | Thesis writing | Rewritten RQs, contribution claims, chapter outline | All subsequent phases |
| 2 | System design | AdminVerifier spec, field-state schema, ablation plan | Phases 3–6 |
| 3 | Data pipeline | Clean masters, fixed training set, audit report | Phase 4, 5 |
| 4 | SFT training | Fine-tuned Llama-3-8B checkpoint | Phase 6 |
| 5 | AdminVerifier + Option B layer | Verified/Inferred/Unverifiable state assignment | Phase 6 |
| 6 | Inference pipeline | End-to-end normalization system | Phase 7 |
| 7 | Evaluation | Ablation results, per-field metrics, real-world test | Phase 8 |
| 8 | Final thesis writing | Complete, submitted manuscript | — |

---

## Phase 1 — Thesis Reframing (Start Here)

**Goal:** Restructure the entire academic argument around the two-error-mode framework and null-honesty principle *before* writing a single line of code. The novelty lives in the framing, not the architecture.

### 1.1 Rewrite the Problem Statement

**Current framing (vulnerable):** "We normalize Jakarta addresses using a fine-tuned LLM + database lookup."  
**Target framing:** "Jakarta address normalization presents a *compound uncertainty problem*: simultaneous syntactic chaos in informal text AND semantic incompleteness in the authoritative geographic database. Existing systems treat these as one problem; we treat them as two structurally distinct failure modes requiring different resolution strategies."

**Key paragraph to draft:**
> Address normalization in developing-world megacities like Jakarta faces a dual challenge absent from prior work. First, *syntactic chaos*: addresses are written in highly informal, multilingual, abbreviation-heavy text that defies rule-based parsing. Second, *semantic incompleteness* (or *rumpang* data): the authoritative administrative database itself is incomplete — 30.1% of registered streets lack kelurahan mappings, and 266 landmark entries carry no wilayah anchor at all. A system that assumes a complete ground truth will silently hallucinate plausible-sounding but incorrect geographic fields when the ground truth is absent. We formalize this as the *null-honesty principle* and design our pipeline accordingly.

**Action items:**
- [ ] Draft 2–3 paragraphs for Section 1.1 (Problem Background) around the compound uncertainty framing
- [ ] Replace any language that positions the contribution as "we use an LLM" with language that positions it as "we identify a new failure taxonomy and design for it"
- [ ] Introduce "rumpang" (Indonesian term for incomplete geographic reference data) as a formal concept in Chapter 1 — this is local color that differentiates the work from Chinese/English address papers

---

### 1.2 Rewrite the Research Questions

**Current RQs (typical, weak):** Likely something like "Can an SLM normalize Jakarta addresses?" or "How does fine-tuning improve address parsing?"

**Target RQs (specific, defensible):**

**RQ1:** What are the distinct failure modes in Jakarta address normalization, and how do syntactic and semantic sources of error differ in their frequency, distribution across address fields, and amenability to resolution?
- *Why this works:* Forces an error taxonomy contribution; no prior work on Jakarta or Indonesian addresses has done this.

**RQ2:** Can a small language model (≤8B parameters) reliably extract a structured 12-field representation from informal Jakarta address text, and what is the marginal gain attributable to domain-specific fine-tuning over a general-purpose baseline?
- *Why this works:* Directly answers the "why not GPT-4?" question preemptively; ties to StructuredRAG finding (71.7% base accuracy).

**RQ3:** How does deterministic administrative verification (AdminVerifier) change normalization accuracy relative to SLM-only output, and how does the distribution of per-field confidence states (Verified / Inferred / Unverifiable) characterize the epistemic limits of the Jakarta administrative database?
- *Why this works:* This IS Option B's contribution — the confidence state distribution is itself a research finding about Jakarta's data infrastructure.

**RQ4 (optional, strengthens originality):** What proportion of Jakarta addresses are structurally unresolvable given the current Pergub street database and BPS administrative hierarchy, and what does this imply for systems that require complete normalization as a precondition for downstream use?
- *Why this works:* Turns the data quality limitation into a *finding* rather than a *limitation*. Reframes "our DB is incomplete" into "we measured how incomplete it is, and the implications are significant."

**Action items:**
- [ ] Align RQs with your institution's thesis format requirements (some require exactly 3 RQs)
- [ ] Map each RQ to a specific section in Chapter 3 (Methodology) and Chapter 4 (Results)
- [ ] Ensure RQ3 language matches the exact field-state vocabulary you'll use in the code

---

### 1.3 Rewrite the Contribution Claims

Replace vague architectural claims with falsifiable, measurable contributions. This is what your examiner will hold you to.

**Contribution 1 — Error Taxonomy (conceptual):**
> We present the first formal error taxonomy for Indonesian address normalization, distinguishing *syntactic extraction failures* (recoverable by domain-adapted SLM) from *semantic verification failures* (caused by rumpang data in the administrative database). We quantify the relative prevalence of each failure mode on a real Jakarta address corpus.

**Contribution 2 — Hybrid Pipeline with Null-Honesty (system):**
> We design and implement a hybrid normalization pipeline combining supervised fine-tuning of Llama-3-8B with a deterministic AdminVerifier module. The pipeline embodies the null-honesty principle: fields that cannot be verified against the authoritative database are returned as null rather than hallucinated, with an explicit confidence state (Verified / Inferred / Unverifiable) assigned per field.

**Contribution 3 — Per-Field Uncertainty Quantification (Option B, measurable):**
> We introduce a per-field confidence state schema (4 states: Verified, Inferred, Unverifiable, Free-form) and report the distribution of these states across a Jakarta test set. This distribution constitutes an empirical measurement of the epistemic completeness of Jakarta's administrative reference data — a finding with implications beyond address normalization.

**Contribution 4 — Dataset and Benchmarks (practical):**
> We construct and release [if you plan to release]: (a) a 231k-sample synthetic SFT training dataset for Indonesian address normalization; (b) a manually curated evaluation set of [N] real Jakarta addresses with gold-standard 12-field annotations; (c) a cleaned and geocoding-enriched version of the Jakarta street and landmark master databases.

**Action items:**
- [ ] Decide now whether you will release datasets — this affects how Contribution 4 is worded and requires ethics/data provenance consideration
- [ ] For each contribution, identify the table/figure in Chapter 4 that provides the evidence — if you can't name it yet, the contribution isn't concrete enough

---

### 1.4 Rewrite the Related Work Positioning

This is where you neutralize the critical threat papers. For each, the strategy is to acknowledge → narrow the gap → assert the residual difference.

**AddrLLM (Yang et al., 2024) — nearest structural analog:**
> AddrLLM applies SFT + semantic retrieval to Chinese address rewriting with complete administrative databases. Our work extends this paradigm to the linguistically distinct Indonesian context and, critically, addresses a structural problem AddrLLM does not encounter: an authoritative administrative database that is itself incomplete. We operationalize this distinction through the null-honesty principle and per-field confidence states — a design choice orthogonal to AddrLLM's semantic retrieval strategy.

**Hu et al. (IJGIS 2024) — fine-tuned LLM + external geocoder:**
> Hu et al. demonstrate that domain-adapted lightweight LLMs outperform base models on address parsing when combined with external geocoder validation. We confirm this finding in the Indonesian context and additionally characterize the behavior of the validation stage when the external geocoder (administrative database) has partial coverage — a regime Hu et al. do not evaluate.

**ReaGeo (April 2026) — most dangerous, argues multi-stage is obsolete:**
> ReaGeo proposes end-to-end geocoding via chain-of-thought + RL, positioning multi-stage pipelines as unnecessarily complex. We distinguish our task: ReaGeo targets coordinate-level geocoding; our system targets structured field normalization per SNI 9037:2021, which requires per-field verifiability for downstream administrative use. An end-to-end system producing lat/long cannot satisfy the auditability requirements of the target application domain.

**Action items:**
- [ ] Write 1 paragraph per critical threat paper in the Related Work section using the acknowledge→narrow→assert structure
- [ ] Do NOT write these as "X does Y, we do Z" — examiners see through parallelism. Write them as genuine scholarly engagement.
- [ ] Add ReaGeo to your reference list (April 2026, arXiv 2604.21357) — it is newer than your folder

---

### 1.5 Revise the Chapter Outline

**Proposed chapter structure aligned with A+B:**

```
Chapter 1: Introduction
  1.1 Problem Background — compound uncertainty framing, rumpang concept
  1.2 Research Questions (RQ1–RQ3, optionally RQ4)
  1.3 Scope and Limitations
  1.4 Contributions
  1.5 Thesis Structure

Chapter 2: Literature Review
  2.1 Address Normalization — from rule-based to neural
  2.2 LLMs for Structured Information Extraction
  2.3 Retrieval-Augmented Generation for Geographic Resolution
  2.4 Indonesian NLP — unique challenges (code-switching, abbreviation density)
  2.5 Administrative Data Quality in Developing Contexts
  2.6 Positioning this Work [the neutralization section — RW paragraphs from 1.4 above]

Chapter 3: Methodology
  3.1 System Overview — two-stage pipeline diagram
  3.2 Error Taxonomy (RQ1) — define syntactic vs. semantic failure modes
  3.3 Dataset Curation
    3.3.1 Administrative master database (full2.csv)
    3.3.2 Street master (USED_master_data_final.csv) — document null rates
    3.3.3 Complexes/landmarks master — document anchor-less entries
    3.3.4 SFT training data construction — document augmentation logic, nomor fix
  3.4 Stage 1: SFT Fine-Tuning
    3.4.1 Base model selection rationale (Llama-3-8B vs. alternatives)
    3.4.2 LoRA/QLoRA configuration
    3.4.3 Training/validation split design (prevent augmentation leakage)
    3.4.4 Output schema — 12-field JSON
  3.5 Stage 2: AdminVerifier (Option B)
    3.5.1 Field-state schema — Verified / Inferred / Unverifiable / Free-form
    3.5.2 Verification logic — exact match, two-hop chain, null-honesty rule
    3.5.3 KODEPOS dtype handling
    3.5.4 Anchor-less complex handling policy

Chapter 4: Results and Analysis
  4.1 Baseline Comparisons (ablation baselines — see Phase 2)
  4.2 Field-Level Accuracy — per-field breakdown (RQ2)
  4.3 AdminVerifier Impact — accuracy delta pre/post verification (RQ2, RQ3)
  4.4 Confidence State Distribution (RQ3) — what fraction of fields are Verified vs. Unverifiable?
  4.5 Error Analysis — taxonomy distribution on test set (RQ1 answered empirically)
  4.6 Real-World Evaluation — hand-labeled test cases from actual Jakarta addresses

Chapter 5: Discussion
  5.1 Two-Error-Mode Framework — does the taxonomy hold empirically?
  5.2 Null-Honesty vs. Hallucination — quantified comparison
  5.3 Rumpang Data — implications for address normalization systems in data-scarce contexts
  5.4 Limitations

Chapter 6: Conclusion
  6.1 Summary of Contributions
  6.2 Future Work — Option C (constrained generative normalization)
```

**Action items:**
- [ ] Verify this structure matches your institution's thesis format requirements
- [ ] Check if Section 2.5 (Administrative Data Quality) has adequate literature — this is a weaker area for existing papers and you may need to cite urban informatics / GIS literature
- [ ] Confirm Section 3.3.4 will be explicit about the nomor null bug fix — document it as a design decision, not a bug

---

### Phase 1 Deliverables Checklist

Before moving to Phase 2, you should have:
- [ ] Revised abstract (1 paragraph, covers compound uncertainty + null-honesty + per-field states)
- [ ] Final RQs (3–4, locked, mapped to results sections)
- [ ] Contribution claims (4 claims, each with a named result table/figure as evidence)
- [ ] Related work draft paragraphs for all 4 critical threat papers
- [ ] Chapter outline approved (or adapted to institution format)

---

## Phase 2 — System Architecture Design

**Goal:** Fully specify the AdminVerifier module, the field-state schema, and the ablation baseline plan on paper — before writing code. Design decisions made here propagate into Phases 4, 5, and 6.

### 2.1 AdminVerifier Module Spec

Build this as a standalone class — not wired to the SLM, not tied to the inference loop. This is the C-compatibility requirement.

```
AdminVerifier
  Input:  raw SLM output dict (12 fields, some null, some strings)
  Output: annotated dict — each field gains a `_state` key

  Internal databases:
    - full2.csv: 267 rows (after Maluku row removed), indexed by Kelurahan
    - USED_master_data_final.csv: 4,190 rows (after 2 null NAMA_JALAN rows removed)
    - USED_master_complexes_final_v2.csv: 2,718 rows (anchor-less handling TBD)

  Verification chain (for street addresses):
    1. Lookup nama_jalan in street master (fuzzy, threshold TBD)
    2. If match found → read KELURAHAN_REF, KODEPOS_REF (cast from float to int)
    3. Cross-ref KELURAHAN_REF against full2.csv → get Kecamatan, KodePos
    4. Assign states per field (see 2.2)

  Verification chain (for complex/landmark addresses):
    1. Lookup nama_kompleks_atau_gedung in complexes master (fuzzy)
    2. Get KELURAHAN_REF from complexes master
    3. Cross-ref full2.csv → infer Kecamatan, KodePos
    (Note: KECAMATAN_REF in complexes master is 99.4% null — do not rely on it)

  Null-honesty rule:
    If a geographic field (kelurahan, kecamatan, kodepos) cannot be verified
    through any chain → output null, state = Unverifiable
    NEVER substitute the SLM's proposed value without verification
```

**Design decision — fuzzy match threshold:**
- For nama_jalan: recommend RapidFuzz token_sort_ratio ≥ 80 as starting point
- For kompleks: recommend ≥ 75 (more name variation in landmark data)
- Document these as hyperparameters in the thesis; evaluate sensitivity in Chapter 4

**Design decision — anchor-less complexes (266 entries with no wilayah):**
- Option (i): Exclude from complexes master entirely → safe, reduces coverage
- Option (ii): Include, but return Unverifiable for all geographic fields → honest, preserves any non-geographic fields (e.g., building name normalization)
- **Recommendation:** Option (ii) — the null-honesty principle handles this cleanly, and you get to report "X% of complex addresses resolved to Unverifiable due to anchor-less entries" as a finding

### 2.2 Field-State Schema (Option B)

Four states, assigned per output field:

| State | Meaning | Example |
|-------|---------|---------|
| `Verified` | Field value confirmed in full2.csv or street/complex master | kelurahan = "Menteng" → found in full2.csv |
| `Inferred` | Field derived via two-hop chain (not directly in SLM output or directly matched) | kecamatan derived from KELURAHAN_REF → full2.csv join |
| `Unverifiable` | Field cannot be confirmed or derived — DB gap | nama_jalan not in street master, no KELURAHAN_REF available |
| `Free-form` | Field has no geographic constraint (free text by nature) | nama_kompleks_atau_gedung, detail_unit, blok_kavling, nomor |

**Implementation note:** Store states in a parallel dict or as field suffixes:
```python
{
  "kelurahan": "Menteng",
  "kelurahan_state": "Verified",
  "kecamatan": "Menteng",
  "kecamatan_state": "Inferred",
  "kodepos": null,
  "kodepos_state": "Unverifiable",
  ...
}
```

### 2.3 Ablation Baseline Plan

You need at minimum 3 baselines to support RQ2 and RQ3:

| Baseline | What it tests | How to implement |
|---------|--------------|-----------------|
| B0 — Base Llama-3-8B (no SFT) | Marginal gain from fine-tuning | Run inference with base model, same prompt |
| B1 — SFT only (no AdminVerifier) | Marginal gain from AdminVerifier | Run fine-tuned model, skip Stage 2 |
| B2 — Rule-based parser | Shows SLM is needed at all | Regex/heuristic extractor (simple, not competitive) |
| **Full system (A+B)** | The proposed approach | SFT + AdminVerifier with confidence states |

**B2 note:** A rule-based baseline does NOT need to be competitive. Its purpose is to show that informal Jakarta address text defeats naive approaches, motivating the SLM. Even 40% accuracy on B2 vs. 85%+ on the full system makes the point.

### Phase 2 Deliverables Checklist
- [ ] AdminVerifier class interface fully specified (inputs, outputs, state logic, fuzzy threshold rationale)
- [ ] Field-state schema documented (4 states, definitions, assignment rules)
- [ ] Anchor-less complex policy decided and documented as a design choice
- [ ] Ablation baseline plan written into Section 3.1 of thesis draft

---

## Phase 3 — Data Pipeline Fixes

**Goal:** Clean all three master databases and fix the training data before any model training begins. These are not minor issues — the nomor bug will teach the model to hallucinate.

### 3.1 Critical Fix: nomor Null Bug

**Problem:** Training data has 0% null nomor values across 231k rows. The address augmentation logic is forcing a placeholder house number even when the source address text contains none.

**Impact:** The model will learn `P(nomor ≠ null | any address input) ≈ 1.0`. On real addresses without a house number (common in Jakarta informal addresses), it will hallucinate a plausible number. This directly violates the null-honesty principle.

**Fix:**
1. Audit the augmentation script — find where nomor is assigned and check the null-assignment branch
2. Estimate the true base rate of null nomor in real Jakarta addresses (likely 20–40%)
3. Regenerate affected training samples with corrected null assignment
4. Verify fix: after regeneration, null rate in nomor should be ≥ 15%
5. Document both the bug and the fix in Section 3.3.4

**This is the highest-priority data task. Do not begin SFT until this is fixed.**

### 3.2 Administrative Master Fixes

**full2.csv:**
- [ ] Remove the 1 stray "Kabupaten Seram Bagian Timur" row (Maluku) — this is not Jakarta data and will corrupt AdminVerifier lookups

**USED_master_data_final.csv (streets):**
- [ ] Drop 2 rows with null NAMA_JALAN — they cannot be matched
- [ ] Cast KODEPOS_REF from float64 to nullable Int64 (preserve nulls, compare correctly with full2.csv KodePos integer strings)
- [ ] Document null rates in thesis: 30.1% null KELURAHAN_REF, 34.2% null KODEPOS_REF — these are findings, not just limitations

**USED_master_complexes_final_v2.csv:**
- [ ] Implement the anchor-less complex policy decided in Phase 2
- [ ] Cast any KODEPOS_REF columns as above

### 3.3 Training Data Validation

After fixing the nomor bug:
- [ ] Recheck all 12 fields for anomalous null rates (0% or 100% anywhere unexpected)
- [ ] Verify augmentation uniqueness: sample 100 records sharing the same base address — confirm no near-duplicates leak across train/val split (the ~33x multiplier makes this a real risk)
- [ ] Implement stratified split by base address, not by row index, to prevent leakage

### Phase 3 Deliverables Checklist
- [ ] Cleaned full2.csv (267 rows, no Maluku entry)
- [ ] Cleaned streets master (4,190 rows, no null NAMA_JALAN, correct KODEPOS_REF dtype)
- [ ] Complexes master with anchor-less policy applied
- [ ] Regenerated training dataset with corrected nomor null distribution
- [ ] Data audit report (can be a notebook) documenting before/after statistics — this becomes Section 3.3 of the thesis

---

## Phase 4 — SFT Training Setup

**Goal:** Fine-tune Llama-3-8B on the cleaned 231k training set. Focus on correctness of the data pipeline and evaluation setup; hyperparameter tuning is secondary at thesis scale.

### 4.1 Base Model Selection Rationale

Lock this reasoning into Section 3.4.1 now:
- **Llama-3-8B vs. Llama-3.1-8B:** Use 3.1 — longer context window (128k), same parameter count, strictly better
- **Why not Mistral-7B?** Hu et al. (2024) use Mistral-7B; using Llama-3 differentiates while remaining in the same compute tier
- **Why not IndoBERT?** Encoder-only, cannot generate structured JSON; extraction task requires generation
- **Why not GPT-4/Claude?** (i) Cost prohibitive at 231k samples; (ii) inference latency incompatible with deployment context; (iii) no fine-tuning available for GPT-4; (iv) thesis requires demonstrating a *small* LM can solve the task

### 4.2 LoRA Configuration (Starting Point)

```python
LoraConfig(
    r=16,                          # rank — start here, ablate to r=8 if VRAM-constrained
    lora_alpha=32,                 # alpha = 2*r is standard starting point
    target_modules=["q_proj", "v_proj"],  # attention layers only
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)
```

**Training configuration:**
- Batch size: 4 per GPU (gradient accumulate to effective 16–32)
- Learning rate: 2e-4 with cosine schedule
- Epochs: 3 (monitor val loss; stop if plateau)
- Sequence length: 512 tokens max (addresses are short; pad to 512)
- Precision: bfloat16 (QLoRA: 4-bit NF4 quantization for base weights)

### 4.3 Prompt Template

Lock the prompt template before training — it must match the inference template exactly.

```
[INST] Normalize the following Jakarta address into a structured JSON format.

Address: {raw_address_text}

Output a JSON object with exactly these fields: nama_jalan, nama_kompleks_atau_gedung, blok_kavling, nomor, rt, rw, kelurahan, kecamatan, kota_kabupaten, provinsi, kodepos, detail_unit.

Use null for any field that cannot be determined from the address text. [/INST]

{json_output}
```

**Note:** The null instruction is critical — this is where the null-honesty principle gets baked into the model's behavior at training time.

### 4.4 Validation Split Design

- Split by unique base address, not by row — the ~33x augmentation multiplier means random row-level splits will leak augmented variants of the same address into both train and val
- Target: 90/10 train/val by base address count
- Hold out a separate test set from real addresses (not synthetic) for final evaluation — this becomes the "real-world evaluation" in Section 4.6

### Phase 4 Deliverables Checklist
- [ ] Base model selection rationale documented (Section 3.4.1)
- [ ] LoRA config documented (Section 3.4.2)
- [ ] Prompt template locked and documented (Section 3.4.3)
- [ ] Training/val/test split documented with leakage-prevention rationale (Section 3.4.3)
- [ ] Trained checkpoint saved with config file

---

## Phase 5 — AdminVerifier Implementation + Option B Layer

**Goal:** Implement the AdminVerifier class as specified in Phase 2. Keep it modular. Test it independently of the SFT model.

### 5.1 Implementation Checklist

```python
class AdminVerifier:
    def __init__(self, full2_path, streets_path, complexes_path, fuzzy_threshold=80):
        # Load and index all three masters at init time
        # full2: index by Kelurahan (lowercase normalized)
        # streets: index by NAMA_JALAN
        # complexes: index by complex name
        pass

    def verify(self, slm_output: dict) -> dict:
        # Returns same 12 fields + _state suffix for each
        pass

    def _verify_street_address(self, slm_output):
        # Step 1: fuzzy match nama_jalan → street master
        # Step 2: get KELURAHAN_REF, KODEPOS_REF
        # Step 3: cross-ref full2.csv → Kecamatan
        # Step 4: assign states
        pass

    def _verify_complex_address(self, slm_output):
        # Step 1: fuzzy match nama_kompleks → complexes master
        # Step 2: get KELURAHAN_REF
        # Step 3: cross-ref full2.csv → two-hop resolution
        # Step 4: assign states
        pass
```

**Key implementation details:**
- Lowercase normalize all string comparisons before lookup — "JALAN SUDIRMAN" vs "Jalan Sudirman" must match
- Strip common Indonesian street prefixes ("Jl.", "Jln.", "Jalan") before fuzzy matching — these vary wildly in informal text
- KODEPOS dtype: `int(float(kodepos_ref_value))` pattern; handle pd.NA explicitly
- For RapidFuzz, `process.extractOne(query, choices, scorer=fuzz.token_sort_ratio)` is more robust than `ratio` for address strings (handles word-order variation)

### 5.2 Unit Tests (Minimum Required for Thesis Credibility)

Write these before the full inference pipeline. Test the AdminVerifier in isolation.

- Test 1: Known kelurahan in full2.csv → state = Verified, correct kecamatan returned
- Test 2: Kelurahan not in full2.csv → state = Unverifiable, null returned
- Test 3: KODEPOS_REF float → correct int cast
- Test 4: Anchor-less complex → Unverifiable states for all geographic fields
- Test 5: Two-hop chain resolution (complex → kelurahan → kecamatan)
- Test 6: Stray Maluku row removed (assert it is not present in loaded DB)

### Phase 5 Deliverables Checklist
- [ ] AdminVerifier class implemented and passing unit tests
- [ ] Field-state assignment logic matches the schema from Phase 2 exactly
- [ ] Fuzzy threshold documented and justified (will become a hyperparameter table in the thesis)
- [ ] KODEPOS dtype handling tested explicitly

---

## Phase 6 — Inference Pipeline Integration

**Goal:** Wire Stage 1 (SFT model) and Stage 2 (AdminVerifier) together into a single end-to-end normalization function. This should be thin glue code — the heavy lifting is already done in Phases 4 and 5.

### 6.1 Pipeline Interface

```python
def normalize_address(raw_text: str, model, tokenizer, verifier: AdminVerifier) -> dict:
    # Stage 1: SFT inference
    prompt = build_prompt(raw_text)
    raw_output = generate(model, tokenizer, prompt)
    slm_output = parse_json(raw_output)  # handle parse failures gracefully

    # Stage 2: AdminVerifier
    verified_output = verifier.verify(slm_output)

    return verified_output  # 12 fields + 12 state fields
```

**JSON parse failure handling:** The SLM will occasionally produce malformed JSON (~5–15% at inference time, depending on training quality). Define a fallback:
- Attempt `json.loads` first
- On failure, attempt regex extraction of field-value pairs
- On second failure, return all fields as null with state = "ParseFailure" (a fifth state)
- Log all parse failures — this is data for Section 4.5

### Phase 6 Deliverables Checklist
- [ ] End-to-end normalization function implemented
- [ ] Parse failure handling documented
- [ ] Parse failure rate logged and reported in Chapter 4

---

## Phase 7 — Evaluation Framework

**Goal:** Produce the numbers that answer RQ1–RQ3. The evaluation design must match the RQs exactly.

### 7.1 Metrics

**Per-field accuracy (answers RQ2):**
- Exact match accuracy per field across all 12 fields
- Aggregate accuracy: proportion of records where all verified fields match
- Report separately for: high-constraint fields (kelurahan, kecamatan, kodepos, provinsi) vs. free-form fields (nomor, detail_unit, blok_kavling)

**AdminVerifier impact (answers RQ2, RQ3):**
- Δ accuracy per field: (accuracy_after_verification − accuracy_SLM_only)
- This should show: high-constraint geographic fields improve substantially; free-form fields unchanged

**Confidence state distribution (answers RQ3):**
- For each geographic field: % Verified, % Inferred, % Unverifiable across the test set
- This IS a research finding — it characterizes the epistemic completeness of the Jakarta administrative database

**Error taxonomy distribution (answers RQ1):**
- For each error, classify as: Syntactic extraction failure (SLM failed to parse correctly) vs. Semantic verification failure (SLM extracted correctly, but AdminVerifier returned Unverifiable)
- Report relative prevalence — this validates the two-error-mode framework empirically

### 7.2 Test Sets

**Synthetic test set:** Held-out portion of the training distribution — measures SFT performance under controlled conditions.

**Real-world test set (critical for credibility):** Hand-labeled real Jakarta addresses. Target: 100–200 addresses, diverse across:
- Address types: street vs. complex/landmark
- Districts: spread across Jakarta's 5 kotamadya
- Formality: formal (complete) vs. informal (abbreviated, multilingual, missing fields)

**Collection suggestion:** Jakarta municipal open data portals, delivery service datasets (with permission), or manual collection from public listings.

### Phase 7 Deliverables Checklist
- [ ] Evaluation script implemented (per-field accuracy, state distribution, error taxonomy classification)
- [ ] Synthetic test results for all 4 baselines (B0, B1, B2, full system)
- [ ] Real-world test set collected and annotated (gold-standard labels)
- [ ] Real-world test results reported
- [ ] All result tables and figures generated (these go directly into Chapter 4)

---

## Phase 8 — Final Thesis Writing

**Goal:** Write Chapters 3, 4, and 5 using the results from Phases 4–7. Chapter 1 and 2 drafts should already exist from Phase 1.

**Chapter 3 (Methodology):** Should be largely draftable from Phase 2 design documents + Phase 3 audit report. Insert actual code snippets where appropriate.

**Chapter 4 (Results):** One section per RQ. Each section: table/figure → prose interpreting it → connection back to the RQ.

**Chapter 5 (Discussion):** The two-error-mode framework — does the data support it? What does the confidence state distribution tell us about rumpang data? What are the honest limitations?

**Abstract (write last):** ~250 words covering: problem (compound uncertainty), approach (SFT + AdminVerifier + null-honesty), contribution (error taxonomy + per-field states), key finding (X% of Jakarta addresses contain at least one Unverifiable geographic field).

---

## Option C — Future Extension (Constrained Generative Normalization)

If time permits after A+B are complete, Option C can be added without rework because the AdminVerifier is already modular.

**What it is:** Instead of generating free-form JSON and then verifying post-hoc, use a grammar-enforced decoding library (Outlines or lm-format-enforcer) to constrain the model's output at decoding time — only allowing kelurahan tokens that exist in full2.csv, for example.

**Why the AdminVerifier first approach sets this up cleanly:**
- AdminVerifier already has the lookup logic; Option C just moves it into the decoding loop
- The field-state schema already distinguishes "constrained" from "free-form" fields — these map directly to grammar-constrained vs. unconstrained slots in the decoding grammar
- You can report Option C as future work in Section 6.2 with a concrete architecture sketch

**Honest caveat:** Grammar-constrained decoding adds significant inference latency and may conflict with LoRA-adapted model behavior. It is a genuine research contribution but also a genuine implementation challenge. Do not promise it in the contributions unless you have implemented it.

---

## Quick Reference: Decision Log

These decisions should be documented in the thesis as explicit design choices, not buried in code:

| Decision | Choice | Rationale |
|---------|--------|-----------|
| Base model | Llama-3.1-8B | Differentiates from Hu et al. (Mistral), same compute tier |
| Fine-tuning method | LoRA/QLoRA | Parameter-efficient; necessary for academic compute constraints |
| Verification approach | Deterministic lookup (not probabilistic RAG) | Geographic hierarchy is fixed; probabilistic retrieval adds noise with no benefit |
| Fuzzy match library | RapidFuzz | token_sort_ratio handles word-order variation in Indonesian addresses |
| Anchor-less complex policy | Include, return Unverifiable | Null-honesty principle; preserves non-geographic normalization value |
| nomor null fix | Regenerate affected training samples | Prevents hallucination; documents augmentation design decision |
| Evaluation | Per-field accuracy + state distribution | Matches RQ structure; state distribution is a novel reporting contribution |
