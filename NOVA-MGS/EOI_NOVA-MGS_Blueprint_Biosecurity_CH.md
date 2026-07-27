# 意向书（Expression of Interest）

**Blueprint Biosecurity —— 无差别病原体生物威胁检测征集（Pathogen-Agnostic Biothreat Detection RFP）**

**技术领域：** TA1 —— 在宏基因组测序（MGS）数据中对已知与新颖病原体的计算检测

**项目名称：** **NOVA-MGS** —— 面向实战化宏基因组生物监测的无参考新颖性评分、概率校准与证据核验系统

**项目负责人：** Sihua Peng, PhD —— 佐治亚大学公共卫生学院研究科学家；佐治亚州先进计算资源中心（GACRC）附属研究人员

**依托单位：** 佐治亚大学（University of Georgia），Athens, GA, USA

**执行周期：** 12 个月

**申请经费（约）：** $249,700（直接费用 + 间接费用，详见第 6 节）

**日期：** [填写]  |  **联系方式：** [填写邮箱 / 电话]

> *说明：本文为中文对照版本，供内部评审与合作方沟通使用。正式提交给 Blueprint Biosecurity 的版本须为英文（见 `EOI_NOVA-MGS_Blueprint_Biosecurity.md`）。*

---

## 1. 问题陈述

宏基因组测序在实验台上取消了"必须事先知道要找什么"这一前提，却没有在键盘上取消它。当前污水与混合人群样本监测中广泛部署的分类器——Kraken2/Bracken、GOTTCHA2、Centrifuge——本质上都通过与参考序列匹配来归属 reads。一个真正新颖或高度发散的病原体，按定义就没有可匹配的参考。它不是被错分，而是被直接丢进 unclassified 部分——在真实污水数据中，这一部分通常占全部 reads 的 60%–90%。

领域内的实际应对方式是降低判定严格度、产出更多候选。这就催生了第二个、且目前已成为真正瓶颈的问题：**分析人员的处理能力**。一个每周产出数百份样本的生产级污水监测项目，标记出的候选 contig 数量远超任何专家团队的复核能力。候选序列往往由当时有空的人用临时的 BLAST 检索来判断，既无审计轨迹，也无经过校准的置信度。CDC Biothreat Radar、NWSS、mSCAPE 这类项目无法靠"专家注意力"扩展。

NOVA-MGS 直接针对这两处失效：一是对数据库中根本不存在的序列给出经校准的新颖性概率的无参考评分引擎；二是把原始候选转化为可审计、可决策证据包的确定性核验环节——分析人员不必再花时间去重建一条被标记的 contig 究竟是什么。

## 2. 技术方案

![**图 1. NOVA-MGS 处理链路。** contig 由两条正交通道并行评分——核酸层 FracMinHash containment（通道 A）与逐条 ORF 的蛋白嵌入局部密度（通道 B）——聚合至 contig 层后融合，再经等渗校准映射为新颖性与关切度评分。只有超过阈值的候选才进入确定性证据核验。](NOVA-MGS-4.png){width=5.2in}


### 目标 1 —— 双通道新颖性评分、概率校准与证据核验（第 1–9 月）

我们对序列的评分依据是其**与已知病毒空间的距离**，而非**是否归属其中**，采用两条失效模式互不重叠的正交通道。

*通道 A（核酸层）。* 通过 sourmash 的 FracMinHash containment 与经过整理的病毒 sketch 数据库比对 [1]，保留完整的 containment 分布而非阈值判定结果。当一条短 contig 与大型数据库比较时，containment 而非 Jaccard 才是正确的度量 [2]；我们采用 Hera 等 [3] 的去偏估计量与置信区间，而非在小 sketch 尺度上存在偏差的朴素二项形式。该通道用于捕获那些与已知病毒相关、但落在分类器归属阈值以下的病原体。

