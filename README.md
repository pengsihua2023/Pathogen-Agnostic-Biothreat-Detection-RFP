# Pathogen-Agnostic-Biothreat-Detection-RFP

- 对于计算方法和样本处理方法的开发，我们重点考虑的因素包括：快速的周转时间（从样本采集到获得生物信息学分析结果）、扩展至处理数百个样本和数十亿条测序读段的能力，以及对环境样本或人群混合样本的适用性，例如污水、空气、鼻拭子、唾液和血液样本。
“For computational and sample processing methods development, some of our key considerations include rapid turnaround time (sample collection to bioinformatic results), ability to scale to hundreds of samples and billions of reads, and applicability to environmental or pooled-human samples such as wastewater, air, nasal swabs, saliva, and blood.”
- 我们预计，有竞争力的计算方法类项目应开发开放、易于获取、设计良好、文档完善且用户友好的工具。这些工具应强调处理速度和并行计算能力，并且能够部署在每天处理数十亿对测序读段的检测流程中。
  “We anticipate that competitive computational proposals will develop open, accessible, well-designed, documented, and user-friendly tools that emphasize speed and parallelizability, and are deployable in detection pipelines processing billions of read pairs per day.”

- processing billions of read pairs per day
每天处理数十亿个双端测序 read pairs。
 - billions of reads：数十亿条测序读段
 - billions of read pairs per day：每天数十亿对双端测序读段
 - hundreds of samples：数百个样本
 - rapid turnaround time：快速的分析周转时间

## 三个方向
● Computational detection of known and novel pathogens, including zoonotic and engineered, in MGS data
● Sample processing methods for more sensitive, sequencing-based detection and rapid validation of sequences of concern
● Modeling pathogen-agnostic biosurveillance system features such as cost, sensitivity, and optimal deployment

● 基于宏基因组测序（MGS）数据，对已知及新型病原体（包括人兽共患病原体和经人工改造的病原体）进行计算检测 （NOVA，SIEVE）
● 针对基于测序的检测，开发可提高灵敏度的样本处理方法，并实现对关注序列的快速验证
● 对不针对特定病原体的生物监测系统进行建模，评估其成本、灵敏度及最佳部署方案等特性 （DPD）
