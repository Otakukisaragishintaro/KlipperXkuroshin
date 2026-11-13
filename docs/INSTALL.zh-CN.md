# 中文安装指南：重复 Rapid Scan + MAD 异常值过滤

[返回中文项目说明](../README.zh-CN.md)

> **重要：当前补丁仍处于实验阶段。** 尚未完成自动化回归测试和打印机实测。第一次安装请严格按照本文的“隔离副本测试”操作，不要直接修改正在控制打印机的 Klipper。

## 你需要准备什么

- 一台可以运行 Linux 命令的电脑或 Linux 手机主机；
- 已安装 `git`、`python3` 和 `curl`；
- 能访问 GitHub；
- 了解你的 Klipper 源码目录在哪里（如果只是先做隔离测试，则不需要已有目录）。

下面所有命令都可以逐段复制执行。命令中的目录名可以按需修改，但同一流程中要保持一致。

## 方案 A：先在独立目录测试（推荐）

此方案会另外下载一份 Klipper 源码，不会改动你当前正在使用的 Klipper。

### 第 1 步：下载指定版本的 Klipper

打开终端，逐行执行：

```bash
mkdir -p ~/klipperxkuroshin-test
cd ~/klipperxkuroshin-test

git clone https://github.com/Klipper3d/klipper.git klipper
cd klipper

git checkout 29d4ec2d6d840ee839333306003a400da186d41d
```

最后一条命令固定到本补丁开发时使用的上游基准版本。不要跳过这一步，也不要直接拿任意最新版套用补丁。

### 第 2 步：下载补丁文件

仍在刚才的 `klipper` 目录中执行：

```bash
curl -fL https://raw.githubusercontent.com/Otakukisaragishintaro/KlipperXkuroshin/main/patches/0001-bed-mesh-rapid-scan-mad-filter.patch -o /tmp/klipperxkuroshin-rapid-scan.patch
```

检查文件是否下载成功：

```bash
ls -lh /tmp/klipperxkuroshin-rapid-scan.patch
```

如果提示下载失败，先不要继续；检查网络或 GitHub 访问是否正常。

### 第 3 步：先检查补丁能否应用

确认当前目录是刚下载的 Klipper 源码目录，然后执行：

```bash
git apply --check /tmp/klipperxkuroshin-rapid-scan.patch
```

- **没有任何输出且返回成功**：表示补丁在当前源码上通过了应用检查，可以继续。
- **出现 error / failed**：表示补丁无法干净应用。请停止，不要强行修改文件；先确认是否检出了上面指定的 commit，以及源码是否被其他修改污染。

### 第 4 步：正式应用补丁

只有第 3 步成功后才执行：

```bash
git apply /tmp/klipperxkuroshin-rapid-scan.patch
```

检查改动范围：

```bash
git status --short
git diff --check
git diff -- klippy/extras/bed_mesh.py
```

本补丁预期只修改 `klippy/extras/bed_mesh.py`。如果出现意料之外的文件变更，请先停止并检查。

### 第 5 步：进行 Python 语法检查

```bash
python3 -m py_compile klippy/extras/bed_mesh.py
```

命令无输出且退出码为 0，表示该文件通过 Python 语法编译检查。**这不等于功能测试通过**，也不能代替 Klipper 回归测试。

### 第 6 步：查看新增配置项

本补丁的配置项属于 `[bed_mesh]`：

```ini
[bed_mesh]
rapid_scan_repeats: 5
rapid_scan_mad_multiplier: 3.0
rapid_scan_outlier_min: 0.020
```

参数含义：

- `rapid_scan_repeats`：同一条扫描路径重复次数。默认 1（保持原行为）；允许 1 或大于等于 3 的整数；2 不允许；4、6 等偶数次数允许。
- `rapid_scan_mad_multiplier`：MAD 阈值系数，默认 3.0。
- `rapid_scan_outlier_min`：异常值判断的最小绝对阈值，单位 mm，默认 0.020。这个值目前只是实验初值，尚未经过多机器噪声标定。

第一次验证建议先保持默认的 1 次扫描，确认默认配置兼容；之后再在**测试机器**上尝试 3 次或 5 次。不要直接把实验配置用于无人看管的正式打印。

### 第 7 步：如何撤销补丁

在这个隔离测试目录中执行：

```bash
git apply -R /tmp/klipperxkuroshin-rapid-scan.patch
git diff --check
git status --short
```

如果补丁是唯一改动，撤销后 `git status --short` 应不再显示该文件的补丁改动。

## 方案 B：应用到现有 Klipper 源码（暂不建议）

只有在隔离副本测试完成、确认你的 Klipper 版本与补丁基准完全一致，并且已经备份后，才考虑这样做。

### 第 1 步：进入现有 Klipper 源码目录

把下面路径替换成你机器上的真实路径：

```bash
cd ~/klipper
```

确认版本：

```bash
git rev-parse HEAD
```

输出必须是：

