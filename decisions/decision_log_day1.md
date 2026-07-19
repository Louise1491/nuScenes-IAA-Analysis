# Decision Log - Day 1
**Date:** 2026-06-27  
**Project:** nuScenes IAA Analysis

## Schema 理解（中文复述）

我们分析的一个 scene 一般是一个 20 多秒的短片，2Hz（约 0.5 秒一帧）
能提取出大约 40 个左右的关键帧，即 sample。每个 sample 的 sample_data
是固定 12 条，包含了每个传感器（6 相机 + 1 LiDAR + 5 Radar = 12 个）
的一份原始数据文件，n 条 sample_annotation，n 取决于场景中有多少物体。
每个 sample_annotation 通过 instance_token 指向其对应 instance，
每个 instance 再通过 category_token 指向其 category。每个物体 instance
的 category 默认是不会变的，而物体的 attribute 会因为物体状态在
不同 sample 中不同而有变化。

## 数据集 Metadata 观察（list_scenes 输出）

- 10 个 scene 覆盖 Boston seaport 和 Singapore 4 个区域
- 夜间场景占 3/10，雨后场景 1/10
- Annotation 数量方差极大：最高 4622（scene-0061 施工+路口），
  最低 592（scene-0757 繁忙路口）—— 时长几乎相同，密度差 7.8 倍
- PM insight：vendor 计费应按 annotation 数量而非 scene 数量

## 关键字段发现（tutorial 实战中识别）

1. **sample_annotation 的 IAA 核心字段**：translation, size, rotation
   → 这三个决定 IoU 计算
2. **num_lidar_pts 是 IAA 的强信号**
   → 假设：点数越少 → 物体越不清晰越难标 → IAA 越低
   → Step 2 验证，成立则进入 vendor scorecard
3. **is_key_frame 用于过滤 sweeps**
   → IAA 分析只看 is_key_frame=True 的数据

## 方案选择（A vs B）

（明天 Step 2 开工时再决定。今天 schema 理解到位，下次直接进 Cohen's Kappa + IoU 实现。）

## 今天卡住的点 / 明天开工前要补的

- 无重大卡点
- 下次开工前需了解：scikit-learn 的 cohen_kappa_score 函数签名
- 下次开工前需想清楚：方案 A（模拟两个标注员）vs 方案 B（用真实多源数据）

## 待办认知（Step 2 开工时自然展开）

这些是 Day 1 学习中浮现的产品决策问题。不在今天解决，留作 Step 2 / Step 4 写
PM Recommendations 时的原料。

### 1. "一致" 怎么定义？—— Binary Agreement → Continuous IoU

- 两个标注员独立标同一物体，translation/size/rotation 数字**不可能完全相等**
- 业界不用"数字相等"判断一致，用 **IoU（两框重叠体积 / 两框并集体积）**
- IoU 是 0-1 连续值：完全重合=1，完全不重合=0
- "一致" 的判定 = IoU ≥ 某阈值

→ Case study 对应章节：**IAA Methodology: From Binary Agreement to Continuous IoU**

### 2. IoU 阈值哪来？—— Technical Cost × Business Cost 交集

- 业界没有统一标准（KITTI 用 0.5，Waymo 用 0.7，无 IAA 专用阈值）
- 正确推法：
  - **技术成本曲线**：标注严格度 vs ML 模型表现
  - **商业成本曲线**：标注严格度 vs vendor 单价
  - 两条曲线的最优交点 = 该公司的阈值
- Portfolio 实操：跑 IoU = 0.3 / 0.5 / 0.7 / 0.9 四档，展示"阈值-一致率"敏感性曲线
- 重点不是给出唯一答案，是展示方法论

→ Case study 对应章节：**Threshold Derivation Framework: Technical × Business Cost**

### 3. num_lidar_pts 是系统算的，不是标注员标的

