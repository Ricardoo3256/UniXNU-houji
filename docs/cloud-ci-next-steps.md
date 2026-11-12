# 公开云端 CI 下一步

依据：2026-11-12 的 [最终有效探针 Run 43238476789](https://github.com/Ricardoo3256/UniXNU-houji/actions/runs/43238476789)，Commit `8ceccee9beb7fb8485737ceddef7bfa66dae98e5`。工作流已启用管道失败传递；Linux portable、Mac all host suite、两平台锁定 Mesa/Turnip 源码组装均通过。这不是 XNU 编译或实机验证。

1. **保留证据。** GitHub 上两份公开日志工件各保留 14 天；本机副本在 `out/cloud-ci-run-43238476789/`，属于 Git 忽略目录。若将来重新跑，记录新的 run URL、Commit、runner 实际版本和工件摘要，不用旧结论替代新 runner。
2. **把公开源码组装作为下一道门。** 可在新的、只处理公开源码的 CI 中锁定 XNU `1b191cb58250d0705d8a51287127505aa4bc0789`，在 Apple Silicon runner 重新运行 `tools/prepare-sources.py kernel`、`tools/check-trial-bindings.py`，并核验 `config/MASTER.arm64.sweet`。这仍是源码组装，不运行 `tools/build-kernel.sh`，不把 Windows `.deps/kernel` 复制到 Mac。
3. **将正式 XNU 编译单独立项。** 本次 Mac runner 是 arm64 / macOS 27.0.1 / Xcode 27.1，macOS SDK 27.0、iPhoneOS SDK 27.1；项目文档记录过 Xcode 27.2 与 Darwin 22。版本差异对 XNU 8796.141.3 的影响尚未验证。runner 上指定目录没有 KDK，PATH 与所选 Xcode 应用内也没有三个 CTF 工具。正式编译仍需经合法渠道确认 KDK、从可信公开源码构建/验证 CTF、在用户私有环境取得与 `kernel/firmware-inputs.json` 完全匹配的 GMU/SQE/AQE。不得把 KDK、OEM 固件或私有头上传到公开 CI。
4. **Windows Turnip 差异单独诊断。** 两个云端平台用相同锁定 Mesa Commit 和原版 `prepare-sources.py` 均组装成功；Windows 先前的 `StreamError: seeking backwards is not allowed` 未在这两平台复现。若要支持 Windows 本地组装，先用同一 Python 版本、文件系统和 Git 归档重现并定位，不通过跳过 Patch 或改锁文件制造通过。
5. **维护工作流。** 本轮 Linux job 出现 Node.js 20 废弃提示，涉及 `checkout@v4`、`setup-python@v5`、`upload-artifact@v4`；下一次维护时核对官方兼容版本后升级并重新跑完整探针。当前成功结果不因该警告失效。

进入 XNU 正式编译前的验收门槛：公开源码固定 SHA 与 Mac 上重组装可复现；KDK/CTF/OEM 三类输入均有合法来源和本地校验；目标配置、工具链版本、产物日志可追溯。**本任务到此不执行内核正式构建、镜像打包或任何手机操作。**
