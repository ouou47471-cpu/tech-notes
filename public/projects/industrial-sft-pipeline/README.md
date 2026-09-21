# 面向工业质检的多模态 SFT 数据构建与质量过滤管线

## 项目一句话

把工业缺陷生成器的产出，经过「程序化标注 → VLM 改写 → 去重校验 → 质量过滤 → 版本化」，变成可以直接对接 LLaMA-Factory 的多模态 SFT 数据资产。

这个项目是在“基于 Focus-StyleGAN 的工业缺陷图像增广系统”基础上扩展出来的下游项目。它继承了基础项目的生成能力，同时进一步覆盖数据管线、语义质量过滤、异常检测评测和 Web 演示，重点解决“生成出来的图到底能不能用于训练”。

## 项目背景

工业质检场景存在两个现实问题：

- 真实缺陷样本数量有限，类别分布不均衡，直接训练容易出现过拟合。
- 统计指标并不一定代表数据可用。FID 下降时，生成器仍可能发生类别漂移，输出与目标产品无关的纹理。

基础项目 Focus-StyleGAN 负责生成可控伪异常图像，本扩展项目继续把生成结果转化为可复现、可追溯的数据生产与质量评估管线，以及一个标准化的 SFT 数据集。

## 与基础项目的关系

| 基础项目：Focus-StyleGAN | 扩展项目：多模态 SFT 数据管线 |
| --- | --- |
| 负责学习缺陷分布并生成伪异常图像 | 接收基础项目生成的图像，继续构建标准训练数据 |
| 解决样本不足和缺陷形态单一问题 | 解决标注、改写、去重、质量过滤和版本追溯问题 |
| 使用 FID、IS、LPIPS、PPS、Pixel-AUC、PRO-AUC 评估生成效果 | 使用真实图与生成图盲测验证数据是否真正可用 |
| 保留四种图像增广方式与 Flask 演示系统 | 在生成结果上新增 pipeline A、pipeline B 与 dataset.jsonl |

扩展项目复用基础项目的生成器、增广入口和评估结论，但不重新包装为新的 GAN 训练成果。它的新增价值集中在数据资产生产和质量评测。

## 扩展项目结构

![数据生产与质量过滤流程](public/projects/industrial-sft-pipeline/pipeline-flow.png)

基础项目负责提供图像来源。扩展项目在此基础上增加程序化标注、VLM 改写、MD5 去重、JSONL 校验、VLM 三维质量过滤、盲测对照和版本化目录。

![基础项目 Focus-StyleGAN 架构](public/projects/industrial-sft-pipeline/architecture.png)

上图属于基础项目，作为扩展项目的模型来源说明保留。

## 我具体做了什么

- 复用基础项目生成的伪异常图像，并以 MVTec AD 的 ground-truth mask 提取缺陷类型、九宫格位置、面积和 bbox。
- 清理并统一数据格式，通过 Qwen-VL / 模板降级两条路径生成自然语言 instruction 和 response。
- 修复 dHash 在真实数据上误删 39% 样本的问题，改为 MD5 精确去重，并增加 JSONL 字段校验、版本目录和配置哈希。
- 搭建合成数据质量过滤模块，对 GAN 生成图做自然度、融合度、可用性三维评分，低于 3.0 自动剔除。
- 设计真实缺陷图、真实正常图、GAN 合成图三组盲测对照实验，统一使用中立提示词，避免向 VLM 泄露图片来源。
- 将基础项目已有的 FID、IS、LPIPS、PPS、Pixel-AUC、PRO-AUC 结果作为生成质量证据，而不是重复宣称为扩展项目产出。
- 复用基础项目的 Flask Web 与四种增广模式，扩展项目负责把其输出接入标准化数据管线。

## 核心数据

| 指标 | 结果 |
| --- | --- |
| 真实 MVTec AD 测试集 | 15 个品类，1725 张 |
| 程序化模板 SFT 样本 | 1725 条 |
| bottle VLM 改写样本 | 83 张，83/83 成功 |
| dHash 去重缺陷 | 误删 678/1725，约 39% |
| 修复后的 MD5 精确去重 | 0 误删 |
| 盲测对照实验 | 133 张 |
| 真实缺陷图通过率 | 63/63，100% |
| 真实正常图通过率 | 20/20，100% |
| GAN 合成图通过率 | 24/50，48% |
| 真实图与合成图均分 | 4.72 / 4.88 vs 3.11 |

## 运行截图与真实样本

### 真实缺陷图

真实缺陷图全部通过质量过滤，平均分 4.72。VLM 在 83 张真实图中，有 82 张自发描述为“瓶”或瓶口结构。

![真实缺陷样本](public/projects/industrial-sft-pipeline/real-defect-samples.jpg)

### 真实正常图

真实正常图平均分 4.88，作为分布对照和过滤器召回结果的重要参照。

![真实正常样本](public/projects/industrial-sft-pipeline/real-good-samples.jpg)

### GAN 合成图

GAN 输出出现明显的类别漂移。50 张合成图中只有 24 张通过过滤，而且通过的 24 张里，0 张被描述为瓶身，23 张被描述为轴承、密封圈或内窥结构。

这说明问题不是合成图不够多样，而是生成器持续输出错误类别。此现象应称为“类别漂移 / 身份丢失”，不能称为 mode collapse。

