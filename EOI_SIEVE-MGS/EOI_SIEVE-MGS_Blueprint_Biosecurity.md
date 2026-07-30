# Expression of Interest

**Blueprint Biosecurity — Request for Proposals: Pathogen-Agnostic Biothreat Detection**

**Technical Area:** 1 — Computational detection of known and novel pathogens in MGS data

**Project Title:** **SIEVE-MGS** — Distilled Read-Level Novelty Screening with Guaranteed Recall for Production-Scale Metagenomic Biosurveillance

**Principal Investigator:** Sihua Peng, PhD — Research Scientist, College of Public Health, University of Georgia; affiliated with the Georgia Advanced Computing Resource Center (GACRC)

**Institution:** University of Georgia, Athens, GA, USA

**Requested Period of Performance:** 12 months

**Approximate Budget Request:** $269,500 (direct + indirect; see Section 6)

**Date:** [INSERT]  |  **Contact:** [INSERT EMAIL / PHONE]

---

## 1. Problem Statement

A production metagenomic program generates on the order of 10¹⁰ reads per week. The models that are best at recognising a virus no one has seen before cannot be run on them.

This is a structural constraint, not a resource complaint. Genome foundation models have measurably improved novel-virus recognition — ViraLM, built on DNABERT-2, is the clearest demonstration [1] — but they cost milliseconds of GPU time per sequence, which is four to five orders of magnitude too slow for weekly operation at metropolitan-catchment scale. The field's standard response is to shrink the problem by assembling first and scoring contigs instead of reads. That response fails precisely where early warning lives: an agent at 10⁻⁷ relative abundance contributes a handful of reads to a sample and will never form a contig. Assembly is not a neutral efficiency step. It is a sensitivity floor, and it is placed exactly under the signal we are trying to catch.

Operational pipelines are therefore forced to choose between two options that both fail on the case of interest: fast reference-based classifiers that by construction cannot score what is absent from the database, and accurate reference-free models that cannot be afforded at scale and are usually gated behind the assembly step that destroys the signal. Deep read-level classifiers such as DeepVirFinder [2] sit between these poles but were trained before the foundation-model era and are not calibrated for novelty as distinct from viral character.

SIEVE-MGS dissolves the dilemma instead of splitting it, using one observation: **the cost of a foundation model is paid at training time, not at inference time, if the model is used as a teacher rather than as a predictor.** We distil foundation-model representations into a compact read-level screener that runs at production throughput without a reference index, and — because a filter that discards 99.99% of the data is only deployable if you can bound what it discarded — we attach a provable upper bound on the fraction of true novel reads it throws away.

The result is not a replacement for anything. It is a pre-filter that any existing program can place in front of the stack it already runs.

## 2. Technical Approach

### Aim 1 — Distilling a read-level novelty model with a recall guarantee (Months 1–7)

**Why distillation rather than a small model trained directly.** This is the scientific core of the project and the one claim that can be falsified. A compact network trained from scratch on leave-one-clade-out data learns what the clades it was shown look like, and degrades on clades genuinely outside that experience. Forcing the student to match the *representation geometry* of a foundation-model teacher [6,7] transfers the abstract structure of "viral-ness" rather than memorised clade identity. The training objective therefore carries two terms: a supervised loss on held-out-clade simulated data, and a representation-matching loss against frozen teacher embeddings. **A month-3 ablation tests directly whether the representation-matching term is the source of cross-clade generalisation.** If it is not, the premise of this project is wrong and we will know early enough to act.

**Choice of teacher is a technical decision, not a default.** The obvious candidate is the wrong one: Evo 2 deliberately excluded genomic sequences of viruses infecting eukaryotic hosts from its training corpus as a biosecurity measure, and consequently exhibits high perplexity on exactly the eukaryotic viral sequences that matter here [3]. We therefore build the teacher as an ensemble of a protein channel (ESM-2 [5] applied to six-frame translations of reads, which retains signal where nucleotide identity has diverged past recognition) and a nucleotide channel from the DNABERT-2/ViraLM and Nucleotide Transformer families [1,4]. The teacher is used offline, on training data only, and is not part of the deliverable.

**Labels come from construction, not from the teacher.** Entire clades are removed from every reference, training, and teacher-fitting set; reads are then simulated from the held-out genomes under realistic platform error profiles at 75–300 bp and embedded in real wastewater backgrounds. Ground truth is exact by construction. Difficulty is indexed continuously by ANI to the nearest retained neighbour rather than by a binary novel/known label, and all results are reported per difficulty stratum, because pooling hides the sub-80% regime where a genuinely novel agent actually sits.

**The model has two heads, because most of the unknown in wastewater is not viral.** We estimate p(viral) and p(outside known viral space) separately; only reads scoring high on both enter the retained set. Architecturally the student is a reverse-complement-equivariant dilated convolutional network [8] of ≤5M parameters, quantised to INT8, handling both strands and both nucleic-acid types, with long reads processed as overlapping chunks.

