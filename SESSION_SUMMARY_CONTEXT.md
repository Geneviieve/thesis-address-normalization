# THESIS CONTEXT SUMMARY
**Chat Session: May 20, 2026 | Dimas Aryaputra**

---

## 1. THESIS OVERVIEW

**Title:** "From Unstructured Text to Verified Geospatial Data: A Hybrid Small Language Model with Deterministic Administrative Grounding for the Jakarta Address Normalization Problem"

**Current Phase:** Foundational & planning stage (data review complete, no model training started yet)

**Core Problem:** 
- Jakarta addresses are highly informal, inconsistent, abbreviated
- Administrative reference data is incomplete ("rumpang data"): 30.1% null kelurahan, 34.2% null kodepos
- Standard address tools (ArcGIS) fail on Indonesian context (RT/RW micro-administrative layer)

**Goal:** Develop system that normalizes informal Jakarta addresses to SNI 9037:2021 standard with explicit honesty about data completeness

---

## 2. YOUR SYSTEM ARCHITECTURE

### Two-Stage Pipeline:

**Stage 1: Extraction (SLM)**
- Model: Llama-3.1-8B with QLoRA fine-tuning
- Input: Messy Jakarta address text
- Output: 12-field JSON (kelurahan, kecamatan, kota_kabupaten, provinsi, nama_jalan, nama_kompleks, blok_kavling, nomor, rt, rw, kodepos, detail_unit)
- Issue: Can hallucinate (30% of time estimated)

**Stage 2: Verification (AdminVerifier)**
- NOT RAG (deterministic indexed lookup, not vector retrieval)
- Input: SLM's 12-field JSON
- Output: Same JSON + 4-state confidence per field + corrections
- Process: Lookup each field against Jakarta admin masters
  - full2.csv (268 rows admin hierarchy - Provinsi to Kelurahan)
  - USED_master_data_final.csv (4,192 streets - 30% null kelurahan, 34% null kodepos)
  - USED_master_complexes_final_v2.csv (2,718 landmarks - 99% null kecamatan)
- Confidence States:
  - **Verified**: Field found exact match in DB
  - **Inferred**: Derived from other field lookup (e.g., kecamatan inferred from kelurahan hierarchy)
  - **Unverifiable**: Field not found in DB (honest rejection, no hallucination)
  - **Free-form**: Field doesn't apply (e.g., blok_kavling for street addresses)

**Key Design Principle: Null-Honesty**
- System explicitly reports missing data instead of inventing it
- Operationalized in TWO places:
  1. Training: Frozen prompt template forces model to see "use null for fields you can't determine" 184k times
  2. Inference: AdminVerifier outputs "Unverifiable" for missing fields

---

## 3. DATA PIPELINE

### 3.1 Training Dataset (training_dataset_v2.csv - 230k rows)
- **Generation Method:** Hybrid synthetic
  - Rule-based: Inject realistic admin gaps (30% kelurahan null, per real Jakarta data)
  - External LLM: Generate conversational prefixes, typos, abbreviations (syntactic noise ONLY)
  - NOT using LLM to generate location data (that would be hallucination)
- **Structure:** input_text (messy) → output_json (structured)
- **Engineering Philosophy:** Two-error-mode taxonomy
  - Syntactic errors (typos, abbreviations, formatting)
  - Semantic errors (missing/wrong admin hierarchy)

### 3.2 Data Preparation (slm_data_prep.ipynb)
- **Input:** training_dataset_v2.csv
- **Process:**
  - Classify: street vs complex vs generic addresses
  - Prevent leakage: identical JSON outputs grouped so no data duplication across splits
  - Split: 80% train, 10% val, 10% test (80/10/10)
  - Wrap with frozen Llama-3.1 prompt template (LOCKED, identical for training & inference)
- **Output:** train.jsonl, val.jsonl, test.jsonl, split_manifest.json
- **Why JSONL?**
  - Line-by-line processing (memory efficient for 230k rows)
  - Robust to malformed entries (one bad row ≠ parse failure)
  - Frozen template ensures consistency (training format = inference format)

