# 论文+实验记录

[TOC]

## Dynamic Coupon Targeting Using Batch Deep Reinforcement Learning: An Application to Livestream Shopping

-- Liu, Xiao. "Dynamic coupon targeting using batch deep reinforcement learning: An application to livestream shopping." *Marketing Science* (2022).

1. ### **数据**

- 数据：2019年3个月的1,020,898名淘宝直播消费者的数据。这些消费者在观看200,568条不同的直播时，获得了25,886,094张优惠券，完成了1,539,424笔交易，由销售男装、女装、童装、化妆品和珠宝5类产品的11,926名主播创造。数据集中的所有消费者都是"活跃的"，平台定义为在样本期间内至少收到10张淘宝直播优惠券。

  <img src="C:\Users\13560\AppData\Roaming\Typora\typora-user-images\image-20230604234542060.png" alt="image-20230604234542060" style="zoom:67%;" />

  - **user_id**
  - **time_id:**优惠券接收频次，即消费者在样本期内收到优惠券的次数
  - **consumer:**消费者的静态和动态特征矢量(4.2.1)
    - 静态：性别、年龄、购买力、产品类别偏好、对特定卖家的偏好
    - 动态：与优惠券接收行为相关：消费者自上次收到优惠券以来的天数( recency _优惠券)，消费者自样本期开始以来收到优惠券的数量( frequency _优惠券)，自样本期开始以来收到优惠券的折扣比例和阈值比例的平均值、最小值和最大值( money _优惠券)。
  - **product:**消费者观看的流媒体视频中推广的每种产品的静态和动态特性矢量(4.2.1)
    - 静态：产品种类、价格、市场占有率、评级等
    - 动态：消费者上次购买产品以来的(优惠券接收发生率)期数( recency _ product )，样本期开始以来购买的产品数量( frequency _ product )，购买产品的平均价格、最高价格和最低价格( money _ product )，以及样本期开始以来的累计支出( money _ product )。
  - **host/seller:**消费者观看的每个直播视频主播的静态和动态特征向量(4.2.1)
    - 静态：主播性别、年龄、卖家知名度（如月收入、质量评级等）
    - 动态：样本期开始以来消费者访问的卖家总数( frequency _ sale )
  - **livestream channel:**消费者观看的每个直播频道的静态和动态视频特征向量(4.2.1)
    - 静态：参与度得分，例如，访问网页和将产品添加到购物车中的消费者的平均数量；非结构化数据如网页美学特征
    - 动态：消费者上次访问网页的时期数( recency _ website )和样本期开始以来访问网页的次数( frequency _ website )。
  - **coupon type:**优惠券类型，由折扣(discount)和门槛率(threshold ratios)定义(3.3)
    - 每张优惠券由**折扣比率**（ratio of the discount value to the threshold value）和**门槛阈值比率**（ratio of the threshold value to the average price in the livestream）两个属性来表征：如果平均价格为120元，那么一张" 15元代99元"的优惠券的折扣比例为15.15 % ( = 15 / 99 )，门槛比例为82.5 % ( = 99 / 120 )
    - 我们使用这两个指标对平台的优惠券选择行为空间进行离散化
    - 将连续的discount和threshold ratios分别离散成5个水平，排列组合后总共有25中优惠券类型
    - “平均价格”包括同一直播视频中显示的所有产品。在淘宝直播上，消费者每个直播视频只能收到一张优惠券，但优惠券可以应用于视频中的任何产品。
  - **revenue:**未购买则revenue为0，购买了则revenue等于（使用了优惠券后的）支付金额
  - **terminal:**一旦消费者停止回到平台，终端状态就发生了。这种状态捕捉了消费者的流失。



- 基于以下五个假设开展本研究
  - 假设1：卖家对平台的优惠券分配决策没有控制权
  - 假设2：优惠券供应无限量
  - 假设3：何时使用优惠券不在本研究考虑范围内，也无法累积优惠券并进行比较
  - 假设4：消费者看哪个直播不受优惠券的影响
  - 假设5：消费者无法预期未来的变化，消费者的目标是最大化当前效用

