# houji 公开云端 CI 实测报告

日期：2026-11-12。范围：公开源码完整性、ARM64 Linux 与 Apple Silicon macOS 主机检查，以及锁定 Mesa/Turnip 源码组装。**未执行 XNU 正式编译、镜像打包或手机操作**；没有将 Apple/OEM 私有输入、设备标识或凭据提交或上传。

## 身份、仓库与运行证据

- GitHub CLI 已通过用户浏览器授权；已确认登录账号 `Ricardoo3256`，具有本人 Fork 的管理与 Git 推送权限。公开 Fork：[Ricardoo3256/UniXNU-houji](https://github.com/Ricardoo3256/UniXNU-houji)。本地 `origin` 指向该 Fork，`upstream` 指向 `lonzzi/UniXNU-houji`；只向 `origin` 推送 `codex/cloud-ci-probe`。
- 首次 push [Run 43237113438](https://github.com/Ricardoo3256/UniXNU-houji/actions/runs/43237113438) 在工作流解析阶段失败：job 级 `env` 使用了该位置不可用的 `${{ runner.temp }}`，**没有启动任何 runner 检查**。修正为相对日志目录后，[Run 43237396336](https://github.com/Ricardoo3256/UniXNU-houji/actions/runs/43237396336) 两作业成功；随后发现 `tee` 管道未传递左侧命令的失败码，不能仅凭该 run 的 job 结论作严格验收。
- 已为探针、依赖准备、源码检查、host suite 和 Turnip 组装启用 Bash `set -e -o pipefail`，并对 Mac 探针管道显式核验退出码。以修正后的 [最终有效 Run 43238476789](https://github.com/Ricardoo3256/UniXNU-houji/actions/runs/43238476789) 为本报告主要证据：push Commit `8ceccee9beb7fb8485737ceddef7bfa66dae98e5`，两个 job 均为 **success**，每个检查步骤和日志上传步骤亦为 **success**。
- 工作流仅由 `codex/cloud-ci-probe` 上指定源码/工作流路径的 push 或手动 `workflow_dispatch` 触发。纯文档与 `source-manifest.json` 更新不触发本工作流。Actions 使用只读仓库权限；日志工件保留 14 天。

## 实测结果

| 项目 | 结果 | 证据边界 |
|---|---|---|
| ARM64 Linux job | **PASS** | `ubuntu-24.04-arm` 实测 `uname -m = aarch64`；Ubuntu 24.04，内核 `7.0.0-1017-azure` |
| Apple Silicon job | **PASS** | `xcode-27` 实测 `uname -m = arm64`；不是仅依赖 runner 标签推断 |
| source check | **PASS / PASS** | 两平台均为 `2396 source hashes; public tree contains no image/build inputs` |
| portable host suite | **PASS** | Linux 日志明确 `PASS: portable suite completed`；不是 GPU 或实机资格测试 |
| Apple host suite | **PASS** | Mac `--suite all` 日志明确 `PASS: all suite completed`；不是 GPU 或实机资格测试 |
| Turnip assembly | **PASS / PASS** | 两平台均校验锁定 Mesa HEAD `85fda3b36405bcaea8bffc64aa4f2651dc90770d`，原版 `prepare-sources.py graphics/turnip` 输出 `Prepared` |
| KDK | **MISSING（探测范围内）** | Mac `/Library/Developer/KDKs` 不存在；未搜索/下载其它私有位置 |
| CTF 工具 | **MISSING（探测范围内）** | `ctfconvert`、`ctfmerge`、`ctfdump` 不在 PATH；所选 Xcode 应用内的有界查找无结果 |

Linux 实测：Python `3.12.16`、Ubuntu Clang `18.1.3`、Git `2.56.0`、Rust `1.98.0`。Mac 实测：macOS `27.0.1`（build `26A434`，Darwin `27.0.0`），所选 Xcode `27.1`（build `27A9275`，`/Applications/Xcode_27.1.app/Contents/Developer`），macOS SDK `27.0`、iPhoneOS SDK `27.1`，Apple Clang `21.0.0`（`clang-2100.3.34.2`）、Rust `1.98.0`。Mac 原始探针在 `actions/setup-python` 之前执行，显示系统 Python `3.15.2`；工作流随后选用 Python `3.12`，但探针日志没有打印切换后的完整补丁版本，故不声称该版本已被日志直接量到。

公开日志工件（GitHub Actions artifact 的压缩包摘要，不等同其中各文件的哈希）：

| 工件 | 大小 | GitHub 提供的 digest |
|---|---:|---|
| `houji-arm64-linux-probe-43238476789` | 6922 B | `sha256:b5535cf1a6f67bb120ac0ac12b498e6ca399aafe70ef62f3bcc477f42d06ce22` |
| `houji-xcode-27-probe-43238476789` | 7403 B | `sha256:af4e0e700e4bcf85a433876be1615f21054866a76fd994c0bb147d4a5763229d` |

日志本地副本在被 Git 忽略的 `out/cloud-ci-run-43238476789/`；以 GitHub 上的 run 和工件为远程证据。两份工件分别包含环境、依赖、源码校验、host suite 与 Turnip 组装完整日志。工作流的 `continue-on-error` 只允许后续上传日志和汇总继续执行，汇总步骤逐一检查实际 outcome 并决定 job 终态；修正后的 `pipefail` 确保检查命令失败不会被 `tee` 掩盖。

## 失败分类与局限

最终有效 run 没有检查失败；以下分类仅用于今后失败时按原始日志诊断，**不能把未发生的失败写成已发生**。

| 类别 | 诊断边界 |
|---|---|
| `MISSING_PACKAGE` | 公开 Python/Clang/Rust/Git 等工具缺失或安装失败 |
| `APPLE_API_FAILURE` | Apple host suite 中 Foundation/Metal/IOSurface/Mach 等接口检查失败 |
| `ARCHITECTURE_FAILURE` | `uname -m` 不为 Linux `aarch64` 或 Mac `arm64` |
| `SOURCE_FAILURE` | source manifest、固定上游 SHA 或补丁应用失败，须继续按日志定位 |
| `TOOLCHAIN_VERSION_FAILURE` | 实际工具链版本不满足明确的项目要求 |
| `REPOSITORY_ASSUMPTION_FAILURE` | 固定输入下暴露归档流、路径或权限等跨平台假设问题 |

Windows 本地曾在 Turnip 组装时出现 `StreamError: seeking backwards is not allowed`。在同一锁定 Mesa SHA 下，Linux 和 Mac 均用原版脚本组装成功，说明**本轮两个云端平台没有复现该错误**；Windows 根因仍未定位，不能由此声称任意 Windows 环境都失败或云端所有情况都成功。

GitHub Actions 对 `actions/checkout@v4`、`actions/setup-python@v5`、`actions/upload-artifact@v4` 给出 Node.js 20 废弃警告，当前被 runner 强制使用 Node.js 24；不影响本次 PASS，但需在后续维护中处理。Mac 的 KDK/CTF 探测范围有限；“MISSING”表示当前公共 runner 探针未找到，不证明任何合法私有构建环境也缺失。当前 macOS/Xcode/SDK 版本也不自动满足项目历史 XNU 8796.141.3 构建要求。

**状态：CLOUD_MAC_PARTIALLY_USABLE。** Apple Silicon 公共 runner 已实测可用于公开源码和 host 检查；缺少本项目正式 XNU 构建所需且合法取得/校验的 KDK、CTF 与匹配 OEM 输入，且本次没有尝试 XNU 编译。后续安排见 `docs/cloud-ci-next-steps.md`。
