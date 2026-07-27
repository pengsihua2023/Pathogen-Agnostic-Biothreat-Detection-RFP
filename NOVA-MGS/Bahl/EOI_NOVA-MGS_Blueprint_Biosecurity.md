# Expression of Interest

**Blueprint Biosecurity — Request for Proposals: Pathogen-Agnostic Biothreat Detection**

**Technical Area:** 1 — Computational detection of known and novel pathogens in MGS data

**Project Title:** **NOVA-MGS** — Reference-Free Novelty Scoring, Probabilistic Calibration, and Evidence Verification for Operational Metagenomic Biosurveillance

**Principal Investigator:** Justin Bahl, PhD — Professor, [INSERT: department / center], University of Georgia

**Project Scientist:** Sihua Peng, PhD — Research Scientist, College of Public Health, University of Georgia; affiliated with the Georgia Advanced Computing Resource Center (GACRC)

**Institution:** University of Georgia, Athens, GA, USA

**Requested Period of Performance:** 12 months

**Approximate Budget Request:** $249,700 (direct + indirect; see Section 6)

**Date:** [INSERT]  |  **Contact:** [INSERT EMAIL / PHONE]

---

## 1. Problem Statement

Metagenomic sequencing removes the requirement to know what to look for at the bench. It does not remove it at the keyboard. Every widely deployed classifier in operational wastewater and pooled-human surveillance — Kraken2/Bracken, GOTTCHA2, Centrifuge — assigns reads by reference matching. A genuinely novel or substantially divergent agent, by construction, has no reference to match. It is not misclassified; it is discarded into the unclassified fraction, which in real wastewater runs routinely comprises 60–90% of reads.

The field's practical response has been to accept lower stringency and generate more candidates. This creates the second, now-binding bottleneck: **analyst throughput**. A production wastewater program generating hundreds of samples per week produces far more flagged contigs than any expert team can adjudicate. Triage falls to whoever has time, via ad hoc BLAST, with no audit trail and no calibrated confidence. Programs such as CDC Biothreat Radar, NWSS, and mSCAPE will not scale on expert attention.

NOVA-MGS addresses both failures directly: a reference-free scoring engine that assigns calibrated novelty probabilities to sequences no database contains, and a deterministic verification stage that converts a raw candidate into an auditable, decision-ready evidence package — no analyst time spent reconstructing what a flagged contig actually is.

## 2. Technical Approach

![**Figure 1. NOVA-MGS processing chain.** Contigs are scored by two orthogonal channels — nucleotide-level FracMinHash containment (Channel A) and per-ORF protein-embedding local density (Channel B) — then aggregated to contig level, fused, and isotonically calibrated to a Novelty and Concern Score. Only candidates above threshold enter deterministic evidence verification.](NOVA-MGS-4.png){width=4.8in}


### Aim 1 — Dual-channel scoring, calibration, and evidence verification (Months 1–9)

We will score sequences by **distance from** known viral space rather than **membership in** it, using two orthogonal channels whose failure modes do not overlap.

*Channel A (nucleotide).* FracMinHash containment via sourmash against a curated viral sketch database [1], retaining the full containment distribution rather than a threshold call. Containment rather than Jaccard is the correct measure when a short contig is compared against a large database [2], and we use the debiased estimator and confidence intervals of Hera et al. [3] rather than the naive binomial form, which carries a small-sketch bias. This captures agents related to known viruses but falling below classifier assignment thresholds.

*Channel B (protein).* Six-frame ORF prediction on assembled contigs and high-complexity read clusters, followed by protein language model embedding (ESM-2) [4] and local-outlier-factor density estimation [5] against a manifold built from known viral proteomes. Because nearest-neighbour distances concentrate at an embedding dimension near 1280 [6], density is estimated under cosine distance on L2-normalised embeddings with prior dimension reduction, not raw Euclidean volume. This channel recovers agents whose nucleotide sequence has diverged past recognition but whose capsid, polymerase, or glycoprotein folds retain viral character — precisely the regime where Channel A fails.