- 关键澄清：num_lidar_pts = 系统自动 count 框内 LiDAR 点数，与标注员无关
- 角色：不是"被比较的对象"，是"标注难度的代理指标"
- 业界经验阈值（<5 / <10 可疑）的推导逻辑：
  - 按 num_lidar_pts 分桶 → 每桶抽样人工复审 → 统计错误率
  - 错误率跳变的桶边界 = 阈值
- 两类低点数标注：标注员脑补 / 物体出 LiDAR 有效距离 → 处理方式相同（质疑或降权）

→ Case study 对应章节：**Lidar-Aware Quality Filtering: Difficulty Stratification**

### 4. IAA 只是三层验证的第一层

- **Intra-Vendor IAA**：vendor 内部标注员之间的一致性 → 一致 ≠ 对
- **Vendor vs Gold Standard**：vendor 标注 vs Nuro 专家标注 → 才是准确性
- **Inter-Vendor Agreement**：多 vendor 交叉 → 校准 gold standard

**关键失效模式识别**：
- 低 IAA + 任何情况 → 随机错误 → vendor 训练不足
- **高 IAA + 低 Gold Standard IoU → 系统性错误 → vendor 被教错了（最危险，伪装成高质量）**
- 高 IAA + 高 Gold Standard IoU → 真高质量

系统性错误的三种真实起源：training 规范偏差 / 工具默认设置偏差 / 认知偏差

→ Case study 对应章节：**Multi-Layer Vendor Quality Framework**

### 5. Vendor 合同层面的产品设计

- 要求 5%-10% 数据 double-blind annotation（否则 Kappa 无从算起）
- 要求 vendor 交付 raw 独立标注，不接受 vendor 自算的 Kappa
- 明确 gold standard 的定义权归 Nuro

→ Case study 对应章节：**QA-Enabling Contract Design**

---

# Decision Log - Step 2
**Date:** 2026-07-11 ~ 07-12
**Session:** IoU + Kappa 实现,首张核心图产出

## 方案拍板:方案 A(模拟第二标注员)

锁定理由(三条):
1. nuScenes 不公开 raw 多标注员数据 → 方案 B 前提不成立
2. "扰动 = 模拟 vendor 失真",恰好对应三层验证框架的第 2 层
   (vendor 交付 vs gold standard)
3. 时间成本:A 一天完成,B 光找数据 3 天+

## Metric 拍板:BEV 2D IoU(不用 3D IoU)

- 把 3D 框投影到俯视图,算旋转矩形重叠(shapely 实现)
- 理由:快 10 倍 / AV 决策关心 x-y 平面 / Waymo & nuScenes 官方评测标准做法
- 面试备用答案:"为什么校准锚定 IoU 不锚定 σ?" → σ 依赖工具/尺度/传感器,
  跨数据集不可迁移;IoU 无量纲,才具备跨研究可比性

## 扰动函数的四个设计决策(面试必问)

1. **位置加性** `+N(0, σ_pos)`:手抖是绝对量(厘米级),不随物体大小缩放
   —— 这是物理真实,不做"公平化修正"
2. **尺寸乘性** `×N(1, 5%)`:估尺寸的参照物是物体自身轮廓,误差按比例走
   —— 加性和乘性对应标注动作里两种不同的手部行为
3. **clip(0.8, 1.2) 护栏**:高斯尾部可能抽出负尺寸(物理荒谬,shapely 崩溃),
   护栏设在 4σ 外,只拦 <0.3% 极端值,几乎不扭曲分布 —— 防御性编程
4. **z 方向 σ 减半**:标注工具有地面吸附(车轮/脚贴地),垂直误差天然更小
   —— 体现"知道标注是在什么工具环境里发生的"
5. **CONFUSION_MAP 只混淆语义相近类**:真人会把 adult 看成 construction_worker,
   不会把行人看成公交车;均匀随机混淆会让 Kappa 假低
6. `np.random.seed(42)`:结果可复现 —— case study 必需

## σ 校准记录(核心方法论:初值来自物理直觉,终值由目标 IoU 区间校准)

