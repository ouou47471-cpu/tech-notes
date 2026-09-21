# 基于 Focus-StyleGAN 的工业缺陷图像增广系统

## 项目一句话

面向工业异常检测中的缺陷样本稀缺问题，设计双分支解耦生成器 Focus-StyleGAN，把缺陷生成和背景保持分开建模，再通过注意力融合与多尺度判别器生成可控的伪异常图像。

这个项目是基础项目。“多模态 SFT 数据构建与质量过滤管线”是在它基础上扩展出来的下游项目：基础项目解决伪异常图像如何生成，扩展项目继续解决 GAN 输出如何变成可追溯、可评测的训练数据资产。

## 项目背景

工业质检中最常见的问题不是模型结构，而是异常样本太少：

- 生产线稳定运行，缺陷本身稀少，某些类别只有个位数样本。
- 传统几何变换和亮度变化只能复制已有缺陷，无法产生新的形态与纹理。
- 人为制造缺陷成本高，破坏性测试周期不可控。

项目利用 GAN 从正常样本分布生成伪异常，并额外评估生成缺陷是否保持产品背景、边缘和物理结构。

## 模型架构

![Focus-StyleGAN 模型架构](public/projects/focus-stylegan-augmentation/architecture.png)

Focus-StyleGAN 由四部分组成：

- 缺陷聚焦分支：渐进式上采样和 AdaIN 风格注入，控制缺陷类型、形态和严重程度。
- 背景保持分支：编码器-解码器结构，保留产品结构、光照和背景纹理。
- 注意力融合模块：学习空间权重，将缺陷分支和背景分支平滑融合。
- 多尺度判别器：三个 PatchGAN 子判别器并行工作，每个特征层集成 CBAM 通道和空间注意力。

## 训练策略

- 损失函数：WGAN-GP 对抗损失、VGG19 感知损失、L1 重构损失、LPIPS 损失。
- 权重设置：`L_G = 1.0 * L_adv + 10.0 * L_perc + 50.0 * L_recon + 1.0 * L_lpips`。
- 优化器：Adam，`beta1 = 0.5`、`beta2 = 0.999`。
- 训练节奏：每训练 5 次判别器，训练 1 次生成器；100 epoch，batch size 8。
- 超参数搜索：Optuna 贝叶斯优化 50 次试验，以验证集 FID 最小化为目标。

![训练损失与收敛曲线](public/projects/focus-stylegan-augmentation/training-curve.png)

![Optuna 超参数搜索过程](public/projects/focus-stylegan-augmentation/optuna-history.png)

## 生成质量结果

| 方法 | FID ↓ | IS ↑ | LPIPS ↓ | PSNR ↑ | SSIM ↑ | PPS ↑ |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 标准 GAN | 52.3 | 2.1 | 0.32 | 18.5 | 0.72 | 0.43 |
| DCGAN | 45.2 | 2.1 | 0.28 | 19.8 | 0.75 | 0.51 |
| WGAN-GP | 38.6 | 2.5 | 0.25 | 20.5 | 0.78 | 0.59 |
| StyleGAN2 | 26.3 | 2.7 | 0.21 | 22.3 | 0.82 | 0.68 |
| FocusGAN | 31.5 | 2.6 | 0.23 | 21.5 | 0.80 | 0.64 |
| Focus-StyleGAN | 22.5 | 2.9 | 0.18 | 23.8 | 0.85 | 0.79 |

![生成图像质量对比](public/projects/focus-stylegan-augmentation/quality-comparison.png)

项目中还设计了 PPS 物理合理性得分，从几何一致性和光照一致性两个角度评价生成缺陷是否符合材料规律。Focus-StyleGAN 的 PPS 为 0.79，优于对比方法。

## 下游异常检测提升

每个类别生成 1000 张伪异常图像，与原始正常样本一起训练 PaDiM：

| 指标 | 原始数据 | 增广后 | 提升 |
| --- | ---: | ---: | ---: |
| Pixel-AUC 平均值 | 0.852 | 0.943 | +10.7% |
| PRO-AUC 平均值 | 0.828 | 0.925 | +11.7% |

缺陷样本越少，提升越明显：

- Cable：Pixel-AUC 0.78 → 0.92，提升 17.9%。
- Capsule：Pixel-AUC 0.82 → 0.94，提升 14.6%。
- Pill：Pixel-AUC 0.80 → 0.91，提升 13.8%。
- Screw：Pixel-AUC 0.83 → 0.93，提升 12.0%。
- 全部 15 个类别都取得正向提升。

