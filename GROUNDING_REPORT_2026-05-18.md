# Thesis Grounding Report — 2026-05-18

**Project:** Address Normalization for the Jakarta Region using Small Language Models
**Phase:** Foundational/planning — pivoting to write Chapter 3 + paper draft before training
**Purpose:** Re-synchronize a single, defensible research direction across all artifacts in the project folder; lock the academic argument before any compute is spent.

---

## 0. What was synchronized

Materials reviewed for this report:

- `Outline Prethesis.pdf` — current outline (Sections 1–6 including the existing Methodology and Roadblocks chapter)
- `NewestPaperPosition.docx` and `NewestPaperPosition_revised.docx` — note: only the *revised* one is up-to-date; it already introduces "AdminVerifier", "null-honesty principle", "rumpang data", and the per-field state schema
- `AB_implementation_roadmap.md` — eight-phase roadmap; Phase 1 (thesis reframing) is what this report operationalizes
- `thesis_paper_cheatsheet.html` — 33-paper threat-tier matrix (4 Critical / 8 Significant / 8 Contextual / 13 Supportive)
- Dataset state from memory: `full2.csv` (268→267 rows after Maluku fix), `USED_master_data_final.csv` (4,192 streets, 30.1% null KELURAHAN_REF, 34.2% null KODEPOS_REF), `USED_master_complexes_final_v2.csv` (2,718 landmarks, 99.4% null KECAMATAN_REF, 266 anchor-less entries), `training_dataset_v2.csv` (231,000 SFT rows, known nomor null bug)
- Existing system flowchart in the outline (SLM → JSON parse → kelurahan_map lookup → fuzzy fallback → null cleanup)
- Notebooks: `02_sft_address_parser.ipynb`, `REVAMPED_dataGenerator.ipynb`, `admin_verifier.py` and `admin_verifier_v2.ipynb` (confirms AdminVerifier scaffolding already exists)

Three observations from synchronization that should drive the rewrite:

1. **The revised paper position has already moved beyond the outline.** The Outline Prethesis still uses "Rule-Based Knowledge Retrieval" and lists Roadblocks as an open problem. The revised paper position introduces the AdminVerifier, the four-state schema, and rumpang data as a *finding*. Chapter 3 should be written from the revised position, not the outline.
2. **The cheatsheet has done the related-work positioning work.** The four critical-tier papers (AddrLLM, AddrKG-LLM, Hu et al. IJGIS 2024, ReaGeo) each have specific counters drafted. Chapter 3's positioning paragraphs should reuse these directly.
3. **No code change is needed for Chapter 3 to be written.** The methodology can be specified completely from the AB roadmap + the existing data state. The pre-training delay is not a blocker.

---

## 1. The Unified System Blueprint

### 1.1 Single-sentence definition

> The system is a two-stage hybrid normalization pipeline in which a parameter-efficient fine-tuned small language model performs *syntactic-to-structural* parsing of informal Jakarta address text into a 12-field intermediate representation, and a deterministic post-processing module performs *structural-to-administrative* verification of that representation against an authoritative geographic hierarchy, returning per-field confidence states under a null-honesty constraint.

### 1.2 Formal specification

Let the input space be `X = Σ*` (raw Indonesian address strings over a character alphabet Σ), and let the output space be a partial function

```
y : F → V ∪ {⊥}
```

where `F = {nama_jalan, nama_kompleks_atau_gedung, blok_kavling, nomor, rt, rw, kelurahan, kecamatan, kota_kabupaten, provinsi, kodepos, detail_unit}` is the 12-field schema and `⊥` denotes a null value (the field is intentionally undetermined). The system is the composition of two functions:

```
Stage 1 (Probabilistic):    f_θ : X → Y_draft
Stage 2 (Deterministic):    g_K : Y_draft → Y_verified × S^|F|
Full pipeline:               h = g_K ∘ f_θ
```

where:

