# Expression of Interest

**Blueprint Biosecurity — Request for Proposals: Pathogen-Agnostic Biothreat Detection**

**Technical Area:** 1 — Computational detection of known and novel pathogens in MGS data

**Project Title:** **NOVA-MGS** — Reference-Free Novelty Scoring and Agentic Triage for Operational Metagenomic Biosurveillance

**Principal Investigator:** Sihua Peng, PhD — Research Scientist, College of Public Health, University of Georgia; affiliated with the Georgia Advanced Computing Resource Center (GACRC)

**Institution:** University of Georgia, Athens, GA, USA

**Requested Period of Performance:** 12 months

**Approximate Budget Request:** $385,000 (direct + indirect; see Section 6)

**Date:** [INSERT]  |  **Contact:** [INSERT EMAIL / PHONE]

---

## 1. Problem Statement

Metagenomic sequencing removes the requirement to know what to look for at the bench. It does not remove it at the keyboard. Every widely deployed classifier in operational wastewater and pooled-human surveillance — Kraken2/Bracken, GOTTCHA2, Centrifuge — assigns reads by reference matching. A genuinely novel or substantially divergent agent, by construction, has no reference to match. It is not misclassified; it is discarded into the unclassified fraction, which in real wastewater runs routinely comprises 60–90% of reads.

The field's practical response has been to accept lower stringency and generate more candidates. This creates the second, now-binding bottleneck: **analyst throughput**. A production wastewater program generating hundreds of samples per week produces far more flagged contigs than any expert team can adjudicate. Candidates are triaged by whoever has time, using ad hoc BLAST searches, with no audit trail and no calibrated confidence. Programs such as CDC Biothreat Radar, NWSS, and mSCAPE will not scale on expert attention.

NOVA-MGS addresses both failures directly: a reference-free scoring engine that assigns calibrated novelty probabilities to sequences no database contains, and a deterministic multi-agent verification layer that converts a raw candidate into an auditable, decision-ready dossier in minutes rather than hours.

## 2. Technical Approach

### Aim 1 — Dual-channel reference-free novelty scoring (Months 1–6)

We will score sequences by **distance from** known viral space rather than **membership in** it, using two orthogonal channels whose failure modes do not overlap.

*Channel A (nucleotide).* FracMinHash containment via sourmash against a curated viral sketch database, retaining the full containment distribution rather than a threshold call. This captures agents that are related to known viruses but fall below classifier assignment thresholds.

*Channel B (protein).* Six-frame ORF prediction on assembled contigs and high-complexity read clusters, followed by protein language model embedding (ESM-2) and density-based outlier detection against an embedding manifold built from known viral proteomes. This channel is designed to recover agents whose nucleotide sequence has diverged past recognition but whose capsid, polymerase, or glycoprotein folds retain viral character — precisely the regime where Channel A fails.

Channel outputs are fused and calibrated by isotonic regression against held-out ground truth (Aim 3), yielding a per-sequence **Novelty and Concern Score (NCS)** that is an interpretable probability rather than an arbitrary index. Calibration, not raw discrimination, is the deliverable that makes downstream alert thresholds defensible.

Implementation is Nextflow with Apptainer containers, engineered for SLURM clusters, targeting throughput of hundreds of samples and billions of reads. Preliminary profiling on our existing GACRC pipelines indicates a per-sample marginal cost consistent with routine weekly operation at metropolitan-catchment scale; hard turnaround figures will be reported as a primary deliverable.

### Aim 2 — Deterministic agentic triage and dossier generation (Months 4–10)

We will build an evidence-assembly layer over the scoring engine using LangGraph, structured as a **fixed directed acyclic graph, not an autonomous agent**. This distinction is deliberate: reproducibility and auditability are non-negotiable in a public health alerting context, and free-running agents provide neither.

For each candidate exceeding an NCS threshold, the graph executes in parallel: remote-homology search (DIAMOND, MMseqs2), viral hallmark gene profiling (HMMER against Pfam/VOG), structure-level homology detection (Foldseek against AlphaFold DB) for cases where sequence homology has been erased, cross-tool concordance checks (geNomad, VirSorter2), time-series abundance trajectory retrieval for the same sequence cluster across prior samples, and sample-metadata context. A dedicated critic node performs explicit contradiction detection across evidence streams.

Output is a structured **Sequence-of-Concern dossier**: calibrated confidence, every supporting and contradicting line of evidence, and a complete tool-invocation log. Language models are constrained to reason only over structured tool outputs and may not assert unsupported claims; every statement in a dossier is traceable to a specific tool call. The system is model-agnostic and validated on locally hosted open-weight models (Gemma, Llama), so that agencies with data-residency constraints can deploy it without external API calls.

