# 🚀 Solana 项目快速创建指南

## 📋 前置要求检查

```bash
# 检查必要工具版本
anchor --version    # 应该是 0.30.1 或兼容版本
solana --version    # 应该是 2.2.18 或兼容版本
node --version      # 应该是 16+
yarn --version      # 或 npm
```

---

## 🛠️ 第一步：创建新项目

```bash
# 创建新的Anchor项目
anchor init my-solana-project
cd my-solana-project

# 检查项目结构
ls -la
```

**期望输出**：

```
├── Anchor.toml
├── Cargo.toml
├── app/
├── migrations/
├── programs/
│   └── my-solana-project/
├── tests/
└── package.json
```

---

## ⚙️ 第二步：配置项目依赖

### 2.1 配置根目录 package.json

```bash
# 删除默认的 package.json 内容，替换为：
cat > package.json << 'EOF'
{
  "license": "ISC",
  "scripts": {
    "lint:fix": "prettier */*.js \"*/**/*{.js,.ts}\" -w",
    "lint": "prettier */*.js \"*/**/*{.js,.ts}\" --check",
    "test": "npx ts-mocha -p ./tsconfig.json -t 1000000 tests/**/*.ts"
  },
  "dependencies": {
    "@coral-xyz/anchor": "^0.30.1",
    "dayjs": "^1.11.13"
  },
  "devDependencies": {
    "@types/chai": "^4.3.0",
    "@types/mocha": "^9.1.0",
    "chai": "^4.3.4",
    "mocha": "^9.0.3",
    "prettier": "^2.6.2",
    "ts-mocha": "^10.0.0",
    "typescript": "^4.9.0"
  }
}
EOF
```

### 2.2 创建 tsconfig.json

```bash
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "types": ["mocha", "chai"],
    "typeRoots": ["./node_modules/@types"],
    "lib": ["es6"],
    "module": "commonjs",
    "target": "es6",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "allowJs": true,
    "outDir": "dist",
    "strict": true,
    "moduleResolution": "node",
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "ts-node": {
    "esm": true,
    "experimentalSpecifierResolution": "node"
  }
}
EOF
```

### 2.3 配置程序的 Cargo.toml

```bash
# 编辑 programs/my-solana-project/Cargo.toml
cat > programs/my-solana-project/Cargo.toml << 'EOF'
[package]
name = "my-solana-project"
version = "0.1.0"
description = "Created with Anchor"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]
name = "my_solana_project"

[features]
default = []
cpi = ["no-entrypoint"]
no-entrypoint = []
no-idl = []
no-log-ix-name = []
idl-build = ["anchor-lang/idl-build"]
custom-heap = []
custom-panic = []
anchor-debug = []

[dependencies]
anchor-lang = { version = "0.30.1", features = ["init-if-needed"] }

[dev-dependencies]
tokio = { version = "1.39", features = ["macros", "rt-multi-thread"] }
solana-program-test = "1.17"
solana-sdk = "1.17"
solana-banks-client = "1.17"
EOF
```

---

## 📝 第三步：创建基础程序结构

### 3.1 创建基础的 lib.rs

```bash
cat > programs/my-solana-project/src/lib.rs << 'EOF'
#![allow(unexpected_cfgs)]
use anchor_lang::prelude::*;

// 先使用占位符，稍后会更新
declare_id!("11111111111111111111111111111111");

const ANCHOR_DISCRIMINATOR_SIZE: usize = 8;

#[program]
pub mod my_solana_project {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        msg!("Hello, Solana!");
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_basic() {
        // 基础测试
        assert_eq!(ANCHOR_DISCRIMINATOR_SIZE, 8);
        println!("✅ 基础测试通过！");
    }
}
EOF
```

---

## 🔑 第四步：生成和配置程序 ID