| 轮次 | SIGMA_POS | IoU 中位数 | IoU<0.5 占比 | 判定 |
|---|---|---|---|---|
| 1 | 0.20m | 0.680 | 30.6% | ❌ 低于 0.7,左尾过重(小物体受创) |
| 2 | 0.12m | 0.768 | 14.6% | ✅ 通过,锁定终值 |

- 校准目标:模拟 IoU 分布落在公开文献真人标注员区间 0.7-0.9
- 校准效率原则:第一步迈大、包夹目标,再二分(0.20→0.12,不是 0.20→0.17)
- 剩余 14.6% 低 IoU 不追杀 —— 它们主要是小物体,是要展示的现象不是要消除的噪音

## 核心机制:加性位置噪声 × 相对 IoU = 尺寸不对称

- 同样 0.2m 偏移:car(2m 宽)挪 10%,IoU≈0.85;trafficcone(0.3m)挪 67%,IoU≈0.2
- IoU 是相对指标,绝对手误除以小分母 → 相对伤害大
- 第一轮校准失败正是这个机制的暴露 —— 校准失败即 insight

## Kappa 结果与 Limitation(必须写进 case study)

- 原始一致率 0.952 / Kappa 0.936 / 运气成分仅 0.016
- **Limitation**: Kappa=0.936 主要是混淆参数(5%)的回声,不是发现。
  模拟中 95% 类别判定完全相同(同一份答案抄 95%),P_o 极高压缩了运气修正空间。
  真实 vendor 数据的分歧率和分歧模式未知 —— 这正是需要真实 raw 数据的原因。
  **本框架的价值在 pipeline 本身:数据接入→IoU→Kappa→分层报告,真实数据到位直接复用。**
- 附:实测混淆率 4.9% 而非 5%,因 CONFUSION_MAP 外类别(占 2.4%)永不混淆
  —— 期望值 5%×97.6%≈4.88%,数字对得上

## Step 2 三条核心 Insights

1. **尺寸梯度**:物体平均宽度 vs IoU 中位数 **r=0.844**。
   排序呈教科书级阶梯:米级车辆 0.8+ → 半米两轮 ~0.7 → 行人 ~0.6 → 锥桶 0.47
2. **一刀切阈值不公平实锤**:IoU 0.7 红线下,pedestrian.adult(n=4765)、
   trafficcone(n=1378)等高频安全类别整体"不及格"
   → threshold 必须按类别/尺寸分层,否则错误惩罚 vendor 或诱发过度返工
3. **child 双重困境**:最低 IoU(0.43)× 最少样本(n=46)× 最高安全权重
   → 定向采集 + 专项 QA 规则的直接依据
   (关联认知:police_officer 不是"一种行人",是"会发指令的交通控制源"——
   误标后果 = AV 不服从人工交通指令,且集中爆发在管制/事故等高危场景。
   nuScenes 把它单独建类,分类学本身就在标记安全攸关的区分)

## 数据分布观察(Cell 1)

- 极度长尾:car(7619)+pedestrian.adult(4765)占 67%;尾部 police_officer 仅 11
- 长尾即 Kappa 存在的理由:分布不平衡下随机猜也能靠 car 蹭高一致率
- mini 只覆盖 17/23 类(无 animal 等)—— case study 如实标注
- 灰显线 n<50:child(46) / personal_mobility(25) / debris(13) / police_officer(11)

## 产出清单

- `step2_iaa_analysis.ipynb`(Cell 1-5 全部跑通)
- `iaa_by_category.png`(核心图:分类别 IoU + 误差线 + 0.7 红线 + 灰显)
- 锁定参数:σ_pos=0.12m, σ_size=5%, σ_yaw=5°, p_confusion=5%, seed=42

## 待梳理清单(汇总版,截至 7/12)

