# Bastion-Security: Reasoning-Distilled PowerShell Threat Classifier

A research post-mortem on a small (~460M) decoder-only model trained to classify PowerShell as
malicious/benign *with an explicit reasoning trace*, then map malicious samples to attacker TTPs.
The project reached a hard performance ceiling. This repo documents what was built, what was
measured, and where the evidence localizes the ceiling. It does **not** release the model,
data, or pipeline (see [Why nothing is released](#why-nothing-is-released)).

**Bottom line:** the classifier hit ~93% malicious-correct but did not meet its design target.
The evidence localizes the ceiling to **pretraining substrate scale**. The domain corpus was
saturated, but the model lacked the world-knowledge representations needed to generalize over
the entities real commands reference. The confirming ablation was compute-gated and never run,
so this is a *localized* root cause, not a proven one.

<!-- ARTIFACT: none needed here. Keep this summary tight; it's the part most readers actually read. -->

---

## Architecture

Custom decoder-only transformer, trained from scratch.  A small parameter model scaled to AMSI speed inference utilizing FlashAttention2.

<!-- ARTIFACT: config table. Safe to publish in full — hyperparameters are not weaponizable.
     Drop a clean table of: params (~460M), layers (18), d_model (1280), heads (10), FF (6144),
     vocab (32768), RoPE base (50000), FlashAttention-2, ~9.6B training tokens.
     Source: your training config. Do NOT include data-recipe or corpus-construction details. -->

Tokenizer: extended from a huggingface tokenizer base with domain-specific additions (details withheld).

## Method

Supervised fine-tuning on **frontier reasoning distillation**: a frontier teacher produced the
reasoning traces, and critically **the teacher was never shown the ground-truth label**. The
traces are the teacher's own reasoning toward a verdict, not post-hoc justification of a known
answer. This is the same principle as open-source reasoning distillation.

<!-- ARTIFACT: optional — a single sanitized example (benign command + trace + verdict).
     If you include one, use a BENIGN sample only. Never a working malicious sample. -->

---

## Results

<!-- ARTIFACT: eval table. Publish the STABLE no_thinking figures, not the best-run checkpoint
     with the provenance issue. Columns: malicious-correct %, false-positive %, TTP-root %.
     State the split size. This is the defensible headline; metrics alone are not weaponizable. -->

The classifier reached **~93% malicious-correct** on held-out evaluation with a non-trivial
false-positive rate. **It did not meet its design target.** The number matters here not as a
success claim but as a locator: the method reaches ~93% *while starved of a world-knowledge
substrate*, which points the ceiling at pretraining scope rather than classifier design (below).

---

## Investigation

Three findings, in order of strength.

### 1. Qualitative error analysis: missing world knowledge

The residual errors concentrate on commands referencing real products and services the model has
no concept of, package managers (e.g. Chocolatey), common services (e.g. Google), and similar
named entities. Lacking a representation of what these *are*, the model surface-matches on tokens
and misclassifies. This is consistent with the knowledge concentration observed in the
weights.

<!-- ARTIFACT: 2–4 sanitized error cases. BENIGN commands only — benign invocations that were
     misclassified because the model didn't recognize the referenced entity. Never a working
     malicious sample. A small table (command → true label → predicted → likely reason) is ideal. -->

### 2. A hard, recipe-invariant validation floor

Training loss descends below the validation floor, but the val floor is a single hard bottom
(~1.4) that does not move across learning-rate schedules, regularization, or the weight-space
path connecting the two best checkpoints. The loss basin is deep and narrow with one minimum,
and that minimum sits *above* reachable training loss. A floor that is invariant to the training
recipe and sits outside the reachable region is the signature of information missing from the
objective — not a tuning problem.

<!-- ARTIFACT: several figures here, all safe (they visualize loss geometry, not the model):
     - Filter-normalized 3D loss landscape
     - Two-minima basin analysis / basin-width comparison
     - LR range test
     - Per-token CE decomposition
     - Separability probe (ROC). KEEP the caption noting AUC 1.000 was a leakage artifact —
       flagging your own confound reads as honest and is the right call.
     Interleave each figure next to the sentence it supports. -->

<img width="820" height="780" alt="landscape3d" src="https://github.com/user-attachments/assets/e3f3ebb4-2011-4786-8b2a-9b988c1eeb6a" />





A representation of the loss basin



<img width="820" height="780" alt="basin2d" src="https://github.com/user-attachments/assets/55500e7b-c5cd-4b53-90b5-38c56b3c4604" />


A visualation of the basin where both a regularized and unregularized finetune land in the same basin

### 3. Reasoning-to-verdict coupling is weak, and regime-dependent

Causal swap tests: substituting the reasoning block and measuring the effect on the CLASSIFY verdict. This show weak coupling in the baseline (causal strength +0.066): CE-SFT admits imitating the trace without it becoming load-bearing, 
but this is not uniform across training regimes. The regularized, wider-basin checkpoint weighted the reasoning more heavily than the non-regularized one. How much the trace drives the verdict is a function of the training setup, not a fixed property.
This characterizes the SFT objective's behavior, but it does not appear to be the performance ceiling. The ceiling is set by pretraining substrate (below), independent of whether the reasoning is load-bearing.

<!-- ARTIFACT: swap-test methodology + result. Publish the METHOD (how you swapped, what you
     measured) and the +0.066 number. A small before/after table or bar figure works.
     This is your strongest safe result — give it room. -->
---

## Root cause (localized, not proven)

The three findings converge: the domain pretrain was **saturated**, more of the same corpus
would not help, but the model was pretrained at Chinchilla-optimal *token count* and far under
the scale needed to build a relational world-knowledge substrate. The information setting the val
floor lives outside the domain distribution, so no fine-tuning recipe can reach it. The likely fix
is a broader pretraining corpus supplying world knowledge, not more domain data and not simply
longer training. Which is where the real compute cost sits.

The honest boundary: the confirming ablation a pretrain with a broader-corpus variant and watch the
floor drop, was compute-gated and never run. So the ceiling is *localized* to pretraining
substrate scale by converging evidence (invariant floor, basin geometry, entity-confusion errors),
not *proven* by ablation.

<!-- ARTIFACT: none — synthesis section, prose only. This paragraph is the spine of the repo;
     the "localized not proven" phrasing is what survives a sharp reader. Don't soften it. -->

---

## What a continuation would test

- Broader-corpus pretrain (world knowledge) at fixed downstream recipe → does the val floor drop?
- A reasoning objective that makes the trace load-bearing (given finding #1, CE-SFT does not).

<!-- ARTIFACT: none. -->

---

## Why nothing is released

The trained weights, dataset, tokenizer, and training/inference pipeline are withheld. Assembled,
they lower the barrier to producing a model that generates malicious PowerShell. The diagnostic
methodology and results below carry no such uplift, so everything that demonstrates the work is
here and everything dual-use is not. As a security professional my primary obligation is ensuring that I 
do not provide any functional exploits that could be weaponized via my ML research.

<!-- ARTIFACT: none. This section is prose only. To a security audience this reads as maturity — keep it. -->

## License / contact

<!-- ARTIFACT: add a license (docs/figures only, since no code/data/weights ship) and contact. -->
