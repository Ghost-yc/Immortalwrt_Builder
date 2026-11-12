# GL-MT5000 ImmortalWrt 固件构建

[![Build mt5000-immortalwrt](https://github.com/Ghost-yc/Immortalwrt_Builder/actions/workflows/Build%20mt5000-immortalwrt.yml/badge.svg)](https://github.com/Ghost-yc/Immortalwrt_Builder/actions/workflows/Build%20mt5000-immortalwrt.yml)

本仓库是一个精简的 GitHub Actions 项目，用于为 **GL.iNet GL-MT5000** 构建
[ImmortalWrt](https://github.com/immortalwrt/immortalwrt) 固件，并以 GitHub Release
的形式发布。

仓库中只保留构建路径必需的内容：一个 workflow、一个发布 action 和一个 OpenWrt `.config`。

> **仅供自用。** 本仓库为我自己的 GL-MT5000 设备维护，产出的固件按“现状”提供，
> 不提供任何担保与支持，也不保证稳定性及安全更新。刷机前请自行审阅配置并了解改动内容。

## 来源与致谢

本项目基于 [m0eak/Openwrt_Builder](https://github.com/m0eak/Openwrt_Builder) 修改而来。
原项目是一个多设备 ImmortalWrt 构建仓库，本项目在其基础上裁剪了其他设备的 workflow
与配置、自定义软件包脚本（`scripts-part1.sh` / `scripts-part2.sh`）以及
`default-settings-m0eak` 包，只保留 GL-MT5000 的构建路径。

感谢 m0eak 开源并维护原项目。本项目的构建流程、发布 action 与配置思路均源自其工作，
没有原项目就不会有这个仓库。原项目的构建流程又源自
[P3TERX/Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt)，在此一并致谢。

本项目为个人修改版本，其中的改动与问题由我自行负责，不代表原项目。

## 目标设备

| 项目 | 值 |
| --- | --- |
| 设备 | GL.iNet GL-MT5000 |
| 目标平台 | `mediatek` / `filogic` |
| Profile | `DEVICE_glinet_gl-mt5000` |
| 软件包架构 | `aarch64_cortex-a53` |
| Rootfs | squashfs + initramfs，7168 MB 分区 |
| 源码 | `https://github.com/Ghost-yc/immortalwrt.git` @ `mt-5000/master` |

## 仓库结构

```
.github/
  actions/openwrt-release-firmware/action.yml   # composite action：整理并发布固件
  workflows/Build mt5000-immortalwrt.yml        # 构建 + 缓存 + 发布流水线
config/mt5000-immortalwrt.config                # 该设备的完整 OpenWrt .config
.gitattributes                                  # 文本文件使用 LF 换行
.gitignore                                      # 忽略本地构建产物
```

## 构建流程

workflow 运行在 `ubuntu-24.04` 上，包含以下步骤：

1. **Checkout** — 检出本仓库。
2. **显示调试上下文** — 输出源码地址/分支与配置路径，并检查所需文件是否存在。
3. **释放磁盘空间** — 删除体积较大的预装 SDK 与 Docker 镜像。
4. **初始化环境** — 安装 `dos2unix` / `libfuse-dev`，执行 ImmortalWrt 的
   `init_build_environment.sh`，并设置时区。
5. **克隆源代码** — 将 ImmortalWrt fork 以单分支方式克隆到 `/workdir/openwrt`，
   并软链接到工作区。若设置了 `GH_PAT` secret，则使用它完成认证克隆（私有源码仓库必需）。
6. **更新源 / 安装源** — `./scripts/feeds update -a` 与 `./scripts/feeds install -a`。
7. **加载自定义配置** — 将 `config/mt5000-immortalwrt.config` 移至 `openwrt/.config`，
   执行 `make defconfig`，并输出生成的 `diffconfig`。
8. **检查并恢复缓存** — 使用 `actions/cache@v4` 缓存 `.ccache`、
   `staging_dir/host*` 和 `staging_dir/tool*`。缓存命中时 touch 所有
   `staging_dir/**/stamp` 文件；未命中时清理 Go module 缓存，并通过 `gh cache`
   删除相同前缀的旧缓存。
9. **编译固件** — 执行 `make -j$(nproc + 1)`，失败时回退到 `make -j1 V=s` 以获取
   完整日志。
10. **上传 bin 目录** — 将 `openwrt/bin` 上传为 workflow artifact。
11. **Release firmware** — 由本地 composite action 整理固件目录、上传、创建带 tag
    的 GitHub Release，并清理旧的 workflow run 与 release。

## 触发构建

可手动触发：

```bash
gh workflow run "Build mt5000-immortalwrt" \
  --repo Ghost-yc/Immortalwrt_Builder
```

或通过 `repository_dispatch`：

```bash
curl -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/Ghost-yc/Immortalwrt_Builder/dispatches \
  -d '{"event_type":"build"}'
```

workflow 中保留了每日定时任务，但目前处于注释状态。

## 凭据与权限

| 名称 | 是否必需 | 用途 |
| --- | --- | --- |
| `GH_PAT` | 可选 | 源码仓库为私有时的克隆认证 |
| `GITHUB_TOKEN` | 自动 | 删除过期缓存、创建 release、删除旧 workflow run |

workflow 申请 `contents: write`（release 与 tag）和 `actions: write`
（workflow run 与缓存）权限。

## 产物

- **Workflow artifact**：`OpenWrt_bin_<device>_<timestamp>`，内容为 `openwrt/bin`。
- **固件 artifact**：`OpenWrt_firmware<device><timestamp>`，内容为
  `openwrt/bin/targets/*/*` 下第一个目录中的文件。
- **GitHub Release**：
  - tag 格式：`GL-MT5000-immortalwrt-main-YYYYMMDD-HHMM`
  - release 正文回链源码仓库
  - 保留最新 **10** 个 release，更早的 release 及其 tag 会被删除
  - 超过 **5 天** 的 workflow run 会被删除，且至少保留 **5** 次

## 自定义

- **配置**：编辑 `config/mt5000-immortalwrt.config`。它是完整的生成式 `.config`，
  `make defconfig` 会保持其一致性。
- **源码项目**：修改 workflow `env` 中的 `REPO_URL_IMM` 与 `REPO_BRANCH`。
- **Release 命名**：调整发布步骤中的 `release-tag-prefix`（默认 `GL-MT5000`）与
  `source-tag`。
- **保留策略**：调整 composite action 的 `keep-latest`，或 workflow run 清理的
  `retain_days` / `keep_minimum_runs`。

构建不会注入任何自定义 rootfs overlay 或默认设置包，镜像完全由源码树与仓库中的
`.config` 决定，因此默认 IP 与默认服务均来自上游源码。

## 说明

- 本固件为个人构建，并非 ImmortalWrt 或 GL.iNet 官方发布，使用风险自负。
- 构建要求 `Ghost-yc/immortalwrt` 中存在 `mt-5000/master` 分支。
- 缓存失效依据 runner OS、workflow 名称、源码分支以及
  `openwrt/feeds.conf.default` 的哈希。仅修改 config 不会使工具链缓存失效。
- 构建产物包含 GPL-2.0 等自由软件组件。若公开分发固件，请按相应许可证要求确保
  对应源码可获取（构建源码地址见 workflow `env` 中的 `REPO_URL_IMM`）。

## 许可证

本项目以 MIT 许可证发布，完整文本见 [LICENSE](LICENSE)。

原项目 [m0eak/Openwrt_Builder](https://github.com/m0eak/Openwrt_Builder) 及其上游
[P3TERX/Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt) 同样采用 MIT
许可证，版权声明为 `Copyright (c) 2019-2020 P3TERX`，该声明已完整保留在
[LICENSE](LICENSE) 中。
