# PCBA 缺陷检测：全流程复现与标注同步增广

## 项目一句话

在工业质检场景里把 PCBA 缺陷检测的全流程跑通（数据体检 → 增广 → 光照校正 → 手写 Darknet53+YOLOv3
训练 → mAP 评估 → 导出推理模型），过程中定位出原方案"增广只改图、不改标注框"的致命缺陷，
自己实现了**标注同步增广**，并用**控制变量的 A/B 实验**量化它到底值不值得做。

## 项目背景

这个项目起点是一套工业缺陷检测的课程实验：给了 15 份实验指导书、一个 PCBA 缺陷数据集和一个 AIStudio 笔记本。
我做的不是"跟着跑一遍"，而是把它当成一个真实项目来对待：

- 原版实验在 GPU 上跑 100 个 epoch、输入 640，最终 mAP 只有 23.38%——先搞清楚**为什么这么低**；
- 逐份读实验指导书时发现数据增广那一步只产出图片、**从不更新 XML 标注框**，等于产出了一批废数据；
- 于是补上标注同步增广，并设计 A/B 实验验证它是否真的有用，而不是"看起来有用"。

## 数据集与一个关键发现

600 张 2448x2048 的 PCBA 表面图像，VOC 格式标注，5 类缺陷：`short`、`skewing`、`tombstoning`、
`solder_bridge`、`open_solder`，共 4552 个标注框。

![数据集体检](public/projects/pcba-defect-detection-repro/dataset-overview.png)

| 指标 | 数值 | 说明 |
| --- | ---: | --- |
| 图像 / 标注框 | 600 / 4552 | 平均每张 7.6 个框 |
| 瑕疵框中位尺寸 | 73x73 px | **仅占整图面积 0.11%** |
| 灰度均值范围 | 97.9 ~ 123.0 | **100% 低于标准灰度 128** |
| 类别分布 | short 1119 / open_solder 1080 / tombstoning 880 / skewing 873 / solder_bridge 600 | 存在不均衡 |

两个发现直接决定了后面的技术路线：**目标太小**（缩到 320 输入后只剩几个像素）→ 精度上不去是必然的；
**整体欠曝** → 采集端就有问题，需要先把数据修好。

## 技术方案

模型不用现成套件，手写 Darknet53 + YOLOv3：

- 骨干：`ConvBNLayer`（conv+BN+LeakyReLU）、`BasicBlock`（1x1 + 3x3 残差）、`DownSample`、
  `LayerWarp`（按 1/2/8/8/4 堆叠残差块）、`DarkNet53_conv_body`；
- 检测头：darknet53 返回 C0/C1/C2（stride 8/16/32），经 `YoloDetectionBlock` 得到 P0/P1/P2，
  P2 上采样与 P1 拼接、再上采样与 P0 拼接，实现三尺度检测；
- 损失：`paddle.vision.ops.yolo_loss`（objectness + 分类 + 定位，`ignore_thresh=0.7`）；
  优化器 Momentum(0.9) + L2(5e-4)，分段学习率 1e-4；
- 参数量 61.6M。

![预测结果（彩色为预测框，灰色为真实框）](public/projects/pcba-defect-detection-repro/prediction-508.jpg)

| 项 | 原版课程实验 | 我的本地复现 |
| --- | --- | --- |
| 平台 | AIStudio + V100 GPU | 8 核 CPU（Windows） |
| Python / Paddle | 3.7 / 2.1.2 | 3.11 / 2.6.2（CPU） |
| 训练规模 | 100 epoch / 640 输入 / batch 1 | 6~12 epoch / 320 输入 / batch 4 |
| mAP(0.50, 11point) | 23.38% | 1.89%（最好一版） |

精度差距来自算力与数据条件，不是流程错误：我的验证 loss 在 90~121 之间波动，
与原版 epoch 99 的 104.47 同量级；而 350 张训练图在第 8 轮附近就开始过拟合（train loss 还在降、val loss 反升）。

## 关键改进：标注同步增广

原实验的增广代码把图片缩放、翻转、裁剪之后直接保存，**完全不管 XML 里的 bndbox**。
这样的增广图拿去训练，框会落在错误的位置，模型学到的是噪声——所以那一步只是"演示"，不能用于训练。

我按照每种变换对坐标的影响重新实现了一遍（`tools/07_annotated_augment.py`）：

| 变换 | 图片 | 标注框 |
| --- | --- | --- |
| 缩放 fx,fy | resize | `xmin*fx, xmax*fx, ymin*fy, ymax*fy` |
| 水平翻转 | flip | `x' = W-1-x`，并交换 xmin / xmax |
| 边界填充 | pad | 加上填充偏移 |
| 裁剪 | crop | 减去裁剪偏移，再 clip |
| 亮度 / 噪声 | 像素变换 | **完全不动** |

变换后的三条清理规则缺一不可：**clip 到图像内**、**剩余面积 <30% 或短边 <8px 的框丢弃**、
**贴边框标 `truncated=1`**；整张图框全被切没了就丢弃该样本。挑图按类别缺口贪心（谁少补谁），
并且**只增广训练集**——验证集增广等于把训练样本泄漏进评估，指标会虚高。

![标注同步增广抽查](public/projects/pcba-defect-detection-repro/annotated-augment-check.jpg)

**怎么证明框没偏？** 不能靠肉眼看抽查图。我写了两层自动校验：

1. 逐框检查坐标落在图像范围内（越界直接报错退出）；
2. 把"原图中该瑕疵的像素块按同样变换算出来"，与增广图中框住的像素块逐一比对——
   实测最大平均绝对差：flip 3.22 / pad 3.33 / resize 4.90 / crop 4.65（0~255 尺度），
   说明框仍精确压在同一个瑕疵上，而不是整体偏移了几个像素。

