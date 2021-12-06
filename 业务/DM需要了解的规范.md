## [临床数据管理员（DM）面试前需要了解的规范、标准（ICH-GCP、GCP、CDISC标准等）](https://zhuanlan.zhihu.com/p/365173408)





1、《赫尔辛基宣言》全称《世界医学协会赫尔辛基宣言》。这个指导原则的来历是，1946年因为二战期间纳粹医生灭绝人性的人体医学研究，诞生了纽伦堡法案，来保护人体医学研究受试者。1964年，在纽伦堡法案基础上起草了《赫尔辛基宣言》。虽然它不具有法律约束力，但已被世界各国纳入各自的法规、指导原则中，已成为人体医学实践及其临床研究的道德准则和临床试验管理规范的核心。

![img](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/Untitled.assets/v2-631a65923aca1fe6808b4e9c400a98fd_b.jpg)

2、ICH-GCPICH：国际人用药物注册技术要求协调会（International Conference on Harmonization of Technical Requirements for Registration of Pharmaceuticals for Human Use）我国在2017年成为ICH正式成员，这利于我国的监管标准化，成员国之间研发、审批互认，减少药品研发和上市的时间和成本。这个国际协调会议有一系列的临床试验指导文件，其中ICH E6这个文件叫做《临床试验质量管理规范》（GCP），它是在《赫尔辛基宣言》基础上补充修改得到。目前最新版是2016版。![img](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/Untitled.assets/v2-61e0c0b7a819f85617668b05f445f993_b.jpg)

ICH-GCP包括上图9个部分，详细概述了临床试验各方（申办方，研究者和伦理委员会）的角色和责任。第二章中阐述了ICH-GCP的13条原则，可以概括为3条，也是前言中提到的。（1）保证受试者的权益、安全和健康（2）符合《赫尔辛基宣言》要求的伦理原则*（因为它是从《赫尔辛基宣言》基础上完成的，很容易理解这点）*（3）确保临床试验数据的可信性3、GCP![img](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/Untitled.assets/v2-32c2b61d318b587ee5c64f2962921693_b.jpg)

我们来看看新版GCP的结构，对比下ICH-GCP的结构，是不是很像？现行的ICH-GCP是2016版的，2017年我国加入ICH，2020年实施新版GCP，也就是我国的GCP来源于ICH-GCP，但也同时结合了我国的国情。总则第一条中概括了GCP的原则，有两条（可以和ICH-GCP原则一起理解记忆）：

（1）保证临床试验过程规范，**数据**和结果的**科学、真实、可靠**

（2）保护**受试者**的**权益和安全**

4、CDISC标准

CDISC：临床数据交换标准协会（Clinical Data Interchange Standards Consortium）这个协会是个非盈利机构，它就如何收集数据、收集什么类型的数据以及如何将数据提交给负责审批新药的机构建立起了一套标准。美国FDA在2016年强制要求必须使用CDISC标准递交电子文档。CDISC标准中包含一系列标准，和DM最密切的是CDASH和SDTM。CDASH是数据采集标准，SDTM是数据递交标准，我们是根据CDASH标准来设计CRF的。还可以了解下ADaM（分析数据模型）是统计分析时需遵循的标准。SDTM：研究数据列表模型（Study Data Tabulation Model）CDASH：临床数据获取协调标准（Clinical Data Acquisition Standards Harmonization）目前使用的CDASHIG V2.1的内容基于SDTMIG V3.2（IG is Implementation Guide，实施指南），SDTM和CDASH是相互关联的，CDASH数据采集的字段或变量是为了助于映射到SDTM。收集的提交数据在两个标准相同时，CDASHIG中将采用SDTMIG中的变量名，两个标准大部分是交叉的，CDASH标准也包括SDTMIG中没有的数据收集字段，SDTMIG中也有CRF中不会设计的变量。在实际中CDASH可能不会包括所有在设计CRF时需要的变量，我们会去STDM中找，如果还是没有，就靠编了，当然也不是瞎编，也有规则的。我们国家现在还没有强制要求遵循CDISC标准，但是很多公司都是按这个标准来的，前几年不规范的现象会多一点。

5、《临床数据质量管理规范》（GCDMP）

这是美国临床试验数据管理学会（SCDM）提出的数据管理规范指导文件，共20个章节，从最低标准和最佳实践两方面阐述临床数据管理的各环节。

6、ALCOA原则、ALCOA+原则这是GCP和GCDMP都提出的适用于临床试验的质量和数据可信性的全球标准。ALOCOA原则：（1）溯源性（attributable）（2）易读性（legible）（3）同时性（contemporaneous）（4）原始性（original）（5）准确性（accurate）ALOCOA+原则，是在以上基础上增加：（1）持久性（enduring）（2）完整性/一致性（complete/consistent）（3）可获得性和可用性（available)