**The guarantee is what makes it deployable.** Using conformal risk control [9], we select the retention threshold λ so that the expected fraction of true novel reads falling in the *discarded* set is at most ε with confidence 1−δ. The headline deliverable of Aim 1 is not a single operating point but the **(ε, compression-ratio) curve**: state the miss rate you can tolerate, and we state how many orders of magnitude of data disappear.

We will also state plainly where the guarantee weakens. Conformal risk control assumes exchangeability, and catchments, seasons, and library chemistries are not exchangeable with one another. We therefore calibrate per site on a rolling window and report, as a named deliverable rather than an appendix, how far the bound degrades under deliberate distribution shift across sites, seasons, and sequencing platforms.

### Aim 2 — Production characterisation and drop-in deployment (Months 4–12)

The decisive experiment is one sentence: **feed a downstream pipeline 0.01% of the reads and ask whether it still finds the same things.** We install SIEVE-MGS in front of nf-core/mag, mgs-workflow, Kraken2/Bracken, and geNomad, and compare full-input against screened-input results for concordance of recovered taxa, contigs, and flagged candidates, alongside end-to-end wall-clock and dollar cost. "The same answers for a thousandth of the compute" is the claim that determines whether a program such as CDC Biothreat Radar, NWSS, or mSCAPE can actually adopt this, and it is measured rather than asserted.

Three further characterisations run in parallel.

*Throughput and cost.* Reads per second per GPU and per CPU core, dollars per billion reads on commodity cloud, resident memory, and a streaming mode that scores reads as they come off the instrument with no index to load. Many state public health laboratories have no GPU at all, so a pure-CPU path is a requirement rather than a courtesy. Our target is ≥5×10⁵ reads/s on an A100-class GPU — a ten-billion-read weekly run screened in roughly five GPU-hours — but the measured figure is itself a primary deliverable, and we will publish it whatever it turns out to be.

*Retention on real backgrounds.* A 10⁴× compression ratio on simulated data proves nothing. We measure retention on CASPER (PRJNA1247874), NWSS (PRJNA747181), Tisza et al. 2023 (PRJNA966185), Wolfe et al. 2026 (PRJNA1438722), and Zephyr/Broad pooled nasal swabs (PRJNA1052714), which also tests whether the screener transfers across sample types with different complexity and host fraction. If real wastewater compresses only 100×, we report 100×, because 100× is still operationally decisive.

*Recall on historical emergence events.* For mpox 2022 and H5N1-in-wastewater 2024, we ask the narrow question this project is responsible for: at the earliest timepoints where the relevant reads exist in archived data, **did the screener retain them?** Whether and when to alert is a different question, and not one this proposal claims to answer.

## 3. Deliverables and Open Availability

`sieve` will be released under MIT/Apache-2.0 on GitHub from month 3, developed in the open rather than dumped at project end: PyTorch training code, ONNX/TensorRT deployment paths, a pure-CPU fallback, and an nf-core-conventional Nextflow module so that programs already running nf-core/mag can adopt it by adding one step. Pretrained weights and the per-site calibration artefacts go to Hugging Face and to Zenodo with a persistent DOI, together with the throughput and retention benchmark data. We anticipate two peer-reviewed publications, both preprinted at submission.

We deliberately do not propose to build another benchmark dataset. Where shared benchmarks for novel-pathogen detection exist or emerge, we will evaluate against them and contribute our leave-one-clade-out splits back; the contribution here is the model, the guarantee, and the measured operating cost.

## 4. Speed of Execution and Risk Posture

The project requires no wet-lab work, no new sample collection, and no new hardware. Every primary dataset is public and already downloaded by our group; GACRC provides GPU, CPU, storage, and container infrastructure. Analysis begins in week one. The month-3 ablation is a genuine go/no-go with a defined fallback rather than a checkpoint that cannot fail, and the retention measurement on real data is deliberately scheduled at month 2, before substantial effort is committed to a compression ratio that real backgrounds may not support.

## 5. Team and Capability

The PI has 40+ peer-reviewed publications (h-index 24) spanning computational biology and biostatistics, combining formal training in statistical methodology and experimental design with current production metagenomics work across Illumina, PacBio HiFi, and Nanopore platforms. Existing pipelines run at scale on the GACRC SLURM cluster via Nextflow and Apptainer, using sourmash, geNomad, VirSorter2, GOTTCHA2, and Kraken2/Bracken. The combination this project requires is unusual: deep-learning engineering on one side and distribution-free uncertainty quantification on the other, joined to hands-on experience of what a metagenomic pipeline actually costs to run at scale. The guarantee in Aim 1 is not decoration — it is the reason a public health program would be willing to discard 99.99% of its data, and stating it correctly is a statistics problem before it is a machine-learning problem.

GACRC provides all compute, storage, and container infrastructure; no new hardware is requested. We welcome Blueprint's introduction to operational partners (NWSS-participating utilities, Georgia DPH, CASPER, Zephyr) for deployment testing, and are open to teaming with groups submitting complementary TA1 or TA2 proposals — a calibrated pre-filter is a natural integration point for any downstream detection method.

