# houji 公开云端构建探针报告

日期：2026-11-12。范围仅为公开源码、ARM64 Linux 和 Apple Silicon macOS 的环境/主机检查，以及 Mesa/Turnip 源码组装；**不执行 XNU 正式构建**，不使用手机、Apple/OEM 私有输入或设备标识。

## 本轮结果与证据边界

本地已创建 `.github/workflows/cloud-build-probe.yml`，目标主仓库基线 Commit 为 `99beb109d66d24d3a1780901cb8958ec3fb24b06`。Windows 上通过了 YAML 解析。2026-11-12 经 GitHub CLI 网页授权，身份确认为 `Ricardoo3256`；个人公开 Fork 为 [Ricardoo3256/UniXNU-houji](https://github.com/Ricardoo3256/UniXNU-houji)，Actions API 显示已启用，针对该 Fork 的 `git push --dry-run` 成功。**截至本次修订仍未执行首次远程 push 或新 workflow**，等待用户确认推送目标和文件清单。因此没有本轮 `xcode-27` runner 的真实探针结果，也没有本轮 Turnip 跨平台组装结果。以下“未运行”不能写作 PASS 或 FAIL。

可独立核查的历史基线：[主项目 GitHub Actions run 41324111854](https://github.com/lonzzi/UniXNU-houji/actions/runs/41324111854) 在**相同主仓库 Commit** 上于 2026-11-01 完成；其 `ubuntu-24.04-arm` 与 `macos-15` 两个 job 均成功，公开源码校验和对应 host suite 步骤均成功。这不是本次 `xcode-27` 探针；历史 workflow 没有记录本次要求的完整环境版本、KDK/CTF 状态，也没有执行 Turnip 组装。

| 项目 | 本轮新 workflow | 历史同 Commit 基线 | 说明 |
|---|---|---|---|
| ARM64 Linux | NOT RUN | PASS（`ubuntu-24.04-arm` job） | 新探针未推送 |
| Apple Silicon / `xcode-27` | NOT RUN | 不适用（历史为 `macos-15`） | 实际架构、macOS、Xcode、SDK 版本均未知 |
| source check | NOT RUN | PASS（两 job） | 新增 workflow/报告后需重建 source manifest 再跑 |
| portable host suite | NOT RUN | PASS（历史 Linux job） | 本轮未重跑 |
| Apple host suite | NOT RUN | PASS（历史 `macos-15` job） | 不能外推至 `xcode-27` |
| Turnip assembly：ARM64 Linux | NOT RUN | NOT RUN | 需本轮锁定 Mesa Commit 组装 |
| Turnip assembly：Apple Silicon | NOT RUN | NOT RUN | 同上 |
| KDK（`xcode-27` runner） | UNKNOWN / 未探测 | UNKNOWN | 工作流只检查 `/Library/Developer/KDKs`，绝不下载 |
| CTF 三工具（`xcode-27` runner） | UNKNOWN / 未探测 | UNKNOWN | 限定 PATH 与所选 Xcode 应用内查找 |

状态：**CLOUD_MAC_PARTIALLY_USABLE（暂定，仅表示公开 CI 方案已准备、历史 macOS 15 主机检查通过；`xcode-27` 可用性未实测）**。不能由 GitHub runner 镜像清单替代实际输出。GitHub 官方当前列出 `xcode-27` 为 arm64 标签，但运行时版本、KDK/CTF 是否存在必须以本 workflow 的日志为准：[runner-images 标签表](https://github.com/actions/runner-images/blob/main/README.md)、[GitHub-hosted runners 文档](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)。

## 工作流设计与取证

- 两个 job 分别锁定 `ubuntu-24.04-arm` 与 `xcode-27`；只读仓库权限，`checkout` 不保存凭据，不传递秘密。Linux 探测 `uname -a/-m`、Python、Clang、Git；Mac 探测 `uname`、`sw_vers`、Xcode、所选 Developer 目录、macOS/iPhoneOS SDK、Clang、Python、KDK 目录及 CTF 工具。Mac 上的 `find` 仅限制在所选 `/Applications/Xcode*.app` 内。
- 使用 `actions/setup-python` 准备 Python 3.12；host suites 使用项目既有 Rust 1.98.0 要求。Linux 缺少 Clang 时才通过 apt 安装公开包；Rustup 缺失时使用官方安装入口。没有安装 KDK/SDK 私有文件、OEM 固件或 Apple 账号输入。
- 两个 job 分别执行 `tools/check-source-tree.py`、相应的 `run-host-checks.py --suite portable/all`。从 `sources.lock.json` 读取 Mesa URL 和固定 SHA，先核验 checkout HEAD，再使用未改动的 `tools/prepare-sources.py graphics/turnip`；输出写入 runner 临时目录。
- 每一步的 stdout/stderr 用 `tee` 留存到 runner 临时目录；`actions/upload-artifact@v4` 即使检查失败也上传日志（保留 14 天）。末尾汇总各步骤真实 outcome 并将失败反映为 job 失败，不能用 `continue-on-error` 冒充成功。
- 工作流没有 `adb`、`fastboot`、XNU 构建、刷机命令或私有输入路径。GitHub Actions 构建日志/工件仅含公开内容。

## 失败分类规则（待实测）

| 类别 | 判定依据 |
|---|---|
| MISSING_PACKAGE | 缺少 Python、Clang、Rustup/Rust 1.98、Git 等公开工具且准备步骤未能补齐 |
| APPLE_API_FAILURE | `--suite all` 的 Foundation/Metal/IOSurface/Mach 等 Apple 接口检查在架构与工具链合格后报错 |
| ARCHITECTURE_FAILURE | runner 实测 `uname -m` 不为 Linux `aarch64` 或 Mac `arm64` |
| SOURCE_FAILURE | source manifest 不匹配、锁定 Mesa Commit 不可获取、补丁校验/应用失败，须以日志细分 |
| TOOLCHAIN_VERSION_FAILURE | Xcode/SDK/Clang/Python/Rust 版本或可用性不符合项目已明确的要求 |
| REPOSITORY_ASSUMPTION_FAILURE | `prepare-sources.py` 的 tar 流/文件模式假设等跨平台缺陷，在固定输入下复现 |

Windows 既有 `graphics/turnip` 组装失败为 `StreamError: seeking backwards is not allowed`，发生于 Python tar 流解包、Patch 前。**目前只能记为 Windows 已观察失败；不能预先判定是 Windows 独有或跨平台缺陷。**本轮两个 CI job 的日志将给出对照；不修改 `prepare-sources.py`、不跳过 Patch/锁定 SHA。

## 后续获取真实结果的最小操作

1. 首次远程 push 前，向用户展示目标公开 Fork 和确切文件清单，取得明确确认。本地分支为 `codex/cloud-ci-probe`，`origin` 为个人 Fork，`upstream` 为原作者；不得向 `upstream` 推送。
2. 经确认后向 `origin` 推送包含 workflow 和更新后 `source-manifest.json` 的分支，等候两 job 完成；核对 run URL、Commit SHA、runner 的实际架构/版本、每步 outcome 和上传日志。
3. 若失败，按上表逐项分类并保存完整 traceback/编译输出；不删减检查或改测试以制造绿灯。以实测更新本报告的 NOT RUN / UNKNOWN 项和最终状态。

本报告不证明 XNU 可编译、可启动、可刷入或设备兼容。
