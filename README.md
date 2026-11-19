# harmonyos-cli

HarmonyOS 命令行构建工具链镜像,供 GitHub Actions(及任何 CI)构建 HarmonyOS NEXT 应用使用。

华为官方只通过 [developer.huawei.com](https://developer.huawei.com/consumer/cn/download/command-line-tools-for-hmos) 登录后分发工具链,CI 无法自动化获取;本仓库把 DevEco Studio 内置的工具链裁剪打包后以 GitHub Release 分卷发布,多项目共用。

## 内容

以 `toolchain-<版本>` 形式发布,版本对应 DevEco Studio 的 SDK 版本(`sdk-pkg.json` 的 `version` + 打包序号):

| 目录 | 内容 | 版本(26.0.0.105-1 示例) |
|---|---|---|
| `tools/hvigor` | hvigor + hvigor-ohos-plugin | 6.26.8 |
| `tools/ohpm` | OpenHarmony 包管理器 | 26.0.0.630 |
| `sdk/default/openharmony` | ets / js / native / toolchains | API 26 |
| `sdk/default/hms` | ets / js / toolchains | API 26 |

**相对完整 DevEco 安装裁剪了**(~7.6G → ~3.4G,gzip 后 ~1.5-1.9G):

- `tools/node`(939M)— CI 上用 `actions/setup-node` 装 Node 24
- `openharmony/previewer`、`hms/previewer`(~820M)— `assembleHap`/`assembleApp` 不需要预览器
- `hms/native`(1.8G)— HMS NDK;应用 CMake 链接的 NDK 库(ace_napi / avplayer / native_media / hilog 等)全部位于 `openharmony/native`

注意:保留的 `native/llvm` 与 `ets/build-tools` 是 **arm64 macOS** 二进制,只能在 Apple Silicon 的 runner(macos-14/15/26)上运行。

## 在 workflow 中使用

```yaml
env:
  TOOLCHAIN_MIRROR_REPO: monster-echo/harmonyos-cli
  TOOLCHAIN_VERSION: '26.0.0.105-1'

steps:
  - uses: actions/setup-node@v4
    with: { node-version: '24' }
  - uses: actions/cache@v4
    with:
      path: ~/harmony-cli
      key: harmony-cli-${{ env.TOOLCHAIN_VERSION }}
  - name: Download & unpack toolchain
    if: steps.toolchain-cache.outputs.cache-hit != 'true'
    run: |
      gh release download "toolchain-${TOOLCHAIN_VERSION}" \
        --repo "$TOOLCHAIN_MIRROR_REPO" \
        --pattern 'harmony-cli-mac-*.tar.gz.part-*' \
        --dir "$RUNNER_TEMP/tc"
      cat "$RUNNER_TEMP"/tc/harmony-cli-mac-*.tar.gz.part-* > "$RUNNER_TEMP/tc/toolchain.tar.gz"
      tar -xzf "$RUNNER_TEMP/tc/toolchain.tar.gz" -C "$HOME"
```

构建时设置 `DEVECO_SDK_HOME="$HOME/harmony-cli/sdk"`,直接 `node` 调用 `$HOME/harmony-cli/tools/hvigor/hvigor/bin/hvigor.js`,参考 [prismplayer 的 workflow](https://github.com/monster-echo/prismplayer/blob/main/.github/workflows/harmony-release-github-runner.yml)。

## 重新打包 / 更新版本

升级 DevEco Studio 后,在装有 DevEco 的 macOS 上执行 [`scripts/pack-toolchain.sh`](scripts/pack-toolchain.sh)(递增 `TOOLCHAIN_VERSION` 尾部序号),`UPLOAD=1` 直接上传新 release;使用方只需改 workflow 里的 `TOOLCHAIN_VERSION`。

## 声明

本仓库仅为开发工具链的搬运与裁剪,便于 CI 自动化,版权归华为所有,使用须遵循华为 HarmonyOS SDK 许可协议。不包含任何应用源码、签名材料或凭据。
