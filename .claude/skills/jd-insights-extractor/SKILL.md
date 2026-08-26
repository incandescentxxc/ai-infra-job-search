---
name: jd-insights-extractor
description: Extracts candidate-facing prep signal (technical skills, soft skills, named projects/techniques) from a single AI/LLM-infra job description, for later aggregation across many postings. Use per-posting inside /summarize-roles.
---

# JD Insights Extractor

This skill defines what to pull out of **one** job description so `/summarize-roles` can aggregate it with others. It only runs on postings that already have a `strong_fit` (or `adjacent`, if included) verdict from `.claude/skills/ai-infra-role-classifier/SKILL.md` - this is a prep tool for roles worth preparing for, not a general JD parser.

## The framing rule: candidate's perspective, not recruiter's

A JD lists **requirements** ("must have experience with X"). The output of this skill is **prep guidance** ("be ready to speak to X, and here's what that could mean"). Same underlying fact, opposite voice - never just relabel the requirements list.

**Bad (recruiter voice, just relabeled):**
> - Requires experience with speculative decoding
> - Must know quantization techniques (FP8, INT4)

**Good (candidate-prep voice):**
> - Be ready to speak to speculative decoding work - implementing a speculator, training one against production traffic, or reasoning about acceptance-rate/latency tradeoffs
> - Understand the quantization landscape well enough to discuss FP8 vs. INT4 tradeoffs for inference, even if you haven't shipped it yourself

The test: does the line tell the candidate *what to be ready to discuss or demonstrate*, not just *what the posting asked for*?

## Extraction schema

Pull four things per JD:

### 1. Technical skills
The concrete technical fluency areas the role expects, reframed as prep items. Group loosely by area (serving stack, training infra, systems/observability, etc.) if the JD covers several. Keep each item specific enough to be useful for a resume bullet or interview prep note - "distributed systems" is too vague; "KV-cache and memory management for LLM serving" is useful.

### 2. Soft skills
Collaboration/working-style expectations, reframed the same way. These are usually thinner in an infra JD than technical skills - don't pad them if the JD genuinely only implies one or two (e.g. "works directly with customers alongside Forward Deployed Engineers" or "partners with research to productionize frontier techniques").

### 3. Named projects, papers, and techniques
Anything the JD names specifically enough to go look up: an open-source project (SGLang, vLLM, FlashAttention), an internal-but-disclosed project name (DFlash), a named technique with enough specificity to research (speculative decoding, disaggregated prefill/decode, blockwise parallel drafting), a paper, or a named piece of hardware/kernel work (Flash Attention 4 kernels). Record what the company is actually doing with it, not just the name - "our work with SGLang on specdec and multimodal inference performance" is more useful than "SGLang" alone.

These are the highest-value output of this whole skill: they tell a candidate concretely what to read, and what kind of project experience or open-source contribution would resonate if they built or contributed to something similar.

### 4. Bare technique/topic keywords (for frequency counting)
A flat list of short, matchable technique/topic strings this JD touches (e.g. `speculative decoding`, `quantization`, `KV-cache management`, `autoscaling`, `disaggregated prefill/decode`, `distributed training`, `GPU kernel optimization`). This is what `/summarize-roles` counts across postings to produce quantitative stats like "mentioned in 4/5 (80%) of high-fit JDs." Keep these normalized and short so near-duplicates cluster naturally (`quantization` not `FP8/INT4 quantization techniques for inference`).

## Worked example

Posting: Member of Technical Staff - Research, Inference (Modal), `https://jobs.ashbyhq.com/modal/73c97bbc-8e27-4c5d-b38b-90b3afdb0d93`

```json
{
  "url": "https://jobs.ashbyhq.com/modal/73c97bbc-8e27-4c5d-b38b-90b3afdb0d93",
  "company": "Modal",
  "title": "Member of Technical Staff - Research, Inference",
  "technical_skills": [
    "Fluency across the full LLM serving stack, from kernels and quantization up to schedulers and autoscaling: speculative decoding, disaggregated prefill/decode, FP8/INT4 quantization, KV-cache and memory management, autoscaling for spiky serverless traffic",
    "Ability to train custom speculators against real production traffic and feedback loops (speculative decoding, applied)"
  ],
  "soft_skills": [
    "Work directly with customers alongside Forward Deployed Engineers to deploy and tune models",
    "Partner with engineering to turn frontier serving research into shipped product"
  ],
  "named_projects": [
    {"name": "DFlash (with ZLab)", "context": "a speculator design built on KV injection and blockwise parallel drafting"},
    {"name": "SGLang", "context": "collaboration on speculative decoding and multimodal inference performance"},
    {"name": "Flash Attention 4", "context": "kernel-level work"}
  ],
  "technique_keywords": [
    "speculative decoding", "disaggregated prefill/decode", "quantization", "KV-cache management",
    "autoscaling", "GPU kernel optimization", "multimodal inference"
  ]
}
```

## What NOT to do

- Don't invent skills the JD doesn't support - if a JD is thin, the extraction is thin. A short technical_skills list is honest signal, not a failure to extract.
- Don't drop the company/context on a named project - "SGLang" alone is a trivia fact; "our work with SGLang on specdec and multimodal inference" tells the candidate what angle to read it from.
- Don't paraphrase a named project's name - copy it verbatim so it's searchable later.
