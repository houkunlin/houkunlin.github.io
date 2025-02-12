---
title: Rust 在 Win11 下开发编译 Win7 exe
date: 2025-02-12 15:16:00
updated: 2025-02-12 15:16:00
tags:
  - rust
---

- 安装每日版本 `rustup toolchain install nightly`
- 切换到每日版本 `rustup default nightly`
- 安装 `rust-src` 组件 `rustup component add rust-src --toolchain nightly-x86_64-pc-windows-msvc`
- 下载 `https://github.com/PLC-lang/llvm-package-windows/releases/tag/v14.0.6` 解压后，设置 `LIBCLANG_PATH` 环境变量指向 `$LLVM_PATH\bin` 路径
- ~~安装 `rustup target add x86_64-win7-windows-msvc`~~ _该步骤好像不需要_
- 使用命令 `cargo build --release -Z build-std --target x86_64-win7-windows-msvc` 打包编译

其他命令

- 查看目标平台 `rustc --print=target-list`
- 列表已按照的目标平台 `rustup target list`

参考文档

- [rust-bindgen报错 ‘Unable to find libclang的解决办法](https://www.cnblogs.com/zhyp/p/18702396)
- [怎么样在>=rustc 1.78.0 stable 下编译 exe 在 win7 下可以运行](https://www.v2ex.com/t/1084204)
- [Rust升级到1.78翻车急救](https://rustcc.cn/article?id=cfb78a4d-9828-44f6-93fc-986c45725977)