*通道 B（蛋白层）。* 对组装 contig 与高复杂度 read 簇进行六框 ORF 预测，随后用蛋白语言模型（ESM-2）[4] 生成嵌入，并针对由已知病毒蛋白组构建的流形做局部离群因子（LOF）密度估计 [5]。由于在接近 1280 的嵌入维度下最近邻距离会出现集中现象 [6]，密度估计在 L2 归一化后的余弦距离下进行并预先降维，而非使用原始欧氏体积。该通道用于找回那些核酸序列已经发散到无法识别、但衣壳、聚合酶或糖蛋白折叠仍保留病毒特征的病原体——这恰恰是通道 A 失效的区间。

由于通道 A 在 contig 层打分、通道 B 在单条 ORF 层打分，融合之前需先将 ORF 层证据聚合到各自所属的 contig，使两条通道以同一统计单元进入模型。随后由逻辑斯蒂融合把对齐后的特征及其交互项合成为标量分数，再由在独立数据划分（见目标 2）上拟合的等渗回归（isotonic regression）映射为每条 contig 的 **新颖性与关切度评分（Novelty and Concern Score, NCS）**——一个可解释的概率，而非任意量纲的指标。融合、校准与可靠性评估分别在互相分离的数据划分上完成（或采用嵌套交叉拟合），以避免报告出过于乐观的概率；等渗回归将与 Platt scaling、beta calibration 对照，按交叉验证 Brier 分数择优 [7]。真正的交付价值在于**校准质量**而非原始区分能力——只有经过校准，下游的告警阈值才站得住脚。

工程实现采用 Nextflow + Apptainer 容器，面向 SLURM 集群优化，目标吞吐量为数百份样本、数十亿条 reads。基于我们在 GACRC 现有流程上的预分析，单样本边际成本可支撑都市级汇水区的常规每周运行；确切的周转时间指标将作为主要交付物之一予以报告。

*证据核验。* 一个经过校准的分数是必要的，但还不够：运维人员仍然需要判断这条被标记的 contig 到底是什么。因此每个超过阈值的候选在报出之前，都会经过一个固定的核验子流程——针对组装 contig 的 read 支持度与覆盖广度/深度、远源同源检索（DIAMOND、MMseqs2）、病毒标志基因谱比对（HMMER 对 Pfam/VOG）、结构层同源检测（Foldseek 对 AlphaFold DB，用于序列同源已被抹除的情形）、跨工具一致性核查（geNomad、VirSorter2），以及同一序列簇在既往样本中的丰度轨迹。这些全部是确定性工具，以 Nextflow 子流程的形式串接，因此相同输入必然给出相同的证据包，其中每一条都可回溯至具体的工具调用。

输出为结构化的 **证据核验档案（evidence-verified dossier）**：经校准的 NCS 及其置信区间、每一条支持与反对证据、以及完整的工具调用日志。正是这一层让分数从"算出来了"变成"可以据以行动"——而它是报告层，不是告警层。NOVA-MGS 负责排序与举证，判断由人来做。

### 目标 2 —— WW-NOVEL-BENCH：公开基准与回溯验证（第 2–12 月）

本领域目前没有共享的新颖病原体检测基准，导致各家工具的性能主张无从核实，资助方的投入产出也无法衡量。我们将建立这一基准。

以 CASPER（PRJNA1247874）、NWSS（PRJNA747181）、Tisza et al. 2023（PRJNA966185）、Wolfe et al. 2026（PRJNA1438722）的真实污水背景为底，在计算机中注入三类真值序列：（i）真实病毒基因组，但将其所在进化支从全部参考数据库中剔除，构成诚实的 leave-one-clade-out 难度；（ii）经演化模拟生成的发散变体，核酸差异分别为 20%、40%、60%；（iii）在数据库中确无代表序列的真实孤儿病毒。注入丰度覆盖 10⁻⁴ 至 10⁻⁸ 相对丰度区间，并跨 Illumina、PacBio HiFi、Nanopore 三个测序平台——这与本团队既有的多平台流程经验相匹配。

