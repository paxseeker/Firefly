---
title: NixOS 开发环境搭建方法总结
category: linux
published: 2026-08-05
draft: false
description: NixOS 开发环境搭建方法总结
tags:
  - linux
  - nixos
---

# NixOS 开发环境搭建方法总结

## 目录

1. [概述](#概述)
2. [方法一：传统 `nix-shell` + `shell.nix`](#方法一传统-nix-shell--shellnix)
3. [方法二：Flake + `nix develop`](#方法二flake--nix-develop)
4. [方法三：FHS 兼容环境 `buildFHSEnv`](#方法三fhs-兼容环境-buildfhsenv)
5. [方法四：`direnv` 自动激活](#方法四direnv-自动激活)
6. [核心概念辨析](#核心概念辨析)
7. [方法对比总表](#方法对比总表)
8. [场景化选择建议](#场景化选择建议)
9. [实际案例：Buildroot 开发环境](#实际案例buildroot-开发环境)

---

## 概述

NixOS 与传统 Linux 发行版最大的区别在于：
- **没有 FHS（Filesystem Hierarchy Standard）**：不存在 `/usr/bin`、`/bin`、`/lib64` 等传统目录
- **一切皆在 `/nix/store`**：所有软件包以不可变哈希路径存储
- **声明式与可复现**：环境配置通过 Nix 表达式声明，理论上任意机器都能复现相同环境

这些特性带来了极致的可复现性，但也导致某些**硬编码传统路径**的软件（如 Buildroot、Steam、某些闭源 IDE）无法直接运行。因此，NixOS 提供了多种开发环境搭建方案，以适应不同需求。

---

## 方法一：传统 `nix-shell` + `shell.nix`

### 核心概念

`nix-shell` 是 Nix 的经典命令，用于进入一个临时环境，其中包含指定的依赖包。`shell.nix` 是描述该环境的 Nix 表达式文件。

`mkShell` 是 Nixpkgs 提供的**函数**，用于方便地定义开发环境，管理 `PATH`、环境变量、构建钩子（`shellHook`）等。

### 典型配置

```nix
# shell.nix
{ pkgs ? import <nixpkgs> {} }:

pkgs.mkShell {
  name = "my-project";

  buildInputs = with pkgs; [
    gcc
    gnumake
    python3
    nodejs
    git
  ];

  nativeBuildInputs = with pkgs; [
    pkg-config
  ];

  shellHook = ''
    export MY_PROJECT_ROOT=$(pwd)
    echo "🚀 进入开发环境"
  '';

  # 禁用某些编译器加固选项（如 Buildroot 需要）
  hardeningDisable = [ "format" ];
}
```

### 使用方式

```bash
# 进入 shell.nix 定义的环境
nix-shell

# 直接运行命令（不进入交互式 shell）
nix-shell --run "make && ./myapp"

# 使用特定 nixpkgs 版本
nix-shell -I nixpkgs=https://github.com/NixOS/nixpkgs/archive/nixos-24.11.tar.gz
```

### 优点

- ✅ 简单直观，学习曲线低
- ✅ 无需 Flakes，兼容旧版 Nix
- ✅ 适合个人快速搭建临时环境

### 缺点

- ❌ 无锁文件（lockfile），依赖版本可能漂移
- ❌ 无法直接利用 Flake 的输入复用
- ❌ 对硬编码 `/usr/bin` 路径的软件无能为力

---

## 方法二：Flake + `nix develop`

### 核心概念

**Flakes** 是 Nix 2.4+ 引入的实验性功能（已广泛稳定使用），通过 `flake.nix` 和 `flake.lock` 实现：
- **声明式输入管理**：明确指定 nixpkgs 版本、外部 Flake 依赖
- **可复现锁文件**：`flake.lock` 锁定所有依赖的确切版本
- **标准化输出**：`packages`、`devShells`、`apps` 等统一接口

`nix develop` 是 Flake 时代进入开发环境的命令，功能上替代了 `nix-shell`。

### 典型配置

```nix
# flake.nix
{
  description = "My Project Development Environment";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system: let
      pkgs = nixpkgs.legacyPackages.${system};
    in {
      devShells.default = pkgs.mkShell {
        buildInputs = with pkgs; [
          gcc
          gnumake
          python3
          nodejs
          git
        ];

        shellHook = ''
          echo "🚀 进入 Flake 开发环境"
        '';
      };

      # 同时可以定义可构建的包
      packages.default = pkgs.hello;
    });
}
```

### 使用方式

```bash
# 进入开发环境
nix develop

# 进入特定 shell（如果有多个）
nix develop .#my-shell

# 直接运行命令
nix develop --command make

# 更新锁文件
nix flake update

# 格式化 flake.nix
nix fmt
```

### 优点

- ✅ **完全可复现**：`flake.lock` 确保任何人在任何时间得到相同环境
- ✅ **版本锁定**：明确知道用的是哪个版本的 nixpkgs
- ✅ **输入复用**：多个项目可以共享同一个 nixpkgs 输入，节省磁盘空间
- ✅ **标准化**：社区工具（如 `direnv`、CI 系统）对 Flake 有良好支持

### 缺点

- ❌ 需要 Nix 2.4+ 并启用 Flakes
- ❌ 学习曲线略高于 `shell.nix`
- ❌ 对硬编码 FHS 路径的软件仍然无能为力

---

## 方法三：FHS 兼容环境 `buildFHSEnv`

### 核心概念

`buildFHSEnv`（全称 build Filesystem Hierarchy Standard Environment）是 Nixpkgs 提供的 **FHS 模拟器**。它通过 `bubblewrap`（或 chroot）创建一个临时的、遵循传统 Linux 目录结构的文件系统视图。

**为什么需要它？**

NixOS 没有 `/usr/bin`、`/bin`、`/lib64`。许多软件（尤其是构建系统如 Buildroot、游戏平台 Steam、闭源 IDE）硬编码了这些路径，导致直接运行时报错：

```
/usr/bin/file: No such file or directory
/lib64/ld-linux-x86-64.so.2: No such file or directory
```

`buildFHSEnv` 通过符号链接和 bind mount，在临时命名空间中"伪造"出这些路径。

### 典型配置

```nix
# fhs-shell.nix
{ pkgs ? import <nixpkgs> {} }:

(pkgs.buildFHSEnv {
  name = "fhs-env";

  # 目标架构包（64位）
  targetPkgs = pkgs: with pkgs; [
    gcc
    gnumake
    binutils
    perl
    python3
    which
    gnused
    gnutar
    wget
    cpio
    unzip
    rsync
    bc
    file
    patch
    pkg-config
    ncurses
  ];

  # 多架构包（32位兼容，可选）
  multiPkgs = pkgs: with pkgs; [
    zlib
  ];

  # 进入环境后执行的脚本
  runScript = "bash";

  # 额外环境变量
  profile = ''
    export C_INCLUDE_PATH=/usr/include:$C_INCLUDE_PATH
    export CPLUS_INCLUDE_PATH=/usr/include:$CPLUS_INCLUDE_PATH
  '';
}).env
```

### 使用方式

```bash
# 传统方式
nix-shell fhs-shell.nix

# Flake 方式
nix run .#fhs-env
```

进入后验证：

```bash
$ ls /bin/sh
/bin/sh  # ← 存在！指向 /nix/store/...-bash

$ ls /usr/bin/gcc
/usr/bin/gcc  # ← 存在！

$ file /bin/sh
/bin/sh: symbolic link to /nix/store/...-bash-5.2-p15/bin/sh
```

### 与 `mkShell` 的关键区别

| 特性 | `mkShell` | `buildFHSEnv` |
|------|-----------|---------------|
| 管理 `PATH` | ✅ | ✅ |
| 管理环境变量 | ✅ | ✅（通过 `profile`） |
| 创建 `/usr/bin`、`/bin` | ❌ | ✅ |
| 创建 `/lib`、`/lib64` | ❌ | ✅ |
| 使用容器/命名空间 | ❌ | ✅（bubblewrap） |
| 适合 Buildroot/Steam | ❌ | ✅ |

### 优点

- ✅ 解决 FHS 硬编码路径问题
- ✅ 对"传统"软件兼容性最好
- ✅ 可与传统/Flake 方式结合使用

### 缺点

- ❌ 启动稍慢（需要创建命名空间）
- ❌ 环境隔离性较强，某些 Nix 工具链特性受限
- ❌ 不适合纯 Nix 生态开发（杀鸡用牛刀）

---

## 方法四：`direnv` 自动激活

### 核心概念

`direnv` 是一个 shell 扩展，能在进入目录时**自动加载/卸载环境变量**。`nix-direnv` 是其 Nix 集成插件，支持自动激活 `shell.nix` 或 Flake 定义的环境。

**核心价值**：无需手动运行 `nix-shell` 或 `nix develop`，进入项目目录即自动进入开发环境，离开目录自动退出。

### 配置方式

#### 1. 安装 direnv

```nix
# configuration.nix
{ pkgs, ... }: {
  programs.direnv = {
    enable = true;
    nix-direnv.enable = true;  # 使用 nix-direnv 替代原生实现，更快
  };
}
```

#### 2. 项目根目录创建 `.envrc`

**传统方式：**

```bash
# .envrc
use nix
```

**Flake 方式（推荐）：**

```bash
# .envrc
use flake
```

#### 3. 首次授权

```bash
direnv allow
```

### 使用体验

```bash
$ cd ~/projects/my-project
direnv: loading ~/projects/my-project/.envrc
direnv: using flake
direnv: nix-direnv: Using cached dev shell
🚀 进入 Flake 开发环境

$ which gcc
/nix/store/...-gcc-wrapper-13.3.0/bin/gcc

$ cd ..
direnv: unloading
```

### 优点

- ✅ **零摩擦**：进入目录 = 进入环境，离开 = 退出
- ✅ **缓存加速**：`nix-direnv` 缓存 shell 环境，二次进入几乎瞬时
- ✅ **IDE 友好**：VS Code、JetBrains 等支持 direnv 插件
- ✅ **团队一致**：`.envrc` + `flake.nix` 确保所有人环境相同

### 缺点

- ❌ 需要安装并配置 direnv
- ❌ 首次进入需要构建/下载，可能较慢
- ❌ 需要信任 `.envrc`（安全机制，需手动 `direnv allow`）

---

## 核心概念辨析

### `mkShell` vs `buildFHSEnv`

- **`mkShell`** = 定义"有哪些包在 PATH 里、有哪些环境变量"。它**不修改文件系统视图**。
- **`buildFHSEnv`** = 创建一个**虚拟的 FHS 文件系统**。它**修改了文件系统视图**，让 `/usr/bin` 等路径存在。

两者可以组合：在 Flake 的 `devShells.default` 中使用 `mkShell`，其中 `buildInputs` 包含一个 `buildFHSEnv` 包。

### `nix-shell` vs `nix develop`

- **`nix-shell`** = 传统命令，读取 `shell.nix`，兼容所有 Nix 版本
- **`nix develop`** = Flake 命令，读取 `flake.nix` 中的 `devShells`，需要 Flakes 支持

功能上两者等价：都进入一个包含指定依赖的临时 shell。

### `buildFHSEnv` vs `buildFHSEnvBubblewrap`

在较新的 Nixpkgs 中，`buildFHSEnv` 默认使用 `bubblewrap` 实现（之前叫 `buildFHSUserEnv`）。现在通常统一使用 `buildFHSEnv`，无需关心底层实现。

---

## 方法对比总表

| 维度 | `shell.nix` + `nix-shell` | Flake + `nix develop` | `buildFHSEnv` | `direnv` |
|------|---------------------------|----------------------|---------------|----------|
| **本质** | Nix 表达式 + CLI 命令 | Nix 表达式 + CLI 命令 | FHS 模拟器 | Shell 扩展 |
| **可复现性** | ⚠️ 无锁文件 | ✅ `flake.lock` | ⚠️ 无锁文件 | 依赖底层方案 |
| **FHS 兼容** | ❌ | ❌ | ✅ | 依赖底层方案 |
| **自动激活** | ❌ 手动进入 | ❌ 手动进入 | ❌ 手动进入 | ✅ 自动 |
| **学习成本** | ⭐ 低 | ⭐⭐ 中 | ⭐⭐ 中 | ⭐⭐ 中 |
| **社区趋势** | ⬇️ 逐步迁移 | ⬆️ 主流推荐 | 特定场景 | ⬆️ 强烈推荐 |
| **典型场景** | 临时测试、旧项目 | 团队项目、CI/CD | Buildroot、Steam | 日常开发 |

---

## 场景化选择建议

### 场景 1：纯 Nix 生态开发（Rust、Haskell、Node.js 等）

**推荐**：Flake + `nix develop` + `direnv`

```nix
# flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  outputs = { self, nixpkgs }: let
    pkgs = nixpkgs.legacyPackages.x86_64-linux;
  in {
    devShells.default = pkgs.mkShell {
      buildInputs = with pkgs; [ cargo rustc rustfmt clippy ];
    };
  };
}
```

```bash
# .envrc
use flake
```

### 场景 2：编译 Buildroot / 嵌入式 Linux

**推荐**：Flake + `buildFHSEnv` + `direnv`

Buildroot 硬编码了 `/usr/bin/file`、`/bin/sh` 等路径，且编译器加固选项会导致失败，必须用 FHS 环境：

```nix
# flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

  outputs = { self, nixpkgs }: let
    pkgs = nixpkgs.legacyPackages.x86_64-linux;
    fhs = pkgs.buildFHSEnv {
      name = "buildroot-env";
      targetPkgs = p: with p; [
        gcc gnumake binutils perl python3
        file patch wget cpio unzip rsync bc
        ncurses pkg-config which gnused gnutar
      ];
      runScript = "bash";
      profile = ''
        export C_INCLUDE_PATH=/usr/include:$C_INCLUDE_PATH
        export CPLUS_INCLUDE_PATH=/usr/include:$CPLUS_INCLUDE_PATH
      '';
    };
  in {
    devShells.default = pkgs.mkShell {
      buildInputs = [ fhs ];
      hardeningDisable = [ "format" ];
      shellHook = ''
        echo "Run 'buildroot-env' to enter FHS environment"
      '';
    };
  };
}
```

### 场景 3：运行 Steam / 游戏 / 闭源软件

**推荐**：直接使用 `buildFHSEnv`

NixOS 官方提供了 `programs.steam.enable`，底层就是 `buildFHSEnv`。对于其他闭源软件，可以：

```nix
nix run nixpkgs#steam-run -- ./my-closed-source-app
```

`steam-run` 是 NixOS 预配置的 FHS 环境，包含大量常见库。

### 场景 4：临时测试某个工具

**推荐**：`nix-shell` 或 `nix run`

```bash
# 临时进入包含 python3 和 numpy 的环境
nix-shell -p python3 python3Packages.numpy

# 或者直接运行
nix run nixpkgs#python3 -- -c "print('hello')"
```

---

## 实际案例：Buildroot 开发环境

以下是一个完整的、可直接使用的 Buildroot 开发环境配置：

```nix
# flake.nix
{
  description = "Buildroot Development Environment";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system: let
      pkgs = nixpkgs.legacyPackages.${system};

      # 创建 FHS 环境
      buildrootFHS = pkgs.buildFHSEnv {
        name = "buildroot-fhs";

        targetPkgs = p: with p; [
          # 基础构建工具
          gcc
          gnumake
          binutils
          perl
          python3
          which
          gnused
          gnutar
          wget
          cpio
          unzip
          rsync
          bc
          file
          patch
          pkg-config

          # 交互式配置
          ncurses

          # 版本控制
          git
        ];

        multiPkgs = p: with p; [
          zlib
        ];

        runScript = "bash";

        profile = ''
          export C_INCLUDE_PATH=/usr/include:$C_INCLUDE_PATH
          export CPLUS_INCLUDE_PATH=/usr/include:$CPLUS_INCLUDE_PATH
          export PS1="[buildroot-fhs] $PS1"
        '';
      };

    in {
      devShells.default = pkgs.mkShell {
        buildInputs = [ buildrootFHS ];

        # 禁用导致 Buildroot 编译失败的加固选项
        hardeningDisable = [ "format" ];

        shellHook = ''
          echo "=========================================="
          echo "  Buildroot Development Environment"
          echo "=========================================="
          echo ""
          echo "Run 'buildroot-fhs' to enter FHS environment"
          echo "Then: make menuconfig && make"
          echo ""
        '';
      };
    });
}
```

```bash
# .envrc
use flake
```

使用流程：

```bash
# 1. 进入项目目录（自动激活 direnv）
cd ~/buildroot-project

# 2. 进入 FHS 子环境
buildroot-fhs

# 3. 正常编译 Buildroot
make menuconfig
make -j$(nproc)
```

