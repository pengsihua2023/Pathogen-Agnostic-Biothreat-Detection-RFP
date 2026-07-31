# Expression of Interest

**Blueprint Biosecurity — Request for Proposals: Pathogen-Agnostic Biothreat Detection**

**Technical Area:** 1 — Computational detection of known and novel pathogens in MGS data

**Project Title:** **SIEVE-MGS** — Distilled Read-Level Novelty Screening with Guaranteed Recall for Production-Scale Metagenomic Biosurveillance

**Principal Investigator:** Sihua Peng, PhD — Research Scientist, College of Public Health, University of Georgia; affiliated with the Georgia Advanced Computing Resource Center (GACRC)

**Institution:** University of Georgia, Athens, GA, USA

**Requested Period of Performance:** 12 months

**Approximate Budget Request:** $269,500 (direct + indirect; see Section 5)

**Date:** [INSERT]  |  **Contact:** [INSERT EMAIL / PHONE]

---

## 1. Problem Statement

A production metagenomic program generates on the order of 10¹⁰ reads per week. The models that are best at recognising a virus no one has seen before cannot be run on them.

This is a structural constraint, not a resource complaint. Foundation models have measurably improved novel-virus recognition — ViraLM is the clearest demonstration [1] — but cost milliseconds per sequence, four to five orders of magnitude too slow for weekly catchment-scale operation. The standard response is to assemble first and score contigs. That fails precisely where early warning lives: an agent at 10⁻⁷ relative abundance contributes a handful of reads and will never form a contig. Assembly is not a neutral efficiency step — it is a sensitivity floor, placed exactly under the signal we are trying to catch.

Pipelines are thus forced between two options that both fail on the case of interest: fast reference-based classifiers that cannot score what is absent from the database, and accurate reference-free models that cannot be afforded at scale. Read-level classifiers such as DeepVirFinder [2] sit between these poles but predate the foundation-model era and do not separate novelty from viral character.

SIEVE-MGS dissolves the dilemma rather than splitting it, on one observation: **the cost of a foundation model is paid at training time, not at inference time, if the model is used as a teacher rather than as a predictor.** We distil those representations into a compact read-level screener that runs at production throughput without a reference index, and — because a filter discarding 99.99% of the data is only deployable if you can bound what it discarded — we attach a provable upper bound on the true novel reads it throws away. It replaces nothing: it is a pre-filter any program can place in front of its existing stack.

## 2. Technical Approach

### Aim 1 — Distilling a read-level novelty model with a recall guarantee (Months 1–7)

![Figure 1. Aim 1 workflow for distilling a compact read-level novelty model from a frozen ESM-2 teacher through representation matching, supervised learning, and positive–unlabeled learning.](figures/aim1-distillation-framework-new-final-2.png){width=3.86in}

**Why distillation rather than a small model trained directly.** The project's core claim, and the one that can be falsified. A compact network trained from scratch on leave-one-clade-out data learns what the clades it was shown look like, and degrades outside that experience. Forcing the student to match the *representation geometry* of a foundation-model teacher [6,7] transfers the abstract structure of "viral-ness" rather than memorised clade identity. The objective carries two terms — a supervised loss on the constructed labels, plus representation matching against the stored teacher embeddings — and **a month-3 ablation tests directly whether the representation-matching term is the source of cross-clade generalisation**; if it is not, the premise of this project is wrong and we will know early enough to act. Note what the student learns: **it never translates anything and never sees a protein.** It learns to recognise, directly in raw nucleotide space, the protein-level character ESM-2 sees only after translation.

**The teacher is a single protein channel, and that is a technical decision.** Nucleotide language models are unusable here for two independent reasons: their corpora under-represent what we need — Evo 2 deliberately excluded viruses infecting eukaryotic hosts as a biosecurity measure and shows high perplexity on exactly those sequences [3], and the Nucleotide Transformer family is built from host rather than viral genomes [4] — and a 150 bp read is 30–40 tokens against models trained for kilobase context, in a regime where below 80% ANI the signal has largely left the nucleotide alphabet anyway. Protein has not. The teacher is therefore **ESM-2 [5] alone**, applied to all six translation frames and pooled across them: its UniRef corpus does contain viral proteins, and its representations survive the divergence range where early warning operates. Reads with no coding content are not a blind spot: all six frames embed far from protein space. Embedding quality at ~50 aa is a month-1 check. The teacher is frozen — never fine-tuned, so its representation is a fixed prior rather than something fitted to the retained clades — runs offline on training data only, and is not a deliverable.

