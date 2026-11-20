# SO101

[English](README.md) | [简体中文](README_zh.md)

SO101 robot assets, a Dora hardware node, serial-port tools and a PyInstaller executable.

## Setup and commands

See [robot description](description.md) and `assets/mjcf/SO101.xml`. Copy `configs/config.example.yaml` to `configs/config.yaml` and set the actual serial port. Configuration precedence is `--config`, `SO101_NODE_CONFIG`, then dataflow `config`/`config_path`.

```bash
uv sync
uv run robots-so101 --config configs/config.yaml
uv run robots-so101 list-devices --json
uv run robots-so101 activate-devices --json
```

`robots-so101` and `robots_so101` are CLI aliases for `robots_so101.main:main`; no subcommand and `run` both start the node. Port tools match `ttyACM*`. JSON uses `ports` with `name`, `address` and `status`; status records whether permissions are already `666`. Activation uses Linux `pkexec` to change permissions temporarily; membership in `dialout` is the persistent alternative. `robots-so101 test --config configs/config.yaml --duration 15` moves the gripper: use only with the correct device and a prepared robot.

## Packaging

```bash
uv sync --extra pyinstaller
./scripts/build_pyinstaller.sh
./dist/robots_so101 --help
```

The `uv_build` module is `robots_so101`. The PyInstaller spec is `scripts/robots_so101.spec`; add genuinely missing runtime imports to `hiddenimports`. Dependencies include Dora, Forge robot/message APIs, `pyserial` and MuJoCo. See [license](LICENSE) and repository notices.

## Versioned releases

**Mandatory tag-first release rule:** merge the version/lock change, create an annotated version tag on that reviewed commit, build that exact tag, validate the artifacts, then publish. Moving branch names, untagged commit IDs and replacement of published tags are not release inputs.

| Release line | Source | Node/package version | Runtime |
| --- | --- | --- | --- |
| Forge 1.x | `stable/1.x` | `1.x`, tag `v1.X.Y` | Forge 1.x / Dora 0.4.x |
| Forge 2.x | `main` | `2.x`, tag `v2.X.Y` | Forge 2.x / Dora 1.x |

**Legacy normalization:** nodes previously at 0.x on `stable/1.x` release as **1.0.0**, retaining Forge 1.x / Dora 0.4.x dependencies and runtime behavior. Existing 1.x package versions are retained. Node/package majors now follow the Forge compatibility line; a 0.x tag is rejected.

Run **Build versioned release** from `main` for either line. First merge the version/lock changes into the selected source branch, then create an unused **annotated** tag on that reviewed commit. The tag must match package metadata. Never move existing tags or reuse a published version. Dispatch uses the workflow's reviewed tooling with the exact tagged source, so legacy tags need not contain the new release helper.

```bash
# Replace with an existing, reviewed and unpublished tag matching source metadata.
TAG=v1.0.0
gh workflow run release.yml --ref main -f release_line=1.x -f tag="$TAG"
TAG=v2.X.Y
gh workflow run release.yml --ref main -f release_line=2.x -f tag="$TAG"
```

The workflow verifies tag type, source commit, branch ancestry, source version and locked Forge/Dora compatibility. It builds source archives and wheel/sdist where supported, checks Python distribution metadata and SHA-256, and creates or attaches to a **draft GitHub Release**. Published releases/assets are not overwritten; review all build results and acceptance evidence before publishing. This workflow does not publish node packages to PyPI.

Set `build_binaries=true` to call the existing Ubuntu 20.04 binary workflow; its existing architecture, CLI and archive checks remain in force. You can also run `build-ubuntu20.yml` directly with `ref`, `version` and `release_line`; `ref` must be an annotated tag, not a moving branch. Binary artifacts are attached only after all selected jobs succeed.

The source archive preserves the tagged tree; external SDKs/submodules, model downloads and site configuration remain separate prerequisites. A dependency or SDK build failure is not successful release acceptance. Build success does not establish single-node or Gateway-chain `dora run`, hardware, real-model, sustained-run or rollback acceptance. Every acceptance chain must include `adapter-forge-gateway` from the same compatibility line. Forge 2.x is a **breaking change**: align Forge libraries, node versions and Dora across the entire deployment; keep 1.x on its maintenance branch.

## Dependency maintenance

This maintenance line uses Forge Msgs 1.3.1 and Pillow 13 while retaining Forge 1.x and Dora 0.4.x compatibility. Dependency vulnerability auditing remains enabled.

The lockfile includes fsspec 2026.6.0 to address PYSEC-2026-4217. This dependency update does not change robot commands or device configuration.
