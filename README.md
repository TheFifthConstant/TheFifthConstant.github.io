# TheFifthConstant.github.io


## 生存分析（任务 2.1 和 2.2）

本部分基于 IBM Telco Customer Churn 数据集，使用 PySpark 进行数据预处理，借助 Python 库 `lifelines` 完成生存分析建模。所有代码均已重写并添加详细注释（见 `Project1_2.ipynb`）。

### 5.1 数据预处理

1. 从 GitHub 下载 `Telco-Customer-Churn.csv`，用 PySpark 加载。
2. 清洗：将 `Churn` 列转为数值（1=流失，0=留存），筛选出月度合同且订购了互联网服务的客户，最终得到 3,351 个有效样本。
3. 将 Spark DataFrame 转为 Pandas，供生存分析库使用。

### 5.2 Kaplan-Meier 分析

- **总体生存曲线**：绘制所有目标客户的 Kaplan-Meier 曲线，中位生存时间约 34 个月，即一半客户预计在 34 个月内流失。
![总体生存曲线](/assets/images/Kaplan_curve_all.png)
- **分组比较**：按 `InternetService` 分组绘制生存曲线。Fiber optic 用户的生存曲线明显低于 DSL 用户，表明光纤用户流失风险更高。
![InternetService 存曲线](/assets/images/Kaplan_curve_internet.png)
- **Log-rank 检验**：对 `InternetService` 进行两两比较，p 值 < 0.001，差异具有统计学显著性。

### 5.3 Cox 比例风险模型

**编码与建模**：对 5 个解释变量 (`Dependents`, `InternetService`, `OnlineBackup`, `TechSupport`, `PaperlessBilling`) 进行独热编码，拟合 Cox 模型。

| 变量 | 系数 (coef) | 风险比 (exp(coef)) | p 值 |
|------|------------|------------------|------|
| `Dependents_Yes` | -0.32 | 0.73 | <0.005 |
| `InternetService_Fiber optic` | 0.20 | 1.22 | <0.005 |
| `OnlineBackup_Yes` | -0.79 | 0.46 | <0.005 |
| `TechSupport_Yes` | -0.64 | 0.53 | <0.005 |
| `PaperlessBilling_Yes` | 0.15 | 1.16 | 0.02 |
![Cox风险系数](/assets/images/Cox_model_hazard_ratios.png)

- 拥有受抚养人（`Dependents_Yes`）的客户流失风险显著降低（HR = 0.73）。
- 订购光纤互联网（`InternetService_Fiber optic`）的客户流失风险略高（HR = 1.22）。
- 订购 `OnlineBackup` 和 `TechSupport` 服务的客户流失风险大幅降低（HR 分别为 0.46 和 0.53）。
- 开通电子账单（`PaperlessBilling`）的客户流失风险略高（HR = 1.16）。
- 模型的 Concordance 指数为 0.64，似然比检验高度显著，整体拟合较好。

**比例风险假定检验**

通过 Schoenfeld 残差检验发现，`InternetService_Fiber optic`、`OnlineBackup_Yes` 与 `TechSupport_Yes` 三个变量的 p 值小于 0.05，提示它们随时间变化的效应可能不满足完全的比例风险假设，因此在解释这些变量的长期风险关系时需谨慎。

### 5.4 加速失效时间 (AFT) 模型

作为半参数模型的补充，本代码拟合了 Log-Logistic AFT 模型。为全面考察变量影响，模型纳入了以下解释变量（均进行独热编码）：  
`Partner`、`MultipleLines`、`InternetService`、`OnlineSecurity`、`OnlineBackup`、`DeviceProtection`、`TechSupport`、`PaymentMethod`。

主要结果如下：

1. `PaymentMethod_Electronic check` 与 `PaymentMethod_Mailed check` 的系数分别为 **-0.88** 和 **-0.96**（均 p < 0.005），表明使用电子支票或邮寄支票支付的客户显著“加速”了流失时间（有效缩短了客户存续周期）。
2. `OnlineSecurity_Yes` 的系数为 **1.05**（p < 0.005），表明订购在线安全服务的客户能够显著延长生存时间（“减速”流失）。
3. `Partner_Yes` 的系数为 **0.87**（p < 0.005），拥有伴侣的客户同样表现出更长的存续时间。
4. 其余变量（如部分互联网服务类别、其他增值服务及支付方式）在 AFT 框架下的效应不显著或贡献较小。

该模型的一致性（Concordance）为 0.67，AIC = 14,083.47，对数据有良好的拟合。

### 5.5 客户终身价值（CLV）分析

作为生存模型的延伸应用，本代码基于已拟合的 Cox 模型，对特定客户队列进行终身价值估算。

定义一个 `get_payback_df()` 函数，输入队列的关键特征（是否拥有受抚养人、是否使用 DSL、是否订购 OnlineBackup 及 TechSupport）及年折现率，函数内部：
1. 使用 `predict_survival_function()` 预测各月存活概率；
2. 假定月均利润为固定值（代码中为 $30），计算期望月利润；
3. 按月度折现率折现后，得到各月净现值（NPV）和累计 NPV。

生成的结果包含 **累计 NPV 柱状图**（12/24/36 个月）与 **该队列的生存概率曲线**，直观展示客户随时间流失对价值的影响。

![累计 NPV 柱状图](/assets/images/Cumulative_NPV.png)
![该队列的生存概率曲线](/assets/images/Survival_probability.png)

通过对比不同特征组合的累计 NPV，可量化留存策略（如推广增值服务）的长期经济收益，为精准营销和预算分配提供数据支持。

### 5.6 综合分析结论

针对月度合同且使用互联网服务的客户群体：
1. Fiber optic 用户流失风险显著高于 DSL 用户。
2. 推广在线备份和技术支持等高附加值服务能有效降低流失。
3. 电子账单客户流失倾向更高，可能因为减少了纸质接触点。
4. 应重点对光纤用户和电子账单用户实施保留策略，同时鼓励订购技术支持和在线备份服务。
4. 应重点对光纤用户和电子账单用户实施保留策略，同时鼓励订购技术支持和在线备份服务。
