# 01 · 份额价格操控攻击（Share Price Inflation Attack）

**类别**：算术攻击 / 精度攻击  
**影响协议**：所有基于 ERC-4626 / LP Token 的 Vault、借贷池、DEX  
**危险等级**：🔴 严重（可导致用户资金全损）  
**发现来源**：学习 Maple Finance / OpenZeppelin ERC4626 源码

---

## 攻击原理

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

---

## 完整攻击步骤

```
Step 1：攻击者存入 1 wei（极小金额）
        → 获得 1 share（totalSupply = 1）

Step 2：攻击者直接 transfer 大量 USDC 到 Vault 地址
        （不走 deposit()，绕过 mint shares）
        → totalAssets 暴涨，但 totalSupply 还是 1
        → sharePrice = 天价

Step 3：受害者存入 X USDC
        → shares = X / sharePrice，整数除法后 = 极小值（可能为 0 或 1）
        → 受害者损失大量资金

Step 4：攻击者赎回 1 share
        → 拿走池子里一半资产，包括受害者的钱
```

---

## 防御方案对比

### 方案 A：Uniswap V2 — 死锁最小流动性

```solidity
uint public constant MINIMUM_LIQUIDITY = 10**3; // 永久锁定 1000 LP token

function mint(address to) external returns (uint liquidity) {
    if (_totalSupply == 0) {
        liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
        // 第一次 mint 时，永久锁定 1000 LP token 到 address(0)
        // address(0) 无私钥，永远无法赎回
        _mint(address(0), MINIMUM_LIQUIDITY);
    }
    _mint(to, liquidity);
}
```

**效果**：totalSupply 永远 ≥ 1000，攻击成本大幅提高  
**缺点**：第一个 LP 需要承担 MINIMUM_LIQUIDITY 的实际损失

---

### 方案 B：OpenZeppelin ERC4626 v5 — 虚拟份额偏移

```solidity
function _convertToShares(uint256 assets, Math.Rounding rounding)
    internal view virtual returns (uint256)
{
    // 分子加 10^offset，分母加 1
    // 即使 totalSupply = 0，分母不为 0，sharePrice 不会被无限拉高
    return assets.mulDiv(
        totalSupply() + 10 ** _decimalsOffset(),
        totalAssets() + 1,
        rounding
    );
}

// 子类可以覆盖，设置更大的 offset 来增强保护
// 建议实际项目设为 3-9
function _decimalsOffset() internal view virtual returns (uint8) {
    return 0;
}
```

**效果**：纯数学防御，零额外成本  
**优势**：不需要实际 burn token，更优雅

---

### 方案 C：Maple Finance — BOOTSTRAP_MINT

```solidity
// MaplePool.sol
uint256 public immutable BOOTSTRAP_MINT;  // 部署时配置

function _mint(uint256 shares_, uint256 assets_, address receiver_, address caller_) internal {
    if (totalSupply == 0 && BOOTSTRAP_MINT != 0) {
        // 死锁一批 shares 到 address(0)，永久无法赎回
        _mint(address(0), BOOTSTRAP_MINT);
        // 用户实际收到的 shares 扣除 BOOTSTRAP_MINT
        shares_ -= BOOTSTRAP_MINT;
    }
    _mint(receiver_, shares_);
}
```

**效果**：和 Uniswap V2 类似，BOOTSTRAP_MINT 数量可在部署时灵活配置

---

## 三种方案对比

| 方案 | 方式 | 额外成本 | 保护强度 | 适用场景 |
|------|------|---------|---------|---------|
| Uniswap V2 | 死锁固定量 | 第一个 LP 损失固定量 | 中 | DEX LP |
| OZ ERC4626 | 数学虚拟偏移 | 零 | 高（可调） | 通用 Vault |
| Maple Bootstrap | 死锁可配置量 | 第一个存款者损失 | 中高 | 机构借贷池 |

---

## 审计检测方法

```solidity
// 🚩 红旗代码（危险，没有保护）：
function convertToShares(uint256 assets) public view returns (uint256) {
    if (totalSupply == 0) return assets;
    return assets * totalSupply / totalAssets();  // 无偏移保护！
}

// ✅ 安全代码：
function convertToShares(uint256 assets) public view returns (uint256) {
    return assets.mulDiv(
        totalSupply() + 1,
        totalAssets() + 1,
        Math.Rounding.Floor
    );
}
```

审计时检查清单：
- [ ] `totalSupply` 能否回到 0（全部赎回后）
- [ ] 是否有途径不走 `deposit()` 直接增加 `totalAssets`
- [ ] `convertToShares()` 在极端 sharePrice 下是否会取整为 0
- [ ] 是否有 Bootstrap 保护或虚拟偏移

---

## 参考
- [OZ ERC4626 安全说明](https://docs.openzeppelin.com/contracts/4.x/erc4626)
- [Uniswap V2 白皮书](https://uniswap.org/whitepaper.pdf)
- [EIP-4626 原文](https://eips.ethereum.org/EIPS/eip-4626)
