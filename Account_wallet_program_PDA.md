# 1. 概念

## 1.1 Program（合约）

- **Solana** 上的合约称为 **Program**，本质是一个“可执行账户”，通过 **BPF** 方式实现。
- 部署时，你会得到一个 **Program ID**（在链上是一个公钥，标记 `executable = true`）。
- **Program** 本身只负责逻辑，**不直接**保存合约状态。合约需要在额外的“数据账户”里存储状态信息。

## 1.2 账户（Account）

- **Solana** 中一切都是“Account”，包括用户钱包、数据存储账户、Program 可执行账户等。
- **普通账户**（钱包）可以由用户掌握私钥，存放 SOL；
- **数据账户** 用来保存具体的数据（由某个 Program 拥有）；
- **可执行账户（Program）** 存储合约代码，`executable = true`。

## 1.3 PDA（Program Derived Address）

- **PDA = Program Derived Address**：由 **Program ID + seeds** 计算出来的一个“无私钥”地址。
- 只有该 Program 能使用 **invoke_signed**（传入 seeds + bump）来对 PDA 做“签署”，从而读写 PDA 的数据。
- 常用于：**给每位用户建立子账户**、托管代币、管理合约状态等。

### 1.3.1 Program 可以使用任意多个 PDA

- 一个 **Program（合约）** 并不只有一个 PDA，而是可以**派生出很多个**不同的 PDA 地址。
- PDA 的地址由种子（seeds）和 Program ID 经过哈希推导得出，只要种子不同，就可以得到不同的 PDA。

#### 1.3.1.1 例子：多用户、多子账户

假设你写了一个合约，要给**每个用户**都创建一份单独的数据存储（比如用户的自定义配置、积分、白名单信息等）。可以这样设计：

1. **PDA Seeds**：`["user-data", user_pubkey]`
   - 其中 `user_pubkey` 是指用户的钱包地址。
2. **PDA Address**：`PDA = hash( "user-data" + user_pubkey + ProgramID )`
3. 当你用 `Anchor` 或者底层 `invoke_signed` 创建账户时，就会根据这个 seeds 生成一个唯一的 PDA。
4. 不同的 `user_pubkey` 派生出的地址不一样，于是你就能针对**每个用户**在链上拥有一个独立的**PDA**。

#### 1.3.1.2 多个用途的 PDA

- 你不仅能给用户创建 PDA，也能给合约本身的“全局状态”创建一个 PDA，用来保存整个合约的配置数据；
- 也可以派生出一个“资金托管账号”之类的 PDA，用来存储代币（SPL Token）。
- **只要你能定义 seeds**，就可以让合约“自助”派生出各种专属账户地址。

### 1.3.2 这些 PDA 都归同一个 Program 管理吗？

**是的**。

- 当我们说一个 PDA “归属”某个 Program 时，意思是**它的 owner 字段 = Program ID**。
- 只有该 Program（使用 `invoke_signed` 并提供正确 seeds）才能对这个 PDA 进行写操作。
- 别的合约或用户没有 PDA 的私钥（因为它根本没有私钥），也无法冒充这个 PDA 来签名。

因此，你可以理解为：**同一个 Program 可以衍生出若干 PDA，每个 PDA 的所有权都写着“这个 Program”。** 这样就形成了很多“子账户”，由 Program 来管理。

### 1.3.3 “为每个用户生成子账户”更具体的流程

以一个典型的 Anchor 场景为例：

1. **用户调用合约函数** `initialize_user_data(ctx, ...)`
2. 在这个函数对应的 `#[derive(Accounts)]` 结构里，你写：
   ```rust
   #[account(
       init,
       seeds = [b"user-data", user.key().as_ref()],
       bump,
       payer = user, // 由用户付费来创建这块存储
       space = 8 + size_of::<UserData>(),
   )]
   pub user_data_account: Account<'info, UserData>,
   ```
3. Anchor 在编译时会根据你声明的 seeds，给你算出一个 PDA 地址，并自动完成：
   - 创建该账户（需要足够 SOL 交租金）
   - 设置账户的 owner = 你的 Program
   - 初始化里面的数据
4. 这样，你就有了一个 PDA，专门存这个用户的资料。

> 如果你需要更多的子账户，比如“用户的物品清单”、“用户积分账本”，也可以定义新的 seeds： `["user-items", user.key()]`、`["user-points", user.key()]` 等等。每个 seeds 对应一个**独立**的 PDA 地址。