### A 档:写 case study Methodology 时必须走读(写作即复习)
- [ ] Cell 2 扰动函数逐行(加性/乘性/clip/z减半——概念已懂,代码要能独立讲)
- [ ] Cell 3 shapely BEV IoU:corners 构造 + 旋转矩阵 + intersection/union
- [ ] Cell 5 groupby/agg 出图代码(errorbar、灰显逻辑)
- [ ] Cell 7 逐行 σ 技巧:normal(0,1,n) × sigma_per_row 为什么这么写

### B 档:面试前过一遍即可(概念题,非代码)
- [ ] 四元数→yaw 公式(定位:知道是 4 数降维 1 角度,不需推导)
- [ ] Cohen's Kappa 公式手算一个 2×2 小例子(防面试白板题)

### C 档:可选增强(有余量再做)
- [ ] 小样本敏感性检验的正规做法(multi-seed / bootstrap 概念,
      对应 construction 格子的发现)

---

# Decision Log - Step 3
**Date:** 2026-07-12

## Step 3 产出 (7/12 下午)

- 环境感知扰动上线:σ_per_row = 0.12 × night(1.5) × rain(1.3),假设系数待校准
- 修正:补 z/h 扰动(数据完整性原则),重跑后数字微变,正常
- 夜昼对比:0.766 vs 0.676 —— 环境盲版本测不出,环境感知版本立刻显形
  → 演示价值:同一 pipeline 换误差模型即输出分场景预警
- iou_drop 排名 = 环境脆弱性排名:pedestrian.adult 0.233 / bicycle 0.214
  vs car 0.046 / bus 0.033 —— 夜间×小物体 = 质量塌陷区,QA 抽检率应倾斜
- child 夜间 NaN:最高安全权重类别在夜间零样本 → coverage 缺口最狠证据
- construction 夜间 n=11:补 z/h 扰动(等价换一批随机数)后 iou_drop 从
  -0.005 翻转为 +0.013 —— 实证:小样本格子的结论不可复现,符号都保不住。
  → scorecard 护栏精确化:n<阈值的格子显示 "insufficient data",
    不输出伪结论数字,防止运营 chasing noise
- 雨天嵌套在夜间(无白天雨景)→ 雨/夜效应 confounded,mini 不支持单因素分析
- 条件组合分解:无雨夜仅掉 0.02,雨夜掉 0.235 —— 聚合稀释极端子群,
  scorecard 必须下钻到条件组合粒度

---

# Decision Log - Step 4: PM Recommendations (2026-07-15)

## R1-R5 终稿
### R1 分层 IoU 阈值(替代一刀切)

**Finding**: 物体尺寸与 IAA 强相关(物体物理平均宽度 vs IoU 中位数 r=0.844)。
统一 0.7 阈值下,米级车辆(car 0.83 / truck 0.84)全员达标,而高频安全类别
系统性"不及格":pedestrian.adult 0.61 (n=4765)、trafficcone 0.47 (n=1378)。
机制:标注手误是绝对量,IoU 是相对量——同样误差除以小分母,伤害数倍放大。

**Recommendation**: 按尺寸族设 3-4 层差异化阈值(如:大型车辆/小型车辆/
行人与两轮/小型静物)。各层阈值以该层真实 IoU 分布的分位数(如 P25)锚定,
使"不及格"在每层都意味着"显著差于同行水平",而非"物理上不可达"。

**Caveat**: ①分层结构可信,各层具体数字须真实数据校准后方可入 SLA;
②低阈值≠低要求,小物体层配更高 gold standard 抽检率作为补偿,
防止被解读为"小物体可随便标";③分层粒度以 3-4 族为上限,保证 SLA 可执行。

### R2 条件×类别粒度的 QA 资源倾斜

**Finding**: 聚合指标稀释极端子群——夜间整体 IoU 0.676(vs 白天 0.766)
看似"略差",下钻后:无雨夜 0.747(基本正常),雨夜 0.531(塌陷)。
环境伤害高度集中:iou_drop 排名 pedestrian.adult 0.233 / bicycle 0.214,
而 car 仅 0.046——夜间对小物体的杀伤是大物体的 5 倍量级。

