---
name: ai-infra-role-classifier
description: Classifies a job posting as genuine AI infrastructure / LLM infrastructure / LLM inference-serving work, or not, by reading the actual job description rather than trusting the title. Use whenever /search-roles needs a fit verdict on a posting.
---

# AI/LLM Infra Role Classifier

This is the domain rubric this repo exists to apply well. A title is a hint, never a verdict - several of the calibration examples below were only correctly classified after reading the full JD.

## Core principle: read the JD, don't trust the title

Titles lie in both directions:
- **Undersells:** "Member of Technical Staff" (Modal, Fireworks, Anthropic) is a generic company-wide IC title covering every specialization and seniority level - it says nothing about fit on its own. Read the team/description.
- **Oversells:** "Member of Technical Staff, Data Platform Engineer" at Fireworks AI turned out to be an Order-to-Cash billing/revenue-engineering contract role. "Member of Technical Staff, Enterprise Foundations" at Fireworks AI turned out to be enterprise identity/security/compliance work (SSO, SCIM, IAM, KMS, SOC 2), not core inference infra, despite not being titled "Security Engineer." Always open the full JD before classifying anything as a strong match.

## Verdict scale

- **Strong fit** - Core AI/LLM infrastructure at any layer of the stack: inference serving, model serving platforms, training infrastructure, GPU/cluster orchestration, ML platform/feature infra, distributed systems underpinning model training or serving. **This explicitly includes deep GPU-kernel/CUDA/HPC-level work** (kernel optimization, GPU networking/RDMA, low-level performance systems) - the hardware/kernel layer is a core part of the AI infra stack, not a diminished tier. Requiring deep systems knowledge is not a reason to downgrade a role; it's a reason it's a serious infra role.
- **Adjacent** - Real infra work, but the role's center of gravity is *operating* or *supporting* the platform rather than *building* the inference/training/serving stack itself - e.g. SRE/reliability engineering for an inference platform, general cloud-platform/deployment engineering, or a role split between infra and a genuinely separate discipline (like open-ended ML research).
- **Excluded** - Fails one of the hard-exclude patterns below, or is a different domain entirely (sales, marketing, recruiting, product design, finance, facilities, physical data-center design/construction).

## Hard-exclude patterns (apply regardless of company or how good the rest of the title looks)

