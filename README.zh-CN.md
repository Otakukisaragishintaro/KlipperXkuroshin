# KlipperXkuroshin（中文说明）

[English](README.md) | **简体中文**

这是一个针对 Klipper `bed_mesh` 快速扫描（rapid scan）流程的独立补丁提案，目标上游项目为 [Klipper3d/klipper](https://github.com/Klipper3d/klipper)。

> **项目状态：实验性开发中。** 当前代码已合并到本仓库的 `main` 分支，但尚未完成补丁应用检查、自动化回归测试和打印机实测，也尚未被 Klipper 官方接受。请勿将其视为稳定版本。

## 当前功能提案

本项目尝试为 Eddy 涡流探针的 rapid scan 增加可配置的重复扫描，以及基于中位数绝对偏差（MAD，Median Absolute Deviation）的逐点异常值过滤。

主要设计：

- 首先生成一次完整扫描路径，然后每轮重复使用同一条路径。
- 根据扫描路径点的顺序对多轮结果进行对应，不跨点混合数据。
- 过滤仅针对 `ProbeResult.bed_z` 高度值。
- 保留的高度样本取算术平均值；每个扫描点最终只向现有 bed mesh 完成回调提交一个结果。
- 不修改 Eddy 单个采样窗口内部的频率平均，也不修改频率到高度的转换过程。
- 默认重复次数为 1，目的是保持原有单次扫描行为。

## 配置提案

在配置文件的 `[bed_mesh]` 段中加入：

```ini
[bed_mesh]
rapid_scan_repeats: 5
rapid_scan_mad_multiplier: 3.0
rapid_scan_outlier_min: 0.020
```

参数说明：

| 参数 | 含义 | 当前默认值 / 规则 |
|---|---|---|
| `rapid_scan_repeats` | 每条扫描路径重复采集的次数 | 默认 1；允许 1 或大于等于 3 的整数；拒绝 2；偶数次数（如 4、6）也允许 |
| `rapid_scan_mad_multiplier` | MAD 阈值系数 | 默认 3.0，必须大于 0 |
| `rapid_scan_outlier_min` | 异常值判定的最小绝对阈值，单位 mm | 默认 0.020，必须大于 0；目前是暂定值，尚未通过不同打印机的实测标定 |

### MAD 过滤方法

对同一个扫描路径点收集多轮高度值：

```text
center = median(z)
MAD = median(abs(z - center))
threshold = max(outlier_min, multiplier * 1.4826 * MAD)
```

偏离中位数超过阈值的样本会被剔除。剩余有效样本必须至少有两个，然后计算这些样本的算术平均值。

这里的 MAD 只用于判断同一个扫描点的多次测量是否存在异常，不会对相邻网格点做空间平滑，也不会用邻点数据替换当前点。

## 源码审查记录

目前已对照项目记录的 Klipper 基准源码检查部分调用流程：

- Python 会先执行命令处理器中的 `finally`，之后 Klipper 才在命令处理流程中分发 `gcode:command_error` 事件。
- Eddy 的错误处理器会检查 `_gather` 是否仍然存在，因此正常清理路径不会必然导致二次结束探针会话。
- 扫描路径会先物化，再供每一轮重复扫描使用，以保持各轮路径顺序一致。
- 在启动探针会话前增加空扫描路径检查，避免没有任何测量点时仍然开启探针会话。

更完整的变更和审查记录见 [CHANGELOG.md](CHANGELOG.md)。

## 补丁与基准版本

- 上游项目：[Klipper3d/klipper](https://github.com/Klipper3d/klipper)
- 当前补丁基准 commit：`29d4ec2d6d840ee839333306003a400da186d41d`
- 补丁文件：[`patches/0001-bed-mesh-rapid-scan-mad-filter.patch`](patches/0001-bed-mesh-rapid-scan-mad-filter.patch)
- 测试计划：[`docs/testing.md`](docs/testing.md)
- 开发记录：[`CHANGELOG.md`](CHANGELOG.md)

本项目是独立补丁提案，不是 Klipper 官方仓库，也不代表已经获得官方认可。不同 Klipper 版本之间可能存在源码差异，应用补丁前必须核对基准版本。

## 安装、更新日志与测试

- **[完整替换文件 bed_mesh.py](manual-replacement/bed_mesh.py)**：适用于指定基准版本的完整文件，可用于不方便执行 `git apply` 的手动替换方式；必须先备份原文件并核对版本。
- **[中文傻瓜式安装指南](docs/INSTALL.zh-CN.md)**：从下载独立 Klipper 测试副本、检查补丁、应用补丁，到语法检查和撤销补丁，提供逐步复制执行的命令。
- [中文更新日志](CHANGELOG.zh-CN.md)：记录每次功能变化、源码审查结论和未完成事项。
- [中文测试清单](docs/TESTING.zh-CN.md)：逐项记录静态检查、行为回归和受控测试。

第一次使用请先按安装指南的**方案 A：隔离副本测试**操作，不要直接修改正在运行的打印机 Klipper。

## 测试状态与参与开发

当前仍需完成：

1. 在干净的 Klipper 基准源码副本上执行 `git apply --check`，确认补丁能够正确应用。
2. 执行 Python 语法 / 编译检查。
3. 对 MAD 算法进行自动化测试，包括偶数和奇数重复次数、MAD 为零、异常值、有效样本不足等情况。
4. 回归检查普通扫描、TAP、轴扭曲补偿、故障区域、overshoot 和 zero reference 等原有流程。
5. 在隔离和可控的条件下进行打印机硬件测试，并记录扫描重复性和网床结果。

欢迎中文开发者参与源码审查、测试设计、问题复现和文档改进。提交问题时请尽量附上：

- Klipper 的具体版本或 commit；
- 完整配置片段（请移除个人隐私信息）；
- 完整错误日志；
- 复现步骤和预期 / 实际结果；
- 是否使用 Eddy rapid scan，以及探针和主板的相关信息。

**在自动化测试和硬件测试完成前，请将此功能视为实验性代码。** 特别是 `rapid_scan_outlier_min: 0.020` 仍是待标定的初始建议值，不是适用于所有机器的最佳参数。