### 1.3.4 PDA 可以存储代币吗？

**可以**。在 Solana 里，代币通常使用“**SPL Token**”合约实现，而每个代币账户都是一个“Account”结构，也有 owner、数据等。

- 如果你想让合约托管代币，可以用**PDA**来创建或控制一个 **SPL Token Account**。
- 这样就能保证**只有该 Program**能动用这个 Token Account 里的资产。

### 1.3.5 总结

1. **一个 Program 可以衍生多个 PDA** 吗？
   - **是的**，而且往往就是这么做。
2. **这些 PDA 都只能被该 Program 管理** 吗？
   - **对**，只要 PDA 的 owner = Program ID，就只能通过 Program 逻辑（合约函数）来写入或修改里面的数据。
3. **如何区分不同用户的数据**？
   - 通过在 seeds 里包含用户公钥或其他唯一标识，就能为每个用户生成独立的子账户（PDA）。

这样你就可以把“**Program = 大管家**，PDA = 各种子账户\*\*”的模型想象成：

- 每个用户在合约系统里都有一个“私人抽屉”（PDA），只属于他们，但抽屉上锁了，只有**大管家**（Program）能打开或往里放东西。
- 大管家（Program）自己也可以有若干抽屉来存储全局的内部状态或资金。

## 1.4 Program keypair vs. 钱包 keypair

1. **钱包 keypair**
   - 相当于“用户账户”，用于付费、签名日常交易；存放在 `~/.config/solana/id.json` 或你自己指定的文件。
2. **Program keypair**
   - 为合约生成的一对公私钥，公钥会成为 **Program ID**；私钥用于“部署和升级合约”时签名，**不**用于支付交易费。

### 1.4.1 任何一对公私钥都可以充当“普通钱包”吗？

**是的，理论上可以**。在 Solana 链上，“公钥”就是一个唯一地址。如果你给这个地址转入一些 SOL，那么它就可以像普通钱包地址一样持有余额、支付交易费等等。这里并不强制区分“这是给人用的”还是“给程序用的”。从纯粹的区块链视角看，它们都是**账户公钥**。

### 1.4.2 部署后 `program_id` 那个账户会变成 “executable = true”，这时还是能当钱包用吗？

当你用 `my_program-keypair.json` 的公钥部署合约后：

1. 链上会出现一个“可执行账户”（Program Account），标记 `executable = true`。
2. 代码会存储在这个账户里（或者，如果是**可升级**合约，则还会关联到一个 `ProgramDataAccount`，稍后会提到）。

**能不能继续当钱包用？**

- 从**理论**上，你依然可以往这个公钥打 SOL，但由于它的所有者是 `BPFUpgradeableLoader`（或 `BPFLoader`），而且 `executable = true`，系统基本不会让你把它当成“普通系统账户”来随意转进转出或写数据。
- 一旦标记为可执行账户，这个账户的“数据布局”已经被 Loader 所管理，不能像普通系统账户那样存放自定义数据。
- 因此，在**实践**中，我们不会再把这个可执行账户当钱包地址来使用，也**几乎不会**往里放额外资金。

所以，你说的“只要是一对公私钥，都可以当作一个普通钱包来用”，**在部署之前**确实没区别；但一旦用来部署合约，那个地址就被“改造成”程序地址了。后续就不再是一个常规钱包账户。

## 1.5 declare_id!("...")

- 在 Anchor 中，`declare_id!("...")` 用来在源码里**声明**这个 Program 的 ID。
- 这样 Anchor 在编译时就知道“这个合约地址是某个固定公钥”，便于进行验证和元数据生成。
- 一定要保证和实际部署使用的公钥一致，否则合约调用会失败。

## 1.6 可升级合约与租金

- **可升级合约**：Solana 原生支持可升级 Program，如果你保留了 Program keypair 的私钥（升级权限），就可以在未来替换合约代码。
- **租金**：任何在链上分配空间（无论是合约代码还是数据账户）都要付一定的 SOL，当 SOL 足够多时可一次性付到“免租金”的阈值。

---

# 2. declare_id!("...")：它的意义与配置

1. 在 **`lib.rs`**（合约入口）中添加：

   ```rust
   use anchor_lang::prelude::*;

   declare_id!("FndYourOwnProgramId111111111111111111111111111111111");
   ```

   - 告诉 Anchor：**我这份合约的 Program ID 就是这个公钥**。
   - 当你后续 `anchor build` 或 `anchor deploy` 时，会校验是否与部署用的 keypair 公钥匹配。