- `f_θ` is a fine-tuned Llama-3.1-8B with LoRA parameters θ, returning a *draft* JSON `y_draft` (probabilistic; may contain hallucinated geographic fields).
- `g_K` is the AdminVerifier, parameterized by three reference databases `K = {D_admin, D_streets, D_complexes}`. It returns a verified JSON `y_verified` and a parallel state vector `s ∈ S^|F|` with `S = {Verified, Inferred, Unverifiable, Free-form}`.
- The composition `h` is what gets evaluated end-to-end.

### 1.3 The two-error-mode decomposition

Define the error of the full system on input `x` with gold annotation `y*` as:

```
err(x) = e_syn(x) + e_sem(x)
```

- `e_syn(x)` — *syntactic extraction error*: `f_θ` failed to extract a value present in `x` (or extracted the wrong substring). Recoverable by improving SFT.
- `e_sem(x)` — *semantic verification error*: `f_θ` extracted a value that is not present in the authoritative database `K`. Resolved by `g_K` returning `(⊥, Unverifiable)` rather than the SLM's proposed value.

This decomposition is the central conceptual contribution of the work, and it is what justifies the two-stage architecture: any system that conflates these two failure modes (e.g., a single end-to-end LLM, or a pure rule-based parser) cannot diagnose its own errors and must trade one off against the other.

### 1.4 The null-honesty constraint

For any geographic field `f ∈ {kelurahan, kecamatan, kota_kabupaten, provinsi, kodepos}`:

```
y_verified(f) =
    y_draft(f)          if (y_draft(f), context) is verifiable in K
    derived value       if a value is inferrable via a two-hop chain in K
    ⊥                   otherwise
```

The third clause is non-negotiable: under no circumstance does `g_K` accept the SLM's proposed value for a geographic field without an independent match in `K`. This is what eliminates geographic hallucination — not at the model level (which is hard) but at the output gate (which is cheap and provably correct).

### 1.5 How rumpang data is solved

Because the street master has 30.1% null KELURAHAN_REF and the complexes master is 99.4% null KECAMATAN_REF, *any* system whose evaluation criterion requires complete output will either (a) score itself as failing whenever the database is incomplete, or (b) hallucinate to fill the gap. Both are wrong.

The four-state schema reframes the problem: a field returned as `(⊥, Unverifiable)` is *correctly* returned — it is honest about what the data can and cannot support. Evaluation is then split into:

- **Accuracy on Verified fields** — the standard correctness metric, restricted to the subset where ground truth exists.
- **State distribution** — the proportion of test addresses for which each geographic field is Verified / Inferred / Unverifiable. This *is itself a finding* about the epistemic completeness of Jakarta's administrative data, independent of the SLM.

This is what makes the rumpang phenomenon a contribution rather than a limitation.

### 1.6 On synthetic data and dataset roles — yes, include them in Methodology

Short answer: **yes, you must explain both the synthetic data generation pipeline and every master dataset's role in Chapter 3**, but they belong in the data-curation section (3.3), not the system-design section (3.4–3.5). The reasoning:

- The reverse-generation strategy *is itself* a methodological contribution. It encodes the null-honesty principle at training time (when the source has no kelurahan, the augmented record has `kelurahan = null` — assuming the nomor-bug-style issue is fixed), and it sidesteps a real privacy/regulatory constraint (PII in real addresses). Without explaining it, examiners cannot evaluate the validity of your SFT.
- Each master plays a structurally distinct role in `K`:
  - `full2.csv` is the **canonical hierarchy** — the only source of truth for Kelurahan↔Kecamatan↔Kota↔Provinsi↔KodePos joins. Used in *every* verification path.
  - `USED_master_data_final.csv` is the **street resolution table** — maps free-text street names to a `KELURAHAN_REF` that joins back into `full2.csv`. This is the first hop for street-style addresses.
  - `USED_master_complexes_final_v2.csv` is the **landmark resolution table** — handles addresses anchored on a building, mall, school, or compound name rather than a street. Provides the first hop for complex-style addresses, but because its KECAMATAN_REF is 99.4% null, all kecamatan-level resolution must go via the KELURAHAN_REF → `full2.csv` two-hop chain.

You will be asked why you have three databases instead of one. The answer is: they implement different *resolution paths* (street vs. landmark) over the same canonical hierarchy. They are not redundant; they are typed entry points.