### 3.3 Stage 2 Verification (admin_verifier_v2.ipynb)
- **Input:** SLM output JSON + Jakarta master data files
- **Process:**
  - Lookup each field against appropriate master
  - Hierarchical traversal (Kecamatan → Kelurahan → RT/RW)
  - Auto-correct if SLM wrong (e.g., kecamatan wrong but kelurahan correct → infer correct kecamatan)
  - Assign confidence state
- **Output:** Verified JSON + 4-state confidence + empirical distribution for thesis validation

---

## 4. YOUR 8 NOVELTY CLAIMS (N1-N8)

| # | Claim | Description |
|----|-------|-------------|
| N1 | Two-Error-Mode Framework | Decomposes address failures into (i) syntactic extraction errors (noisy input) vs (ii) semantic verification errors (data rumpang in DB). No competitor explicitly decomposes this. |
| N2 | Deterministic Admin Grounding | Post-hoc indexed exact-match lookup (NOT RAG, NOT vector retrieval). Eliminates retrieval failure + hallucination risk simultaneously. |
| N3 | 4-State Confidence Schema | {Verified, Inferred, Unverifiable, Free-form} as first-class output. Reports epistemic state per field. No competitor outputs this. |
| N4 | Null-Honesty Principle | System explicitly reports missing data (Unverifiable) instead of hallucinating. Operationalized in training (frozen template) + inference (AdminVerifier). |
| N5 | Data Rumpang as Design Constraint | 30% kelurahan null, 34% kodepos null explicitly measured & designed for. Most competitors assume complete reference data. |
| N6 | Hybrid Synthetic Corpus | Rule-based admin gap injection + external LLM (syntactic noise only). 230k samples engineered per two-error-mode taxonomy. Realistic gaps, not random. |
| N7 | Proof of Sufficiency | 8B SLM + deterministic grounding achieves competitive performance (ideally within 5-10% of GPT-4) at 1000x lower cost & privacy. |
| N8 | Indonesian-Specific | Jakarta RT/RW micro-administrative layer + SNI 9037:2021 conformance + Pergub-derived master data. Unique to Indonesian governance context. |

**Key Insight:** No single competitor has ALL 8 together. Each piece exists elsewhere; the **combination** designed for Jakarta's constraints is novel.

---

## 5. COMPETITIVE THREAT ANALYSIS

| Rank | Paper | Why Threatening | Your Defense |
|------|-------|-----------------|--------------|
| **CRITICAL** | **AddrLLM** (Yang et al. 2024) | SFT + retrieval for address normalization; nationwide scale (100M+ addresses) | Different constraints. You: deterministic + rumpang-aware for Jakarta. They: nationwide + flexible RAG. Both valid. Your 4-state schema is orthogonal. |
| **CRITICAL** | **AddrKG-LLM** (Li et al. 2024) | Two-stage pipeline (KG retrieval + constrained LLM → JSON); most similar structure | Their Top-K retrieval can fail if gold not ranked high. Your deterministic: found or Unverifiable. They assume complete KG; you model incomplete data. Your confidence schema unique. |
| **HIGH** | **GeoAgent** (Huang et al. 2024) | LLM + external tools for address standardization; similar problem framing | They optimize flexibility + external scale. You optimize privacy + offline + transparency. Different design philosophies. |
| **HIGH** | **SLOT** (Wang et al. 2025) | Post-hoc LLM post-processor for schema compliance (99.5% accuracy) | They fix JSON structure (Layer 0). You verify field correctness (Layer 1). Complementary, not competing. |
| **MODERATE** | **GeoIndia** (Singhal et al. 2024) | QLoRA Llama-3-8B for address processing; similar model + domain | Different task (geocoding vs normalization), different output (geohash vs JSON). You verify; they predict. Indonesian-specific RT/RW. |

---

## 6. ADVISOR'S VALIDATION CHECKLIST (From Consultation Prep Notes)

**Must have before final consultation:**

1. ✅ Real validation data: 200-500 manually labeled Jakarta addresses (NOT synthetic)
2. **Hallucination metrics:**
   - % SLM hallucinates
   - % AdminVerifier successfully rescues