2. **为什么要先生成并声明 Program ID？**

   - **可控性**：如果每次让系统随机分配 ID，那么你的前端/配置文件得反复更新；固定一个 Program ID 可以省事且可预先写死。
   - **权限管理**：可升级合约需要用该 Program keypair 做升级交易签名；如果你没有那把私钥，就不能升级。
   - **部署一致**：`declare_id!` 中的 ID 要和实际部署用的 keypair 公钥一致，否则在合约调用时会报错。

3. **如果不想手动指定 ID**
   - 在测试或 devnet 环境下，可以让 Anchor 自动生成 Program ID；
   - 会写入到 `.anchor/program-ids.json` 文件；
   - 缺点是：每次都随机生成 ID，不适合正式上线产品场景。

---

# 3. 常见的开发流程

下面是一个**完整的** Solana + Anchor 开发流程示例，帮助你把概念和实操对上号。

## 3.1 生成 Program 的密钥对（离线/本地）

1. 在项目根目录或 `target/deploy` 下，运行：

   ```bash
   solana-keygen new --outfile target/deploy/my_program-keypair.json
   ```

   - 这会在 `target/deploy/` 下生成一个 JSON 文件，其中包含 Program 的公私钥。
   - 这里的密钥对**不是**普通用户的钱包，而是**Program 专用**的 keypair。

2. 查看公钥：
   ```bash
   solana-keygen pubkey target/deploy/my_program-keypair.json
   ```
   - 会输出一串形如：`FndYourOwnProgramId111111111111111111111111111111111`，这就是要用作 Program ID 的公钥。

## 3.2 在 Anchor 项目的 `lib.rs` 中写 `declare_id!`

- 拿到刚才的公钥后，在 `programs/<program_name>/src/lib.rs` 文件里写：

  ```rust
  use anchor_lang::prelude::*;

  declare_id!("FndYourOwnProgramId111111111111111111111111111111111");

  #[program]
  pub mod my_program {
      // ...
  }
  ```

- 这样，编译时会将该 ID 作为“本合约的地址”进行绑定。

## 3.3 编译、部署到链上

1. **配置 Anchor.toml（可选）**

   - 例如，指定 Program Keypair 路径、目标网络（devnet / localnet / mainnet-beta）等。
   - 示例：

     ```toml
     [programs.localnet]
     my_program = "FndYourOwnProgramId111111111111111111111111111111111"

     [registry]
     url = "https://anchor.projectserum.com"

     [provider]
     cluster = "localnet"
     wallet = "~/.config/solana/id.json"
     ```

   - 上述 `wallet` 就是你**付款用**的普通钱包。

2. **编译**：

   ```bash
   anchor build
   ```

   - 生成 `.so` 文件（在 `target/deploy/` 下）。
   - Anchor 会尝试在 `.anchor/program-ids.json` 里记录当前 Program ID。

3. **部署**：
   ```bash
   anchor deploy
   ```
   - Anchor 会读取 `Anchor.toml` 配置，找到你的 Program keypair 和钱包 keypair：
     - **Program keypair** → 确定 Program ID；
     - **钱包 keypair** → 支付“写入合约代码”的费用。
   - 部署成功后，命令行会显示类似：
     ```
     Program Id: FndYourOwnProgramId111111111111111111111111111111111
     ```

## 3.4 调用程序

- 部署完成后，你就能在测试脚本或前端里通过这个 Program ID 调用合约。
- 典型的调用方式：
  - **Anchor TypeScript**：`@project-serum/anchor` 库
  - 或者直接用 `@solana/web3.js` 与 Program 交互。

> **提示**：
>
> - 你也可以用 `anchor test` 跑单元测试，测试会自动编译并在本地测试验证合约功能。

---

# 4. 进一步解读：为什么要这样做？

1. **与以太坊对比**

   - 在以太坊里，合约地址是由部署者的 EOA（外部账户）通过 `CREATE` 计算产生，合约本身没有私钥。
   - 在 Solana 里，我们“给合约”分配一对公私钥，但只是用来**部署和升级**签名，合约日常并不主动发交易。

2. **固定 Program ID 的好处**

   - 可以把它写死在前端 / 配置里，省去反复更新地址的麻烦；
   - 可以**控制升级**：如果合约是可升级的，只能用这把私钥来执行替换代码的操作。

