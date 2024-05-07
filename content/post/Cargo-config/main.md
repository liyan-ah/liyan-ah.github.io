---
title: "Cargo config 使用备注"
description: 本文备注下如何配置 Cargo dependencies 为指定仓库。
date: 2024-05-07T11:39:00+08:00
categories:
    - Code
tags:
    - Rust
    - Cargo
---

本文旨在回答如下问题：
1. Rust 项目中，如何配置 dependencies 的路径为指定仓库。
2. 在不指定 branch 或 version 的情况下，dependencies 会拉取什么版本/仓库里的数据。
3. Rust 项目中，配置 dependencies 的最佳实践。

以下为正文。

## 如何配置 dependencies 的路径？
出于稳定性考虑，工作中使用的项目，尤其是一些开源的项目，往往无法和社区时刻保持一致。甚至，在选定了一个基线版本后，以周或者年计，主干会持续保持这个版本。有一些效果较好的“新特性”需要合入时，就需要以 patch 的形式合入。同样出于稳定性考虑，
或者兼容性考虑，服务的依赖，甚至依赖的依赖都是无法进行升级的。如果项目的新特性需要依赖库进行变更，就只能将依赖库的基线版本单独维护，按需合入依赖库的 patch。这样的维护方式看起来较为繁琐，但是能够有效避免项目因依赖库的升级而引入新的问
题。  
有了这样的需求，对应到 Rust 项目中，就需要对 Cargo.toml 里的 dependencies 配置进行调整，将部分依赖调整为自有库。
参照[reference/specifying-dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html)，可以通过如下方式来指定依赖库：
```Rust
[dependencies]
regex = { version = "1.10.3", git = "https://github.com/rust-lang/regex.git", branch = "next", rev = "0c0990399270277832fbb5b91a1fa118e6f63dba", tag = "11.10.4" }
```
除了 git，version、branch、rev 以及 tag 都是可选的。补充一个参数即可实现分支提交控制的目的。在实践过程中，考虑到分支上的提交是时刻进行的，在确认了基础的功能后，可以考虑发布一个版本。一来可以通过 tag 来进行版本/功能的
发布控制，同时也便于后续的功能梳理以及项目整理关键里程碑可视化。通过 branch 来指定依赖库会比较方便，这种方式对仓库的开发流程管理较为依赖：如果分支上提交了功能异常的代码，项目的编译就会异常。rev 方案也可以唯一指定代码版本，只是看起来
不像 tag 那样直观。

## dependences 默认拉取什么版本？
笔者在近期的工作中，遇到了如下的配置方式：
```Rust
[dependencies]
regex = { git = "https://github.com/rust-lang/regex.git" }
```
这里并没有指定 version, branch 或者 rev。但是在编译项目时也会进行拉取的动作。默认会拉取目标仓库的最新提交（效果和直接 git clone 一致）。  
在复杂的项目中，通常会出现依赖库和依赖库的依赖有重合的情况。而Rust 工程在编译时，会通过校验目标库的 source 信息（Cargo.lock 中生成）来识别两个库是相同。当项目的依赖库的 dependence 配置和项目依赖库的依赖的 dependence 配置不同时，
就会出现种种类型不兼容的情况。比如，目前有这样的一个项目[dep-check](https://github.com/liyan-ah/dep-check)，其依赖
[trait-lib](https://github.com/liyan-ah/trait-lib)和[middle-lib](https://github.com/liyan-ah/middle-lib)两个库。同时，[middle-lib](https://github.com/liyan-ah/middle-lib)又对
[trait-lib](https://github.com/liyan-ah/trait-lib)存在依赖。当[dep-check](https://github.com/liyan-ah/dep-check)使用了`trait_lib::Check`，并且将该`trait`通过引用
[middle-lib](https://github.com/liyan-ah/middle-lib)里的函数进行处理时，就涉及到[dep-check](https://github.com/liyan-ah/dep-check)和[middle-lib](https://github.com/liyan-ah/middle-lib)对[trait-lib](https://github.com/liyan-ah/trait-lib)的引用校验问题：两个库里涉及的`Check`是否是同一个`Trait`？就笔者目前的理解来看，由于 rust 里的 trait 采用的不是 duck-typing，因此就需要编译器在编译时对类型进行强校验。在这个小示例中，如果 middle-lib
的 dependencies 设置如下：
```Rust
[dependencies]
trait-lib = { git = "https://github.com/liyan-ah/trait-lib.git", tag = "1.0.0" }
```
而 dep-check 的 dependencies 设置如下：
```Rust
[dependencies]
trait-lib = { git = "https://github.com/liyan-ah/trait-lib.git", tag = "1.0.0" }
middle-lib = { git = "https://github.com/liyan-ah/middle-lib.git" } 
```
dep-check 编译时，其引用的`trait_lib::Check`和 middle-lib 里使用的`trait_lib::Check`就无法认为是同一个。依赖库的设置可以通过 Cargo.lock 里的 source 来确认。当 dep-check 和 middle-lib 的 Cargo.lock 对 trait-lib 的 source
配置相同时，就不会出现类型不一致的问题。

配置的问题在这里有描述：[The dependency resolution is confused when using git dependency and there's a lockfile](https://github.com/rust-lang/cargo/issues/11490)。

## dependences 配置的最佳实践？
golang 和 rust 都支持通过配置 git 仓库的地址来直接引用，个人还是比较喜欢 rust 的配置方式：通过 Cargo.toml 能够简洁、明了的声明各种依赖的信息，在工程里可以直接使用库名（而非 golang 里的项目地址）。此外，rust 里的 workspace 机制
对仓库里存在多个 sub-lib 时也能较好的处理依赖的管理。  
下面是一个实践示例[dep-check](https://github.com/liyan-ah/dep-check)：
```Rust
[workspace]
members = [
    "dep-check",
    "dep-run",
]

default-members = ["dep-check"]
resolver = "2"

[workspacepackage]
name = "dep-check"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[workspace.dependencies]
trait-lib = { git = "https://github.com/liyan-ah/trait-lib.git", tag = "1.0.0" }
middle-lib = { git = "https://github.com/liyan-ah/middle-lib.git", tag = "1.0.0" }
```
对于其中的一个 member:dep-check，其配置为：
```Rust
[package]
name = "dep-check"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
trait-lib = { workspace = true }
middle-lib = { workspace = true }
```
这样，就能很便捷的对项目里的依赖进行管理了。

以上。