Because Channel A scores contigs while Channel B scores individual ORFs, ORF-level evidence is aggregated within each contig before fusion, so that both channels enter the model as the same statistical unit. A logistic fusion combines the aligned features and their interaction into a scalar score, which an isotonic regression fitted on an independent split (Aim 2) maps to a per-contig **Novelty and Concern Score (NCS)** — an interpretable probability rather than an arbitrary index. Fusion, calibration, and reliability assessment are fitted on separate splits, or by nested cross-fitting, so reported probabilities are not optimistic; isotonic regression is benchmarked against Platt scaling and beta calibration and selected by cross-validated Brier score [7]. Calibration, not raw discrimination, is the deliverable that makes downstream alert thresholds defensible.

Implementation is Nextflow with Apptainer containers on SLURM, targeting hundreds of samples and billions of reads. Profiling on our existing GACRC pipelines indicates a per-sample marginal cost consistent with routine weekly operation at metropolitan-catchment scale; hard turnaround figures are a primary deliverable.

*Evidence verification.* A calibrated score is necessary but not sufficient: an operator still has to decide what a flagged contig is. Every candidate above threshold therefore passes through a fixed verification subworkflow before it is reported — read support and coverage breadth/depth against the assembled contig, remote-homology search (DIAMOND, MMseqs2), viral hallmark gene profiling (HMMER against Pfam/VOG), structure-level homology (Foldseek against AlphaFold DB) for cases where sequence homology has been erased, cross-tool concordance (geNomad, VirSorter2), and abundance trajectory for the same sequence cluster across prior samples. These are deterministic tools composed as a Nextflow subworkflow, so the same input yields the same evidence package and every line in it is traceable to a specific tool invocation.

The output is a structured **evidence-verified dossier**: the calibrated NCS, its confidence interval, each supporting and contradicting line of evidence, and the full tool log. This is what makes the score actionable rather than merely computed — and it is a reporting layer, not an alerting one. NOVA-MGS ranks and documents; a human decides.

### Aim 2 — WW-NOVEL-BENCH: open benchmark and retrospective validation (Months 2–12)

The field currently has no shared benchmark for novel-pathogen detection, which makes competing tool claims unverifiable and funder ROI unmeasurable. We will build one.

Real wastewater backgrounds from CASPER (PRJNA1247874), NWSS (PRJNA747181), Tisza et al. 2023 (PRJNA966185), and Wolfe et al. 2026 (PRJNA1438722) will be spiked in silico with three classes of ground truth: (i) real viral genomes with their clade removed from all reference databases, providing honest leave-one-clade-out difficulty; (ii) evolutionarily simulated divergent variants at 20%, 40%, and 60% nucleotide divergence; and (iii) genuine orphan viruses lacking database representation. Spike-ins span 10⁻⁴ to 10⁻⁸ relative abundance across Illumina, PacBio HiFi, and Nanopore platforms, matching our group's established multi-platform pipeline experience.

Leakage control is the part of benchmark design most often skipped, so we specify it explicitly: the entire clade of each spiked genome is removed from every reference, sketch, and training set; difficulty is then indexed continuously by ANI to the nearest retained neighbour rather than by a binary novel/known label, and results are reported per difficulty stratum rather than pooled. Pooling hides the sub-80% regime where a genuinely novel agent actually sits. A parallel time-split protocol — fit and calibrate only on sequences deposited before a cutoff date, evaluate strictly after it — tests generalisation without relying on synthetic divergence at all.

Because positives are a vanishing fraction of contigs, evaluation reports AUPRC rather than ROC-AUC [8], together with the analyst workload implied at the chosen threshold: the alert budget, not a default cutoff, sets the operating point. Probability quality is assessed separately from ranking via reliability diagrams and a Brier-score decomposition, since a well-ranked score can still be badly calibrated.

Retrospective validation will ask the operationally decisive question — *would this have caught it, and how much earlier?* — against mpox 2022 and H5N1-in-wastewater 2024 signals in archived public data.

The benchmark, evaluation harness, and a public leaderboard will be released independently of NOVA-MGS's own performance, so that competing tools benefit whether or not our method wins.

## 3. Deliverables and Open Availability

All software will be released under MIT/Apache-2.0 on GitHub from month 3, developed in the open rather than dumped at project end. Containers will be published to a public registry; the benchmark dataset and calibration data to Zenodo with a persistent DOI. We anticipate two peer-reviewed publications, both preprinted on bioRxiv/medRxiv at submission. Pipelines will follow nf-core conventions to lower the adoption barrier for programs already running nf-core/mag.

## 4. Responsible Disclosure

