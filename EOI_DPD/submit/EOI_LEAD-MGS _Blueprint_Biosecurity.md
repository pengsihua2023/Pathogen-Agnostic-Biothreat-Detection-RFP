# Expression of Interest

**Blueprint Biosecurity — Request for Proposals: Pathogen-Agnostic Biothreat Detection**

**Technical Area:** 3 — Modeling pathogen-agnostic biosurveillance system features such as cost, sensitivity, and optimal deployment

**Project Title:** **LEAD-MGS** — Lead-time Estimation and Deployment Optimisation: Sequential Detection Statistics and Calibrated Budget-to-Lead-Time Modeling for Metagenomic Biosurveillance

**Principal Investigator:** Justin Bahl, PhD — Professor, [INSERT: department / center], University of Georgia

**Project Scientist:** Sihua Peng, PhD — Research Scientist, College of Public Health, University of Georgia; affiliated with the Georgia Advanced Computing Resource Center (GACRC)

**Institution:** University of Georgia, Athens, GA, USA

**Requested Period of Performance:** 12 months

**Approximate Budget Request:** $267,300 (direct + indirect; see Section 6)

**Date:** [INSERT]  |  **Contact:** [INSERT EMAIL / PHONE]

---

## 1. Problem Statement

Every operational biosurveillance program faces the same budget meeting: someone asks what another million dollars buys, and no one can answer in days of warning. The question is not rhetorical — it determines site counts, sampling cadence, and sequencing depth for CDC Biothreat Radar, NWSS, TGS, and their international counterparts — and it is currently answered by intuition.

Two gaps make it unanswerable. First, **the detection endpoint itself is undefined.** Cost-sensitivity models must assume a bioinformatic detection threshold, because no one has characterized one operationally: three reads at 10⁻⁷ relative abundance in one sample cannot be distinguished from noise, and essentially all deployed MGS analysis is single-sample, discarding the time dimension where the signal actually lives. An agent growing exponentially produces a *trajectory*, and a trajectory is detectable long before any single timepoint clears a threshold. Second, **the modeling chain is broken at the joints.** Shedding models, P2RA relative-abundance factors, cost models, and siting analyses exist as separate literatures with incompatible parameterizations, so no one can propagate uncertainty from a shedding parameter through to a deployment decision.

LEAD-MGS closes both. We will build a sequential detection framework whose *measured* operating characteristic — detection delay at a fixed false-alert rate — becomes the calibration input to a differentiable end-to-end simulator that maps budget directly to lead time, then validate the whole chain against events that actually happened.

## 2. Technical Approach

### Aim 1 — Anytime-valid sequential detection on MGS time series (Months 1–7)

We will treat detection as a **sequential change-point problem over thousands of parallel hypotheses**, not a per-sample classification.

*Cluster construction.* Sequence clusters are built by this project, not assumed: reads are assigned by a fixed minimizer/k-mer hashing scheme frozen at project start, so cluster identity is stable across time by construction rather than by post-hoc matching — the precondition without which Yi(t) is not a time series at all. We deliberately avoid assembly-based clustering, which fails at the 10⁻⁷ abundances where early warning lives. The step is streaming and single-pass at roughly 10⁷–10⁸ bp/s/core, so a program already processing billions of read pairs per day adds a couple of core-hours and no GPU; the sequential layer on top is O(1) per cluster per timepoint and costs seconds. A cold-start rule handles clusters first appearing mid-series, and building the count time series for the NWSS and CASPER archives is a one-time cost borne on the GACRC cluster.

*Observation model.* Read counts for each cluster are modeled as negative-binomial / Dirichlet-multinomial conditional on library depth, with explicit terms for run and batch effects, fecal-strength normalization (PMMoV, crAssphage), flow and dilution covariates, and extraction-control efficiency. Compositionality is handled directly rather than assumed away — a taxon's relative abundance can fall while its absolute load rises, and ignoring this manufactures both false alarms and false reassurance. We systematically compare normalization strategies, itself an open question the RFP names.

*Sequential test.* Bayesian online change-point detection and CUSUM on log-abundance trajectories, plus **e-value / test-martingale sequential tests that are anytime-valid**. This is methodologically load-bearing: an operational program looks at its data every week, and classical fixed-sample FDR control is invalid under repeated looks. E-processes with e-BH multiplicity control give valid inference at every peek, at a quantified cost in power we report rather than hide. Site-specific empirical nulls are built from historical baselines so that thresholds are calibrated to each catchment rather than imported.