**Recommendation**: QA 抽检率放弃单维度均匀分配,按"条件×类别"格子加权:
塌陷格子(夜×行人)抽检率数倍于安全格子(昼×car)。同时采用
"加权抽检(70-80%)+ 全域随机兜底(20-30%)"混合结构。
Dashboard 监控下钻到全部格子粒度,但差异化干预只对 Top 3-5 塌陷格子生效。

**Caveat**: ①环境系数(×1.5/×1.3)为假设值,格子间相对排序结构可信,
绝对数字待真实数据校准;②公开的抽检规则必然引发 vendor 行为博弈,
随机兜底保证任意格子被查概率非零,使偷工减料期望成本恒正;
③条件×类别全展开达数十格,全格子差异化管理在抽检样本量上不可行——
监控全量(计算零成本)、干预头部(执行有成本),两者解耦。

### R3 样本量护栏(结论可复现性门槛)

**Finding**: 实证:夜间 construction 格子(n=11),仅因补充 z/h 扰动代码
(等价于更换一批随机数,环境系数未动),iou_drop 从 -0.005 翻转为 +0.013——
符号都不可复现。小样本格子输出的"数字"具备结论的外形,不具备结论的效力,
会诱导运营团队追逐噪音(chasing noise)。

**Recommendation**: 建立分层护栏:可视化层置灰弱化(如分类别图 n<50 灰显)、
报告层拦截(n<阈值的格子输出 "insufficient data" 而非数字)、
接口层返回 NULL。原则统一:样本量不足时,系统不产出"看起来像结论的数字"。

**Caveat**: ①n=50 为经验起点;严格阈值应由敏感性检验反推——格子结论
在多个随机种子下保持符号稳定所需的最小样本量;②若大量格子长期显示
insufficient data,使用方会绕过系统凭感觉决策——护栏必须与 R4 的定向
采集联动(缺数据的格子进采集清单),否则护栏沦为长期免责声明;
③被拦截格子的监控需求仍在:输出"当前样本量/所需样本量"进度而非空白,
使 insufficient data 本身成为可跟踪的状态。

### R4 Coverage 缺口驱动的定向采集

**Finding**: 样本量与安全权重呈反比:child 46 条且夜间 0 条(最高安全权重
类别在最高风险条件下零覆盖),police_officer 全集仅 11 条;数据集中不存在
白天雨景,雨/夜效应 confounded,无法支持单因素质量分析。

**Recommendation**: 建立 coverage 缺口清单驱动采集优先级,
缺口分数 = 安全权重 × 样本缺口 × 条件组合空洞。与 R3 护栏联动:
被拦截为 insufficient data 的格子自动进入采集清单(护栏与采集构成
"不说没把握的话 → 去获得把握"的闭环)。

**Caveat**: ①三因子中样本缺口与条件空洞可客观测量,安全权重是价值判断——
其量化应升级至安全委员会/跨职能层面,非数据 PM 单方定价;
②定向采集使训练集偏离真实世界分布,targeted subset 应独立标记,
采样权重由 ML 团队在训练侧控制——采集决策与配比决策解耦;
③目标场景自然发生率极低(夜间儿童/夜间执勤交警),现实路径为
自然采集 → staged collection(摆拍)→ 合成数据的递进,
选择本身是真实性×成本×速度的三角权衡。

### R5 三层验证框架与合同前置(元建议:R1-R4 的生效前提)

**Finding**: 本项目全部指标(IoU/Kappa)测量的是一致性,而一致≠正确——
高 IAA + 低 gold standard 吻合度 = 系统性错误(vendor 被统一教错,
最危险的失效模式,伪装成高质量)。且模拟仅覆盖幅度型误差,
规范型分歧(如 bendy 转弯的框选取舍)只能靠真实分歧案例定性归因。

