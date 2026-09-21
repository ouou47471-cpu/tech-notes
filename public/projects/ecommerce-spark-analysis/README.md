# 基于 PySpark 的电商经营分析与用户价值分层

## 项目一句话

用 PySpark 处理 2016–2018 年约 10 万条巴西电商订单（Olist 六表关联），构建订单级与商品行级双层宽表，
完成区域、趋势、品类、客户结构四个维度的经营分析，并用 RFM 把 9.47 万真实客户分成四层——
全程在本地 Spark 会话即可跑通。

## 数据与口径

| 项 | 说明 |
| --- | --- |
| 数据源 | Kaggle：Brazilian E-Commerce Public Dataset by Olist（2016–2018） |
| 规模 | 约 10 万条订单记录，订单 / 订单商品 / 商品 / 客户 / 支付 / 品类翻译 共 6 张表 |
| 成交口径 | 过滤无效订单状态；支付金额按订单聚合并剔除异常金额（保留剔除日志） |
| GMV 口径 | 区域 / 趋势 / RFM 用**订单级宽表**；品类用**商品行级宽表**（GMV = price + freight_value） |
| 客户口径 | 用 `customer_unique_id` 识别真实客户，而非订单级的 `customer_id` |

## 分析内容

| 模块 | 说明 |
| --- | --- |
| 数据建模 | 6 表关联，分别构建订单级（一行一订单）与商品行级（一行一商品）双层宽表 |
| 数据清洗 | 金额类型转换、多格式时间戳兼容解析、无效状态过滤、异常金额剔除 |
| 区域分析 | 各州 GMV / 订单量 / 客单价排名 |
| 趋势分析 | 月度 GMV 走势与环比增速（窗口函数 `lag`） |
| 品类分析 | 商品类目 GMV 排名（葡语类目名翻译为英文） |
| RFM 分层 | 按最近消费 R、频次 F、金额 M 打分，划分高价值 / 重要 / 一般 / 低价值四层 |

## 核心结论

- **区域高度集中**：圣保罗州（SP）GMV 约 585 万 BRL、4.1 万单，是第二名里约州（RJ，209 万）的约 2.8 倍；
  头部三州（SP / RJ / MG）贡献全平台约 **62%** 的营收。
- **品类格局**：健康美妆（144 万 BRL）、手表礼品（129 万）、床品卫浴（124 万）为 GMV 前三类目；
  手表礼品的件均价最高（约 218 BRL）。
- **增长趋势**：2017 年起月度 GMV 进入稳定增长通道，2017 年 11 月（黑五）单月 GMV 达 **116.7 万 BRL** 峰值，
  2018 年稳定在月均 100 万 BRL 以上，客单价约 160 BRL。
- **客户结构**：识别出真实客户 **9.47 万人**，复购率仅 **3.0%**——拉新强、留存弱；
  RFM 分层显示高价值客户 603 人、人均消费 436 BRL，是低价值客户（57 BRL）的 7 倍以上。

## 成果展示

| 各州 GMV Top 10 | 月度 GMV 趋势 |
| --- | --- |
| ![各州 GMV](public/projects/ecommerce-spark-analysis/01-state-sales.png) | ![月度趋势](public/projects/ecommerce-spark-analysis/02-monthly-trend.png) |

| 品类 GMV 排名 | RFM 客户分层 |
| --- | --- |
| ![品类排名](public/projects/ecommerce-spark-analysis/03-category-ranking.png) | ![RFM 分层](public/projects/ecommerce-spark-analysis/04-rfm-segmentation.png) |

## 实现要点（踩过的坑）

1. **订单级 ID 不是客户 ID**：Olist 的 `customer_id` 一单一换，直接聚合会把订单当客户、复购率恒为 0。
   改用 `customer_unique_id` 聚合后才得到真实的频次与复购率。
2. **GMV 口径会重复累加**：多商品订单里支付金额被摊在订单级，
   若用它算品类 GMV 会把整单金额重复计到每个商品行上；因此品类分析单独走商品行级宽表。
3. **时间戳有两种格式**：Kaggle 原版是 `yyyy-MM-dd`，部分镜像是 `yyyy/MM/dd`，
   混用会静默丢行——脚本同时兼容两种格式，并输出解析失败行数做自检。
4. **RFM 的 F 维度不能直接四分位**：绝大多数客户只买过一次，`ntile` 会把同值客户随机拆进不同层；
   这里 R / M 用四分位，F 改用阈值打分（≥4 / 3 / 2 / 1 次对应 4 / 3 / 2 / 1 分）。
5. **首尾月份会放大增长率**：首尾是不完整月份，订单量低会把环比算得虚高，剔除后重算。

## 复现方式

```bash
pip install -r requirements.txt
# 从 Kaggle 下载 Olist 数据集，解压到 data/
python analysis.py     # PySpark 分析，生成本地 Spark 会话即可运行
python visualize.py    # 生成 4 张图表到 output/charts/
```

> Windows 下运行时出现 `winutils.exe` / `native-hadoop` 相关 WARN 属正常现象，不影响分析结果。

代码仓库：<https://github.com/MimiJimmy001/Ecommerce-Analysis>

## 复盘

这个项目最花时间的不是写 Spark，而是**确认口径**：同一份数据里，"客户是谁""GMV 怎么算"
换个定义结论就会变。把口径写进脚本注释和 README、并对每次清洗输出自检日志，
比多写几个分析维度更值得。