### 1.7 End-to-end flow (verbal)

A raw informal address such as `"Apt Sudirman Twr, lt 12, Jl Jen Sudirman kav 86, Jaksel"` flows through:

1. **Tokenize and prompt** with the locked instruction template.
2. **SLM inference** (Llama-3.1-8B + LoRA) emits a draft JSON. Some fields may be wrong, hallucinated, or null.
3. **JSON parse** with a fallback regex extractor; on hard failure, all fields → null with state `ParseFailure`.
4. **Branch selection** in `g_K`: if `nama_jalan` is non-null, take the street path; if `nama_kompleks_atau_gedung` is non-null, take the landmark path. Both can run for compound addresses; merge by precedence.
5. **First-hop lookup** (fuzzy, RapidFuzz token_sort_ratio, threshold 80 for streets / 75 for complexes after Indonesian street-prefix stripping) → returns `KELURAHAN_REF` if matched.
6. **Second-hop join** of `KELURAHAN_REF` into `full2.csv` → returns Kecamatan, Kota, Provinsi, KodePos with state `Inferred`.
7. **Cross-check** the SLM's proposed kelurahan/kecamatan/etc against the recovered values. If they agree → `Verified`. If the SLM said something else → still trust the database; flag for error analysis.
8. **Null-honesty fill**: any geographic field not reached by steps 5–7 → `(⊥, Unverifiable)`.
9. **Sanitization**: strip ghost numbers (e.g., `nomor` equal to `kodepos`), normalize RT/RW format, strip stray punctuation.
10. **Return** `(y_verified, s)`.

---

## 2. Resolving the Terminology Confusion

### 2.1 The argument against calling this "RAG"

Retrieval-Augmented Generation, as introduced by Lewis et al. (2020) and as practiced in AddrLLM, AddrKG-LLM, and RACCOON, has a specific operational definition:

> *During* generation, a retrieval step fetches relevant context from an external corpus and *injects* it into the language model's input/prompt, so that the model's next-token distribution is conditioned on the retrieved context. The retrieval is typically dense (vector similarity) and the augmented context is consumed *by the model*.

Your Stage 2 does *none* of this:

- It runs **after** generation, not during.
- It does **not** inject anything into the model's prompt or hidden state.
- It uses **deterministic exact / fuzzy lookups** over indexed tables, not vector retrieval.
- The model never sees the database.

Calling it RAG is technically misleading and strategically dangerous. AddrLLM (your most direct critical-tier threat) *is* a RAG system in the canonical sense — vector retrieval over a Chinese national address database, injected into prompts. If you label your system "RAG", you collapse a meaningful architectural distinction and hand the examiner an easy line of attack: "Your contribution then reduces to using a different retrieval method on a different language. AddrLLM already did this."

### 2.2 Better terminology — choose one and use it consistently

Three candidates, in decreasing order of how I'd rank them for your paper:

1. **Deterministic Administrative Database Grounding** — most precise. "Grounding" is the established term in NLP for binding model outputs to external referents (entity linking, semantic parsing). "Administrative Database" is specific. "Deterministic" eliminates the probabilistic-retrieval confusion. This is what I would use in the abstract and Chapter 3 title.
2. **Geographic Anchoring via Hierarchical Lookup** — slightly more evocative; works well in figures and section headers. The "anchor" metaphor connects naturally to the anchor-less complex problem.
3. **Post-hoc Authoritative Verification** — emphasizes the *when* (after generation, not during) and the *role* (verification, not augmentation). Good for the related-work positioning paragraph.

You can use (1) as the formal term, (2) as the figure caption / section subtitle, and (3) in the related-work contrast paragraph. Do not mix them with "RAG" anywhere.

### 2.3 The defense paragraph (drop into Chapter 3 §3.5)