**Labels come from construction, not from the teacher.** Entire clades are removed from every reference, training and teacher-fitting set; reads are simulated from held-out genomes under realistic error profiles at 75–300 bp and embedded in real wastewater background, so ground truth is exact. Difficulty is indexed by protein-level identity (AAI) to the nearest retained neighbour — nucleotide ANI is unmeasurable below ~80%, precisely where a genuinely novel agent sits — and reported per stratum rather than pooled.

**The model has two heads, and the negative class is not clean.** We estimate p(vertebrate-infecting) and p(outside known viral space) separately — phage dominates the wastewater virome and is background, not signal — and retain only reads high on both. Real background is not clean: it contains known viruses, which we relabel by reference matching, and unknown ones, which we cannot identify and which are precisely our target. We therefore train under a positive–unlabeled formulation [9] and estimate the residual positive fraction, so reported false-positive rates are not inflated by correct detections. The student is a reverse-complement-equivariant dilated CNN [8] of ≤5M parameters, INT8-quantised, handling both strands and nucleic-acid types.

**The guarantee is what makes it deployable.** Using conformal risk control [10] we select the retention threshold λ so that the expected fraction of true novel reads in the *discarded* set is at most ε with confidence 1−δ. The headline deliverable is not a single operating point but the **(ε, compression-ratio) curve**: state the miss rate you can tolerate, and we state how many orders of magnitude of data disappear. We also state where it weakens: conformal risk control assumes exchangeability, and catchments, seasons and library chemistries are not, so we calibrate per site and report how far the bound degrades under deliberate shift.

### Aim 2 — Production characterisation and drop-in deployment (Months 4–12)

The decisive experiment is one sentence: **feed a downstream pipeline 0.01% of the reads and ask whether it still finds the same things.** We install SIEVE-MGS in front of mgs-workflow, TaxTriage, Kraken2/Bracken and geNomad and compare full-input against screened-input results for concordance of recovered taxa, contigs and flagged candidates, alongside wall-clock and cost. "The same answers for a thousandth of the compute" is what determines whether CDC Biothreat Radar, NWSS or mSCAPE can adopt this, and it is measured rather than asserted.

Three further characterisations run in parallel.

*Throughput and cost.* Reads per second per GPU and per CPU core, dollars per billion reads, resident memory, and a streaming mode with no index to load. Many state public health laboratories have no GPU, so a pure-CPU path is a requirement. We target ≥5×10⁵ reads/s on an A100-class GPU, but the measured figure is itself a primary deliverable, published whatever it turns out to be.

*Retention on real backgrounds.* A 10⁴× compression on simulated data proves nothing. We measure retention on CASPER (PRJNA1247874), NWSS (PRJNA747181), Tisza et al. 2023 (PRJNA966185), Wolfe et al. 2026 (PRJNA1438722) and Zephyr/Broad nasal swabs (PRJNA1052714), the last also testing cross-sample-type transfer. If real wastewater compresses only 100×, we report 100× — still operationally decisive.

*Recall on historical emergence events.* For mpox 2022 and H5N1-in-wastewater 2024 we ask the narrow question this project owns: at the earliest timepoints where relevant reads exist in archived data, **did the screener retain them?** Whether and when to alert is a different question this proposal does not claim to answer.

## 3. Deliverables, Open Availability, and Execution Speed

`sieve` will be released under MIT/Apache-2.0 on GitHub from month 3, developed in the open rather than dumped at project end: PyTorch training code, ONNX/TensorRT deployment, a pure-CPU fallback and an nf-core-conventional Nextflow module, so an existing Nextflow pipeline adopts it by adding one step. Weights, calibration artefacts and benchmarks go to Hugging Face and Zenodo with a DOI. Two publications are anticipated, both preprinted at submission.

We do not propose another benchmark: where shared benchmarks exist or emerge we evaluate against them and contribute our leave-one-clade-out splits back. On execution speed: no wet-lab work, no new sample collection, no new hardware; every dataset is public and already downloaded, so analysis begins in week one.

## 4. Team and Capability