![Pixel-AUC 与 PRO-AUC 对比](public/projects/focus-stylegan-augmentation/detection-comparison.png)

![各工业类别 Pixel-AUC 提升](public/projects/focus-stylegan-augmentation/per-class-improvement.png)

## 消融实验

### CBAM 注意力

移除 CBAM 后平均 FID 从 22.5 升到 25.1，相对退化 11.6%。纹理复杂类别退化更明显，说明注意力机制能够引导判别器聚焦缺陷区域。

### 损失函数

| 配置 | FID | 说明 |
| --- | ---: | --- |
| 完整模型 | 22.5 | 所有损失项启用 |
| 移除感知损失 | 29.3 | 语义一致性下降 |
| 移除重构损失 | 27.6 | 缺陷与背景协调性下降 |
| 移除 LPIPS | 24.1 | 感知质量略有下降 |
| 移除全部辅助损失 | 36.8 | 仅使用对抗损失 |

### 双分支结构

| 架构 | FID | IS | LPIPS | 结果 |
| --- | ---: | ---: | ---: | --- |
| 单分支生成器 | 34.2 | 2.5 | 0.27 | 背景常出现扭曲 |
| 双分支生成器 | 22.5 | 2.9 | 0.18 | 缺陷更清晰，背景更自然 |

![消融实验对比](public/projects/focus-stylegan-augmentation/ablation-comparison.png)

## 工程实现

系统提供四种图像增广模式：

- GAN 伪异常生成。
- 真实缺陷迁移。
- 检索式增广。
- 缺陷堆叠增广。

前端使用 HTML、CSS 和 JavaScript，后端基于 Flask、PyTorch 与 RESTful API。系统演示页面支持上传图像、自动匹配类别、生成缺陷、查看分析结果和批量处理。

<video controls preload="metadata" playsinline poster="public/projects/focus-stylegan-augmentation/system-demo.png"><source src="https://github.com/ouou47471-cpu/tech-notes/releases/download/project-media-assets/focus-stylegan-system-demo.mp4" type="video/mp4">你的浏览器不支持视频播放。</video>

## 关键问题与解决方案

### 问题 1：单分支生成器难以同时学习缺陷与背景

现象：

单个生成器同时学习缺陷形态和产品背景时，背景容易扭曲，FID 较高。

解决方案：

将生成任务拆成缺陷聚焦分支和背景保持分支，并用注意力模块完成空间融合。消融实验显示 FID 从 34.2 降到 22.5，背景结构也更稳定。

### 问题 2：判别器对微小低对比度缺陷不敏感

现象：

单尺度判别器主要关注全局结构，容易忽略小面积、低对比度缺陷。

解决方案：

使用三尺度 PatchGAN 并行判别，并在每个特征层加入 CBAM 通道与空间注意力。移除 CBAM 后平均 FID 退化 11.6%。

### 问题 3：旧 checkpoint 与重构后的模型结构不完全兼容

现象：

后续代码重构修改了缺陷分支输出尺寸、判别器结构和融合模块，加载早期 checkpoint 时出现权重键缺失。

解决方案：

保留明确的权重加载告警，不再静默忽略缺失项。旧 checkpoint 仅用于生成路径验证，正式复现实验需要按当前代码重新训练。

### 问题 4：Windows 中文路径会影响部分图像库读取

现象：

OpenCV 在部分 Windows 环境下无法直接处理中文路径。

解决方案：

使用 ASCII 临时目录中转，或直接通过 `imdecode` 读取文件字节，避免路径编码问题影响批处理。

## 可复现边界

- 论文中的 FID、IS、LPIPS、PPS 和检测指标对应论文实验版本与对应训练配置。
- 后续代码重构与早期 checkpoint 不完全兼容，不能用旧权重直接代表当前代码的正式实验结果。
- 项目目前聚焦静态批量增广，论文中预留的“生成→检测→反馈→再生成”动态闭环尚未完成。
- 模型按类别单独训练，跨类别泛化能力有限。
- 生成缺陷仍受训练数据中已见缺陷类型限制。

## 与扩展项目的关系

Focus-StyleGAN 项目的目标是把伪异常图像生成出来。扩展项目“面向工业质检的多模态 SFT 数据构建与质量过滤管线”进一步解决：

- GAN 输出如何转成结构化图像描述和 SFT instruction；
- 如何用 MD5 精确去重、JSONL 校验和版本目录保证数据可追溯；
- 如何用 VLM 对生成图做语义质量过滤；
- 为什么 FID 下降并不代表合成数据一定可用于训练。

两个项目共同构成从“图像生成”到“数据资产生产与质量评估”的完整技术演进。