```bash
# 生成程序密钥
anchor keys list

# 输出类似：
# my_solana_project: 4JX8UgLp5wENBAKBanibpLLoZmyn88ScNR9BukfHHTHy

# 复制显示的程序ID，更新 lib.rs 中的 declare_id!
# 例如：
sed -i '' 's/declare_id!("11111111111111111111111111111111");/declare_id!("4JX8UgLp5wENBAKBanibpLLoZmyn88ScNR9BukfHHTHy");/' programs/my-solana-project/src/lib.rs

# 同步程序ID到 Anchor.toml
anchor keys sync
```

---

## 🧪 第五步：安装依赖并测试

```bash
# 安装TypeScript依赖
npm install

# 测试Rust单元测试
cargo test

# 测试Anchor构建
anchor build

# 测试TypeScript集成测试
anchor test
```

**期望结果**：

```bash
# Rust测试
test result: ok. 1 passed; 0 failed

# Anchor构建
Finished `release` profile [optimized] target(s)

# TypeScript测试
✔ Initialize test (1000ms)
1 passing
```

---

## 📁 第六步：创建文档结构

````bash
# 创建文档目录
mkdir -p docs

# 创建基础README
cat > README.md << 'EOF'
# My Solana Project

## 快速开始

```bash
anchor build
anchor test
cargo test
````

## 项目结构

- `programs/` - Solana 程序代码
- `tests/` - TypeScript 集成测试
- `docs/` - 项目文档
  EOF

# 创建基础的 TypeScript 测试文件

cat > tests/my-solana-project.ts << 'EOF'
import \* as anchor from "@coral-xyz/anchor";
import { Program, BN } from "@coral-xyz/anchor";
import { MySolanaProject } from "../target/types/my_solana_project";
import { expect } from "chai";

describe("my-solana-project", () => {
anchor.setProvider(anchor.AnchorProvider.env());
const program = anchor.workspace.MySolanaProject as Program<MySolanaProject>;

it("Is initialized!", async () => {
const tx = await program.methods.initialize().rpc();
console.log("Your transaction signature", tx);
expect(tx).to.be.a('string');
});
});
EOF

````

---

## ✅ 第七步：验证项目设置

```bash
# 最终验证所有功能
echo "🔍 验证项目设置..."

echo "✅ 检查程序构建:"
anchor build

echo "✅ 检查Rust测试:"
cargo test

echo "✅ 检查TypeScript测试:"
anchor test

echo "🎉 项目创建完成！"
````

---

## 📂 最终项目结构

```
my-solana-project/
├── Anchor.toml                 # Anchor配置
├── Cargo.toml                  # Workspace配置
├── package.json                # Node.js依赖
├── tsconfig.json              # TypeScript配置
├── README.md                   # 项目说明
├── docs/                       # 文档目录
├── programs/
│   └── my-solana-project/
│       ├── Cargo.toml         # 程序依赖配置
│       └── src/
│           └── lib.rs         # 主程序代码
├── tests/
│   └── my-solana-project.ts   # TypeScript测试
├── target/                    # 编译输出
└── node_modules/              # Node.js模块
```

---

## 🎯 下一步开发建议

### 添加复杂功能

1. **数据结构**：添加账户数据结构
2. **指令**：实现 CRUD 操作
3. **PDA**：使用程序派生地址
4. **权限控制**：添加访问控制

### 测试策略

1. **单元测试**：使用 `cargo test`
2. **集成测试**：使用 `anchor test`
3. **性能测试**：测试大数据量操作

### 部署准备

1. **Devnet 测试**：`anchor deploy --provider.cluster devnet`
2. **主网准备**：安全审计和优化
3. **前端集成**：创建 Web3 前端

---

## 🔧 故障排除

### 常见问题

**问题 1**：`anchor keys list` 显示错误

```bash
# 解决方案：重新生成密钥
rm -rf target/deploy/*.json
anchor keys list
```

**问题 2**：TypeScript 编译错误

```bash
# 解决方案：重新安装依赖
rm -rf node_modules package-lock.json
npm install
```

**问题 3**：程序 ID 不匹配

```bash
# 解决方案：同步密钥
anchor keys sync
```

**问题 4**：测试环境问题

```bash
# 解决方案：清理并重建
anchor clean
anchor build
anchor test
```
