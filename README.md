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

- `sqlcipher_ref`: SQLCipher 的 tag、分支、commit，或 `latest`。

使用 GitHub CLI：

```powershell
gh workflow run build-windows.yml `
  --repo ShintoKosei/sqlcipher-windows-builds `
  -f sqlcipher_ref=latest
```

构建成功后会自动发布或更新对应 Release。

## 定时构建

仓库每周自动运行一次构建，默认解析 SQLCipher 最新 `v*` tag。构建成功后会发布或更新对应 Release。

## 构建方式

Actions 使用 MSYS2 UCRT64 环境、GCC 和 OpenSSL 构建 SQLCipher CLI。

主要编译选项：

- `SQLITE_HAS_CODEC`
- `SQLITE_EXTRA_INIT=sqlcipher_extra_init`
- `SQLITE_EXTRA_SHUTDOWN=sqlcipher_extra_shutdown`
- `--with-tempstore=yes`
- `LDFLAGS=-lcrypto`

构建后执行自检：

- `sqlcipher.exe -version`
- `PRAGMA cipher_version;`

如果自检失败，workflow 会直接失败，不会发布 Release。

## Windows 构建补丁

当前 MSYS2 UCRT64 / MinGW GCC 构建 SQLCipher `v4.17.0` 时，可能在 `splitmix64` 的 64 位立即数处触发汇编错误：

```text
value too large for field of 4 bytes
```

workflow 会在源码匹配时应用一个最小构建期补丁：把该 64 位常量改为从 `volatile uint64_t` 变量读取。这个补丁只影响 Windows 构建过程，不改变 SQLCipher 的数据库格式、加密参数或 CLI 使用方式。

## 许可证

本仓库的 workflow 和文档使用 MIT License。

SQLCipher 本身由上游项目维护。构建产物会附带 SQLCipher 上游许可证文件，使用或再分发前请确认符合对应许可证要求。
