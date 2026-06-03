# CloudRedirect 中文自定义版

这是基于 [Selectively11/CloudRedirect](https://github.com/Selectively11/CloudRedirect) 的自定义构建。

上游项目提供 CloudRedirect 的核心实现，包括 Steam 集成、原生 DLL hook、WPF 管理界面和云存档重定向逻辑。本仓库在上游 `v2.1.2` 基础上增加了简体中文界面、WebDAV 支持、WebDAV 连接测试、设置页 DLL 来源识别，以及对自定义 WebDAV DLL 的官方自动更新保护。

## 重要提醒

CloudRedirect 仍然是实验性工具。它会介入 Steam Cloud 存档流程，使用前请备份重要存档。错误配置、云端文件被手动删除、Steam 客户端版本变化或 DLL hook 点失配，都可能导致同步失败、冲突或存档丢失。

如果只是想关闭某个游戏的 Steam Cloud 错误提示，请优先在 Steam 游戏属性里关闭 Steam Cloud，不要把这个工具当成通用修复器。

## 这个精简仓库包含什么

本仓库只保留 Windows 自定义构建需要的核心源码和必要资源：

- `src/common/`：跨平台核心逻辑、配置、同步、序列化和 RPC 处理。
- `src/providers/`：Google Drive、OneDrive、本地目录和 WebDAV provider。
- `src/platform/win/`：Windows DLL 注入、Steam hook、HTTP、日志和平台实现。
- `ui/`：Windows WPF 管理界面。
- `ui/Resources/payloads/`：构建和运行补丁流程需要嵌入的 payload 资源。
- `CMakeLists.txt`、`Version.props`：Windows Release 构建入口和版本信息。
- `UPSTREAM.md`：上游来源与自定义改动说明。

以下内容已从源码树移除：Linux UI、Flatpak 打包文件、旧的独立 .NET CLI 工程、补丁归档、本地工具链缓存、NuGet 缓存、构建输出和临时文件。Release 成品请从 GitHub Releases 下载。

## 功能差异

相对上游 `v2.1.2`，本版本主要增加：

- 简体中文界面和中文语言选项。
- WebDAV 云端存储 provider。
- WebDAV 连接测试按钮，实际执行 `PROPFIND`。
- WebDAV Basic Auth 和 Digest Auth 支持。
- 设置页显示当前 DLL 来源：官方版、自定义 WebDAV 版、哈希不匹配或未安装。
- 检测到自定义 WebDAV DLL 时，避免官方 DLL 自动更新覆盖自定义 DLL。

## 支持的云端

- Google Drive
- OneDrive
- WebDAV
- 本地目录 / 映射盘

## 构建环境

建议在 Windows x64 上构建：

- Visual Studio 2022 或 Build Tools，安装 C++ 桌面开发组件。
- CMake 3.20 或更高版本。
- .NET 8 SDK，包含 Windows Desktop / WPF 构建支持。
- 能访问 NuGet，用于还原 `WPF-UI` 和 `System.Security.Cryptography.ProtectedData`。

## 构建 Release EXE

在仓库根目录执行：

```powershell
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

构建完成后会生成：

- 原生 DLL：`build/Release/cloud_redirect.dll`
- 原生 CLI：`build/Release/cloud_redirect_cli.exe`
- 单文件 WPF 程序：`ui/bin/publish/CloudRedirect.exe`

`ui/CloudRedirect.csproj` 会在发布时自动从 `build/Release/` 复制并嵌入 `cloud_redirect.dll` 和 `cloud_redirect_cli.exe`。如果看到缺少 DLL 或 CLI 的警告，请先确认 CMake 的 Release 配置已经成功构建。

这个 EXE 是 framework-dependent single-file 发布，目标机器需要安装 .NET 8 Windows Desktop Runtime。

## 关于源码构建和 Release 哈希

Release 页面提供的 EXE、DLL、CLI 是已经构建好的成品。源码重新构建时，编译器版本、SDK 版本、构建路径、Git 提交号和时间相关元数据都可能让最终 SHA256 与 Release 资产不同。只要从同一份源码成功构建，功能应与该 Release 对应。

## 使用方式

1. 从本仓库 Releases 下载 `CloudRedirect.exe`。
2. 运行程序，进入 Setup，执行补丁流程。
3. 在 Cloud Provider 页面选择 Google Drive、OneDrive、WebDAV 或本地目录。
4. 如果选择 WebDAV，填写 URL、用户名、密码后先测试连接。
5. 重启 Steam 后检查 `cloud_redirect.log`，确认没有版本、RVA、prologue 或 vtable mismatch。

WebDAV URL 建议使用 HTTPS，并直接指向你希望 CloudRedirect 使用的目录。例如 Nextcloud 通常类似：

```text
https://example.com/remote.php/dav/files/username/CloudRedirect
```

## 上游更新

如果上游发布新版本，优先保留上游对 Steam hook、RVA、signature、Steam 客户端版本支持的修改，再重新移植本仓库的中文界面和 WebDAV provider。相关来源说明见 `UPSTREAM.md`。