效果：33 张原图生成 198 张增广图，训练集从 350 张扩到 548 张；
`solder_bridge` 350→548、`tombstoning` 390→713，所有类别样本量都补到中位数水平以上。

## A/B 实验：增广到底有没有用

很多"我做了数据增广"的项目只给出一个变好的数字，但变量没控制。我按控制变量的方式做：

| 控制项 | 设置 |
| --- | --- |
| 权重初始化 | 相同随机种子（两组首步 loss 4423.84 vs 4424.16） |
| 迭代步数 | 两组都是 528 步（相同算力预算） |
| 超参数 | lr 1e-4、Momentum 0.9、L2 5e-4、batch 4、输入 320 完全一致 |
| 验证集 | 完全相同（100 张原图，绝不增广） |
| 唯一变量 | 训练集：A = 350 张原图；B = 548 张（含标注同步增广图） |

| 组 | 验证 loss（132/264/396/528 步） | mAP@528 | short | skewing | tombstoning | solder_bridge | open_solder |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| A（原图） | 237.46 / 96.16 / 90.63 / 98.61 | 2.07% | 9.09 | 0.63 | 0.54 | 0.08 | 0.00 |
| B（增广） | 107.88 / 99.44 / 88.82 / **90.58** | **2.56%** | 7.14 | **5.05** | 0.47 | 0.07 | 0.08 |

![loss 曲线对比](public/projects/pcba-defect-detection-repro/ab-loss-curve.png)
![mAP 与逐类 AP 对比](public/projects/pcba-defect-detection-repro/ab-map-compare.png)

结论与反思：

- 相同算力下增广组 mAP 高 0.49 个百分点（相对 +24%），验证 loss 更低；
- 增益集中在**被定向增广的少数类**：`skewing` 的 AP 从 0.63% 提到 5.05%；
  而样本最多的 `short` 反而原图组更高（9.09 vs 7.14）——增广补的是短板，不是"全面提升"；
- B 组同一时刻 train loss 更高（数据更杂更难拟合）但验证 loss 更低，说明是泛化收益，不是背得更熟；
- **必须说清的局限**：两组 mAP 都只有 2~3%，绝对量级很小，单次实验噪声不可忽视（264 步时 A 组还领先）。
  所以正确表述是"小幅、可解释的泛化提升，方向正确"，而不是"精度提升 24%"。

## 工程踩坑记录

1. **OpenCV 读不了含中文的绝对路径**：`cv2.imread` 内部走本地 ANSI 编码，遇到中文路径静默返回 `None`
   （不报错，最难查）。解法：`np.fromfile` + `cv2.imdecode`。
2. **Paddle 静态图导出同样栽在中文路径**：`paddle.jit.save` 直接 `mkdir failed!`。
   解法：先导出到 ASCII 临时目录，再用 Python 拷回工程。
3. **Paddle 启动要写 `~/.cache/paddle`**：目录无权限时直接 `[WinError 5]`，需要指定可写目录。
4. **验证环节忘了 `paddle.no_grad()`**：计算图每轮累积，内存涨到 5 GB，单步从 6.6 秒掉到 68 秒。
   自己写训练循环时非常容易踩。
5. **顺手修掉的原脚本隐患**：空预测结果未 `.tolist()` 会让 `json.dump` 崩溃；训练用 640、测试用 608 输入尺寸不一致。

## 项目结构

```
src/     手写 Darknet53 + YOLOv3、数据读取、训练/评估/预测、mAP 计算
tools/   数据体检、增广、光照校正、标注同步增广、A/B 训练、A/B 报告、模型导出、断点续训
docs/    数据体检/增广/标注同步增广/光照/逐类 AP/A-B/完整复现报告 + 结果图
samples/ 2 张示例图片与对应 XML（数据集不随仓库分发）
```

## 复现步骤

```bash
pip install -r requirements.txt
# 数据集放到 data_y/{train,val,test}/{images,annotations}
python tools/01_dataset_audit.py          # 数据集体检
python tools/03_lighting_fix.py           # 光照分析 + 校正数据集
python tools/07_annotated_augment.py      # 标注同步增广（含自动校验）
REPRO_IMG_SIZE=320 REPRO_BATCH_SIZE=4 REPRO_MAX_EPOCH=6 python tools/run_train.py
python evaluate.py                        # 输出 mAP
python tools/04_visualize_prediction.py   # 逐类 AP + 预测可视化
python tools/08_ab_train.py --iters 528   # A/B 训练
python tools/09_ab_report.py              # A/B 报告与图表
python tools/05_export_inference_model.py # 导出部署用推理模型
```

代码与完整报告：<https://github.com/MimiJimmy001/PCBA-Defect-Detection-Repro>

## 复盘

这个项目最大的收获不是"会写 YOLOv3"，而是三件事：

1. **数据的问题要在数据层解决**：先体检再建模，才知道 0.11% 的瑕疵面积占比注定让低分辨率训练效果有限；
2. **改进必须可验证**：标注同步增广如果不写自动校验，很容易在"看起来对"的图上翻车；
3. **实验必须控制变量**：把"看起来有用"变成"在相同算力下更快收敛、少数类 AP 更高，但绝对量级仍小"，
   诚实地写出局限，比夸大结论更能站得住。

下一步计划：把步数拉到 2000+ 并用多个随机种子重复实验，再补上 Mosaic 增广与更高分辨率的对比。
