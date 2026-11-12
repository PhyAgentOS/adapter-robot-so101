# SO101

[English](README.md) | [简体中文](README_zh.md)

## BREAKING CHANGE：Forge 2.x / Dora 1.x 迁移

本次从 Forge 1.x / Dora 0.4.x 升级需要协同部署。节点对接前必须对齐 Forge 包、Dora CLI/coordinator/daemon、本节点及上下游节点的版本；不支持在 Forge 1.x 部署中只替换单个节点。可执行产物也需要使用匹配环境重新构建和部署。仍使用旧技术栈的部署请保留在 `stable/1.x`。

本轮验证的 Python 基线为 `forge-msgs==2.1.0`、`Pillow==13.0.0`、`forge-common==2.1.0`、按需使用的 `forge-robot` / `forge-policy` / `forge-kinematics` / `forge-tool` 为 `2.0.0`，以及 `dora-rs==1.0.1`。同时遵守本仓库的 Python 和 SDK 平台要求。Forge 各包族独立发布，无需强行使用同一个版本号。Rust 节点继续使用已发布的 `forge_msgs` crate `2.0.0` 和 Dora `1.0.1`；Python Msgs `2.1.0` 不要求升级到尚未发布的 Rust `2.1.0`。消息命名空间 `forge_msgs.v1` 保持不变。

这里的 breaking change 指旧技术栈的部署迁移，不代表真机、真实模型或长时间运行已完成验收。实际测试和未完成项见[本轮验证报告](migration/msgs-2.1.0-report.json)。

[查看机器人构型说明](description.md)

SO101 机械臂资源与运行入口，包含 MuJoCo MJCF 描述文件、真机 dora 节点、串口工具和 PyInstaller 单文件打包脚本。

## 目录结构

```text
.
|- assets/
|  |- mjcf/
|     |- SO101.xml
|- configs/
|  |- config.example.yaml
|- scripts/
|  |- build_pyinstaller.sh
|  |- robots_so101.spec
|- src/
|  |- robots_so101/
|     |- main.py
|     |- config.py
|     |- driver.py
|     |- feetech_bus.py
|     |- device_tools.py
|- pyproject.toml
```

## MJCF 资源

主模型文件位于 `assets/mjcf/SO101.xml`，可作为 MuJoCo 场景或机器人节点配置的模型输入。

## 配置

复制 `configs/config.example.yaml` 为 `configs/config.yaml` 并按实际串口修改。配置来源优先级：

1. 命令行 `--config /path/to/config.yaml`
2. 环境变量 `SO101_NODE_CONFIG`
3. dora dataflow 节点的 `config` / `config_path`

## 运行

`robots-so101` 是统一 CLI 入口。无子命令或使用 `run` 时运行节点：

```bash
uv sync
uv run robots-so101 --config configs/config.yaml
uv run robots-so101 run --config configs/config.yaml
```

串口工具与测试命令：

```bash
uv run robots-so101 list-devices --json
uv run robots-so101 activate-devices --json
uv run robots-so101 test --config configs/config.yaml --duration 15
```

`list-devices` 和 `activate-devices` 仅匹配 `ttyACM*` 设备。`list-devices` 的 JSON 结构与 Piper 统一：`{"ports": [{"name", "address", "status"}]}`，其中 `name` 可直接作为串口路径使用，`status` 表示对应字符设备权限是否已是 `chmod 666`。`test` 会循环夹爪开合，用于确认串口与机械臂对应关系。`activate-devices` 会对匹配到的串口设备临时执行 `chmod 666`（Linux，需 `pkexec`）。长期推荐将用户加入 `dialout` 组：

```bash
sudo usermod -aG dialout $USER
# 注销后重新登录
```

## Python 包

项目使用 `uv_build`，包模块名为 `robots_so101`，入口定义在 `pyproject.toml`：

```toml
[project.scripts]
robots-so101 = "robots_so101.main:main"
robots_so101 = "robots_so101.main:main"

[tool.uv.build-backend]
module-name = "robots_so101"
```

## PyInstaller 单文件打包

安装打包依赖：

```bash
uv sync --extra pyinstaller
```

执行打包脚本，产物位于 `dist/robots_so101`：

```bash
chmod +x scripts/build_pyinstaller.sh
./scripts/build_pyinstaller.sh
```

也可以直接运行 PyInstaller：

```bash
uv run --extra pyinstaller pyinstaller --noconfirm --clean scripts/robots_so101.spec
```

打包后示例：

```bash
./dist/robots_so101 --config configs/config.yaml
./dist/robots_so101 list-devices --json
./dist/robots_so101 activate-devices --json
```

若运行时缺少模块，可在 `scripts/robots_so101.spec` 的 `hiddenimports` 中补充。

## 主要依赖

- `dora-rs`：dora 节点运行。
- `robot-core`：机器人驱动协议与节点 runner。
- `forge-msgs`：`JointCommand` / `JointState` 消息与 Arrow 序列化。
- `pyserial`：Feetech STS 舵机串口通信。
- `mujoco`：MJCF 模型与仿真相关工作流。

## Forge 2.x 迁移

本分支准备节点版本 `2.0.0`，适配 Dora 1.x，迁移验证锁定 Dora `1.0.1`。使用正式 Forge 2.x 分发包；Common 至少为 `2.1.0`。此次更新包括依赖锁文件与各独立可执行入口。

每轮报告必须列出仓库、改动和测试证据，包括各节点的 `dora run` 记录，以及包含 `adapter-forge-gateway` 的串联测试。缺少硬件或模型的验收项保持待验证。源码 PR 不等于正式发布；整体验收通过后再同步 GitLab。