> While our two-stage architecture superficially resembles Retrieval-Augmented Generation (RAG), we deliberately avoid this terminology. Canonical RAG, as formulated by Lewis et al. (2020) and exemplified in address-domain systems such as AddrLLM (Yang et al., 2024) and AddrKG-LLM (Li et al., 2026), injects vector-retrieved context into the language model's input distribution during generation. Our verification module, by contrast, operates entirely post-hoc on the model's structured output, performs no vector retrieval, never modifies the model's input or hidden state, and uses exact and bounded-fuzzy matches against indexed administrative tables. We therefore refer to this stage as Deterministic Administrative Database Grounding. The distinction is methodologically substantive: probabilistic retrieval introduces a noise source that propagates into the model's generation distribution; deterministic post-hoc grounding cannot, by construction, introduce hallucination. The null-honesty principle relies on this property — a probabilistically-retrieved context cannot guarantee that ungrounded fields are returned as null.

### 2.4 The honest caveat (do not hide this)

You *do* use fuzzy matching (Levenshtein/token_sort_ratio) at the first hop, which is technically a bounded-tolerance probabilistic relaxation of exact lookup. Do not pretend otherwise — declare it explicitly as a hyperparameter with a defended threshold, and analyze sensitivity in Chapter 4. The deterministic claim survives: fuzzy match returns a *deterministic* result (the same input always yields the same matched entry); it is the matching function that has tolerance, not the retrieval semantics.

---

## 3. Writing Prioritization: Chapter 3 Granular Outline

You can write the entirety of Chapter 3 right now from the materials already on disk. No training results are required. The outline below maps every subsection to (a) the material source you already have, (b) the formal artifact (formula, pseudo-code, flowchart, table) that goes in it, and (c) a length target.

### 3.1 System Overview — ~1.5 pages

- **Source:** §1 of this report; the AB roadmap Phase 1.5
- **Artifacts to include:**
  - A redrawn system diagram (replace the current flowchart with one that shows the two stages as explicitly labeled `f_θ` and `g_K`, with the four state colours)
  - Equation block from §1.2 of this report (define X, Y, F, S, h, g_K)
  - One-paragraph rationale for the two-stage decomposition (the two-error-mode argument)
- **Avoid:** Implementation details. Save them for §3.4 and §3.5.

### 3.2 Error Taxonomy and the Two-Error-Mode Framework — ~1 page

- **Source:** §1.3 of this report; the Roadblocks section of the current outline (the four failure modes — hallucination, random spaces, abbreviations, typos)
- **Artifacts:**
  - A 2×2 table: rows = `e_syn / e_sem`, columns = examples and resolution mechanism
  - The error decomposition equation `err(x) = e_syn(x) + e_sem(x)`
  - Cite Hu et al. (IJGIS 2024) as the closest analog that does *not* characterize verification under partial coverage; that's the gap this taxonomy fills
- **This subsection is the originality lever.** No prior Indonesian address paper has formalized this. Write it carefully.

### 3.3 Dataset Curation — ~3 pages

#### 3.3.1 Reference Database (full2.csv)

- Source: GitHub `teguh02/Wilayah-Indonesia-Beserta-Kode-Pos`, filtered to DKI Jakarta
- Document: 268 → 267 rows after removing the stray "Kabupaten Seram Bagian Timur" Maluku entry
- Table: per-Kota row counts (Jaksel/Jakbar/Jakpus/Jaktim/Jakut/Kepulauan Seribu)
- Schema: Provinsi | Kota/Kabupaten | Kecamatan | Kelurahan | KodePos

#### 3.3.2 Street Master (USED_master_data_final.csv)

- Source: Pergub-derived street list, enriched via OSM Nominatim, augmented with imputed kelurahan from neighbouring streets
- Document the STATUS column distribution: `OSM_Success`, `Imputed_Success`, `OSM_Success_Recovered`, `Not_Found`
- **Critical disclosure:** 30.1% null `KELURAHAN_REF`, 34.2% null `KODEPOS_REF`, 2 null `NAMA_JALAN` rows (dropped)
- Note the KODEPOS_REF dtype issue (float64 due to pandas null inference; cast to nullable Int64 before lookup)
- Frame the nulls as *rumpang* — refer back to §3.2

#### 3.3.3 Complexes/Landmarks Master (USED_master_complexes_final_v2.csv)

