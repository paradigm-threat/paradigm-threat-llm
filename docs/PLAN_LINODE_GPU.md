# Plan: Linode / Akamai GPU credit for Paradigm Threat LLM & agent work

Use promotional credit (e.g. **$100**, or **$100–$200** combined) on **GPU compute** for short, focused training and evaluation—not continuous 24/7 hosting. Official rates change; always confirm **$/hr** in [Akamai Cloud pricing](https://www.linode.com/pricing/) (GPU section) before sizing.

---

## 1. Rough budget → GPU hours

Billing is **per hour the instance exists** (powered on), rounded up, until monthly cap if applicable—see [how billing works](https://techdocs.akamai.com/cloud-computing/docs/understanding-how-billing-works).

| Tier (indicative) | Example $/hr | $100 buys ~ | $200 buys ~ |
|-------------------|--------------|-------------|-------------|
| Entry Ada / smaller GPU | ~$0.5–1.0 | 100–200 h | 200–400 h |
| Mid (e.g. Quadro-class) | ~$1.5–2.0 | 50–67 h | 100–133 h |
| High (e.g. Blackwell x1 listed at **$2.50/hr**) | $2.50 | **40 h** | **80 h** |

**Reserve part of credit** for: extra block storage, **object storage** ($/GB-mo), snapshots/images, egress over quota ($0.005/GB typical), and any non-GPU instances (small CPU box for Git, docs, light jobs).

**Cost discipline:** Create the GPU instance only for **jobs**; delete or resize when idle. Long-running “leave it on” burns credit fast.

---

## 2. What is feasible on $100–$200 (realistic)

### Likely **in scope**

- **Instruction-tuning / SFT** on an **open base model** (e.g. 7B–14B class) using **LoRA / QLoRA** on a curated dataset derived from Paradigm Threat **timeline + files** (public markdown, structured Q&A, style guides). Multiple experimental runs and hyperparameter sweeps *if* you use an efficient stack (e.g. Unsloth, axolotl, or plain HF + PEFT) and modest sequence lengths.
- **Embedding model** fine-tuning or **reranker** training for **RAG** over your corpus (often cheaper than full LLM SFT).
- **Synthetic data generation** (smaller model or API-assisted offline batch) then **filter + train** on GPU.
- **Agent scaffolding**: tool-calling formats, JSON/function schemas, eval sets—**training** small models to follow tools is possible; **frontier-level** agent training is not.
- **Evaluation harness**: fixed prompts, retrieval metrics, regression tests—run repeatedly across checkpoints.

### Likely **out of scope** (same budget)

- **Training a large LLM from random init** or full pretraining at meaningful scale.
- **Full fine-tune** (all parameters) of 70B+ models without multi-GPU and much larger spend.
- **Always-on** API serving at scale (use separate, smaller inference later; GPU credit is for R&D bursts).

**Practical framing:** Budget buys **tens to a few hundred GPU-hours** of **adaptation** (LoRA/RAG/evals), not a proprietary foundation model.

---

## 3. Suggested phases (maximize science per dollar)

1. **Off-GPU prep (cheap CPU instance or local)**  
   Repo layout, dataset build (chunking, dedup, license checks), train/val/test splits, prompt templates, baseline evals, Docker image or `cloud-init` script so GPU time is not spent fixing apt.

2. **Pilot on smallest GPU that fits**  
   Prove one full loop: load base model → train LoRA → save checkpoint → run eval → **upload artifact** (below). Only then scale batch size or model class.

3. **Production-ish runs**  
   Longer runs, ablations, optional better GPU *only* when the pipeline is stable.

4. **Optional CPU-only serving test**  
   Quantized 7B on CPU for demos; keep GPU for training.

---

## 4. Paradigm Threat–specific data strategy

- **Sources:** `paradigm-threat-timeline` content, `paradigm-threat-files` investigations/docs (respect licensing and what must stay private).
- **Artifacts to version:** dataset manifests (hashes), preprocessing scripts, eval sets, **not** raw weights in Git.
- **Governance:** redact PII; document what was included/excluded; align tone with project editorial rules (timeline “book voice” vs assistant behavior).

---

## 5. Storing progress after shutdown (critical)

The **GPU instance disk is ephemeral** in practice: if you **delete** the Linode, local disk is gone unless you saved elsewhere.

### A. Primary: **object storage** (S3-compatible)

Akamai/Linode **Object Storage** is the usual home for:

- **Training checkpoints** (`.safetensors`, adapter weights, optimizer states if needed)
- **Logs** (tensorboard/wandb offline, CSV metrics)
- **Datasets** (parquet/jsonl.gz)
- **Container images** exports if you tarball them

Sync with **`rclone`**, **`aws s3 sync`**, or libraries that speak S3 API. Organize by run id, e.g. `s3://pt-llm/runs/2025-03-24-lora-7b/{checkpoints,logs,config}/`.

**After each meaningful step:** upload; treat the cloud bucket as source of truth.

### B. **Hugging Face Hub** (private repo)

- Push **adapters** and small models with `huggingface-cli upload` or `huggingface_hub`.
- Good for **sharing with collaborators** and **version tags** (`v0.1-lora`).

### C. **Block storage volume** (optional)

- Attach a **Block Storage** volume for fast local checkpoint writes during training.
- Before destroying the instance: **snapshot** the volume or **rclone** contents to object storage. Snapshots/images incur **ongoing storage cost** until deleted.

### D. **Git repository** (this repo: `paradigm-threat-llm`)

- Commit: **configs**, **small scripts**, **dataset recipes**, **eval definitions**.
- Do **not** commit multi-GB weights; use Git LFS only if you accept cost/complexity.

### E. **Training resume**

- Use frameworks that support **resume from step** (save `trainer_state.json`, optimizer if you need exact resume; for LoRA-only, often **restoring weights + continuing** is enough).
- Store **seed**, **git commit hash**, **library versions** (`pip freeze` / conda export) beside each run.

### F. **What to upload how often**

| Frequency | What |
|-----------|------|
| Every N steps / minutes | Latest LoRA checkpoint + loss/metrics |
| End of session | Full run folder: configs, logs, best checkpoint, eval reports |
| Before delete instance | Full sync job; verify byte counts or checksums |

---

## 6. Minimal checklist before first GPU hour

- [ ] Base model + license compatible with your use (commercial vs noncommercial).
- [ ] Dataset v1 frozen with a version id.
- [ ] One-command train script; deterministic seeds documented.
- [ ] Destination bucket/repo + credentials on the instance (prefer **scoped keys**).
- [ ] Automated **post-train upload** (cron, `&&` in job script, or watchdog).
- [ ] **Alert on spend** (Akamai billing notifications / budget if available).

---

## 7. References

- [Akamai Cloud GPU product](https://www.linode.com/products/gpu/) / [GPU instance docs](https://techdocs.akamai.com/cloud-computing/docs/gpu-compute-instances)
- [Pricing](https://www.linode.com/pricing/) (verify GPU $/hr at deploy time)
- [Object Storage](https://www.linode.com/products/object-storage/) (S3-compatible)

---

*This document is a planning aid; numbers are order-of-magnitude until confirmed in your account and region.*
