# Installing and Managing Python | uv

## 目录

- [Getting started](#getting-started)
- [Installing a specific version](#installing-a-specific-version)
- [Reinstalling Python](#reinstalling-python)
- [Viewing Python installations](#viewing-python-installations)
- [Automatic Python downloads](#automatic-python-downloads)
- [Using existing Python versions](#using-existing-python-versions)
- [Upgrading Python versions](#upgrading-python-versions)
- [Next steps](#next-steps)

---

## Installing Python

如果系统中已存在 Python，uv 会自动识别并使用它。但 uv 也提供 Python 安装和版本管理功能。该工具"会在需要时自动安装缺失的 Python 版本——无需预先安装 Python 即可开始使用。"

### Getting started

安装最新 Python 版本：

```bash
$ uv python install
```

> **注意：** Python 没有官方可分发的二进制文件，因此 uv 依赖 Astral 的 `python-build-standalone` 项目提供的发行版。

安装完成后，uv 命令会自动使用该版本，安装的版本也会添加到 `PATH` 中：

```bash
$ python3.13
```

默认情况下，仅安装带版本号的可执行文件。若要同时添加 `python` 和 `python3` 可执行文件，可使用实验性标志：

```bash
$ uv python install --default
```

### Installing a specific version

安装特定版本：

```bash
$ uv python install 3.12
```

同时安装多个版本：

```bash
$ uv python install 3.11 3.12
```

安装其他实现（如 PyPy）：

```bash
$ uv python install pypy@3.12
```

### Reinstalling Python

使用 `--reinstall` 标志重新安装 uv 管理的版本：

```bash
$ uv python install --reinstall
```

这会恢复所有之前安装的版本。由于发行版持续改进，即使版本号不变，重新安装也可以解决一些问题。

### Viewing Python installations

列出可用版本和已安装版本：

```bash
$ uv python list
```

### Automatic Python downloads

使用 uv 无需预先安装 Python。该工具会在需要时自动下载所需版本。例如：

```bash
$ uvx python@3.12 -c "print('hello world')"
```

即使没有指定版本，uv 也会按需下载最新版本：

```bash
$ uv venv
```

### Using existing Python versions

uv 无需任何配置即可检测并使用系统中已有的 Python 安装。若要强制使用系统 Python，可提供 `--no-managed-python` 标志。

### Upgrading Python versions

> **重要：** 补丁版本升级功能处于预览阶段，可能会有变动。

升级到最新补丁版本：

```bash
$ uv python upgrade 3.12
```

升级所有已管理的版本：

```bash
$ uv python upgrade
```

### Next steps

查阅 [Python 版本概念](https://docs.astral.sh/uv/concepts/python-versions/) 页面和[命令参考](https://docs.astral.sh/uv/reference/cli/#uv-python)，或继续阅读[运行脚本](https://docs.astral.sh/uv/guides/scripts/)。

---

*来源：https://docs.astral.sh/uv/guides/install-python/*