泄漏控制是基准设计中最常被跳过的一环，因此我们予以显式规定：每条注入基因组所在的整个进化支，都从全部参考库、sketch 与训练集中剔除；难度不再用"新颖/已知"的二元标签，而是按与最近保留邻居的 ANI 做连续分层；结果按难度分层分别报告，不做合并。合并会掩盖 ANI 低于 80% 的那一段区间——而真正新颖的病原体恰恰落在那里。此外并行采用时间切分协议：仅在某截止日期之前提交的序列上拟合与校准，严格在其之后的序列上评估，从而完全不依赖人工模拟的发散度来检验泛化能力。

由于阳性样本在全部 contig 中占比极低，评估以 AUPRC 而非 ROC-AUC 为准 [8]，并同时报告所选阈值下对应的人工复核工作量——决定工作点的是告警预算，而不是默认的判定阈值。概率质量与排序能力分开评估，通过可靠性图与 Brier 分数分解来考察，因为排序良好的分数完全可能校准很差。

回溯验证将回答那个在实战中真正决定成败的问题——**"这套方法当时能不能抓到？能提前多久？"**——以 2022 年 mpox 与 2024 年 H5N1 进入污水的信号为对象，在归档的公共数据上检验。

基准数据集、评测框架与公开排行榜将独立于 NOVA-MGS 自身的表现予以发布，使得无论我们的方法是否胜出，其他竞争工具都能从中受益。

## 3. 交付物与开放获取

全部软件将以宽松开源许可（MIT / Apache-2.0）发布于 GitHub，自第 3 月起**在开放状态下持续开发**，而非在项目结束时一次性倾倒代码。容器镜像将发布至公共镜像库；基准数据集与校准数据将发布至 Zenodo 并获得持久 DOI。预计产出两篇同行评议论文，投稿同时在 bioRxiv/medRxiv 发布预印本。流程将遵循 nf-core 规范，以降低已在运行 nf-core/mag 的监测项目的采用门槛。

## 4. 负责任披露

目标 1 的一项扩展——针对组装 contig 中工程改造特征的统计学变点筛查——存在合理的信息危害风险：若筛查签名的细节被完整公开，可能为规避检测提供参考。我们选择在事前而非事后主动提出这一点。我们建议该组件在与 Blueprint 协商确定的**分级披露协议**下开发：运行能力向经过审核的监测项目开放，而方法层面的发表时机与披露粒度由双方共同决定。若 Blueprint 认为不妥，我们也完全可以将该组件整体移出项目范围。

## 5. 团队与能力

项目负责人已发表同行评议论文 40 余篇（H 指数 24），研究覆盖计算生物学与生物统计学；当前正在开展跨 Illumina、PacBio HiFi、Nanopore 三平台的宏基因组病毒检测工作，使用 nf-core/mag、sourmash、geNomad、VirSorter2、GOTTCHA2、Kraken2/Bracken 等工具，并通过 Nextflow 与 Apptainer 在 GACRC 的 SLURM 集群上规模化运行。在此之上，项目负责人在生物统计与实验设计方面的系统训练与发表记录，才是目标 1 的校准工作与目标 2 的基准工作真正的立足点：**本提案真正困难的地方不在于把工具跑起来，而在于证明这些工具给出的概率确实是它们声称的那个意思。**

GACRC 可提供所需的计算、存储与容器基础设施，本项目不申请新增硬件。我们欢迎 Blueprint 引荐运行中的监测项目（CASPER、Zephyr、参与 NWSS 的污水处理单位）以开展真实场景验证，也乐于与提交 TA2 或 TA3 互补提案的团队组队合作。

## 6. 预算概要（12 个月，估算）