*Companion submission disclosure:* our group is separately submitting two other EOIs — **NOVA-MGS** (TA1, reference-free novelty scoring and evidence verification on assembled contigs) and **DPD** (TA3, sequential detection statistics and budget-to-lead-time modelling). SIEVE-MGS is scientifically independent of both and differs in the dimension that matters: it operates on unassembled reads rather than contigs, its objective is throughput under a recall guarantee rather than calibrated ranking, and its deliverable is a filter that precedes any detection stack rather than a detector. Each of the three is fully self-contained, independently scoped, and independently budgeted. None depends on any other being funded.

## 6. Budget Summary (approximate, 12 months)

| Category | Amount |
|---|---|
| PI effort (25% FTE) | $40,000 |
| Postdoctoral researcher / machine learning scientist (100% FTE) | $82,000 |
| Graduate research assistant (50% FTE, incl. tuition) | $46,000 |
| Fringe benefits | $37,000 |
| GPU compute for teacher inference, distillation, and throughput benchmarking | $30,000 |
| Publication, dissemination, travel | $10,000 |
| **Total direct** | **$245,000** |
| Indirect costs (10% of direct, per Blueprint policy) | $24,500 |
| **Total requested** | **$269,500** |

The compute line is the largest non-personnel item and is genuinely required: generating teacher embeddings over the training corpus and characterising inference throughput on multiple hardware classes are both GPU-bound, and the throughput numbers are worthless if measured on a single device type. GACRC provides cluster compute, storage, and container infrastructure to the PI at no charge, so this line covers cloud burst capacity and archival hosting only. Indirect costs are budgeted at Blueprint's published 10% cap; the University of Georgia's negotiated F&A rate exceeds this, and the institutional waiver request is being initiated through the Office of Research now rather than at Full Proposal stage. The project is modular: **Aim 1 alone constitutes a coherent standalone effort at approximately $175,000** should Blueprint prefer a tighter scope.

## 7. Risks and Mitigations

**Representation distillation may not transfer across clades.** This is the project's central assumption and is tested first. The month-3 ablation compares the distilled student against an identically sized model trained without the representation-matching term, stratified by ANI to nearest retained neighbour. If the term buys nothing, the fallback is a larger student — still two orders of magnitude faster than the teacher — and the finding itself is publishable and useful to the field.

**Real backgrounds may compress far less than simulated ones.** Measured at month 2 on real wastewater rather than at month 10. We report the achieved compression honestly; the deliverable is the (ε, compression) curve, not a marketing number.

**Exchangeability failure weakens the conformal bound.** Mitigated by per-site rolling calibration, and quantified by deliberate shift stress tests across site, season, and platform. Where the bound does not hold, we say so and report the empirical degradation.

**A sufficiently divergent agent may evade any learned representation.** The screener is deliberately configured for high recall at low precision: we bound what is discarded, not what is retained. Controlling false positives is the downstream stack's job, which is exactly why this is proposed as a pre-filter and not as a detector.

**Read-level scores are weak individually.** Single reads carry little information, so scores are aggregated to k-mer or minimiser-defined read clusters before any candidate is emitted, and both read-level and cluster-level operating characteristics are reported.

## 8. References

1. Peng C, Shang J, Guan J, Wang D, Sun Y. ViraLM: empowering virus discovery through the genome foundation model. *Bioinformatics* 2024;40(12):btae704.
2. Ren J, Song K, Deng C, Ahlgren NA, Fuhrman JA, Li Y, Xie X, Poplin R, Sun F. Identifying viruses from metagenomic data using deep learning. *Quantitative Biology* 2020;8(1):64–77.
3. Brixi G, Durrant MG, Ku J, et al. Genome modelling and design across all domains of life with Evo 2. *Nature* 2026. (Preprint: *bioRxiv* 2025.02.18.638918.)
4. Dalla-Torre H, Gonzalez L, Mendoza-Revilla J, et al. Nucleotide Transformer: building and evaluating robust foundation models for human genomics. *Nature Methods* 2025;22(2):287–297.
5. Lin Z, Akin H, Rao R, et al. Evolutionary-scale prediction of atomic-level protein structure with a language model. *Science* 2023;379(6637):1123–1130.
6. Hinton G, Vinyals O, Dean J. Distilling the knowledge in a neural network. *arXiv:1503.02531*, 2015.
7. Romero A, Ballas N, Kahou SE, Chassang A, Gatta C, Bengio Y. FitNets: hints for thin deep nets. *Proceedings of ICLR* 2015.
8. Shrikumar A, Greenside P, Kundaje A. Reverse-complement parameter sharing improves deep learning models for genomics. *bioRxiv* 103663, 2017.
9. Angelopoulos AN, Bates S, Fisch A, Lei L, Schuster T. Conformal risk control. *Proceedings of ICLR* 2024.
10. Saito T, Rehmsmeier M. The precision-recall plot is more informative than the ROC plot when evaluating binary classifiers on imbalanced datasets. *PLoS ONE* 2015;10(3):e0118432.

*Per the RFP, references do not contribute to the page limit.*

---