- Source: 11 raw category CSVs (apartments, malls, museums, hospitals, schools, universities, etc.) merged and OSM-enriched
- 2,718 entries; 99.4% null KECAMATAN_REF (justifies the two-hop chain via KELURAHAN_REF → full2.csv)
- 266 anchor-less entries: policy decision = include, return `Unverifiable` for all geographic fields (per the null-honesty principle)

#### 3.3.4 SFT Training Data (training_dataset_v2.csv)

- 231,000 (input, output) pairs
- **Document the reverse-generation pipeline** as a formal procedure:

```
Procedure: ReverseGenerate(D_admin, D_streets, D_complexes)
  Input:  three master databases (clean ground truth)
  Output: training pairs (raw_text, y*)

  For each base record b in (D_streets ∪ D_complexes):
    Construct a canonical address string c(b) from the clean fields
    For k = 1 ... K:                       # augmentation multiplier
      noise_ops ← sample subset of {Lex, Perm, Char}
      raw ← Apply(noise_ops, c(b))
      y* ← StructuredJSON(b)               # includes nulls for rumpang fields
      Emit (raw, y*)

  Where:
    Lex  = lexical substitution from a curated dictionary of
           Jakarta-local abbreviations and colloquial forms
    Perm = structural permutation of address component order
    Char = character-level perturbation (insert/delete/swap)
```

- **Disclose the nomor null bug** (currently 0% nulls across 231k; the augmentation forces a placeholder even when the base address has none). Document the planned fix as a methodological decision: regenerate affected samples with corrected null assignment to target ≥15% null rate, matching estimated real-world base rate.
- **Disclose the split strategy:** stratified by base address (not by row), to prevent augmented variants of the same source from leaking across train/val/test under the ~33x multiplier.

### 3.4 Stage 1: Supervised Fine-Tuning — ~2 pages

#### 3.4.1 Base Model Selection

Lock the four-way comparison from the AB roadmap:
- Llama-3.1-8B (chosen): 128k context, open weights, instruction-tuned, differentiates from Hu et al.'s Mistral-7B while staying in the same compute tier
- Mistral-7B (rejected): used by Hu et al. — direct overlap
- IndoBERT (rejected): encoder-only, cannot generate structured JSON
- GPT-4 / Claude (rejected): cost, latency, no fine-tuning, and conceptually wrong for a Proof-of-Sufficiency thesis

#### 3.4.2 LoRA/QLoRA Configuration

A table with the exact hyperparameters from the AB roadmap (r=16, alpha=32, target=q_proj/v_proj, dropout=0.05, 4-bit NF4, bf16, lr=2e-4, cosine, 3 epochs, seq_len=512, effective batch 16–32). Justify each choice with one sentence.

#### 3.4.3 Prompt Template (Locked)

Reproduce the locked instruction template verbatim. **Emphasize the null instruction line** — this is how the null-honesty principle enters the model at training time. Without it, the model has no incentive to emit null.

#### 3.4.4 Output Schema

12 fields, formally specified. A table with `field name | type | nullable | constrained by K? | example`.

### 3.5 Stage 2: Deterministic Administrative Database Grounding — ~3 pages

#### 3.5.1 The Four-State Schema (Verified / Inferred / Unverifiable / Free-form)

Table from §2.2 of the AB roadmap. Worked example: walk through a single input address showing the state assigned to each of the 12 fields.

#### 3.5.2 Verification Logic — Pseudo-code