- 思想：因为消费者会基于历史优惠券水平评估未来的优惠券优惠力度，所以优惠券目标政策可能会影响当前收入和未来收入，那么本文则意图求解一个能够灵活考虑这种“跨期权衡”的最优优惠券目标策略



2. ### **模型框架：**

- **Batch Deep Reinforcement Learning (BDRL) Framework**

  - $S$ 状态
  - $A$ 动作
  - $R_t$ 回报，本文中：$R_t$即消费者GMV，由优惠券如何影响消费者的最终购买决策决定
  - $\delta$ 折扣系数
  - $p\left(R_t, \mathbf{S}_{\mathbf{t}+\mathbf{1}} \mid \mathbf{S}_{\mathbf{t}}, A_t\right)$ 过程
  - 目标：最大化总折扣奖励，即回报 $\sum_{t=\tau+1}^{\infty} \delta^t R_t\left(\mathbf{S}_t, A_t, \mathbf{S}_{t+1}\right)$
  - 在强化学习框架中，"环境"是根据agent的动作和当前状态值产生奖励和状态变化的过程。本文中：环境是三种消费者决策的组合：1 )流失，即是否离开平台且从未回来，2 )搜索，即访问的哪个网页，3 )购买，即收到优惠券后是否购买
  - <img src="C:\Users\13560\AppData\Roaming\Typora\typora-user-images\image-20230605232333330.png" alt="image-20230605232333330" style="zoom:67%;" />

- **State**

  - 静态状态：用户（demographic）、商家、产品、网页 基本情况

    ![image-20230605235024094](C:\Users\13560\AppData\Roaming\Typora\typora-user-images\image-20230605235024094.png)

  - 动态状态：基于RFM模型评估用户、商家、产品、网页情况（eg：用户消费频次、频率、金额）

    ![image-20230605235041014](C:\Users\13560\AppData\Roaming\Typora\typora-user-images\image-20230605235041014.png)

  - 状态转换

    - 静态状态转换：基本上静态状态在整个样本周期中是静止的，比如性别女则一直都是女
    - 动态状态转换：随机的，不受优惠券信息影响，因为消费者是否购买是随机的。
    - 一旦达到终端状态，即用户永久离开平台，就不会有后续的状态转换。

- **Action**

  - 每张优惠券由**折扣比率**和**门槛阈值比率**两个属性来表征：如果平均价格为120元，那么一张" 15元代99元"的优惠券的折扣比例为15.15 % ( = 15 / 99 )，门槛比例为82.5 % ( = 99 / 120 )
  - 我们使用这两个指标对平台的优惠券选择行为空间进行离散化

- **Evaluation**

  - 强化学习文献中的大多数研究使用两类方法中的一种来进行政策评估。
  - 一类是**逆倾向加权**(IPW)方法，也称为**重要性抽样**(IPS)方法。IPS背后的思想是，当对新政策做出反事实预测时，产生的行动分布与数据中观察到的分布不同，我们可以使用倾向得分来重新权衡原始观察结果，这样我们就可以使用重新加权的观察结果作为估计新政策价值的对照组。换句话说，IPS使用重要性加权来纠正历史数据中不正确的操作比例。
  - 第二类**策略评估方法被称为直接方法**，它使用历史数据来学习**状态-行为空间与奖励之间的函数映射**。然后，可以使用估计的奖励函数来计算**价值函数**。
  - 本文采用**双鲁棒估计方法**，它结合了这两种方法，两种方法有一种正确即可
  - 新政策$\pi^e$
  - 值函数$V^{\pi_e}=E_{P, \pi_e}\left[\sum_{t=0}^T \delta^t R_t\right]$

​	

​		**实验开展：**

- - 基于四种状态：用户、卖家、产品、网页计算状态变量
  - 用折扣比率和阈值比率讲行动空间离散化，现用25张优惠券表示此行动空间，且动作集合不随时间变化
  - 训练-测试，二八分
  - 目的：针对三个基准模型：线性回归模型、GBDT和DNN推导出相应的策略
  - 实验结果：表明需求动态模型对优惠券的有效性起着重要作用，因此有必要考虑优惠券目标策略对消费者的影响。最后，使用BDRL()生成的动态瞄准策略使CLV增加了63 %



