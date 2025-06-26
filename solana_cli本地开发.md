## Solana CLI 快速笔记（本地开发常用命令）

> 适用于 **macOS / Linux** 环境，假设你已通过
> `solana-install init <version>` 或 Homebrew 装好 CLI。
> 所有示例默认密钥位于 `~/.config/solana/id.json`。

---

### 1. `solana-test-validator` —— 启动本地单节点链

| 常见参数              | 作用                                       | 示例                                         |
| --------------------- | ------------------------------------------ | -------------------------------------------- |
| _(无)_                | 直接在当前目录下创建 `test-ledger/` 并启动 | `solana-test-validator`                      |
| `-r, --reset`         | 先删除旧账本再重建（干净链）               | `solana-test-validator -r`                   |
| `--ledger <DIR>`      | 指定账本目录                               | `solana-test-validator --ledger /tmp/ledger` |
| `--mint <KEYPAIR>`    | 创世时把全部代币铸给指定 Keypair           | `solana-test-validator --mint ~/mykey.json`  |
| `--rpc-port <PORT>`   | 改 RPC 端口（默认 8899）                   | `solana-test-validator --rpc-port 8898`      |
| `--limit-ledger-size` | 只保存最近若干箭头槽，省磁盘               | 常开                                         |

**启动输出关键字段**

- `Identity`：本节点公钥
- `Genesis Hash / Shred Version`：链 ID
- `JSON RPC URL`：CLI/前端连它的地址
- “`Processed / Confirmed / Finalized Slot`” 每秒刷新 → 数字递增即链在出块

> **TIP**
> 将 validator 放前台看日志，另开终端运行 CLI；或者：
>
> ```bash
> solana-test-validator > validator.log 2>&1 &
> tail -f validator.log
> ```

---

### 2. `solana config get | set` —— 查看 / 修改 CLI 默认 RPC 等

```bash
solana config get
solana config set --url http://127.0.0.1:8899  # 指到本地链
solana config set --keypair ~/.config/solana/devnet.json
```

关键字段：

| 字段            | 含义                          |
| --------------- | ----------------------------- |
| `RPC URL`       | 全部 CLI 子命令的默认目标节点 |
| `WebSocket URL` | 事件订阅用，通常自动推断      |
| `Keypair Path`  | 默认签名人                    |

---

### 3. `solana cluster-version` —— 查看目标节点版本

- **用途**：快速确认自己连到哪条链、当前版本号。
- **常见输出差异**

  - `2.2.x` ⇒ 官方 Devnet / Mainnet Beta
  - `2.1.x` ⇒ 你自己 build/旧版或教程脚本下载的 validator

```bash
solana cluster-version               # 用默认 RPC
solana cluster-version --url localhost
```

---

### 4. `solana balance` —— 查询余额

| 用法                      | 说明         |
| ------------------------- | ------------ |
| `solana balance`          | 默认密钥余额 |
| `solana balance <PUBKEY>` | 查询任意地址 |
| `--url <RPC>`             | 临时切换节点 |

> **在本地 validator 上**：创世账户初始拿到全部 SOL；可用 faucet 无限打钱。

---

### 5. `solana airdrop` —— 申请测试币

| 场景               | 规则                                    |
| ------------------ | --------------------------------------- |
| **Devnet**         | 每地址 ≤ 2 SOL / 8 h                    |
| **本地 validator** | 无限额度；其实是把 SOL 从创世帐号转给你 |

```bash
solana airdrop 1                 # Devnet: +1 SOL
solana airdrop 100 --url localhost  # 本地链随便打
```

---

### 6. 一条典型工作流

```bash
# ① 启动本地链（新项目建议 -r 清干净）
solana-test-validator -r --limit-ledger-size &

# ② 把 CLI 指向本地
solana config set --url localhost

# ③ airdrop 一些测试币
solana airdrop 10

# ④ 部署 / 调试合约或运行前端
anchor deploy
npm run dev
```

结束后 `fg` + `Ctrl-C` 或 `kill %1` 关闭 validator。

---

### 7. 常见坑与排查

