# Anchor 项目文件速查笔记

---

## 1 · 目录总览

```
my_anchor_project/
├── Anchor.toml          ← Anchor CLI 全局配置
├── Cargo.toml           ← *workspace* 虚拟清单
├── Cargo.lock           ← 整个工作区唯一锁文件
└── programs/
    └── simple-program/
        ├── Cargo.toml   ← 该程序自身的 crate 清单
        ├── Cargo.lock   ← 若单独编译时临时生成（可删）
        └── Xargo.toml   ← 为 BPF 目标定制 sysroot
```

---

## 2 · 顶层三件套

| 文件                | 作用                                                                                                                  | 常见字段 / 要点                                            |
| ------------------- | --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **Anchor.toml**     | Anchor CLI 的“控制面板”：设定默认集群、钱包、脚本、各程序 ID 等。                                                     | `[provider] cluster = "mainnet"`, `[programs.localnet]` 等 |
| **Cargo.toml** (根) | Rust _workspace_ 的虚拟清单：自身不编译代码，只用 `[workspace] members = ["programs/*"]` 将所有程序打包进同一工作区。 | 仅含 `[workspace]` 块                                      |
| **Cargo.lock** (根) | 锁定工作区所有 crate 的确切依赖版本，确保可重现构建。Cargo 规定一个 workspace 只保留 **一个** 锁文件。                | 自动生成，勿手改                                           |

---

## 3 · 子目录三件套

| 文件                  | 作用                                                                                                                            | 常见字段 / 要点                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| **Cargo.toml** (程序) | 描述某个 Solana 程序的 crate：指定 `name`、版本、`crate-type = ["cdylib","lib"]` 并列出依赖（`anchor-lang`, `anchor-spl` 等）。 | `[package]`, `[lib]`, `[dependencies]`           |
| **Xargo.toml**        | 指示 Xargo 为 `bpfel-unknown-unknown` 目标构建极简 `core/alloc` 标准库；Solana SDK 在 `anchor build` 流程中会用到。             | `[dependencies.std] default_features = false` 等 |
| **Cargo.lock** (程序) | 若在该子目录里手动 `cargo build` 会生成本地锁；标准 `anchor build` 流程会忽略它，可安全删除。                                   | 可删                                             |

---

## 4 · 两个 Cargo.toml 的关键区别

| 对比维度     | 根目录 Cargo.toml                | programs/\*/Cargo.toml          |
| ------------ | -------------------------------- | ------------------------------- |
| **定位**     | _虚拟_ 清单：只描述 workspace    | _实际_ crate 清单：可编译       |
| **职责**     | 管理成员包、共用输出目录与锁文件 | 说明单个程序如何编译成 BPF      |
| **编译目标** | 宿主机 (x86_64…)；常用来跑测试   | BPF (`bpfel-unknown-unknown`)   |
| **典型字段** | `members = ["programs/*"]`       | `crate-type = ["cdylib","lib"]` |

---

## 5 · 常见问答

1. **子目录的 `Cargo.lock` 可以删吗？**
   可以。保留根目录那一个即可避免版本冲突。

2. **`Xargo.toml` 必须存在吗？**
   是。它让 Xargo 先为 BPF 目标构建 sysroot，否则编译器找不到 `core/alloc`。

3. **添加第二个程序的流程？**

   - 在根目录运行 `anchor init my-second-program --program-name my_second_program`（或 `anchor new …`）。
   - Anchor 会在 `programs/` 下生成新子目录，并自动更新根 `Cargo.toml` 的 `[workspace]` 与 `Anchor.toml` 的 `[programs.*]`。

4. **为什么只用一个锁文件？**
   确保 workspace 内所有程序依赖的库版本完全一致，避免链上 bytecode 不一致及重复下载依赖。

---

## 6 · 速记命令

```bash
# 一次性编译工作区所有程序
anchor build          # = cargo build-sbf --workspace

# 单独调试某个程序（不会用根锁文件）
cd programs/simple-program
cargo build           # 仅建议本地快速测试，结束后删 local Cargo.lock

# 清理生成文件
anchor clean
```