​		**策略建议：**

- - 主播影响力小，需发放高折扣优惠券；反之亦然

  - 对于新顾客，GBDT推荐更多高折扣券，而BDRL推荐更多低折扣券。更一般地，BDRL建议随着时间的推移给予越来越高的折扣，这与参考价格效应一致：如果平台给予新消费者高折扣的优惠券，他们未来可能会将高折扣的价格作为参考，使他们不太可能对未来的激励做出反应。动态政策似乎认识到向新消费者提供高折扣优惠券的长期负面后果，因此采取了更为保守的做法，即随着消费者变得更加忠诚，逐渐提高折扣水平。

    <img src="C:\Users\13560\AppData\Roaming\Typora\typora-user-images\image-20230606003752142.png" alt="image-20230606003752142" style="zoom:67%;" />

  - 建议初期给予低消费群体(相对于高支出者)略高的折扣，随着消费者获得体验，以更快的速度增加折扣。这种优惠券政策可能是为了应对低消费人群比高消费人群更高的价格敏感度，向这些顾客提供越来越大的折扣，以保持他们的持续参与。

    <img src="C:\Users\13560\AppData\Roaming\Typora\typora-user-images\image-20230606003913509.png" alt="image-20230606003913509" style="zoom:67%;" />



​		**研究不足：**

- - 泛化性不强：数据集来自于淘宝，实验均基于淘宝数据展开

  - 周期较短：消费者可能没有时间在这种短期干预中调整偏好或行为

  - 用户群体：只选择了活跃用户进行分析

  - 数据运用：没有将非结构化数据（如图像、音频、视频特征）合并为状态变量

  - 后续考虑将降价定价/加价定价/同质动态政策纳入研究中

  - 未来的研究可以探索两种解决方案：将问题建模为部分可观测的MDP，或者使用多智能体强化学习并将每个消费者段视为单独的智能体



## How causal machine learning can leverage marketing strategies: Assessing and improving the performance of a coupon campaign

-- Langen, Henrika, and Martin Huber. "How causal machine learning can leverage marketing strategies: Assessing and improving the performance of a coupon campaign." *Plos one* 18.1 (2023): e0278937.

**研究目的：**

使用因果机器学习方法来评估营销活动的因果影响，以支持决策。研究旨在评估不同类型优惠券的平均影响，并分析不同客户子群体、不同商品类目之间因果效应的异质性，最终使用最优策略学习来确定应针对哪些客户群体进行优惠券活动，以最大化营销干预的销售效果。

**实验过程：**

本文的实验过程涉及应用因果机器学习算法来评估优惠券活动对零售商销售的因果影响。该研究使用零售商销售数据进行分析，并应用因果分析和最优策略学习算法来确定最优优惠券分配策略。

Step1：**客户细分**，只有优惠券有促进作用的细分消费群体，才列为本次优惠券活动的目标。 举例说明了针对肉类和海鲜商品优惠券的最优政策建议，此类优惠券应发放给以下目标群体：活动前消费未超过一定水平的低收入客户、46岁以上的中高收入客户，活动前一段时间内从商店购买过商品的老年人或老年人，以及活动前一段时间内没有从商店购买过任何商品的至少有 5 名成员的中高收入家庭。

Step2：**因果机器学习**，估计优惠券活动对零售商销售额的因果影响。 具体来说，该研究使用因果森林算法，它是随机森林算法的一种变体，旨在估计因果效应。用于估计优惠券活动对零售商销售额的 ATE，即处理组（收到优惠券的客户）和对照组（未收到优惠券的客户）之间的销售额差异。