**Sihua Peng, PhD — Principal Investigator.** Research Scientist, College of Public Health, University of Georgia; 40+ peer-reviewed publications (h-index 24) across computational biology, biostatistics, and machine learning. The deep-learning half of this project is demonstrated rather than asserted: the PI develops and publicly releases fine-tuned protein and biological language models (huggingface.co/sihuapeng), several at the 3B-parameter scale — the same model family as Aim 1's teacher — and that release history is the most direct evidence we can offer for this RFP's open-availability criterion: `sieve`'s weights and calibration artefacts will be published the same way. The PI also runs production metagenomics across Illumina, PacBio HiFi and Nanopore platforms at scale on the GACRC SLURM cluster. The combination is rare: protein-language-model engineering, distribution-free uncertainty quantification, and hands-on knowledge of what a metagenomic pipeline costs. The guarantee in Aim 1 is not decoration — it is why a public health program would be willing to discard 99.99% of its data, and stating it correctly is a statistics problem before a machine-learning one.

We welcome Blueprint's introduction to operational partners (NWSS utilities, Georgia DPH, CASPER, Zephyr) and are open to teaming with complementary TA1 or TA2 proposals.

*Companion submission disclosure:* our group is separately submitting a TA3 EOI (**LEAD-MGS**, sequential detection statistics and budget-to-lead-time modelling), led by Prof. Justin Bahl as PI. The two are complementary but independent — SIEVE-MGS decides which reads are worth keeping, LEAD-MGS when a trajectory warrants an alert and what it costs — and each is self-contained, independently scoped, budgeted and led. Neither depends on the other being funded.

## 5. Budget Summary (approximate, 12 months)

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

The compute line is genuinely required: teacher embeddings and cross-hardware throughput characterisation are both GPU-bound. GACRC provides cluster compute and storage at no charge, so it covers cloud burst and archival hosting only. Indirect costs are at Blueprint's 10% cap; UGA's negotiated F&A rate exceeds this and the institutional waiver is being initiated now rather than at Full Proposal stage. The project is modular: **Aim 1 alone is a coherent standalone effort at approximately $175,000**.

## 6. Risks and Mitigations

**Distillation may not transfer across clades.** The central assumption, tested first: a month-3 ablation against an identically sized model trained without the representation-matching term, stratified by AAI. If it buys nothing, the fallback is a larger student — still two orders faster than the teacher — and the result is itself publishable.

**Real backgrounds may compress less than simulated ones, and exchangeability failure weakens the conformal bound.** The first is measured at month 2, not month 10; the deliverable is the (ε, compression) curve, not a marketing number. The second is handled by per-site rolling calibration with degradation quantified under shift stress tests. A very divergent agent may still evade any learned representation — which is why the screener is deliberately high-recall, low-precision: we bound what is discarded, not what is retained.

## 7. References

1. Peng C, Shang J, Guan J, Wang D, Sun Y. ViraLM: empowering virus discovery through the genome foundation model. *Bioinformatics* 2024;40(12):btae704.
2. Ren J, Song K, Deng C, Ahlgren NA, Fuhrman JA, Li Y, Xie X, Poplin R, Sun F. Identifying viruses from metagenomic data using deep learning. *Quantitative Biology* 2020;8(1):64–77.
3. Brixi G, Durrant MG, Ku J, et al. Genome modelling and design across all domains of life with Evo 2. *Nature* 2026. (Preprint: *bioRxiv* 2025.02.18.638918.)
4. Dalla-Torre H, Gonzalez L, Mendoza-Revilla J, et al. Nucleotide Transformer: building and evaluating robust foundation models for human genomics. *Nature Methods* 2025;22(2):287–297.
5. Lin Z, Akin H, Rao R, et al. Evolutionary-scale prediction of atomic-level protein structure with a language model. *Science* 2023;379(6637):1123–1130.
6. Hinton G, Vinyals O, Dean J. Distilling the knowledge in a neural network. *arXiv:1503.02531*, 2015.
7. Romero A, Ballas N, Kahou SE, Chassang A, Gatta C, Bengio Y. FitNets: hints for thin deep nets. *Proceedings of ICLR* 2015.
8. Shrikumar A, Greenside P, Kundaje A. Reverse-complement parameter sharing improves deep learning models for genomics. *bioRxiv* 103663, 2017.
9. Kiryo R, Niu G, du Plessis MC, Sugiyama M. Positive-unlabeled learning with non-negative risk estimator. *Advances in Neural Information Processing Systems* 2017;30.
10. Angelopoulos AN, Bates S, Fisch A, Lei L, Schuster T. Conformal risk control. *Proceedings of ICLR* 2024.
11. Saito T, Rehmsmeier M. The precision-recall plot is more informative than the ROC plot when evaluating binary classifiers on imbalanced datasets. *PLoS ONE* 2015;10(3):e0118432.

*Per the RFP, references do not contribute to the page limit.*

---
