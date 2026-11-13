# 更新日志（简体中文）

[English changelog](CHANGELOG.md) | **简体中文**

## 2026-11-14 — 完整替换文件与单文件备份教程

- 在 `manual-replacement/bed_mesh.py` 提供完整的 `bed_mesh.py` 替换文件。该文件由指定上游基准 commit `29d4ec2d6d840ee839333306003a400da186d41d` 的原始文件与本项目补丁生成。
- 新增[单文件替换、备份与恢复教程](docs/SINGLE-FILE-REPLACEMENT.zh-CN.md)，包含版本核对、本地修改检查、下载、语法预检、带时间戳备份、替换、重启和日志检查、恢复备份等步骤。
- 更新中文 README 和安装指南入口，并扩充测试清单，明确区分补丁应用检查与完整文件替换检查。
- 本次文档更新不代表完整文件已经通过编译、自动化回归或硬件测试。

## 2026-11-14 — 源码审查与防御性加固

### 已对照上游源码确认

- 在基准 commit `29d4ec2d6d840ee839333306003a400da186d41d` 下审查了 `klippy/gcode.py`。
- 确认 `_process_commands()` 会在 G-code 命令处理器退出后才捕获命令错误。因此，Python 会先执行 rapid scan 处理函数中的 `finally`，Klipper 随后才分发 `gcode:command_error`。
- 确认 Eddy 的 `_handle_command_error()` 仅在 `_gather` 非 `None` 时调用 `end_probe_session()`。由于 `finally` 已经结束会话并清空 `_gather`，这条正常错误处理路径不会必然发生二次清理。此前提出的“必然二次清理”是审查判断错误，并非已确认的代码缺陷。
- 审查了 `EddyGatherSamples`：探针请求按队列顺序处理，分析结果按请求顺序追加；当每轮扫描都按相同顺序提交路径时，可以按路径点索引分组。
- 审查了 `ProbeManager.iter_rapid_path()`：生成路径包含探测点、非探测移动、故障区域替代点、可选 overshoot 移动，以及可选的 zero-reference 探测点。将生成器先转换为列表，可以让每一轮重复扫描重放相同的路径序列。

### 本次修改

- 增加空扫描路径保护：如果生成的 rapid-scan 路径不包含任何探测点，则在启动探针会话之前报错。
- 在清理代码旁增加注释，说明已确认的 `finally` 与 `gcode:command_error` 执行顺序。
- 扩充验证清单，要求检查上述两项行为。

### 尚未验证

- 尚未在本地 Klipper 源码副本上执行 `git apply --check`。
- 尚未完成 Python 编译检查、自动化测试、硬件测试和测量噪声底标定。
- `rapid_scan_outlier_min: 0.020` 仍是暂定值，不能视为适用于所有机器的通用阈值。

## 2026-11-14 — 初始功能提案

- 为 `[bed_mesh]` 提出重复 rapid scan 配置。
- 为同一扫描路径点的多次 `bed_z` 测量提出 MAD 异常值过滤。
- 默认重复次数为 1，目标是保留原有单次扫描行为。
- 允许重复次数为 1 或大于等于 3 的整数；拒绝 2；允许 4、6 等偶数。
- 对通过过滤的样本取算术平均值，每个路径点只向原有网床完成流程提供一个结果。
- 不修改 Eddy 单个采样窗口内的频率平均和频率到高度的转换过程。

