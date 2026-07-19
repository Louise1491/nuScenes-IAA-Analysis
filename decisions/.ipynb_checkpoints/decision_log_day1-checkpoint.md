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