```
function g_K(y_draft):
    s ← {f: Free-form for f in F}              # default
    geographic_fields ← {kelurahan, kecamatan, kota_kabupaten, provinsi, kodepos}

    # Branch 1: street path
    if y_draft.nama_jalan is not null:
        match ← FuzzyMatch(y_draft.nama_jalan, D_streets.NAMA_JALAN,
                            threshold=80, prefix_strip=True)
        if match is not None:
            kel_ref ← match.KELURAHAN_REF
            if kel_ref is not null:
                row ← D_admin.lookup(kel_ref)
                AssignFromRow(y_verified, s, row, source=Inferred)
                CrossCheckSLM(y_verified, y_draft, s)   # may upgrade to Verified

    # Branch 2: complex path (analogous, with threshold=75)
    if y_draft.nama_kompleks_atau_gedung is not null:
        match ← FuzzyMatch(y_draft.nama_kompleks_atau_gedung,
                           D_complexes.name, threshold=75)
        if match is not None:
            kel_ref ← match.KELURAHAN_REF       # may be null (anchor-less)
            if kel_ref is not null:
                row ← D_admin.lookup(kel_ref)
                AssignFromRow(y_verified, s, row, source=Inferred)

    # Null-honesty enforcement
    for f in geographic_fields:
        if y_verified.f is null and s[f] is Free-form:
            s[f] ← Unverifiable

    # Sanitization
    StripGhostNumbers(y_verified)
    NormalizeRTRW(y_verified)
    return (y_verified, s)
```

#### 3.5.3 Fuzzy Thresholds and Their Defense

Two thresholds (80 for streets, 75 for complexes), one library (RapidFuzz token_sort_ratio), one normalization (Indonesian street-prefix stripping: `Jl. / Jln. / Jalan`). Acknowledge these as hyperparameters and pre-commit to a sensitivity analysis in Chapter 4 (vary each ±10).

#### 3.5.4 Dtype, Anchor-less, and Maluku Handling

A short subsection covering the three pre-registered data-cleaning decisions: KODEPOS dtype cast, anchor-less complex policy (include + Unverifiable), Maluku row removal. These are presented as *design decisions*, not bug fixes.

### 3.6 Evaluation Framework — ~1.5 pages

#### 3.6.1 Metrics

- Per-field exact-match accuracy, computed *conditioned on state*
  - Acc(f) over Verified-state instances of f
  - Acc(f) over (Verified ∪ Inferred) instances of f
- Aggregate full-record correctness
- Δ accuracy attributable to `g_K`: `Acc(h) − Acc(f_θ)` per field
- State distribution: for each geographic field f, `P(s_f = state)` over the test set — this is a first-class reported finding

#### 3.6.2 Test Sets

- Synthetic held-out (from the augmentation pipeline, split by base address)
- Real-world hand-labeled (target ≥100 Jakarta addresses, diverse over the 5 kotamadya and over formality)

#### 3.6.3 Baselines

The four-baseline ablation table from the AB roadmap (B0: base Llama-3.1-8B, B1: SFT only, B2: regex/heuristic, Full: SFT + AdminVerifier).

#### 3.6.4 Error Taxonomy Application

For each Chapter-4 error case, classify into `e_syn` or `e_sem`. Report the distribution. *This is the empirical validation of the framework introduced in §3.2.*

---

## 4. Immediate Action Plan — Today and Tomorrow

This is what to do in the next ~16 working hours to walk into the supervisor consultation with a draftable Chapter 3 and a paper-ready introduction.

### Day 1 (Today, 2026-05-18) — Lock the argument

1. **(60 min) Read this report end-to-end and decide on terminology.** Pick one of the three terms in §2.2 and commit. Do a global find-replace on `NewestPaperPosition_revised.docx` so no occurrence of "RAG" remains in your own positioning text (citations to AddrLLM etc. keep the word).
2. **(45 min) Update the Outline Prethesis title and §1.3 / §4 / §5.** Replace "Rule-Based Knowledge Retrieval" / "Knowledge Retrieval Validator" everywhere with "Deterministic Administrative Database Grounding" (or your chosen variant). Replace "Hybrid SLM-Rule Based" in the subtitle with "Hybrid SLM with Deterministic Administrative Grounding".
3. **(90 min) Draft §3.1 (System Overview)** following the spec in §3.1 above. Include the equation block from §1.2. Redraw the system flowchart with the two stages labeled `f_θ` and `g_K` and the four state colours visible.
4. **(120 min) Draft §3.2 (Error Taxonomy)**. This is the originality section — give it real time. Cite Hu et al. and explicitly contrast: they evaluate verification against complete external geocoders, you evaluate against partial-coverage authoritative databases.
5. **(60 min) Draft §3.3.1–3.3.3 (the three reference databases).** All numbers come from the memory inventory and the data audit. Tables can be built from `df.describe()` style outputs of each master.
6. **(30 min) Write Phase-1 deliverable list status check.** From the AB roadmap §1.7: confirm you now have a revised abstract, locked RQs (use the four-RQ set in roadmap §1.2), four contribution claims (roadmap §1.3), and the related-work paragraphs for the four critical-tier papers. Whichever are still missing, draft them tonight — they are short and the cheatsheet has done 80% of the work already.

