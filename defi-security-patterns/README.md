# DeFi 安全模式与攻击防御手册

> 从真实协议代码中提炼的安全设计模式，每一条都有攻击场景 + 防御实现 + 协议案例

---

## 目录

1. [份额价格操控攻击（Share Price Inflation Attack）](#1-份额价格操控攻击)
2. [重入攻击（Reentrancy Attack）](#2-重入攻击)
3. [权限信息泄露（403 vs 404）](#3-权限信息泄露)

> 持续更新中...

---

## 1. 份额价格操控攻击

**类别**：算术攻击 / 精度攻击  
**影响协议**：所有基于 ERC-4626 / LP Token 的 Vault、借贷池、DEX  
**危险等级**：🔴 严重（可导致用户资金全损）

### 攻击原理

当 Vault 刚部署、`totalSupply` 极小时，Solidity 整数除法的取整误差可以被放大成巨额损失。

```
正常情况（totalSupply 足够大）：
  用户存入 1,000,000 USDC
  shares = 1,000,000 / 1.05 = 952,380 shares
  误差 = 0.0000x USDC  ← 可忽略

攻击者操控后（totalSupply = 1）：
  sharePrice = 1,000,000 USDC / 1 share
  用户存入 1,999,999 USDC
  shares = 1,999,999 / 1,000,000 = 1.999... → 取整 = 1 share
  误差 = 999,999 USDC  ← 被攻击者套走！
```

### 完整攻击步骤

```
Step 1：攻击者存入 1 wei
        → 获得 1 share（totalSupply = 1）

Step 2：攻击者直接 transfer 大量 USDC 到 Vault 地址
        （不走 deposit，所以不 mint shares）
        → totalAssets 暴涨，但 totalSupply 还是 1
        → sharePrice = 天价

Step 3：受害者存入 X USDC
        → shares = X / sharePrice，整数除法后 = 极小值（可能为 0 或 1）
        → 受害者损失大量资金

Step 4：攻击者赎回 1 share
        → 拿走池子里一半资产，包括受害者的钱
```

### 防御方案对比

#### 方案 A：Uniswap V2 — 死锁最小流动性

```solidity
uint public constant MINIMUM_LIQUIDITY = 10**3; // 永久锁定 1000 LP token

function mint(address to) external returns (uint liquidity) {
    if (_totalSupply == 0) {
        liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
        
        // 关键：第一个 LP 永久损失 1000 token，锁入 address(0)
        // address(0) 无私钥，永远无法赎回
        _mint(address(0), MINIMUM_LIQUIDITY);
    }
    _mint(to, liquidity);
}
```

**效果**：totalSupply 永远 ≥ 1000，攻击成本大幅提高  
**缺点**：第一个 LP 需要承担 MINIMUM_LIQUIDITY 的实际损失

#### 方案 B：OpenZeppelin ERC4626 v5 — 虚拟份额偏移

```solidity
function _convertToShares(uint256 assets, Math.Rounding rounding)
    internal view virtual returns (uint256)
{
    // 分子加 10^offset，分母加 1
    // 即使 totalSupply = 0，分母也不为 0，sharePrice 不会被无限拉高
    return assets.mulDiv(
        totalSupply() + 10 ** _decimalsOffset(),
        totalAssets() + 1,
        rounding
    );
}

// 子类可以覆盖，设置更大的 offset 来增强保护
function _decimalsOffset() internal view virtual returns (uint8) {
    return 0;  // 默认 0，实际项目建议设为 3-9
}
```

**效果**：纯数学防御，零额外成本，攻击收益被数学限制  
**优势**：不需要实际 burn token，更优雅

#### 方案 C：Maple Finance — BOOTSTRAP_MINT

```solidity
// MaplePool.sol
uint256 public immutable BOOTSTRAP_MINT;

function _mint(uint256 shares_, uint256 assets_, address receiver_, address caller_) internal {
    // 只在第一次 mint 时执行
    if (totalSupply == 0 && BOOTSTRAP_MINT != 0) {
        // 死锁一批 shares 到 address(0)，永久无法赎回
        _mint(address(0), BOOTSTRAP_MINT);
        
        // 用户实际收到的 shares 扣除 BOOTSTRAP_MINT
        shares_ -= BOOTSTRAP_MINT;
        
        emit BootstrapMintPerformed(caller_, receiver_, assets_, shares_, BOOTSTRAP_MINT);
    }
    _mint(receiver_, shares_);
}
```

**效果**：和 Uniswap V2 类似，但 BOOTSTRAP_MINT 数量可在部署时配置  
**灵活性**：Pool 部署者可以根据资产精度设置合适的锁定量

### 三种方案对比

| 方案 | 方式 | 额外成本 | 保护强度 | 适用场景 |
|------|------|---------|---------|---------|
| Uniswap V2 | 死锁 1000 LP | 第一个 LP 损失固定量 | 中 | DEX LP |
| OZ ERC4626 | 数学虚拟偏移 | 零 | 高（可调） | 通用 Vault |
| Maple Bootstrap | 死锁可配置量 | 第一个存款者损失 | 中高 | 机构借贷池 |

### 检测方法

```solidity
// 审计时检查以下条件：
// 1. totalSupply 能否变为 0（全部赎回后）
// 2. 是否有人可以在不走 deposit() 的情况下增加 totalAssets
// 3. convertToShares(assets) 在极端 sharePrice 下是否会取整为 0

// 红旗代码（危险）：
function convertToShares(uint256 assets) public view returns (uint256) {
    if (totalSupply == 0) return assets;  // 如果没有这行 ↓ 的保护，有风险
    return assets * totalSupply / totalAssets();  // 没有偏移保护！
}

// 安全代码：
function convertToShares(uint256 assets) public view returns (uint256) {
    return assets.mulDiv(
        totalSupply() + 1,   // 虚拟偏移
        totalAssets() + 1,   // 虚拟偏移
        Math.Rounding.Floor
    );
}
```

### 参考链接
- [OZ ERC4626 安全说明](https://docs.openzeppelin.com/contracts/4.x/erc4626)
- [Uniswap V2 白皮书](https://uniswap.org/whitepaper.pdf)
- [以太坊 EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)

---

## 2. 重入攻击

**类别**：执行流攻击  
**影响协议**：任何在转账前更新状态的合约  
**危险等级**：🔴 严重（The DAO 攻击的根本原因，损失 $60M）

### 攻击原理

合约在调用外部合约（如 ETH 转账）时，如果状态还没更新，攻击者可以在回调中重复进入：

```
受害合约 Vault.withdraw(100 ETH)
    │
    ├─1─► 检查余额：user.balance = 100 ETH ✅
    ├─2─► 转账：user.call{value: 100 ETH}()
    │         │
    │         └─► 攻击者的 receive() 被触发
    │                 │
    │                 └─► 再次调用 Vault.withdraw(100 ETH)
    │                         │
    │                         ├─1─► 检查余额：还是 100 ETH！（状态没更新）
    │                         ├─2─► 再次转账...
    │                         └─► 循环直到 Vault 被掏空
    │
    └─3─► 更新状态：user.balance = 0（但已经晚了）
```

### 防御方案

#### 方案 A：Checks-Effects-Interactions 模式（CEI）

```solidity
// ❌ 危险写法：先转账再更新状态
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    (bool success,) = msg.sender.call{value: amount}("");  // 先转账
    require(success);
    balances[msg.sender] -= amount;  // 后更新（危险！）
}

// ✅ 安全写法：CEI 原则
function withdraw(uint256 amount) external {
    // Check
    require(balances[msg.sender] >= amount);
    
    // Effect（先更新状态）
    balances[msg.sender] -= amount;
    
    // Interaction（最后才转账）
    (bool success,) = msg.sender.call{value: amount}("");
    require(success);
}
```

#### 方案 B：重入锁（ReentrancyGuard）

```solidity
// Maple 的实现（用 1/2 而非 false/true，节省 gas）
uint256 private _locked = 1;

modifier nonReentrant() {
    require(_locked == 1, "LOCKED");
    _locked = 2;   // 进入：加锁
    _;
    _locked = 1;   // 退出：解锁
}

// 为什么用 1/2 而不是 0/1？
// Solidity 中 storage 从 0→非零 需要 20,000 gas（SSTORE cold write）
// 非零→非零 只需要 5,000 gas
// 所以用 1/2 比 false/true 更省 gas
```

#### 方案 C：OpenZeppelin ReentrancyGuard

```solidity
// 原理相同，推荐直接使用 OZ 库
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyVault is ReentrancyGuard {
    function withdraw(uint256 amount) external nonReentrant {
        // 受保护
    }
}
```

### 高级变体：跨函数重入

```solidity
// 即使单个函数有重入锁，跨函数重入仍然可能：
// 如果 funcA 调用外部合约，攻击者从回调中调用 funcB

// 防御：确保所有共享状态的函数都有重入锁，或使用全局锁
```

---

## 3. 权限信息泄露

**类别**：信息安全 / API 设计  
**影响场景**：后台接口、管理 API、敏感路由  
**危险等级**：🟡 中（辅助攻击，不直接导致资金损失）

### 问题描述

当用户访问无权限的接口时，返回不同的 HTTP 状态码会泄露不同信息：

```
攻击者试探：GET /admin/users

→ 返回 403 Forbidden：
  攻击者知道：「/admin/users 路由存在，只是我没权限」
  攻击者下一步：继续尝试绕过权限（参数注入、令牌伪造等）

→ 返回 404 Not Found：
  攻击者知道：「可能不存在这个路由，也可能存在但我看不到」
  攻击者无从判断：信息泄露最小化
```

### 防御实现（Spring Boot 示例）

```java
// ❌ 危险：暴露路由存在性
@GetMapping("/admin/users")
public ResponseEntity<?> getAdminUsers(Authentication auth) {
    if (!auth.hasRole("ADMIN")) {
        return ResponseEntity.status(403).body("Forbidden");  // 泄露路由存在
    }
    return ResponseEntity.ok(userService.getAllUsers());
}

// ✅ 安全：统一返回 404
@GetMapping("/admin/users")
public ResponseEntity<?> getAdminUsers(Authentication auth) {
    if (!auth.hasRole("ADMIN")) {
        return ResponseEntity.notFound().build();  // 不泄露路由存在性
    }
    return ResponseEntity.ok(userService.getAllUsers());
}
```

### 设计原则

```
对外暴露的 API：
  ├── 公开接口 → 正常返回 403（让用户知道需要登录）
  └── 敏感/管理接口 → 返回 404（不暴露接口存在）

判断标准：
  「如果攻击者知道这个接口存在，会不会增加攻击面？」
  → 是 → 用 404
  → 否 → 403 也可以
```

### 参考
- OWASP API Security Top 10: API3 - Excessive Data Exposure
- MAS TRM 合规要求：敏感接口需做访问控制审计日志

---

## 如何贡献新条目

每个安全模式包含：
1. **攻击类别 + 危险等级**
2. **攻击原理**（含代码示例）
3. **完整攻击步骤**
4. **多种防御方案**（含真实协议代码）
5. **检测方法**（审计时如何发现）
6. **参考链接**

---

*基于 Maple Finance、Uniswap V2、OpenZeppelin 真实源码整理*  
*仓库：https://github.com/Jx0nathan/stablecoin-research*
