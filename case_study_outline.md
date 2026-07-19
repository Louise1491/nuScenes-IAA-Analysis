# Case Study Outline — nuScenes IAA Analysis
# (Defining Data-Driven QA Thresholds for AV Annotation)

Target length: 3-5 pages / GitHub README + linked notebook
Audience: AV Data PM hiring managers (Nuro/Applied Intuition/Waymo)
Status: outline v1 (7/15) — drafting scheduled for Step 5

---

## 1. Context & Problem (0.5 page)
- 钩子:AV 公司每年花数千万美元买标注,但 "pass/fail threshold" 通常是拍的
- JD 原文引用:"define pass/fail quality thresholds" / "build visibility"
- 本项目回答:阈值应该怎么被推导出来,而不是被拍出来
- 素材指针:Day 1 待办认知 #2(阈值哪来)

## 2. Methodology (1 page)
- 2.1 数据:nuScenes mini,18538 annotations,17/23 类
  → 指针:Step 2「数据分布观察」
- 2.2 模拟双标注员框架:方案 A 机制 + 三种失真(加性/乘性/混淆)
  → 指针:Step 2「扰动函数四个设计决策」;写作时走读 Cell 2(还账 A 档)
- 2.3 σ 校准闭环:初值物理直觉 → 目标 IoU 区间(0.7-0.9)校准 → 两轮记录表
  → 指针:「σ 校准记录」表格直接搬
- 2.4 BEV IoU 选择理由 + Kappa
  → 指针:Step 2「Metric 拍板」;走读 Cell 3(还账)
- 2.5 环境感知升级:环境盲的发现 → σ×条件系数
  → 指针:Step 3 产出第 1-3 条

## 3. Findings (1-1.5 page,图驱动)
- F1 尺寸梯度 r=0.844(图:iaa_by_category.png)
- F2 一刀切 0.7 下高频安全类别系统性不及格(同图,红线叙事)
- F3 环境伤害集中于条件×类别格子(图:iaa_day_vs_night.png;
     雨夜 0.531 vs 聚合 0.676 稀释效应)
- F4 小样本格子不可复现(construction 符号翻转,无图,叙述实证)
- F5 coverage 空洞(child 夜间零样本 / 无白天雨景)
  → 指针:Step 2「三条核心 Insights」+ Step 3 产出全部

## 4. Limitations (0.5 page — 独立成章,放 Recommendations 前)
- L1 Kappa 数字是混淆参数的回声(pipeline 真实,数字待真实数据)
- L2 环境系数为假设值(结构可信,幅度待校准)
- L3 幅度型误差可模拟,规范型分歧不可模拟(bendy 案例)
- L4 mini 数据集:17/23 类、雨/夜 confounded、单城市对
  → 指针:「Kappa 结果与 Limitation」+「模拟的结构性盲区」
- L5 模拟的结构性盲区(第二例):bendy 铰接形变导致的规范模糊型分歧
  无法用"σ×系数"表达——它不是误差幅度问题,是标注规范的取舍问题。
  归类:模拟可覆盖【幅度型误差】(手抖/环境),不可覆盖【规范型分歧】
  (形状失配/类别边界争议)。后者恰恰是真实 IAA 数据最有价值的部分
  → 写进 Limitations:本框架量化幅度型误差;规范型分歧需真实
    multi-annotator 数据 + 分歧案例的定性归因(这是 Data PM 的日常工作)
- 写法基调:每条 limitation 都指明"真实数据到位后如何升级"——
  limitation 章是能力展示,不是道歉信

## 5. PM Recommendations (1-1.5 page)
- R1-R5 直接搬 Step 4 五条(Finding→Recommendation→Caveat 三段式保留)
- 开头加一句框架说明:R5 是 R1-R4 的生效前提(元建议)

## 6. What I'd Do Next / With Real Data (0.25 page)
- 多 seed 敏感性检验正规化(C 档)
- SQL 层重写 + visibility dashboard(7 月下旬已排期,预告即可)
- 真实 multi-annotator 数据接入路径
  → 指针:「Known Unknowns」清单选 2-3 条,展示边界自知

## Appendix
- notebook 链接 / 参数表(σ=0.12, ×1.5/×1.3, seed=42, n≥50)/ 复现说明