### Day 2 (Tomorrow, 2026-05-19) — Convert design into draft

7. **(90 min) Draft §3.3.4 (SFT data and reverse-generation procedure).** Include the boxed pseudo-code from §3 of this report. Disclose the nomor null bug as a methodological decision-in-progress.
8. **(60 min) Draft §3.4 (SFT methodology).** Most of this is reproducing the LoRA config table and the locked prompt template from the AB roadmap. Argue the base model selection in one paragraph.
9. **(120 min) Draft §3.5 (AdminVerifier).** This is the longest subsection. Use the pseudo-code from §3.5.2 of this report. Add a worked example: pick an actual address from `data/training/training_dataset_v2.csv`, run it through verbally, show the resulting `(y_verified, s)`.
10. **(60 min) Draft §3.6 (Evaluation).** This is fully specifiable now even without results — the metrics, test set structure, and baseline definitions are all in the AB roadmap. Leave Chapter 4 stubs for the numbers.
11. **(45 min) Paper-side: update §2.2 of the revised paper position.** It currently says "Geospatial RAG" — change the heading to "Post-hoc Verification vs. Generative Retrieval-Augmented Methods" and use the defense paragraph from §2.3 of this report.
12. **(60 min) Prepare a 1-page supervisor brief.** Cover: (a) the two-error-mode reframing, (b) why "RAG" was dropped, (c) the four-state schema as the measurable contribution, (d) data-quality issues found and planned fixes, (e) timeline: Phase 3 (data fixes) → Phase 4 (training) → Phase 7 (eval) over the next N weeks. Bring the redrawn system flowchart and the draft Chapter 3 §3.1–§3.5 (even if rough).

### Deliverables walking into supervisor consultation

- A revised paper position with no internal use of "RAG", consistent terminology
- A Chapter 3 draft (rough but complete-in-structure) covering §3.1 through §3.6
- A 1-page brief summarizing the architectural pivot
- The redrawn system flowchart with `f_θ` / `g_K` and four state colours
- A short list of the open methodological decisions you still want the supervisor's input on (recommend: fuzzy-threshold defense strategy, anchor-less complex policy, real-world test set size and sourcing)

### What deliberately is *not* on this checklist

- Training. (Phase 4 of the AB roadmap; not a Day-1/2 task.)
- Implementing the AdminVerifier in code. (`admin_verifier_v2.ipynb` already exists; finalize after Chapter 3 §3.5 is drafted, so the spec drives the code.)
- Real-world test set collection. (Phase 7; needs a separate planning conversation.)

---

## Appendix A — Numbered claims for supervisor sign-off

Use these as a checklist when you sit down with your supervisor. Each is a binary decision; getting agreement on these unblocks the rest of the writing.

1. The system will be described as a two-stage hybrid pipeline, not as RAG. ✅ / ❌
2. The verification stage will be called "Deterministic Administrative Database Grounding" (or equivalent neutral term agreed upon). ✅ / ❌
3. The four-state per-field schema (Verified / Inferred / Unverifiable / Free-form) is the central measurable contribution. ✅ / ❌
4. Rumpang data is presented as a finding (with quantified state distribution), not as a limitation. ✅ / ❌
5. The error taxonomy (e_syn vs. e_sem) is the central conceptual contribution. ✅ / ❌
6. The reverse-generation synthetic pipeline is methodologically disclosed in Chapter 3, including the nomor null bug and its fix. ✅ / ❌
7. Three reference databases (`full2.csv`, streets, complexes) are explained as distinct resolution paths over the same canonical hierarchy. ✅ / ❌
8. Evaluation uses the four-baseline ablation (B0/B1/B2/Full) defined in the AB roadmap. ✅ / ❌