![GAN 合成样本](public/projects/industrial-sft-pipeline/gan-samples.jpg)

### 过滤评测结果

![VLM 盲测过滤结果](public/projects/industrial-sft-pipeline/evaluation-summary.png)

## 关键 Bug 与解决方案

### Bug 1：dHash 去重误删 39% 的样本

现象：

最初使用 dHash 和汉明距离判断重复，在真实 MVTec AD 数据上误删 678/1725 张图片，占比约 39%。不同缺陷类型被错误地判定为重复，例如 broken_small 与 broken_large、contamination 与 broken_large。

原因：

MVTec AD 的测试图通常是“同一个物体 + 不同缺陷”。dHash 只感知低分辨率整体结构，无法区分缺陷类型和局部变化。早期 synthetic demo 图片差异较大，掩盖了这个问题。

解决方案：

去重目标被重新定义为“只删除字节级完全相同的文件”。实现从 dHash 改为 MD5 精确哈希，同时保留版本目录、配置哈希和去重记录。修改后在真实数据上实现 0 误删。

解决结果：

避免训练数据被错误删除，也为后续数据版本追溯提供了稳定基础。

### Bug 2：VLM 调用失败后静默降级

现象：

API key 失效时，程序会回退到模板改写或启发式打分，但 manifest 和报告仍可能记录为 VLM 模式，导致实验数据被错误标记。

原因：

早期逻辑只判断“是否配置 API key”，没有记录每次请求的实际成功或失败。

解决方案：

新增 vlm_success、template_fallback、fallback_reason 等统计字段。管线 B 单独统计 VLM 成功数和启发式降级数，全部失败时输出明确警告，报告中增加“VLM 实际成功 N 条”。

解决结果：

实验记录不再把模板结果伪装成 VLM 结果，任何降级都能追溯，面试时可以准确说明哪些数字来自真实模型调用。

### Bug 3：旧 checkpoint 与新模型结构不完全兼容

现象：

2026-08 代码重构后，生成器缺陷分支输出尺寸、判别器结构和融合模块发生变化。加载旧 checkpoint 时出现判别器权重缺失警告。

原因：

旧版训练代码和新版模型定义的结构不一致，部分判别器权重无法直接映射。

解决方案：

保留明确的加载告警，不静默忽略；将旧 checkpoint 仅用于生成路径验证，正式实验重新训练。代码中补充兼容性说明与加载逻辑。

解决结果：

避免把不兼容的旧模型结果包装成新版本实验结果，也保证后续训练配置的可复现性。

## 最重要的结果：FID 下降不等于数据可用

Kaggle bottle 训练从 30 到 100 epoch 时，FID 约从 187 降到 96。但生成图出现整体偏色、模糊和缺陷形态不明显等问题。

后续真实 checkpoint 输出进一步暴露出类别漂移。VLM 在不知道图片来源的情况下，只通过中立打分提示词，一致识别出合成图是环形工业部件而不是瓶身。

这部分验证了一个比“生成图好不好看”更重要的问题：

统计指标只能说明分布接近，不能保证生成样本保留了业务语义。真正可用的数据管线必须同时具备视觉质量评估、语义一致性检查和可追溯的过滤报告。

## 为什么这是扩展项目，而不是重复基础项目

- 基础项目负责“把图生成出来”，扩展项目负责“把图变成可训练的数据”。
- 基础项目的重点是指标和样本多样性，扩展项目的重点是标注结构、语义一致性和质量过滤。
- 基础项目输出图像与模型结果，扩展项目输出 dataset.jsonl、manifest.json、版本目录和过滤报告。
- 扩展项目发现了基础项目中“FID 改善但图像语义已经漂移”的问题，并用 VLM 盲测提供证据。
- 数据管线与生成器解耦。基础项目未来把 GAN 替换成扩散模型，扩展项目的数据管线仍然可以复用。

## 项目边界

- 1725 条程序化样本来自真实 MVTec AD；bottle 83 张额外经过 Qwen-VL 改写验证。
- 合成图没有进入最终 1725 条 SFT 数据集，合成路径主要用于可行性验证和过滤器区分度验证。
- “过滤后回灌训练并提升下游指标”的闭环实验尚未完成。
- VLM 与人工判断的一致性目前只在 50 张合成图上做了完整人工核验。

## 复现命令

```bash
# 管线 A：模板全量数据
python -m datapipe.pipeline_a --root <MVTec_AD_root> --mode template

# 管线 A：bottle VLM 改写
python -m datapipe.pipeline_a --root <MVTec_AD_root> --category bottle --mode vlm

# 管线 B：合成图质量过滤
python -m datapipe.pipeline_b --input <generated_images> --reference <normal_images>

# 测试
python tests/test_datapipe.py
```

## 技术栈

### 扩展项目新增

Python、Qwen-VL、DashScope 兼容接口、VLM 三维质量评分、程序化标注、MD5 去重、JSONL 校验、配置哈希、版本化 manifest、盲测对照实验。

### 基础项目复用

Focus-StyleGAN、WGAN-GP、AdaIN、CBAM、Optuna、PyTorch、MVTec AD、FID、IS、LPIPS、PPS、Pixel-AUC、PRO-AUC、Flask Web 与 RESTful API。