*Primary metric.* Not AUC. We report the **detection-delay distribution at a fixed false-alert rate** — the surveillance analogue of average run length — because that is the quantity an operator and a funder can both act on. We also deliver growth-rate and doubling-time estimates with honest uncertainty, directly answering the RFP's question about a taxon's trajectory.

*Data.* NWSS (PRJNA747181) and CASPER (PRJNA1247874) time series as the primary substrate, Tisza et al. 2023 (PRJNA966185) and Wolfe et al. 2026 (PRJNA1438722) for capture-based contrast, and Zephyr/Broad nasal swabs (PRJNA1052714) to test transfer across sample types with different shedding and pooling structure.

### Aim 2 — `leadtime`: a differentiable end-to-end budget-to-lead-time simulator (Months 3–11)

We will implement the full causal chain as a single differentiable JAX pipeline: pathogen introduction → epidemic growth (branching process with overdispersion *k*, plus SEIR comparison) → sample-type-specific shedding kinetics → catchment capture and collection cadence → wet-lab efficiency including extraction, library complexity, and duplication → sequencing depth and read allocation → **bioinformatic sensitivity curve calibrated by Aim 1's measured operating characteristic** → alerting rule → **lead-time distribution relative to status-quo syndromic detection**.

The coupling to Aim 1 is the point of the project. Existing cost models must *assume* a detection threshold; ours *measures* one on real wastewater and real pooled swabs, so the resulting dollar-to-days curve rests on an empirical foundation rather than a plausible-sounding constant.

We build on and extend Grimm et al. 2025 (P2RA), generalizing a point relative-abundance factor into a time-resolved, uncertainty-propagating quantity, cross-checked against EpiSewer- and wwinference-style formulations adapted to the compositional MGS setting. Because the pipeline is differentiable, budget allocation becomes gradient-based constrained optimization over site count, cadence, depth, and sample-type mix — rather than the coarse grid search current practice permits.

Parameter uncertainty is treated as a deliverable, not an embarrassment: Sobol global sensitivity analysis identifies which parameters actually move the lead-time answer, telling Blueprint where measurement investment would most reduce decision uncertainty. Outputs are intervals; we decline to publish point estimates the data do not support.

### Aim 3 — Empirical deployment optimization and retrospective validation (Months 5–12)

A national optimizer with no real constraints is a toy. We ground the model in one real region.