3. **Schema validation:** Prove "Verified" fields more accurate than "Inferred" (confidence state must be meaningful)
4. **Two-error-mode proof:** Show architecture fixes semantic errors without breaking syntactic
5. **Stress test:** 50 adversarial cases (out-of-Jakarta, extreme typos, OOA terms, etc.)
6. **GPT-4 baseline:** 200 real addresses → GPT-4 zero-shot (target: within 5-10% accuracy at 1000x lower cost)
7. **Statistical rigor:** 95% CI, McNemar's test, 20-case failure analysis with explanation per case

---

## 7. IMPLEMENTATION DETAILS

### QLoRA vs Full Fine-Tuning
- **Q (Quantization):** Compresses model to fit on cheaper GPU
- **LoRA (Low-Rank Adaptation):** Adds learnable "cheat sheet" instead of modifying full weights
- **Cost:** ~Rp 10k/hour (vs millions for superkomputer)
- **Why relevant:** Enables economical training → supports Proof of Sufficiency claim

### SFT (Supervised Fine-Tuning)
- Training: Model sees 184k examples (input address → expected JSON output)
- Model learns pattern mapping: noisy text → structured fields
- Post-training: Weights frozen, model used for inference only
- **NOT retraining on DB updates:** SLM trained once; AdminVerifier handles DB updates

### Training vs Inference
- **Training (one-time, offline):** QLoRA fine-tune with train.jsonl (berjam-jam)
- **Inference (repeated, online):** Load checkpoint, forward pass for each new address (milliseconds)

---

## 8. KEY INSIGHTS & TALKING POINTS

**For Consultation Defense:**

1. **Data Rumpang is Feature, Not Limitation**
   - "30% kelurahan null, 34% kodepos null not hidden. Explicitly modeled. Most competitors assume complete data."

2. **Null-Honesty Operationalized**
   - "Frozen template in training forces model to see 'use null' 184k times. AdminVerifier outputs Unverifiable for missing fields. This is architectural choice, not workaround."

3. **Error Attribution is Clear**
   - "Two-stage pipeline = clear tracing. Is this error from SLM? AdminVerifier rejected it? Integrated approaches (AddrLLM, RAG) harder to debug."

4. **Confidence Schema is First-Class**
   - "Not post-metric, not aesthetic. Tells user: 'this field verified from DB', 'this field missing from DB', 'this field inferred from other field'. Epistemic signal, independent contribution."

5. **Deterministic = Transparent**
   - "AdminVerifier: if field found, trace exact DB row. If not found, explain data gap. Traceability high. RAG/vector retrieval: cosine score ambiguous, hard to explain."

6. **Proof of Sufficiency Matters for Indonesia**
   - "If small model sufficient, SMEs/logistics companies can deploy without cloud rental. Cost-benefit for adoption in developing-country context."

---

## 9. CURRENT STATUS

**Complete:**
- ✅ Data review (identified 5 data quality issues)
- ✅ Architecture design (two-stage pipeline clear)
- ✅ Novelty positioning (8 claims defined)
- ✅ Competitive analysis (8 threat papers analyzed)
- ✅ Consultation prep notes (advisor checklist defined)

**In Progress / Next:**
- Real validation data collection (200-500 manually labeled addresses)
- Hallucination metrics (measure SLM error rate + AdminVerifier rescue rate)
- Schema validation (prove 4-state confidence is meaningful)
- Stress testing (50 adversarial cases)
- GPT-4 baseline comparison
- Statistical analysis + failure case documentation

**Documentation Ready:**
- Thesis outline (Prethesis PDF)
- Novelty threat analysis (Word doc with detailed competitive positioning)
- Comparison cheatsheet (HTML - quick reference for consultation)

---

## 10. NOTES FOR NEXT CONVERSATION

**If you reference this in another chat, mention:**
- Current phase: Foundational/planning (no training started)
- 8 novelty claims are defended against 5 critical/high-threat competitors
- Unique value: Combination of deterministic grounding + null-honesty + 4-state confidence + data rumpang modeling
- Next priority: Execute validation experiments per advisor checklist
- Consultation approach: Emphasize design philosophy (honesty + transparency) not just technical cleverness

**You'll need to provide to new conversation:**
- Updated status (if you've done validation experiments)
- Specific questions where you got stuck
- New competitive papers if you found any
- Latest outline/methodology draft if changed

---

**END SUMMARY**
Generated: May 20, 2026 | For: Thesis Consultation Preparation
