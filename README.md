# sqlcipher-windows-builds

自动构建 Windows x64 UCRT 版 SQLCipher 命令行工具。

本仓库只维护构建流程：从 SQLCipher 上游源码解析指定版本，使用 GitHub Actions 在 Windows runner 上编译，打包 `sqlcipher.exe` 和运行所需 DLL，并在构建成功后发布 Release。

## 构建产物

Release 压缩包命名格式：

```text
sqlcipher-<SQLCipher版本>-windows-x64-ucrt.zip
```

压缩包包含：

- `sqlcipher.exe`
- UCRT64 / OpenSSL 运行时 DLL
- `BUILD_INFO.txt`
- SQLCipher 上游许可证文件

## 手动构建

在 GitHub 页面进入 `Actions`，选择 `Build Windows SQLCipher`，点击 `Run workflow`。

参数：

- `sqlcipher_ref`: SQLCipher 的 tag、分支、commit，或 `latest`（仅解析最新稳定版 `vX.Y.Z`）。

使用 GitHub CLI：

```powershell
gh workflow run build-windows.yml `
  --repo ShintoKosei/sqlcipher-windows-builds `
  -f sqlcipher_ref=latest
```

手动触发始终执行构建，可用于强制重建已经发布过的版本。

构建成功后会自动发布或更新对应 Release。

## 定时构建

仓库每周一 03:17 UTC 检查上游最新稳定版 tag。`v5.0.0-beta.1` 之类的预发布 tag 会被忽略。

若该版本对应的 Release 已存在，定时构建会直接跳过，避免每周重复编译并覆盖同一个 Release；上游发布新的稳定版时才会构建并发布新 Release。

修改 `.github/workflows/build-windows.yml` 并推送到 `main` 时，会自动运行一次验证构建。

## 构建方式

Actions 使用 MSYS2 UCRT64 环境、GCC 和 OpenSSL 构建 SQLCipher CLI。

workflow 会根据源码自动选择编译参数（不依赖 tag 名称，因此分支名和任意 commit 也能正确构建）：

| 上游版本 | 主要编译选项 |
| --- | --- |
| SQLCipher 4.x | `-DSQLITE_HAS_CODEC`、`-DSQLITE_EXTRA_INIT=sqlcipher_extra_init`、`-DSQLITE_EXTRA_SHUTDOWN=sqlcipher_extra_shutdown` |
| SQLCipher 5.x | `-DSQLITE_USE_URI=1`、`-DSQLITE_DIRECT_OVERFLOW_READ=0`、`-DSQLITE_EXTRA_INIT=sqlcipher_extra_init`、`-DSQLITE_EXTRA_SHUTDOWN=sqlcipher_extra_shutdown` |

两者都使用 `--with-tempstore=yes` 和 `LDFLAGS=-lcrypto`。

> SQLCipher 5.x 在 `src/sqlcipher.c` 中用 `#error` 强制要求 `SQLITE_USE_URI` 和 `SQLITE_DIRECT_OVERFLOW_READ=0`，并且不再使用 `SQLITE_HAS_CODEC`。缺少这些宏会在 `make` 阶段直接失败（退出码 2）。

构建后执行自检：

- `sqlcipher.exe -version`
- `PRAGMA cipher_version;`

如果自检失败，workflow 会直接失败，不会发布 Release。

## Windows 构建补丁

当前 MSYS2 UCRT64 / MinGW GCC 构建 SQLCipher 时，可能在 `splitmix64` 的 64 位立即数处触发汇编错误：

```text
value too large for field of 4 bytes
```

workflow 会在源码匹配时应用一个最小构建期补丁：把该 64 位常量改为从 `volatile uint64_t` 变量读取。这个补丁只影响 Windows 构建过程，不改变 SQLCipher 的数据库格式、加密参数或 CLI 使用方式。

## 维护记录

- 2026-11：修复定时构建。2026-09 上游出现 `v5.0.0-beta` / `v5.0.0-beta.1` 预发布 tag，旧逻辑用 `sort -V | tail -n 1` 会把 beta 误判为最新版；而该版本要求新的编译宏，导致 9/21 起连续多次定时构建在 `Configure and build` 步骤失败，期间没有发布任何新 Release。现已改为只解析稳定版 tag、按源码自动选择编译参数，并对已发布版本跳过重复构建。

## 许可证

本仓库的 workflow 和文档使用 MIT License。

SQLCipher 本身由上游项目维护。构建产物会附带 SQLCipher 上游许可证文件，使用或再分发前请确认符合对应许可证要求。
