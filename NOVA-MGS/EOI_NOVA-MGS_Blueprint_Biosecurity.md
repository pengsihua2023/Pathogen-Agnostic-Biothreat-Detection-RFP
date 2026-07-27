# Expression of Interest

**Blueprint Biosecurity — Request for Proposals: Pathogen-Agnostic Biothreat Detection**

**Technical Area:** 1 — Computational detection of known and novel pathogens in MGS data

**Project Title:** **NOVA-MGS** — Reference-Free Novelty Scoring, Probabilistic Calibration, and Evidence Verification for Operational Metagenomic Biosurveillance

**Principal Investigator:** Sihua Peng, PhD — Research Scientist, College of Public Health, University of Georgia; affiliated with the Georgia Advanced Computing Resource Center (GACRC)

**Institution:** University of Georgia, Athens, GA, USA

**Requested Period of Performance:** 12 months

**Approximate Budget Request:** $249,700 (direct + indirect; see Section 6)

**Date:** [INSERT]  |  **Contact:** [INSERT EMAIL / PHONE]

---

## 1. Problem Statement

Metagenomic sequencing removes the requirement to know what to look for at the bench. It does not remove it at the keyboard. Every widely deployed classifier in operational wastewater and pooled-human surveillance — Kraken2/Bracken, GOTTCHA2, Centrifuge — assigns reads by reference matching. A genuinely novel or substantially divergent agent, by construction, has no reference to match. It is not misclassified; it is discarded into the unclassified fraction, which in real wastewater runs routinely comprises 60–90% of reads.

The field's practical response has been to accept lower stringency and generate more candidates. This creates the second, now-binding bottleneck: **analyst throughput**. A production wastewater program generating hundreds of samples per week produces far more flagged contigs than any expert team can adjudicate. Candidates are triaged by whoever has time, using ad hoc BLAST searches, with no audit trail and no calibrated confidence. Programs such as CDC Biothreat Radar, NWSS, and mSCAPE will not scale on expert attention.

NOVA-MGS addresses both failures directly: a reference-free scoring engine that assigns calibrated novelty probabilities to sequences no database contains, and a deterministic verification stage that converts a raw candidate into an auditable, decision-ready evidence package — no analyst time spent reconstructing what a flagged contig actually is.

## 2. Technical Approach

### Aim 1 — Dual-channel novelty scoring, calibration, and evidence verification (Months 1–9)

We will score sequences by **distance from** known viral space rather than **membership in** it, using two orthogonal channels whose failure modes do not overlap.

*Channel A (nucleotide).* FracMinHash containment via sourmash against a curated viral sketch database, retaining the full containment distribution rather than a threshold call. Containment rather than Jaccard is the correct measure when a short contig is compared against a large database, and we use the debiased estimator and confidence intervals of Hera et al. (2023) rather than the naive binomial form, which carries a small-sketch bias. This captures agents related to known viruses but falling below classifier assignment thresholds.

*Channel B (protein).* Six-frame ORF prediction on assembled contigs and high-complexity read clusters, followed by protein language model embedding (ESM-2) and local-outlier-factor density estimation against a manifold built from known viral proteomes. Because nearest-neighbour distances concentrate at an embedding dimension near 1280, density is estimated under cosine distance on L2-normalised embeddings with prior dimension reduction, not raw Euclidean volume. This channel recovers agents whose nucleotide sequence has diverged past recognition but whose capsid, polymerase, or glycoprotein folds retain viral character — precisely the regime where Channel A fails.

Because Channel A scores contigs while Channel B scores individual ORFs, ORF-level evidence is aggregated within each contig before fusion, so that both channels enter the model as the same statistical unit. A logistic fusion combines the aligned features and their interaction into a scalar score, which an isotonic regression fitted on an independent split (Aim 2) maps to a per-contig **Novelty and Concern Score (NCS)** — an interpretable probability rather than an arbitrary index. Fusion, calibration, and reliability assessment are fitted on separate splits, or by nested cross-fitting, so reported probabilities are not optimistic; isotonic regression is benchmarked against Platt scaling and beta calibration and selected by cross-validated Brier score. Calibration, not raw discrimination, is the deliverable that makes downstream alert thresholds defensible.

