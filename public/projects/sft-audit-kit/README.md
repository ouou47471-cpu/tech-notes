# SFT Audit Kit：多模态 SFT 数据集质量审计工具

## 项目一句话

一套面向多模态 SFT 数据发布的质量审计工具：在训练前检查格式、图片完整性、重复样本、数据泄漏和模板文本问题，并输出可复查的 HTML、JSON 与 CSV 报告。

## 为什么做这个项目

多模态模型训练失败时，团队往往先怀疑模型结构和训练参数，最后才发现问题在数据：

- 图片路径失效或文件损坏
- 同一个 ID 对应多条记录
- 相同图片在 train 和 test 中重复出现
- instruction 与 response 大量模板化重复
- 数据版本更新后，旧标注没有同步修改
- 数据规模增大后，人工抽查覆盖不足

SFT 数据管线负责把图片变成数据，Audit Kit 负责判断这些数据能否被稳定发布和训练。

## 与现有项目的关系

| 项目 | 主要职责 |
| --- | --- |
| Focus-StyleGAN | 生成工业缺陷伪异常图像 |
| 多模态 SFT 数据管线 | 完成标注、改写、去重、质量过滤和版本化 |
| SFT Audit Kit | 对最终数据集执行发布前审计并生成质量门禁 |

Audit Kit 不是重新训练模型，而是把数据管线中的一次性检查逻辑抽象成可复用工具。

## 系统流程

![SFT Audit Kit 架构](public/projects/sft-audit-kit/architecture.png)

工具首先读取 `dataset.jsonl`，根据同级 `manifest.json` 或 `--dataset-root` 解析图片路径，然后依次执行格式、图片、重复和泄漏检查，最后输出三份报告。

## 当前功能

- 检查 JSONL 是否可解析
- 检查 `image`、`instruction`、`response` 必填字段
- 缺少 `id` 时使用图片路径生成稳定标识
- 从 `meta.annotation.category` 自动读取类别
- 从图片路径自动推断 train/test 划分
- 检查图片是否存在、能否完整解码、尺寸和颜色模式
- 使用 MD5 检测字节级重复图片
- 检查重复 ID
- 检查完全重复的 instruction/response
- 检查同一图片跨 train/test 泄漏
- 统计类别和数据划分分布
- 输出 `report.html`、`report.json` 和 `issues.csv`
- 支持 `--strict` CI 质量门禁

## 输入格式

最小记录格式：

```json
{
  "id": "sample-001",
  "image": "images/001.png",
  "instruction": "判断产品是否存在表面缺陷",
  "response": "产品正常，未发现明显缺陷",
  "category": "normal",
  "split": "train"
}
```

可以兼容以下变体：

- 缺少 `id`：使用图片路径作为稳定标识
- 缺少 `category`：读取 `meta.annotation.category`
- 缺少 `split`：从路径中的 `train` 或 `test` 推断
- 图片位于其他根目录：读取同级 `manifest.json` 的 `dataset_root`

## 真实数据集完整测试

测试对象为真实 MVTec AD v8 SFT 数据集：

| 指标 | 结果 |
| --- | ---: |
| 总记录数 | 1725 |
| 无错误记录 | 1725 |
| Error | 0 |
| Warning | 116 |
| 唯一图片 | 1725 |
| 重复图片 | 0 |
| 缺失图片 | 0 |
| 重复 ID | 0 |
| train/test 泄漏 | 0 |
| 类别覆盖 | 15 |

116 条警告全部来自模板化 `instruction/response` 完全重复。工具按重复组聚合，避免同一问题被逐条放大。

![1725 条真实数据审计报告](public/projects/sft-audit-kit/audit-report.png)

## 关键问题与解决方案

### 问题 1：真实数据集没有统一顶层 ID

现象：

现有 SFT 数据只在 `image`、`instruction`、`response` 和 `meta` 中保存信息，没有顶层 `id`。旧版工具会把 1725 条记录全部标记为缺少 ID。

解决方案：

把 `id` 改为可推导字段：优先使用顶层 ID，否则使用图片路径，最后回退到行号。这样既能兼容已有数据，也能保持标识稳定。

### 问题 2：图片根目录不在 JSONL 同级目录

现象：

JSONL 中保存的是相对于 MVTec AD 根目录的路径，直接按 JSONL 所在目录解析会导致 1725 张图片全部缺失。

解决方案：

支持 `--dataset-root`，并自动读取同级 `manifest.json` 的 `dataset_root`。工具会先验证根目录和图片路径，再执行完整性检查。

### 问题 3：固定图片尺寸白名单造成大量误报

现象：

第一版只允许 256、512、768、1024 等正方形尺寸，导致 614 张合法 MVTec 图片被误报。

解决方案：

改为检查真正异常的情况：低于 64×64、高于 4096×4096，或图片无法完整解码。最终误报从 614 条降为 0 条。

### 问题 4：重复文本按记录逐条输出，报告噪声过大

现象：

模板化数据会产生大量相同 response，旧实现为每条记录单独生成警告，最终出现 328 条重复告警。

解决方案：

按 instruction/response 组合分组，每组只生成一条聚合警告。最终警告降到 116 组，更适合人工复核。

## 工程决策

- 核心审计不依赖大模型和 API Key，保证在 CI 中稳定运行。
- 先用 MD5 做字节级精确去重，避免感知哈希把不同缺陷误判为同一图片。
- 图片读取使用完整解码，而不是只读取文件头，避免损坏图片被漏检。
- 报告同时提供 HTML、JSON 和 CSV，对应人工查看、程序消费和 Excel 复核。
- `--strict` 只对 Error 返回失败，Warning 不直接阻断数据发布。

## 运行方式

Windows 用户可以直接双击：

- `run_demo.bat`：运行内置示例
- `audit_dataset.bat`：选择或拖入自己的 `dataset.jsonl`

命令行：

```bash
sft-audit audit --input data/dataset.jsonl --output reports/audit
```

指定图片根目录：

```bash
sft-audit audit --input data/dataset.jsonl --output reports/audit --dataset-root /path/to/images
```

CI 严格模式：

```bash
sft-audit audit --input data/dataset.jsonl --output reports/audit --strict
```

## 项目边界

- 当前版本使用 MD5 检测字节级重复，尚未加入图片近重复检测。
- 当前版本没有 VLM 图文语义一致性判断。
- 当前检测规则还没有在人工标注基准集上计算准确率和召回率。
- 当前报告是可读 HTML，不包含交互式数据看板。

## 下一阶段

- 基于感知哈希的图片近重复检测与阈值校准
- 图片、instruction、response 的语义一致性评分
- 数据集版本差异和回归报告
- 规则配置文件和自定义严重级别
- GitHub Actions 自动质量门禁
- 10k 以上记录的稳定性与性能测试