Integrating NWSS site metadata, census demographics, LODES commuting flows, Hartsfield–Jackson passenger volumes (the world's busiest hub and a CDC TGS site), and hospital referral networks, we formulate joint maximum-coverage siting and pooling optimization for Georgia and the Southeast — a framework any state can re-parameterize.

Retrospective validation asks the only question that settles the matter: **would this have caught it, and how much earlier?** We test the optimized site set plus Aim 1's alerting rule against mpox 2022 and H5N1-in-wastewater 2024 in archived data. To prevent hindsight bias, the comparison definition — reference detection dates, alert criteria, success thresholds — is written and timestamped before the analysis is run.

## 3. Deliverables and Open Availability

Three open-source packages released under MIT/Apache-2.0 on GitHub from month 3, developed in the open rather than dumped at project end: **`mgs-sentry`** (Aim 1 sequential detection, R and Python, nf-core-compatible), **`leadtime`** (Aim 2 JAX simulator with an interactive dashboard), and **`sitewise`** (Aim 3 siting optimizer). Calibrated parameter sets, benchmark time series, and all retrospective analysis code go to Zenodo with a persistent DOI. We anticipate two to three publications, each preprinted at submission, plus a plain-language policy brief for program managers rather than methodologists — the audience that actually sets sampling cadence. All outputs are usable independently of one another and of any tool we build.

## 4. Speed of Execution and Risk Posture

The project requires no wet-lab work, no new sample collection, and no new hardware; every primary dataset is public and already downloaded. Analysis begins in week one, and the 12-month timeline carries genuine slack. Aim 1 is method development on data in hand; Aim 2 is simulation; Aim 3's only external dependency — site metadata beyond what is public — is initiated in month 1 and, if it stalls, degrades gracefully to fully public NWSS data without losing the retrospective validation.

## 5. Team and Capability

**Justin Bahl, PhD — Principal Investigator.** Professor, [INSERT: department / center], University of Georgia. Prof. Bahl's group works on the phylodynamics and molecular epidemiology of emerging and zoonotic viruses. [INSERT: 1–2 sentences of specific track record — representative publications, and current or prior awards with funder, period, and value, which the RFP asks applicants to supply.] This expertise is load-bearing rather than nominal. Aim 2's epidemic layer — branching-process growth with overdispersion *k*, shedding kinetics, the SEIR comparison — is an epidemiological model before it is a simulation, and its parameter ranges must be defensible rather than convenient. Aim 3 turns on the same judgment: fixing the true onset of the mpox 2022 and H5N1 2024 signals is what decides whether "we would have caught it X days earlier" means anything at all. Prof. Bahl holds overall scientific and fiscal responsibility for the award.

**Sihua Peng, PhD — Project Scientist.** Research Scientist, College of Public Health, UGA; 40+ peer-reviewed publications (h-index 24) spanning computational biology and biostatistics, with formal training and publication record in statistical methodology and experimental design. Dr. Peng leads implementation and day-to-day execution across all three aims, and brings current production metagenomics work across Illumina, PacBio HiFi, and Nanopore platforms, running at scale on the GACRC SLURM cluster via Nextflow and Apptainer with sourmash, geNomad, VirSorter2, and Kraken2/Bracken. The combination that matters here is rarer than either half: sequential-testing and multiplicity-control statistics on one side, hands-on operational MGS pipeline engineering on the other, which is what allows Aim 2's cost model to be calibrated by Aim 1's measurements rather than by literature guesses.

GACRC provides all compute, storage, and container infrastructure; no new hardware is requested. We welcome Blueprint's introduction to operational partners (NWSS-participating utilities, Georgia DPH, CASPER, Zephyr) for parameterization and validation, and are open to teaming with groups submitting complementary TA1 or TA2 proposals — the `leadtime` sensitivity curve is a natural integration point for any TA1 detection method.

*Companion submission disclosure:* our group is separately submitting a TA1 EOI (**SIEVE-MGS**, distilled read-level novelty screening with a guaranteed recall bound), led by Dr. Peng as Principal Investigator. The two are complementary but independent: SIEVE-MGS decides which reads are worth keeping, LEAD-MGS decides when a trajectory warrants an alert and what that capability costs. Each is fully self-contained, independently scoped, independently budgeted, and separately led. Neither depends on the other being funded.

## 6. Budget Summary (approximate, 12 months)

| Category | Amount |
|---|---|
| PI effort (30% FTE) | $47,000 |
| Project Scientist (Dr. Peng, 50% FTE) + computational scientist (50% FTE) | $78,000 |
| Graduate research assistant (50% FTE, incl. tuition) | $46,000 |
| Fringe benefits | $38,000 |
| Compute, storage, and GPU cloud burst capacity | $22,000 |
| Data acquisition, publication, dissemination, travel | $12,000 |
| **Total direct** | **$243,000** |
| Indirect costs (10% of direct, per Blueprint policy) | $24,300 |
| **Total requested** | **$267,300** |

Indirect costs are budgeted at Blueprint's published 10% cap; the University of Georgia's negotiated F&A rate exceeds this, and we are initiating the institutional waiver request through the Office of Research now rather than at Full Proposal stage. The project is modular: **Aims 1 and 2 constitute a coherent standalone effort at approximately $180,000** should Blueprint prefer a tighter scope, and Aim 1 alone is deliverable at approximately $110,000.

## 7. Risks and Mitigations

**Anytime-valid sequential tests are conservative.** E-processes trade power for validity under continuous monitoring. We will benchmark e-BH against Benjamini–Yekutieli and fixed-sample controls on identical data, report the power cost in explicit terms, and ship both options so programs can choose their own operating point.

**Parameter uncertainty in the end-to-end model is large.** This is the honest state of the field, not a flaw we can engineer away. Mitigation is structural: Sobol global sensitivity analysis makes uncertainty a first-class output, and the model reports which unmeasured parameters most limit confidence — itself a useful finding for Blueprint's future funding priorities.

**Retrospective ground truth is imperfect.** Historical detection dates for mpox and H5N1 are themselves fuzzy and partly retrospective. We mitigate by pre-registering comparison definitions before analysis, reporting results under multiple defensible date conventions, and stating limitations in the main text rather than a footnote.

**Cross-sample-type transfer may fail.** Wastewater and pooled nasal swabs differ in shedding, pooling depth, and background. If the Aim 1 framework does not transfer cleanly, that is a reportable scientific result and the wastewater deliverable stands unaffected.

---

*Submitted in response to Blueprint Biosecurity's Pathogen-Agnostic Biothreat Detection RFP. Contact: [EMAIL], [PHONE].*