Evaluation is head-to-head against human experts on a blinded candidate set, measuring adjudication time, inter-rater agreement (Cohen's κ) between analyst-alone and analyst-with-dossier, and error rates in both directions.

### Aim 3 — WW-NOVEL-BENCH: open benchmark and retrospective validation (Months 2–12)

The field currently has no shared benchmark for novel-pathogen detection, which makes competing tool claims unverifiable and funder ROI unmeasurable. We will build one.

Real wastewater backgrounds from CASPER (PRJNA1247874), NWSS (PRJNA747181), Tisza et al. 2023 (PRJNA966185), and Wolfe et al. 2026 (PRJNA1438722) will be spiked in silico with three classes of ground truth: (i) real viral genomes with their clade removed from all reference databases, providing honest leave-one-clade-out difficulty; (ii) evolutionarily simulated divergent variants at 20%, 40%, and 60% nucleotide divergence; and (iii) genuine orphan viruses lacking database representation. Spike-ins span 10⁻⁴ to 10⁻⁸ relative abundance across Illumina, PacBio HiFi, and Nanopore platforms, matching our group's established multi-platform pipeline experience.

Retrospective validation will ask the operationally decisive question — *would this have caught it, and how much earlier?* — against mpox 2022 and H5N1-in-wastewater 2024 signals in archived public data.

The benchmark, evaluation harness, and a public leaderboard will be released independently of NOVA-MGS's own performance, so that competing tools benefit whether or not our method wins.

## 3. Deliverables and Open Availability

All software will be released under a permissive open-source license (MIT/Apache-2.0) on GitHub from month 3, developed in the open rather than dumped at project end. Containers will be published to a public registry; the benchmark dataset and calibration data to Zenodo with a persistent DOI. We anticipate two peer-reviewed publications, both preprinted on bioRxiv/medRxiv at submission. Pipelines will follow nf-core conventions to lower the adoption barrier for programs already running nf-core/mag.

## 4. Responsible Disclosure

One extension of Aim 1 — statistical change-point screening for engineering signatures in assembled contigs — carries a plausible information-hazard profile, in that detailed publication of screening signatures could inform evasion. We raise this proactively rather than after the fact. We propose to develop this component under a tiered disclosure agreement negotiated with Blueprint: operational capability shared with vetted surveillance programs, with method-level publication timing and detail determined jointly. We are prepared to descope this component entirely if Blueprint prefers.

## 5. Team and Capability

The PI has 40+ peer-reviewed publications (h-index 24) spanning computational biology and biostatistics, with current active work on metagenomic viral detection across Illumina, PacBio HiFi, and Nanopore platforms using nf-core/mag, sourmash, geNomad, VirSorter2, GOTTCHA2, and Kraken2/Bracken, operating at scale on the GACRC SLURM cluster via Nextflow and Apptainer. The PI has separately built and deployed production LangGraph multi-agent systems with locally hosted open-weight models — an unusual combination of production metagenomics engineering and applied agentic-systems experience that is directly load-bearing for Aims 1 and 2.

GACRC provides the compute, storage, and container infrastructure required; no new hardware is requested. We welcome Blueprint's introduction to operational partners (CASPER, Zephyr, NWSS-participating utilities) for real-world validation, and are open to teaming with groups submitting complementary TA2 or TA3 proposals.

## 6. Budget Summary (approximate, 12 months)

| Category | Amount |
|---|---|
| PI effort (40% FTE) | $62,000 |
| Postdoctoral researcher (100% FTE) | $78,000 |
| Graduate research assistant (50% FTE, incl. tuition) | $46,000 |
| Fringe benefits | $41,000 |
| Compute, storage, and cloud burst capacity | $35,000 |
| Publication, dissemination, travel | $12,000 |
| **Total direct** | **$274,000** |
| Indirect costs | $111,000 |
| **Total requested** | **$385,000** |

*Note: indirect cost recovery will be reconciled with Blueprint's published Policy on Indirect Costs prior to full proposal submission; the figure above is a placeholder pending that reconciliation.* The project is modular by design — Aims 1 and 3 constitute a coherent standalone effort at approximately $240,000 should Blueprint prefer a tighter scope.

## 7. Risks and Mitigations

**Protein language model performance on fragmented environmental ORFs is unproven.** A go/no-go feasibility test on Channel B is scheduled for month 2; Channel A alone is sufficient for a reduced-scope deliverable if Channel B underperforms.

**LLM hallucination in triage output.** Mitigated architecturally: the graph is deterministic, models reason only over structured tool returns, and every dossier assertion is traceable. Hallucination rate is measured, not assumed.

**Benchmark realism.** In-silico spike-ins can flatter detection methods. We mitigate through leave-one-clade-out real genomes and retrospective testing against genuine historical signals, and we will state benchmark limitations explicitly rather than in a footnote.

---

*Submitted in response to Blueprint Biosecurity's Pathogen-Agnostic Biothreat Detection RFP. Contact: [EMAIL], [PHONE].*