Implementation is Nextflow with Apptainer containers, engineered for SLURM clusters, targeting throughput of hundreds of samples and billions of reads. Preliminary profiling on our existing GACRC pipelines indicates a per-sample marginal cost consistent with routine weekly operation at metropolitan-catchment scale; hard turnaround figures will be reported as a primary deliverable.

*Evidence verification.* A calibrated score is necessary but not sufficient: an operator still has to decide what a flagged contig is. Every candidate above threshold therefore passes through a fixed verification subworkflow before it is reported — read support and coverage breadth/depth against the assembled contig, remote-homology search (DIAMOND, MMseqs2), viral hallmark gene profiling (HMMER against Pfam/VOG), structure-level homology (Foldseek against AlphaFold DB) for cases where sequence homology has been erased, cross-tool concordance (geNomad, VirSorter2), and abundance trajectory for the same sequence cluster across prior samples. These are deterministic tools composed as a Nextflow subworkflow, so the same input yields the same evidence package and every line in it is traceable to a specific tool invocation.

The output is a structured **evidence-verified dossier**: the calibrated NCS, its confidence interval, each supporting and contradicting line of evidence, and the full tool log. This is what makes the score actionable rather than merely computed — and it is a reporting layer, not an alerting one. NOVA-MGS ranks and documents; a human decides.

### Aim 2 — WW-NOVEL-BENCH: open benchmark and retrospective validation (Months 2–12)

The field currently has no shared benchmark for novel-pathogen detection, which makes competing tool claims unverifiable and funder ROI unmeasurable. We will build one.

Real wastewater backgrounds from CASPER (PRJNA1247874), NWSS (PRJNA747181), Tisza et al. 2023 (PRJNA966185), and Wolfe et al. 2026 (PRJNA1438722) will be spiked in silico with three classes of ground truth: (i) real viral genomes with their clade removed from all reference databases, providing honest leave-one-clade-out difficulty; (ii) evolutionarily simulated divergent variants at 20%, 40%, and 60% nucleotide divergence; and (iii) genuine orphan viruses lacking database representation. Spike-ins span 10⁻⁴ to 10⁻⁸ relative abundance across Illumina, PacBio HiFi, and Nanopore platforms, matching our group's established multi-platform pipeline experience.

Leakage control is the part of benchmark design most often skipped, so we specify it explicitly: the entire clade of each spiked genome is removed from every reference, sketch, and training set; difficulty is then indexed continuously by ANI to the nearest retained neighbour rather than by a binary novel/known label, and results are reported per difficulty stratum rather than pooled. Pooling hides the sub-80% regime where a genuinely novel agent actually sits. A parallel time-split protocol — fit and calibrate only on sequences deposited before a cutoff date, evaluate strictly after it — tests generalisation without relying on synthetic divergence at all.

Because positives are a vanishing fraction of contigs, evaluation reports AUPRC rather than ROC-AUC, together with the analyst workload implied at the chosen threshold: the alert budget, not a default cutoff, sets the operating point. Probability quality is assessed separately from ranking via reliability diagrams and a Brier-score decomposition, since a well-ranked score can still be badly calibrated.

Retrospective validation will ask the operationally decisive question — *would this have caught it, and how much earlier?* — against mpox 2022 and H5N1-in-wastewater 2024 signals in archived public data.

The benchmark, evaluation harness, and a public leaderboard will be released independently of NOVA-MGS's own performance, so that competing tools benefit whether or not our method wins.

## 3. Deliverables and Open Availability