One optional extension of Aim 1 — statistical change-point screening for engineering signatures — carries a plausible information hazard, since publishing screening signatures in detail could inform evasion. We raise this proactively, and propose to develop it under a tiered disclosure agreement: operational capability shared with vetted programs, publication timing and method detail set jointly with Blueprint. We will descope it entirely if Blueprint prefers.

## 5. Team and Capability

**Justin Bahl, PhD — Principal Investigator.** Professor, [INSERT: department / center], University of Georgia. Prof. Bahl's group works on the phylodynamics and molecular epidemiology of emerging and zoonotic viruses. [INSERT: 1–2 sentences of specific track record — representative publications, and current or prior awards with funder, period, and value, which the RFP asks applicants to supply.] This expertise is load-bearing rather than nominal: the leave-one-clade-out design in Aim 2 turns on evolutionary judgments — what constitutes a clade, which divergence levels are realistic for a spillover lineage, which reference sets are honestly curated — that a purely computational team is not positioned to make. Prof. Bahl holds overall scientific and fiscal responsibility for the award.

**Sihua Peng, PhD — Project Scientist.** Research Scientist, College of Public Health, UGA; 40+ peer-reviewed publications (h-index 24) spanning computational biology and biostatistics. Dr. Peng leads implementation and day-to-day execution: current active work on metagenomic viral detection across Illumina, PacBio HiFi, and Nanopore platforms using nf-core/mag, sourmash, geNomad, VirSorter2, GOTTCHA2, and Kraken2/Bracken, at scale on the GACRC SLURM cluster via Nextflow and Apptainer. His training in biostatistics and experimental design is what the calibration and benchmark work actually rests on: the hard part of this proposal is not running the tools but establishing that the probabilities they emit mean what they claim to mean.

GACRC provides all required compute, storage, and container infrastructure; no new hardware is requested. We welcome introductions to operational partners (CASPER, Zephyr, NWSS-participating utilities) for real-world validation, and are open to teaming with complementary TA2 or TA3 proposals.

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

**Protein language model performance on fragmented environmental ORFs is unproven**, and high-dimensional density estimation is fragile in its own right. Both are addressed by a month-2 go/no-go test on Channel B covering the distance metric and dimension-reduction choices, not only the embeddings; Channel A alone supports a reduced-scope deliverable if Channel B underperforms.

**False-positive burden at operational thresholds.** A calibrated score does not by itself bound analyst workload. We therefore report, for every operating point, the implied candidates-per-week at realistic sample volumes, and set thresholds from that budget rather than from a default cutoff.

**Benchmark realism.** In-silico spike-ins can flatter detection methods. We mitigate with leave-one-clade-out real genomes and retrospective testing against historical signals, and state benchmark limitations in the main text.

## 8. References

1. Ondov BD, Treangen TJ, Melsted P, Mallonee AB, Bergman NH, Koren S, Phillippy AM. Mash: fast genome and metagenome distance estimation using MinHash. *Genome Biology* 2016;17:132.
2. Koslicki D, Zabeti H. Improving MinHash via the containment index with applications to metagenomic analysis. *Applied Mathematics and Computation* 2019;354:206–215.
3. Hera MR, Pierce-Ward NT, Koslicki D. Deriving confidence intervals for mutation rates across a wide range of evolutionary distances using FracMinHash. *Genome Research* 2023;33(7):1061–1068.
4. Lin Z, Akin H, Rao R, et al. Evolutionary-scale prediction of atomic-level protein structure with a language model. *Science* 2023;379(6637):1123–1130.
5. Breunig MM, Kriegel H-P, Ng RT, Sander J. LOF: identifying density-based local outliers. *Proceedings of ACM SIGMOD* 2000:93–104.
6. Zimek A, Schubert E, Kriegel H-P. A survey on unsupervised outlier detection in high-dimensional numerical data. *Statistical Analysis and Data Mining* 2012;5(5):363–387.
7. Niculescu-Mizil A, Caruana R. Predicting good probabilities with supervised learning. *Proceedings of ICML* 2005:625–632.
8. Saito T, Rehmsmeier M. The precision-recall plot is more informative than the ROC plot when evaluating binary classifiers on imbalanced datasets. *PLoS ONE* 2015;10(3):e0118432.

*Per the RFP, references do not contribute to the page limit.*

---

*Submitted in response to Blueprint Biosecurity's Pathogen-Agnostic Biothreat Detection RFP. Contact: [EMAIL], [PHONE].*
