---
title: NixOS 下使用 uv + mkShell 搭建 Python 开发环境
category: linux
tags:
  - linux
  - nixos
description: NixOS 下使用 uv + mkShell 搭建 Python 开发环境
published: 2026-08-04
draft: false
---

# NixOS 下使用 uv + mkShell 搭建 Python 开发环境

## 背景

在 NixOS 上使用 `uv` 管理 Python 项目时，pip 安装的包（如 `numpy`、`opencv-python`）是预编译的通用 Linux 二进制文件，它们动态链接了系统库（`libz.so.1`、`libxcb.so.1` 等）。NixOS 的文件系统布局与标准 Linux 不同，这些库不在常规路径下，导致运行时找不到。

## 方案一：nix-ld + 扩展库列表

[nix-ld](https://github.com/Mic92/nix-ld) 为 NixOS 提供了通用 Linux 动态链接库的加载支持。默认只包含基础库（`glibc`、`zlib` 等），但可以通过 `libraries` 选项添加任意库：

```nix
{ pkgs, ... }: {
  programs.nix-ld = {
    enable = true;
    libraries = with pkgs; [
      # 默认基础库
      zlib
      stdenv.cc.cc

      # 按需添加
      glib
      gtk3
      libxcb
      libx11
      libxext
      libxrender
      libxi
      libice
      libsm
    ];
  };
}
```

添加后 `sudo nixos-rebuild switch` 即可全局生效，所有通过 nix-ld 加载的二进制都能找到这些库。

### 重要限制

nix-ld **只对解释器路径为 `/lib64/ld-linux-x86-64.so.2` 的二进制生效**。NixOS 系统自带的 Python（通过 nixpkgs 安装）使用 Nix store 里的 glibc 作为动态链接器，不会走 nix-ld，因此 `NIX_LD_LIBRARY_PATH` 对它无效。

验证方法：
```bash
# 查看 Python 解释器路径
readelf -l .venv/bin/python3 | grep "INTERP"
# 输出类似：/nix/store/...-glibc-2.42-67/lib/ld-linux-x86-64.so.2
# 不是 /lib64/ld-linux-x86-64.so.2，所以 nix-ld 不会拦截

# 查看 nix-ld 位置
ls -la /lib64/ld-linux-x86-64.so.2
# 输出：/lib64/ld-linux-x86-64.so.2 -> /nix/store/...-nix-ld-2.0.6/libexec/nix-ld
```

### 解决方案

nix-ld 对 NixOS 系统 Python 无效。需要手动安装 uv 自带的独立 Python（`python-build-standalone`），它编译时写死 `/lib64/ld-linux-x86-64.so.2`，会被 nix-ld 拦截，自动加载 `programs.nix-ld.libraries` 里的所有库。

```bash
# 当前 venv 用的是系统 Python
readlink -f .venv/bin/python3
# 输出：/nix/store/...-python3-3.13.14/bin/python3.13

# 改用 uv 自带的 Python
rm -rf .venv
uv python install 3.13.14
uv venv --python 3.13.14
uv sync
```

**关键**：系统上不要安装 NixOS 的 Python（或确保 uv 优先使用下载的版本），否则 uv 默认会复用系统 Python。

验证方案一是否生效：
```bash
uv python install 3.13
uv run --python 3.13 python -c "import ssl, zlib; print(ssl.OPENSSL_VERSION)"
```

### 注意

- `glib` 默认输出是 `bin`（不含 `.so`），但 nix-ld 会自动使用 `glib.out` 输出，无需手动指定
- 库列表会全局生效，可能引入不必要的依赖

### 配合 vim LSP（方案一适用）

使用 uv 自带 Python 时，LSP（如 pyright）需要知道 venv 路径。推荐 `nvim-venv-detector`，它会自动扫描项目目录（`.venv`、`venv` 等）并激活对应环境，pyright 自动使用该解释器。

在 lazy.nvim 中添加：

```lua
{
  "tnfru/nvim-venv-detector",
  event = "VimEnter",
  config = function()
    require("venv_detector").setup()
  end,
}
```

## 方案二：mkShell（项目级隔离）

利用 Nix 的 `mkShell` 为**每个项目**提供独立的库环境，按需注入，可复现、可版本管理。

### 目录结构

```
project/
├── pyproject.toml    # uv 项目配置
├── shell.nix         # Nix 开发环境
└── .venv/            # uv 创建的虚拟环境
```

### shell.nix 示例

```nix
{ pkgs ? import <nixpkgs> { } }:

pkgs.mkShell {
  buildInputs = with pkgs; [
    zlib
    stdenv.cc.cc.lib
    libxcb
    libx11
    libxext
    libxrender
    libxrandr
    libxfixes
    libxi
    libsm
    libice
    glib.out
    gtk3
  ];

  shellHook = ''
    export LD_LIBRARY_PATH="\
${pkgs.zlib}/lib:\
${pkgs.stdenv.cc.cc.lib}/lib:\
${pkgs.libxcb}/lib:\
${pkgs.libx11}/lib:\
${pkgs.libxext}/lib:\
${pkgs.libxrender}/lib:\
${pkgs.libxrandr}/lib:\
${pkgs.libxfixes}/lib:\
${pkgs.libxi}/lib:\
${pkgs.libsm}/lib:\
${pkgs.libice}/lib:\
${pkgs.glib.out}/lib:\
${pkgs.gtk3}/lib:\
$LD_LIBRARY_PATH"
  '';
}
```

### 使用方式

```bash
# 进入 nix shell（自动设置 LD_LIBRARY_PATH）
nix-shell

# 在 shell 内运行 Python
.venv/bin/python main.py

# 或单条命令
nix-shell --run '.venv/bin/python main.py'
```

### 关键点

- `glib.out` 而非 `glib`：NixOS 24.11+ 上 `glib` 默认输出是 `bin`，不含 `.so` 文件，需显式指定 `out` 输出
- 库路径写死在 `shellHook` 中，确保每次进入 shell 都生效
- `uv run` 会隔离环境变量，优先用 `.venv/bin/python` 直接运行

## 总结

| 方案 | 优点 | 缺点 |
|------|------|------|
| nix-ld + libraries | 全局生效，一次配置到处可用 | 库列表膨胀，全局污染，不同项目需求冲突时难处理 |
| mkShell | 项目级隔离，精确控制依赖，可版本管理 | 每个项目需要维护 shell.nix |

两种方案本质相同——都是把 nix store 里的库路径暴露给动态链接器。区别在于作用域：

- **nix-ld**：全局配置，适合桌面应用、游戏等通用场景
- **mkShell**：项目级配置，适合开发环境，依赖随项目走
