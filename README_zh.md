# SO101

[English](README.md) | [简体中文](README_zh.md)

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

## 分版本发布

**发布规则（强制）：** 合并版本/锁文件修改 → 在已审查提交上创建 annotated 版本标签 → 从该标签精确构建 → 校验产物 → 发布。禁止使用移动分支、未打标签的提交构建 Release，禁止覆盖已发布标签。

| 维护线 | 源码 | 节点/包版本 | 运行时 |
| --- | --- | --- | --- |
| Forge 1.x | `stable/1.x` | `1.x`，标签 `v1.X.Y` | Forge 1.x / Dora 0.4.x |
| Forge 2.x | `main` | `2.x`，标签 `v2.X.Y` | Forge 2.x / Dora 1.x |

**旧版版本号统一：** `stable/1.x` 原为 0.x 的节点以 **1.0.0** 发布，保留 Forge 1.x / Dora 0.4.x 依赖与运行行为。已有 1.x 版本保留。节点/包主版本与 Forge 兼容线一致，不再接受 0.x 发布标签。

两条线均从 `main` 运行 **Build versioned release**。先将版本和锁文件修改通过 PR 合并至对应源码分支，再在已审查提交上创建未占用的 **annotated** 标签，标签必须匹配包元数据。禁止移动已有标签或复用已发布版本。工作流使用自身已审查的发布工具与精确标签源码，旧标签无需包含新发布脚本。

```bash
# 替换为已存在、已审查、未发布且与源码元数据一致的标签。
TAG=v1.0.0
gh workflow run release.yml --ref main -f release_line=1.x -f tag="$TAG"
TAG=v2.X.Y
gh workflow run release.yml --ref main -f release_line=2.x -f tag="$TAG"
```

流水线校验标签类型、源码提交、分支归属、源码版本及锁定的 Forge/Dora 兼容性；构建源码归档及项目支持的 wheel/sdist，检查 Python 分发元数据与 SHA-256，并创建或附加至 **GitHub Release 草稿**。不覆盖已发布 Release 或产物；发布前检查全部构建结果与验收依据。该流程不发布节点包至 PyPI。

设置 `build_binaries=true` 复用已有 Ubuntu 20.04 二进制工作流，保留原架构、CLI 和归档校验。也可直接运行 `build-ubuntu20.yml`，传入 `ref`、`version`、`release_line`；`ref` 必须为 annotated 标签，不能为移动分支。仅在所有选定任务通过后附加二进制产物。

源码归档保留标签中的文件；外部 SDK/submodule、模型下载与现场配置仍是独立前置条件。依赖或 SDK 构建失败不能视为发布验收通过。构建成功不代表单节点或串联 `dora run`、真机、真实模型、长稳态或回滚验收通过；每条验收链必须包含相同兼容线的 `adapter-forge-gateway`。Forge 2.x 属于 **breaking change**，整个部署的 Forge 库、节点版本与 Dora 必须对齐；1.x 保留在维护分支。