| 症状                                                        | 原因 / 解决                                                                    |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `RPC connection failure` 连续刷                             | 刚启动未 ready；若 >1 min 则检查端口冲突 / 代理                                |
| CLI 版本 2.2.x，validator 2.1.x                             | 装了多套工具；`solana-install init 2.2.16` 统一                                |
| `solana airdrop` 返回 “airdrop failed: blockhash not found” | 你连的是 Mainnet（禁止空投）或 RPC 不支持 `/requestAirdrop`                    |
| balance 仍显示旧值                                          | 切回 Devnet 但忘了改 URL；`solana balance --url https://api.devnet.solana.com` |

---

### 8. 快捷命令备忘

```bash
# 一次性在本地链打100 SOL
solana airdrop 100 --url localhost

# 查询任意账号余额
solana balance <PUBKEY> --commitment finalized

# 查看本机 validator 日志（后台跑时）
tail -f ~/test-ledger/validator.log
```

---

> **记住一句话：** > _CLI 读的版本、余额、空投，全由 `solana config get` 中的 **RPC URL** 决定。_
> 在「本地链 / Devnet / Mainnet」之间切换时，先看它对不对！

### 9 | 必须把 `my-keypair.json` 挪到 `~/.config/solana/` 吗？

**完全不用。**
Solana CLI 读取哪个密钥文件，由下面两种方式决定——你只要给它指向正确路径即可：

| 方法           | 说明                                                           | 示例                                                       |
| -------------- | -------------------------------------------------------------- | ---------------------------------------------------------- |
| **命令行参数** | 每条命令临时指定                                               | `solana balance --keypair /Users/dylan/my-keypair.json`    |
| **全局配置**   | 用一次 `solana config set` 之后，所有 CLI 命令都会默认用这把钥 | `solana config set --keypair /Users/dylan/my-keypair.json` |

> 查看当前生效的是哪把钥：
>
> ```bash
> solana config get
> # 输出中的 “Keypair Path:” 就是 CLI 将要使用的文件
> ```

如果你更喜欢「跟官方教程保持一致，默认不用写 `--keypair`」，那就把它设到 config 里；文件位置放哪都行，无需移动。

---

### 2 | 一个 JSON = 一个账户吗？

**是的。**

- Solana 把私钥 + 公钥（合计 64 byte）以 _JSON 数组_ 存在一个文件里，例如

  ```json
  [12, 199, 34, …, 88]            ← 64 个 0-255 的整数
  ```

- 这 64 byte 描述的是 **一对 Ed25519 keypair**，对应链上的 **一个账户地址**。
- 想要多个账户，就各生成 / 备份各自的 JSON：

  ```bash
  solana-keygen new -o path/to/another-key.json
  ```

> ⚠️ **安全提醒**
> 存在本地磁盘的 JSON 包含私钥，一旦泄露就等于账户全权给了别人。请放在只自己可读的目录（`chmod 600`），重要账户最好离线保存或用硬件钱包。

---

### 3 | 常见操作备忘

```bash
# 生成新钥并直接放进 CLI 默认 (~/.config/solana/id.json)
solana-keygen new

# 把现有 my-keypair.json 设置成默认
solana config set --keypair /Users/dylan/my-keypair.json

# 临时用另一把钥发交易
solana transfer RECEIVER 1 --keypair other.json --url localhost
```

只要记住：**CLI 看的是 “Keypair Path” 配置**——文件放哪、叫什么名字都行。

## phantom 上传本地 帐户

下面直接给出两条 **100 % 可跑通** 的命令——**任选其一** 就能把
`/Users/dylan/.config/solana/test_1.json` 里的 64 byte 私钥转成 Phantom 需要的 **base58 字符串**。

---

### 方案 A —  用 Python（最稳，无需理 Node/bs58）

```bash
python - <<'PY'
import json, base58
secret = json.load(open("/Users/dylan/.config/solana/test_1.json"))
print(base58.b58encode(bytes(secret)).decode())
PY
```

- **输出**：一串 \~90 字符的字母数字（这是完整私钥）。
- **下一步**：Phantom → 头像 → **Add / Connect Wallet** → **Import Private Key**
  → Network 选 **Solana** → 粘贴 → Done。

  > 导入后公钥应当是 `5LwycGT1…` ——与你的 CLI 地址完全一致。

> _如 Python 提示 `No module named base58`，先 `pip install --user base58`，再执行上面命令。_