> 随机森林是一种机器学习算法，属于集成学习的一种类型。集成学习的基本思想是将多个分类器组合，以达到更好的预测效果。随机森林可以同时应用于分类和回归任务。它采用Bagging的思想，即通过有放回地从训练集中取出样本组成新的训练集，并使用多个弱分类器进行集成。最终的预测结果可以通过投票或取均值的方式得到，从而具有较高的精确度和泛化性能[[1]](https://zhuanlan.zhihu.com/p/471494060)[[2]](https://zhuanlan.zhihu.com/p/265703650)。
>
> 因果随机森林是将随机森林算法与因果推断相结合的一种方法。因果推断是计量经济学中的重要问题之一。传统的因果推断方法在数据量较小、对异质性讨论不充分的情况下应用较多。然而，随着对个体处理效应估计的需求增加，使用异质性处理效应的方法面临一些问题，例如可能会选择性地报告处理效应较强的个体。为了解决这些问题，Athey和Imbens提出了因果森林方法，它基于决策树和随机森林的非参数算法。与传统的非参数方法相比，因果森林方法具有维度灾难的问题较小的优势。因果森林将随机森林的预测能力与因果推断相结合，提供了一种强大的工具来估计和推断个体处理效应[[3](https://zhuanlan.zhihu.com/p/46803675)]。
>
> 总结起来，随机森林是一种集成学习算法，通过组合多个弱分类器进行分类和回归任务。因果随机森林则是将随机森林算法应用于因果推断，用于估计和推断个体处理效应。)

$Y(1)$表示用户收到优惠券后的购买行为，$Y(0)$表示没有使用优惠券的购买行为，优惠券的因果效应对应于有无优惠券的购买行为差异$Y(1)-Y(0)$，由于一个用户使用/没使用优惠券进行购买这样的行为是互斥行为，因此本文考虑使用ATE，即优惠券分配对总顾客购买行为Y的平均影响，用Δ表示的ATE对应于平均潜在结果$Y(1)$$和$$Y(0)$中的差异$Δ=E[Y(1)-Y(0)]$，根据本文相关指标定义，ATE平均结果表示为
$$
\theta = E[E[Y|D=1,X] - E[Y|D=0,X]]
$$
Y表示平均每日支出，D表示是否使用优惠券，X是协变量（包括活动前优惠券接收/使用情况，以及活动前日均消费等信息）

Step3：**最优策略学习算法**，以确定优惠券活动应针对哪些客户群，以最大限度地提高其有效性。将用户的基本信息以及过往的优惠券使用行为作为细分群体的依据，将优惠券的条件平均处理效果 (CATE)记为价值函数，然后使用这些估计来构建优惠券分配策略，根据客户的预期 CATE 向客户分配优惠券。 本节还讨论了使用双稳健估计器来估计 CATE，以及平衡探索（即针对不确定是否反馈优惠券的客户）之间权衡的重要性。

> 双稳健估计（Bivariate Robust Estimation）是一种统计估计方法，用于处理具有异常值或离群点的双变量数据。它是对传统的最小二乘估计方法的一种改进，能够在存在异常值的情况下提供更稳健和可靠的估计结果。
>
> 双稳健估计方法的目标是降低异常值对估计结果的影响，并提高估计的稳定性和准确性。它采用一些鲁棒性较强的统计量或模型，来代替传统的最小二乘估计中对数据分布的假设。这些鲁棒性统计量或模型能够减少异常值对估计结果的影响，从而提供更可靠的估计。
>
> 双稳健估计方法常用的统计量包括中位数（median），M-估计（M-estimation），MM-估计（MM-estimation），Huber估计等。这些方法在计算估计量时，对异常值的影响进行了控制，更加关注数据中的典型观测值，而非受异常值影响较大的极端观测值。
>
> 双稳健估计方法在许多统计学应用中都有广泛的应用，特别是在金融风险管理、经济学、环境科学等领域。它能够提供更可靠的估计结果，减少异常值对数据分析的干扰，从而改善决策和推断的准确性。
>
> 双稳健估计是一种针对双变量数据的统计估计方法，通过使用鲁棒性统计量或模型，降低异常值对估计结果的影响，提供更稳健和可靠的估计。

总体而言，本文的实验过程涉及数据驱动的客户细分过程，使用因果机器学习算法来估计优惠券活动对零售商销售的因果影响，并使用最优策略学习算法来确定最优优惠券分配策略。

**实验结论：**

- 药店优惠券对过往购买量高的客户有效；食品优惠券对过往购买量低的客户有效

- 领取任何优惠券对活动周期的日常开支都有正向且统计上显著的影响。给顾客提供优惠券使她的预期日常开支增加了大约60个货币单位。
- 为药店物品和其他食物提供优惠券对日常开支有统计上显著的正向影响，但对整体客户群没有统计上的显著影响。领取这些类别的优惠券分别使有效期内的预期日均支出增加约80和78个货币单位。另一方面，发放适用于其他非食品类产品的优惠券，估计会使顾客的预期日均支出减少约24个货币单位。

​		

[数据结构](https://www.kaggle.com/datasets/vasudeva009/predicting-coupon-redemption?select=item_data.csv)：

- 数据集包含了零售店顾客的社会经济特征、活动期间收到的优惠券以及优惠券兑换和购买行为等信息。
- 将每一次发放优惠券记为一个时序活动，总共有18个部分时序重叠活动。
- 商店分布了208种不同的优惠券，每种类型适用于多达12,000种产品，其中大部分属于同一产品类别。将优惠券分为5大类：即食食品、肉类和海鲜、其他食品、药店物品和其他非食品产品的优惠券
- 优惠券的发放方式是每位顾客每次活动获得0 ~ 37张不同的优惠券，总共有1582个用户。
- 定义33个优惠券活动周期（原文叫做优惠券活动周期campaign period）。目的是为了探究：可以将购买行为从一个人工竞选期到另一个竞选期的变化完全归因于在各自时期内有效的优惠券。为了考虑竞选期持续时间的差异，我们考虑每个客户和竞选期的**平均每天支出**作为损失函数。为了估计优惠券提供对购买行为的因果效应，我们将顾客在不同活动期间的具体购买行为进行合并，每个顾客得到33个观测值。

​		**train.csv**[id, campaign_id, coupon_id, customer_id, redemption_status(0/1)]
​		**campaign_data.csv**[campaign_id, campaign_type(匿名X/Y), start_date, end_date]
​		**item_data.csv**[item_data, brand, brand_type, category]
​		**coupon_item_mapping.csv**[coupon_id, item_id]
​		**customer_demographics.csv**[customer_id, age_range, marital_status, rented, family_size, no_of_children, income_bracket]
​		**customer_transaction_data.csv**[date, customer_id, item_id, quantity, selling_price, other_discount, coupon_discount]
​		**test.csv**[id, campaign_id, coupon_id, customer_id]

![image-20230708114634206](C:\Users\13560\AppData\Roaming\Typora\typora-user-images\image-20230708114634206.png)

![image-20230604153221524](C:\Users\13560\AppData\Roaming\Typora\typora-user-images\image-20230604153221524.png)

​		



## Understanding Users' Coupon Usage Behaviors in E-Commerce Environments

数据[Coupon Usage Data for O2O_数据集-阿里云天池 (aliyun.com)](https://tianchi.aliyun.com/dataset/59)
[天池新人实战赛o2o优惠券使用预测_学习赛_天池大赛-阿里云天池 (aliyun.com)](https://tianchi.aliyun.com/competition/entrance/231593/information)

问题：为了帮助商家提高优惠券的使用率，通过预测用户使用优惠券的概率，以便确定商家是否对用户发放优惠券

模型：尝试了朴素贝叶斯、支持向量机、随机森林、GBDT、XGBOOST等机器学习模型
随机森林效果最差，XGBOOST效果最好

数据预处理过程：

- 特征分为六种类型，包括用户特征、商家特征、优惠券特征、组群特征、类别特征、时间特征



代码[wepe/O2O-Coupon-Usage-Forecast: 1st Place Solution for O2O Coupon Usage Forecast (github.com)](https://github.com/wepe/O2O-Coupon-Usage-Forecast)