**Recommendation**: 质量体系分三层:Intra-Vendor IAA(稳定性)→
Vendor vs Gold Standard(准确性)→ Inter-Vendor(交叉校准)。
配套合同前置:5-10% 数据 double-blind 标注、vendor 交付 raw 独立标注
(不接受 vendor 自算指标)、gold standard 定义权归买方。
R1-R4 的一切阈值与抽检设计,都以本条的数据可得性为前提。

**Caveat**: ①gold standard 自身也有误差与成本,专家标注 ≠ 绝对真值,
高分歧案例应保留多方标注而非强制归一;②合同条款增加 vendor 成本,
会反映进单价——double-blind 比例是质量置信度与预算的权衡参数;
③三层体系的运维本身需要人力,小团队可从"第二层抽检 + 合同条款"
的最小版起步,三层全量是成熟态非起点。

## Step 4 过程中新增的方法论认知

- **Caveat 公式**:①数字的效力边界 ②被滥用的防线 ③操作复杂度上限
- **分位数锚定及格线**:P25 = 后 25% 挂科;线越靠中间越严;
  "不及格"应指向"显著差于同行"而非"物理上不可达"
- **冷启动 vs 成熟方法谱系**(两例):分位数锚定→成本曲线法;
  n=50 经验阈值→多 seed 敏感性检验反推
- **多 seed 敏感性检验**:结论对随机种子敏感 = 噪音,不敏感 = 真实机制;
  construction 翻转是无意中完成的 n=2 版本
- **随机数消耗序列错位**:seed 固定的是数字流起点,代码插入一次抽取,
  后续全部错位;可复现契约仅对逐字节相同的代码成立
- **三层拦截强度**(置灰/报告拦截/接口 NULL):拦截强度与受众误用能力
  成正比;与 Hellman Phase1/2 受众分离同一设计哲学
- **抽检博弈**:公开规则必然引发 vendor 行为优化;
  加权抽检 + 全域随机兜底,保证任意格子被查概率非零
- **监控与干预解耦**:监控全量(计算零成本),干预头部(执行有成本)
- **taxonomy 设计逻辑**:车辆按运动学切(bendy=双刚体铰接 vs rigid=单刚体，car不再细分因运动学同质),
  行人按行为语义切（child/officer）;标注分类是下游模型需求的上游翻译；
  QA 规则也应继承这个逻辑(如 bendy 转弯时矩形框的固有失配是规范问题,非标注员问题)

## Known Unknowns(本项目触碰不到的真实工作面)

### 数据与方法侧(可通过阅读/对话补)
- [ ] 规范型分歧的定性归因流程:真实团队怎么开 disagreement review 会,
      分歧案例怎么归档、怎么反哺规范修订
- [ ] 标注规范(labeling spec)的真实形态:一份工业级 spec 的结构、
      粒度、随 edge case 演化的机制
- [ ] Taxonomy 变更治理:新类别由谁提议、怎么评审、
      历史数据怎么迁移(变更 = 全量重标?)
- [ ] 工具链取舍:Scale/Labelbox/自建的决策逻辑,
      工具对 IAA 的影响(如地面吸附这类特性的实际普及度)

### 组织与人侧(只能靠 networking 对话采集)
- [ ] Vendor 谈判实操:SLA 怎么定价、返工怎么扯皮、
      double-blind 条款在真实合同里的接受度
- [ ] 标注员流失/训练衰减对质量的影响曲线,以及 vendor 怎么隐藏它
- [ ] Data PM 与 ML 工程师的优先级冲突怎么裁
      (采集预算、taxonomy 变更、质量 vs 交付速度)

用途:①case study 第 6 章素材 ②秋季 networking(Rohan/Molly/Amanda)
提问弹药——每条都是"我走到了边界、知道边界外是什么形状"的内行钩子


## 收工快照(7/15)
- 停在哪:Step 2/3/4 全部完成。产出:两张图 + R1-R5 + case study outline
- 下一步第一个动作:Step 5 英文写作日——照 outline 搬运,写作中还 A 档代码账
- 需加载:本 log 全文 + case_study_outline.md + notebook