```text
29d4ec2d6d840ee839333306003a400da186d41d
```

如果不是这个 commit，**不要继续套用**。不要通过删除补丁检查或强制覆盖来绕过版本差异。

### 第 2 步：确认没有未提交的源码修改并备份目标文件

```bash
git status --short
cp klippy/extras/bed_mesh.py ~/bed_mesh.py.before-kuroshin-patch
```

如果 `git status --short` 显示你自己已有的改动，先妥善保存；不要用 reset 或 checkout 强行清除。

### 第 3 步：下载、检查并应用

```bash
curl -fL \
  https://raw.githubusercontent.com/Otakukisaragishintaro/KlipperXkuroshin/main/patches/0001-bed-mesh-rapid-scan-mad-filter.patch \
  -o /tmp/klipperxkuroshin-rapid-scan.patch

git apply --check /tmp/klipperxkuroshin-rapid-scan.patch
```

只有检查成功才执行：

```bash
git apply /tmp/klipperxkuroshin-rapid-scan.patch
python3 -m py_compile klippy/extras/bed_mesh.py
git diff --check
```

### 第 4 步：重启 Klipper 主机服务

应用 Python 主机端代码后，通常需要重启 Klipper 主机服务才能加载新代码；**不需要因此重新编译或刷写 MCU 固件**。

如果你的系统确实使用名为 `klipper` 的 systemd 服务，可以执行：

```bash
sudo systemctl restart klipper
sudo systemctl status klipper --no-pager
```

如果你使用的是手机上的 Beam Klipper、容器或其他启动管理方式，请使用对应方式重启 Klipper 主机进程；不要盲目执行上面的 systemd 命令。重启后先检查 Klipper 日志和配置加载结果，不要立即开始打印。

## 方案 C：直接替换完整文件（不使用 git apply）

如果你不熟悉 Git 补丁操作，也可以下载仓库提供的完整 `bed_mesh.py`，手动替换 Klipper 中对应文件。

**注意：这个完整文件不是通用版。** 它由 Klipper 基准 commit `29d4ec2d6d840ee839333306003a400da186d41d` 的原始 `klippy/extras/bed_mesh.py` 加上本项目补丁生成。只有源码版本完全一致时才能使用。不同 commit 的完整文件不能直接覆盖。

### 第 1 步：先确认版本

进入你的 Klipper 源码目录：

```bash
cd ~/klipper
git rev-parse HEAD
```

输出必须为：

```text
29d4ec2d6d840ee839333306003a400da186d41d
```

如果不是，停止操作，不要用这个完整文件覆盖。

### 第 2 步：备份原文件

```bash
cp klippy/extras/bed_mesh.py ~/bed_mesh.py.before-kuroshin-patch
```

如果目标文件已有你自己的修改，请先另外保存这些修改；备份只是防止文件覆盖后无法恢复，不会自动合并你自己的改动。

### 第 3 步：下载完整替换文件

```bash
curl -fL https://raw.githubusercontent.com/Otakukisaragishintaro/KlipperXkuroshin/main/manual-replacement/bed_mesh.py -o /tmp/bed_mesh.py.kuroshin
```

### 第 4 步：替换文件

```bash
cp /tmp/bed_mesh.py.kuroshin klippy/extras/bed_mesh.py
```

### 第 5 步：检查语法和文件状态

```bash
python3 -m py_compile klippy/extras/bed_mesh.py
git diff --check
git status --short
```

语法检查通过不代表功能测试通过。请继续按照 [中文测试清单](TESTING.zh-CN.md) 验证；不要直接开始无人看管的正式打印。

### 第 6 步：恢复原文件

如果需要撤销替换：

```bash
cp ~/bed_mesh.py.before-kuroshin-patch klippy/extras/bed_mesh.py
python3 -m py_compile klippy/extras/bed_mesh.py
```

如果替换后你又对 `bed_mesh.py` 做了其他修改，恢复备份会覆盖这些后续修改，请先自行保存。

## 常见问题

**提示 patch does not apply / patch failed**

通常是源码版本不匹配、文件已有改动，或补丁文件下载不完整。回到方案 A，重新检出指定基准 commit，再从头检查。

**Python 编译成功，是否代表可以打印？**

不是。语法检查只排除部分语法错误。仍需完成 MAD 单元测试、bed_mesh 回归测试、TAP / faulty region / overshoot / zero reference 等行为检查，以及受控硬件验证。

**如何恢复原版？**

隔离副本使用 `git apply -R`。如果已经应用到现有源码，优先使用同一补丁执行反向应用：

```bash
git apply -R /tmp/klipperxkuroshin-rapid-scan.patch
```

若反向应用失败，不要继续覆盖文件；使用事先备份的 `bed_mesh.py.before-kuroshin-patch`，并先确认期间没有其他人或更新修改过该文件。

## 下一步测试

请按照 [中文测试清单](TESTING.zh-CN.md) 记录每项结果。当前项目尚未宣称通过完整测试。