1. **Forward Deployed Engineer** (any variant) - customer-embedded delivery/consulting, not infra ownership. Also watch for the same *shape* under a different name: "Applied Machine Learning Engineer" roles built around GTM/customer PoC delivery are functionally the same pattern even without "Forward Deployed" in the title.
2. **Security Engineer** (any variant, including "GRC Analyst", "Security Operations Lead") - different domain. Also apply this to roles that aren't titled "Security" but are functionally identity/security/compliance platform work (SSO, SCIM, IAM, KMS, SOC 2 as the primary scope) - judge by the JD's actual responsibilities, not the absence of the word "Security" in the title.
3. **Product-facing engineer roles** - a posting scoped around customer-facing product features/UI rather than core infra, even when it says "Member of Technical Staff" or "Software Engineer." Signals: "product engineering, developer experience" framing as the role's center of gravity; frontend stack (React/TypeScript) as a primary skill; "Product (Backend)" or "Product Engineer" in the title.
4. **Staff-level seniority tier** - "Staff Engineer", "Staff Software Engineer", "Staff Machine Learning Engineer", "Staff Platform Engineer" as an explicit seniority modifier before a standard title. **Exception: do not exclude "Member of Technical Staff" (MTS)** - that is a generic company-wide IC title (Modal, Fireworks, Anthropic all use it this way), not the "Staff" seniority tier. The test: does removing the word "Staff" still leave a complete, sensible title ("Staff Software Engineer" → "Software Engineer": yes, exclude) or does it break the phrase ("Member of Technical Staff" → "Member of Technical": no, don't exclude)?
5. **Research Scientist / pure research roles** - PhD-preferred, publication-track-record-preferred, "foundational research" framing, novel-architecture-design as the primary responsibility. Distinguish from **Research Engineer** roles that are explicitly infra-adjacent (e.g. "operating at the intersection of model research and training infrastructure," CUDA/NCCL/distributed-systems as required skills) - those can be Strong fit or Adjacent depending on how much of the role is infra vs. open-ended research; read the actual split of responsibilities rather than the title alone.
6. **Management / people-lead roles** - "Engineering Manager", "Manager", "Head of Engineering", or any title whose primary day-to-day is people management rather than hands-on engineering. This applies **regardless of how on-target the team's technical scope is** - an "Engineering Manager, Inference Runtime" role over a perfect-fit team is still excluded on the management-track pattern, the same way Staff-tier is excluded on seniority alone. (Confirmed exclusion, not a judgment call: this was decided explicitly rather than inferred - management-track roles are out of scope for this rubric even when the team is squarely core AI infra.)

## Calibration examples (from verified real postings)

| Posting | Verdict | Why |
|---|---|---|
| Senior Machine Learning Engineer, LLM Inference Optimization (Nebius) | Strong fit | Core inference-serving optimization work |
| ML Infrastructure Engineer (Nebius) | Strong fit | Core AI infra |
| Member of Technical Staff - Research, Inference (Modal) | Strong fit | Inference-focused despite generic MTS title |
| Member of Technical Staff - ML Performance (Modal) | Strong fit | Performance/serving-focused |
| Software Engineer, LLM Infrastructure (Fireworks AI) | Strong fit | Backend infra for LLM CI/CD, control plane, model serving |
| Member of Technical Staff, Cloud Infrastructure (Fireworks AI) | Strong fit | Distributed training/inference/data infra, Kubernetes/Terraform |
| Member of Technical Staff, AI Training Infrastructure (Fireworks AI) | Strong fit | Core training infra |
| Senior Backend Engineer, Inference Platform (Together AI) | Strong fit | Routing, load balancing, autoscaling for an inference platform |
| Senior Software Engineer - Together Cloud Infrastructure (Together AI) | Strong fit | GPU cluster IaaS, Kubernetes, observability |
| Machine Learning Engineer - Inference (Together AI) | Strong fit | Production inference-engine systems |
| Member of Technical Staff - Reliability Engineering (Fireworks AI) | Adjacent | SRE/reliability for an inference platform - keeps it running, doesn't build the inference stack itself |
| LLM Inference Frameworks and Optimization Engineer (Together AI) | Strong fit | Deep CUDA/TensorRT/MoE-parallelism kernel work - the hardware/kernel layer is core AI infra, not a diminished tier |
| Member of Technical Staff, Performance Optimization (Fireworks AI) | Strong fit | CUDA/Triton kernel-level, HPC-specific - core to inference performance |
| Systems Research Engineer, GPU Programming (Together AI) | Strong fit | GPU-kernel depth - core AI infra at the hardware layer |
| Member of Technical Staff, Research (Fireworks AI) | Excluded (research scientist) | PhD-preferred, foundational-research framing, not infra-engineering |
| MTS, Research Engineer (Fireworks AI) | Adjacent | Split role: research fundamentals (loss landscapes, novel architectures) plus distributed training infra - genuinely dual-scope, judge case by case |
| Member of Technical Staff, Data Platform Engineer (Fireworks AI) | Excluded (misleading title) | Actually Order-to-Cash billing/revenue engineering |
| Member of Technical Staff, Enterprise Foundations (Fireworks AI) | Excluded (security-adjacent) | SSO/SCIM/IAM/KMS/SOC2 as primary scope despite non-"Security" title |
| Member of Technical Staff, Evals & Post-Training Product (Fireworks AI) | Excluded (product-scoped) | JD explicitly frames it as "product engineering, developer experience" |
| Applied Machine Learning Engineer (Fireworks AI) | Excluded (forward-deployed pattern) | Customer-facing GTM/PoC delivery role |
| Staff Software Engineer, Inference / Compute Infrastructure Engineering (Together AI) | Excluded (Staff tier) | Otherwise a strong topical match - excluded purely on seniority tier |
| Infrastructure Design Engineer (Together AI) | Excluded (different domain) | Physical data-center design (rack layout, cooling, AutoCAD/Revit) - facilities/hardware engineering, not software |
| Sr. Forward Deployed Engineer (Databricks) | Excluded (forward-deployed) | Customer-embedded delivery |
| Senior Software Engineer - Enterprise Platform, CustomerLake (Databricks) | Strong fit | Backend-leaning full-stack on a 0-to-1 enterprise platform team; not infra-specific but strong systems-engineering signal |
| Software Engineer - Training Infrastructure (Baseten) | Strong fit | Architects the training platform enabling developers to deploy/scale/monitor workloads - core training infra |
| Software Engineer - Baseten Inference Stack (Baseten) | Strong fit | Distributed runtime powering large-scale LLM inference - exact core match |
| Software Engineer - Model Products (Baseten) | Strong fit | Despite "Products" in the title, JD confirms it's the infra behind hosted model API endpoints (distributed systems, model serving) - title oversells caution cuts both ways |
| Software Engineer - Voice AI (Inference Runtime) (Baseten) | Strong fit | Domain-specific (voice) but still building production inference runtime |
| AI Inference Engineer (Baseten) | Excluded (forward-deployed) | Title reads like a perfect infra match, but the JD body opens "As a Forward Deployed Engineer at Baseten, you will partner directly with customers..." - always read past the title into the actual body text |
| Software Engineer - Dedicated Inference (Baseten) | Excluded (product-scoped) | Title suggests core inference work, but JD explicitly says "As a Product Engineer on the Dedicated Inference team" building CLI/SDK/observability tooling for customers |
| Frontend Engineer - Dedicated Inference (Baseten) | Excluded (product-facing) | Explicit frontend/web-app UI work despite "Dedicated Inference" in the title |
| Software Engineer - Enterprise Platform (Baseten) | Excluded (security/compliance-adjacent) | Enterprise security, compliance, and procurement capabilities for large customers - same pattern as Fireworks' "Enterprise Foundations" |
| Software Engineer - Training Product (Baseten) | Excluded (product-scoped) | JD explicitly frames it as "customer-obsessed," shipping "products," architecting "API and UI down to infrastructure" |
| Software Engineer - Billing & Internal Tooling (Baseten) | Excluded (finance/billing) | Owns pricing/billing/revenue infrastructure - same pattern as Fireworks' "Data Platform Engineer" |
| Software Engineer - Capacity (Baseten) | Excluded (internal business tooling) | Manages customer lifecycle/SLAs on the Internal Tooling team - business operations tooling, not core AI infra |
| Software Engineer - GPU Kernels / GPU Networking & Distributed Systems / Model Performance Systems (Baseten) | Strong fit | Deep GPU-kernel/HPC/RDMA specialization - the hardware/kernel layer is core AI infra, not a diminished tier (corrected 2026-08-26 - previously miscategorized as Adjacent) |
| Site Reliability Engineer / Cloud Platform Engineer (Baseten) | Adjacent | Day-2 ops and general platform reliability for the ML infra platform - keeps it running, not building the inference stack itself |
| Engineering Manager, Runtime Fabric / Engineering Manager, Cloud Platform (Baseten) | Excluded (management track) | Confirmed exclusion per pattern 6 above, even though both teams are squarely core AI infra by domain |

## How to use this when scoring a posting

1. Fetch the full JD (never classify from title + snippet alone).
2. Check it against the five hard-exclude patterns first. If it matches one, verdict is **Excluded** - record which pattern and quote the JD line that triggered it.
3. If it clears all five, judge Strong fit vs. Adjacent by asking: is the role's center of gravity building/operating the AI/LLM infra itself (serving, training infra, GPU orchestration, ML platform), or is it one step removed (reliability for such a platform, GPU-kernel-level optimization requiring deep HPC background, a research role with partial infra scope)?
4. Record the verdict with a one-line reason, quoting the JD where the reasoning isn't obvious from the title. This is what makes the classification auditable later, the same way `seen_jobs.json`'s `source`/`portal` provenance fields make a stale-link report diagnosable.
