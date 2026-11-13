# 单文件替换与备份教程（简体中文）

[返回中文 README](../README.zh-CN.md) | [中文安装总指南](INSTALL.zh-CN.md) | [中文测试清单](TESTING.zh-CN.md)

> **重要：这是实验性代码。** 完整替换文件只对应指定的 Klipper 基准版本。请先在独立副本中验证；不要把它当作适用于任意 Klipper 版本的通用文件。

## 适用范围

- 仓库文件：`manual-replacement/bed_mesh.py`
- 目标文件：`klippy/extras/bed_mesh.py`
- 唯一适用的基准 commit：

```text
29d4ec2d6d840ee839333306003a400da186d41d
```

如果你的 Klipper commit 不完全相同，**停止操作，不要覆盖**。即使文件名相同，不同版本的 `bed_mesh.py` 也可能有其他代码变更。这个替换操作会覆盖目标文件中的本地修改，不会自动合并。

## 一、先确认 Klipper 版本和文件状态

进入你的 Klipper 源码目录。以下假设目录是 `~/klipper`，如果实际位置不同，请修改第一行：

```bash
cd ~/klipper

git rev-parse HEAD
```

输出必须逐字等于：

```text
29d4ec2d6d840ee839333306003a400da186d41d
```

再检查目标文件有没有你自己的修改：

```bash
git status --short -- klippy/extras/bed_mesh.py
git diff --quiet -- klippy/extras/bed_mesh.py
git diff --cached --quiet -- klippy/extras/bed_mesh.py
```

前一条命令应无输出；后两条命令都应正常结束且返回码为 0。若有输出或报错，先保存、提交或另外备份自己的修改，不要继续覆盖。

> 这几条只检查 Git 能识别的改动。若你曾经用其他方式改过文件，请自行确认这些修改已经备份。

## 二、下载完整替换文件到临时位置

仍在 Klipper 源码目录执行：

```bash
curl -fL --retry 3 \
  https://raw.githubusercontent.com/Otakukisaragishintaro/KlipperXkuroshin/main/manual-replacement/bed_mesh.py \
  -o /tmp/bed_mesh.py.kuroshin
```

确认下载文件存在且非空：

```bash
test -s /tmp/bed_mesh.py.kuroshin && echo "下载成功"
wc -l /tmp/bed_mesh.py.kuroshin
```

先对下载文件做 Python 语法检查：

```bash
python3 -m py_compile /tmp/bed_mesh.py.kuroshin
```

无报错表示这个文件通过了语法编译检查；它**不代表**功能测试或硬件测试通过。

## 三、备份原始 bed_mesh.py（关键步骤）

建议每次备份都使用独立文件名，避免覆盖以前的备份：

```bash
BACKUP="$HOME/bed_mesh.py.backup-$(date +%Y%m%d-%H%M%S)"
cp -a klippy/extras/bed_mesh.py "$BACKUP"
echo "原文件备份位置：$BACKUP"
```

请把终端显示的备份路径保存下来。确认备份文件存在：

```bash
ls -l "$BACKUP"
```

如果 `cp` 报错或备份文件不存在，**不要继续替换**。

## 四、替换文件

如果 Klipper 正在运行，先通过你当前使用的服务管理方式停止 Klipper 主机进程。不要在不了解启动方式时盲目执行 systemd 命令。替换 Python 主机端文件不需要重新编译或刷写 MCU 固件。

确认仍在 Klipper 源码目录，然后执行：

```bash
cp /tmp/bed_mesh.py.kuroshin klippy/extras/bed_mesh.py
```

检查目标文件：

```python
python3 -m py_compile klippy/extras/bed_mesh.py
git diff --check
git status --short -- klippy/extras/bed_mesh.py
```

如果语法检查失败，或发现文件内容/状态不符合预期，先按下一节恢复备份，不要启动打印任务。

## 五、如何恢复原始文件

将第三步显示的备份路径填入下面命令。示例路径仅供说明，请换成你终端实际显示的路径：

```bash
cp -a "$HOME/bed_mesh.py.backup-20261114-120000" klippy/extras/bed_mesh.py
python3 -m py_compile klippy/extras/bed_mesh.py
```

恢复后检查：

```bash
git diff -- klippy/extras/bed_mesh.py
```

如果备份是在任何修改之前创建的，恢复后该文件相对 Git 基准应不再显示差异（前提是原文件当时干净）。恢复完成后，再按原来的方式启动 Klipper 主机进程，并检查日志。

**注意：** 不要把示例中的时间戳照抄；必须使用你实际创建的备份文件名。恢复备份会覆盖替换后对该文件做的所有新修改，因此如有后续改动，先另行保存。

## 六、重启与日志检查

通过你设备原有的管理方式重新启动 Klipper 主机进程。若使用 systemd 且服务确实叫 `klipper`，才使用：

```bash
sudo systemctl restart klipper
sudo systemctl status klipper --no-pager
```

若使用 Beam Klipper、容器或其他启动器，请从对应管理界面/脚本重启，不要照搬 systemd 命令。

重启后先检查 Klipper 主机日志和配置加载结果，确认没有 Python traceback、配置解析错误或探针初始化错误。不要因为进程启动成功就立即进行无人看管的打印。

## 七、常见问题

**版本不匹配怎么办？**

停止操作。不要强行覆盖，也不要通过忽略检查来绕过。请使用与当前 Klipper commit 对应的补丁，或在匹配的基准源码副本中测试。

**我已经改过原版 bed_mesh.py，怎么办？**

不要直接替换。先单独保存自己的改动，再比较差异并手动移植；完整替换文件不会自动保留本地修改。

**Python 编译通过，是否代表可以正式打印？**

不是。还需要完成 MAD 合成数据测试、单次扫描兼容性检查、bed_mesh 行为回归、Eddy 探针流程检查和受控硬件验证。请逐项填写[中文测试清单](TESTING.zh-CN.md)。

**替换文件和 .patch 有什么区别？**

- 完整替换文件：直接替换一个完整的 `bed_mesh.py`，操作简单，但必须严格匹配基准版本，并先备份。
- `.patch`：只记录代码差异，适合 Git 工作流；应用前可用 `git apply --check` 检查，通常更容易审查变更。

## 安全提醒

- 不要把此文件用于不同 Klipper commit。
- 不要覆盖未备份的本地修改。
- 不要在未检查日志和测试结果时开始正式打印。
- `rapid_scan_outlier_min: 0.020` 仍是暂定值，不是已经完成噪声标定的通用参数。