3. **不想手动管理 Program keypair？**
   - **Anchor** 支持自动生成，但在正式环境上，最好自己管理 keypair 以确保可控。

---

# 5. 其他命令行工具

1. `anchor keys list`
   - 列出当前 Anchor 项目中使用的所有 keypair（包括 Program keypair）。
2. `anchor build`
   - 编译合约，生成 `.so` 文件 + 更新 `.anchor/program-ids.json`。
3. `anchor deploy`
   - 部署到指定网络（localnet / devnet / testnet / mainnet-beta），需要你有足够 SOL 支付费用。
4. `solana program deploy <program.so>`
   - 不用 Anchor CLI，直接用 solana CLI 的方式部署；相比 `anchor deploy` 手动步骤更多。

---

# 6. 参考示例与 Q&A

### 6.1 示例：部署一个简单计数器 Program

1. **生成 Program keypair**
   ```bash
   solana-keygen new --outfile target/deploy/counter-keypair.json
   ```
2. **查看公钥，得到 Program ID**
   ```bash
   solana-keygen pubkey target/deploy/counter-keypair.json
   # 假设输出: 1234Counter111111111111111111111111111111111
   ```
3. **在 `programs/counter/src/lib.rs`** 中：

   ```rust
   use anchor_lang::prelude::*;

   declare_id!("1234Counter111111111111111111111111111111111");

   #[program]
   pub mod counter {
       use super::*;

       pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
           Ok(())
       }
   }

   #[derive(Accounts)]
   pub struct Initialize<'info> {
       // ...
   }
   ```

4. **Anchor.toml** (节选)：

   ```toml
   [programs.localnet]
   counter = "1234Counter111111111111111111111111111111111"

   [provider]
   cluster = "localnet"
   wallet = "~/.config/solana/id.json"
   ```

5. **编译 & 部署**：
   ```bash
   anchor build
   anchor deploy
   ```
   - 这会把编译好的 `.so` 文件部署到 localnet，费用由 `~/.config/solana/id.json` 里存的私钥支付。
6. **验证**：
   - 链上出现一个可执行账户，地址即 `1234Counter111111111111111111111111111111111`。
   - 你可以通过 `solana program show <ProgramID>` 命令查看详情。

## 6.2 Q&A

1. **Q**：部署费是 Program keypair 出吗？  
   **A**：不是。真正支付 SOL 的是你配置的**外部钱包**(如 `~/.config/solana/id.json`)。Program keypair 只用于声明和升级权限。

2. **Q**：declare_id!("...") 中的地址若与部署时的 keypair 公钥不同，会怎样？  
   **A**：合约在运行时会出现不匹配错误，导致无法调用合约成功。必须保持一致。

3. **Q**：部署后能否改 Program ID？  
   **A**：不行。Program ID 是固定的，若要更换只能重新部署到新地址。

4. **Q**：可升级合约如何升级？  
   **A**：需要用同一个 Program keypair 的私钥签名“升级交易”，并将新的 `.so` 提交到链上。如果你废弃了那把私钥或设置 `upgrade_authority = None`，则无法再升级。

---

# 7. 总结

1. **Program ID 与 declare_id!**
   - 先本地生成 keypair → 拿公钥写进 `declare_id!` → 部署时指定同一 keypair → 链上就注册此 ID 为你的合约地址。
2. **为什么先生成 ID 再部署**
   - 便于项目管理、前端集成、保持合约地址固定，避免随机变化。
3. **整体流程**
   1. 准备好“Program keypair”
   2. 在 `lib.rs` 中声明 ID
   3. 配置 Anchor.toml
   4. `anchor build` + `anchor deploy`
   5. 用 Program ID 调用合约逻辑
4. **费用由普通钱包承担**
   - Program keypair 仅做“所有权”/“升级签名”之用。
5. **自动/手动**
   - 开发阶段常用自动生成 ID；
   - 生产环境中最好手动固定 ID，方便维护和升级。

通过以上内容，你就能理解并实践 **Solana + Anchor** 从**生成 Program keypair**到**声明 ID**、**编译部署**、**使用 PDA**、以及**管理可升级合约**的完整流程。

---

> **更多参考**
>
> - [Solana Docs](https://docs.solana.com/)
> - [Anchor Book](https://www.anchor-lang.com/)
> - [Solana Cookbook](https://solanacookbook.com/)

希望这篇笔记能帮助你在 Solana 合约开发中快速搭建环境、部署合约、并熟悉 **Program ID** 与 **PDA** 的核心机制！