All software will be released under a permissive open-source license (MIT/Apache-2.0) on GitHub from month 3, developed in the open rather than dumped at project end. Containers will be published to a public registry; the benchmark dataset and calibration data to Zenodo with a persistent DOI. We anticipate two peer-reviewed publications, both preprinted on bioRxiv/medRxiv at submission. Pipelines will follow nf-core conventions to lower the adoption barrier for programs already running nf-core/mag.

## 4. Responsible Disclosure

One extension of Aim 1 — statistical change-point screening for engineering signatures in assembled contigs — carries a plausible information-hazard profile, in that detailed publication of screening signatures could inform evasion. We raise this proactively rather than after the fact. We propose to develop this component under a tiered disclosure agreement negotiated with Blueprint: operational capability shared with vetted surveillance programs, with method-level publication timing and detail determined jointly. We are prepared to descope this component entirely if Blueprint prefers.

## 5. Team and Capability

The PI has 40+ peer-reviewed publications (h-index 24) spanning computational biology and biostatistics, with current active work on metagenomic viral detection across Illumina, PacBio HiFi, and Nanopore platforms using nf-core/mag, sourmash, geNomad, VirSorter2, GOTTCHA2, and Kraken2/Bracken, operating at scale on the GACRC SLURM cluster via Nextflow and Apptainer. Alongside this, the PI's training and publication record in biostatistics and experimental design is what the calibration and benchmark work in Aims 1 and 2 actually rests on: the hard part of this proposal is not running the tools but establishing that the probabilities they emit mean what they claim to mean.

GACRC provides the compute, storage, and container infrastructure required; no new hardware is requested. We welcome Blueprint's introduction to operational partners (CASPER, Zephyr, NWSS-participating utilities) for real-world validation, and are open to teaming with groups submitting complementary TA2 or TA3 proposals.

## 6. Budget Summary (approximate, 12 months)

| Category | Amount |
|---|---|
| PI effort (30% FTE) | $47,000 |
| Postdoctoral researcher (100% FTE) | $78,000 |
| Graduate research assistant (50% FTE, incl. tuition) | $46,000 |
| Fringe benefits | $37,000 |
| GPU cloud burst capacity and archival storage | $10,000 |
| Publication, dissemination, travel | $9,000 |
| **Total direct** | **$227,000** |
| Indirect costs (10% of direct, per Blueprint policy) | $22,700 |
| **Total requested** | **$249,700** |

The budget is deliberately lean on non-personnel cost: GACRC provides cluster compute, storage, and container infrastructure to the PI at no charge to the project, so the compute line covers only GPU cloud burst for protein language model embedding and persistent archival hosting of the benchmark release. No new hardware is requested. Indirect costs are budgeted at Blueprint's published 10% cap; the University of Georgia's negotiated F&A rate exceeds this, and we are initiating the institutional waiver request through the Office of Research now rather than at Full Proposal stage. The project is modular by design — **Aim 1 alone constitutes a coherent standalone effort at approximately $165,000** should Blueprint prefer a tighter scope.

## 7. Risks and Mitigations

**Protein language model performance on fragmented environmental ORFs is unproven**, and density estimation in high-dimensional embedding space is fragile in its own right. Both are addressed by a go/no-go feasibility test on Channel B in month 2, which tests the distance metric and dimension-reduction choices as well as the embeddings themselves; Channel A alone is sufficient for a reduced-scope deliverable if Channel B underperforms.

**False-positive burden at operational thresholds.** A calibrated score does not by itself bound analyst workload. We therefore report, for every operating point, the implied candidates-per-week at realistic sample volumes, and set thresholds from that budget rather than from a default cutoff.

**Benchmark realism.** In-silico spike-ins can flatter detection methods. We mitigate through leave-one-clade-out real genomes and retrospective testing against genuine historical signals, and we will state benchmark limitations explicitly rather than in a footnote.

---

*Submitted in response to Blueprint Biosecurity's Pathogen-Agnostic Biothreat Detection RFP. Contact: [EMAIL], [PHONE].*