| 类别 | 金额 |
|---|---|
| 项目负责人投入（30% FTE） | $47,000 |
| 博士后研究员（100% FTE） | $78,000 |
| 研究生助研（50% FTE，含学费） | $46,000 |
| 附加福利（Fringe benefits） | $37,000 |
| GPU 云端弹性算力与长期归档存储 | $10,000 |
| 论文发表、成果传播与差旅 | $9,000 |
| **直接费用合计** | **$227,000** |
| 间接费用（按 Blueprint 政策，取直接费 10%） | $22,700 |
| **申请总额** | **$249,700** |

本预算在非人力成本上刻意从紧：GACRC 向项目负责人免费提供集群算力、存储与容器基础设施，因此算力一项仅覆盖蛋白语言模型嵌入所需的 GPU 云端弹性算力，以及基准数据集发布所需的永久归档托管；不申请任何新增硬件。间接费严格按 Blueprint 公布的 10% 上限编制；佐治亚大学协商 F&A 费率高于此值，我们正**在现阶段**（而非等到 Full Proposal 阶段）通过校科研办公室启动机构豁免申请。本项目在设计上是模块化的——若 Blueprint 倾向更紧凑的范围，**仅目标 1 即可构成一个自洽的独立项目，约 $165,000**。

## 7. 风险与应对

**蛋白语言模型在碎片化环境 ORF 上的表现尚未验证**，而高维嵌入空间中的密度估计本身也很脆弱。两者均由第 2 月的通道 B go/no-go 可行性测试一并处理——该测试不仅检验嵌入本身，也检验距离度量与降维方案的选择；若通道 B 表现不达预期，仅凭通道 A 仍足以支撑一个范围收窄的交付版本。

**运行阈值下的假阳性负担。** 仅有经过校准的分数，并不足以约束分析人员的工作量。因此对每一个工作点，我们都会报告在真实样本通量下所对应的"每周候选数"，并据此设定阈值，而非沿用默认判定值。

**基准的真实性。** 计算机注入的 spike-in 有可能系统性地高估检测方法的表现。我们通过 leave-one-clade-out 真实基因组与针对真实历史信号的回溯测试来缓解，并将把基准的局限性写在正文而非脚注里。

## 8. 参考文献

1. Ondov BD, Treangen TJ, Melsted P, Mallonee AB, Bergman NH, Koren S, Phillippy AM. Mash: fast genome and metagenome distance estimation using MinHash. *Genome Biology* 2016;17:132.
2. Koslicki D, Zabeti H. Improving MinHash via the containment index with applications to metagenomic analysis. *Applied Mathematics and Computation* 2019;354:206–215.
3. Hera MR, Pierce-Ward NT, Koslicki D. Deriving confidence intervals for mutation rates across a wide range of evolutionary distances using FracMinHash. *Genome Research* 2023;33(7):1061–1068.
4. Lin Z, Akin H, Rao R, et al. Evolutionary-scale prediction of atomic-level protein structure with a language model. *Science* 2023;379(6637):1123–1130.
5. Breunig MM, Kriegel H-P, Ng RT, Sander J. LOF: identifying density-based local outliers. *Proceedings of ACM SIGMOD* 2000:93–104.
6. Zimek A, Schubert E, Kriegel H-P. A survey on unsupervised outlier detection in high-dimensional numerical data. *Statistical Analysis and Data Mining* 2012;5(5):363–387.
7. Niculescu-Mizil A, Caruana R. Predicting good probabilities with supervised learning. *Proceedings of ICML* 2005:625–632.
8. Saito T, Rehmsmeier M. The precision-recall plot is more informative than the ROC plot when evaluating binary classifiers on imbalanced datasets. *PLoS ONE* 2015;10(3):e0118432.

*按 RFP 规定，参考文献不计入页数上限。*

---

*本意向书系响应 Blueprint Biosecurity《无差别病原体生物威胁检测》征集而提交。联系方式：[邮箱]、[电话]。*
