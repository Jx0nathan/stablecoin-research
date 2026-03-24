# 02 · 重入攻击（Reentrancy Attack）

**类别**：执行流攻击  
**影响协议**：任何在外部调用前未更新状态的合约  
**危险等级**：🔴 严重（The DAO 攻击根本原因，损失 $60M ETH）  
**发现来源**：学习 Maple Finance nonReentrant 实现

---

## 攻击原理

合约在向外部转账时，如果状态还没更新，攻击者可以在回调中重复进入同一函数：

```
受害合约 Vault.withdraw(100 ETH)
    │
    ├─1─► 检查余额：user.balance = 100 ETH ✅
    ├─2─► 外部转账：user.call{value: 100 ETH}()
    │         │
    │         └─► 攻击者的 receive() 被触发 ← 这里是漏洞入口
    │                 │
    │                 └─► 再次调用 Vault.withdraw(100 ETH)
    │                         ├─1─► 检查余额：还是 100 ETH！（状态还没更新）
    │                         ├─2─► 再次转账...
    │                         └─► 循环直到 Vault 被掏空
    │
    └─3─► 更新状态：user.balance = 0（但已经太晚了）
```

---

## 完整攻击合约示例

```solidity
// 攻击者部署的合约
contract Attacker {
    VulnerableVault public vault;
    
    constructor(address _vault) {
        vault = VulnerableVault(_vault);
    }
    
    function attack() external payable {
        vault.deposit{value: 1 ether}();
        vault.withdraw(1 ether);  // 触发重入
    }
    
    // 每次收到 ETH 时自动重入
    receive() external payable {
        if (address(vault).balance >= 1 ether) {
            vault.withdraw(1 ether);  // 重入！
        }
    }
}
```

---

## 防御方案

### 方案 A：CEI 原则（Checks-Effects-Interactions）

最根本的防御，不依赖任何库。

```solidity
// ❌ 危险写法：先 Interaction 再 Effect
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient");  // Check
    (bool success,) = msg.sender.call{value: amount}("");     // Interaction（先转账）
    require(success);
    balances[msg.sender] -= amount;  // Effect（后更新状态，危险！）
}

// ✅ 安全写法：严格遵循 CEI 顺序
function withdraw(uint256 amount) external {
    // 1. Check（检查）
    require(balances[msg.sender] >= amount, "Insufficient");
    
    // 2. Effect（先更新状态）
    balances[msg.sender] -= amount;
    
    // 3. Interaction（最后才外部调用）
    (bool success,) = msg.sender.call{value: amount}("");
    require(success);
}
```

---

### 方案 B：重入锁（Maple Finance 实现）

```solidity
// Maple 的轻量实现，不依赖 OpenZeppelin
uint256 private _locked = 1;

modifier nonReentrant() {
    require(_locked == 1, "P:LOCKED");
    _locked = 2;   // 进入时加锁
    _;             // 执行函数体
    _locked = 1;   // 退出时解锁
}

// ⚡ 为什么用 1/2 而不是 false/true（0/1）？
// Solidity storage 写入成本：
//   从 0 → 非零（cold write）= 20,000 gas
//   从 非零 → 非零（warm write）= 5,000 gas
//
// 用 false/true（0→1→0）：每次锁定 = 20,000 gas（从 0 开始）
// 用 1/2（1→2→1）：每次锁定 = 5,000 gas（始终非零）
// 节省 15,000 gas/次！

function withdraw(uint256 amount) external nonReentrant {
    // 受保护，重入时 _locked == 2，require 直接 revert
}
```

---

### 方案 C：OpenZeppelin ReentrancyGuard

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyVault is ReentrancyGuard {
    // nonReentrant modifier 由 OZ 提供
    function withdraw(uint256 amount) external nonReentrant {
        // 安全
    }
}
```

---

## 高级变体：跨函数重入

单函数有锁，但两个函数共享状态时仍然可能被攻击：

```solidity
// 假设 transfer 和 withdraw 都没有锁
mapping(address => uint256) balances;

function transfer(address to, uint256 amount) external {
    require(balances[msg.sender] >= amount);
    balances[to] += amount;
    balances[msg.sender] -= amount;
}

function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    (bool success,) = msg.sender.call{value: amount}("");  // 触发回调
    // 攻击者在回调中调用 transfer，转移余额给另一个地址
    // 然后 withdraw 完成，但余额已经被转走
    balances[msg.sender] -= amount;
}

// 防御：所有操作共享状态的函数都加锁，或使用全局锁
```

---

## 只读重入（Read-Only Reentrancy）

更隐蔽的变体，攻击 view 函数返回的中间状态：

```solidity
// 场景：Protocol B 使用 Protocol A 的价格 oracle
// 如果 Protocol A 在转账过程中 oracle 返回错误值
// 攻击者可以在此时调用 Protocol B，利用错误的价格套利

// 防御：在 view 函数中也要确保状态一致性
```

---

## 审计检测方法

检查清单：
- [ ] 所有外部调用（`call`、`transfer`、`send`）之前状态是否已更新
- [ ] 是否遵循 CEI 顺序
- [ ] 有 ETH 转账的函数是否有重入锁
- [ ] 两个共享状态的函数之间是否有跨函数重入风险
- [ ] 是否使用了可能触发回调的 ERC-777 token（比 ERC-20 多了 hooks）

---

## 参考
- [The DAO Hack 事件（2016）](https://www.coindesk.com/learn/2016/06/25/understanding-the-dao-attack/)
- [Solidity 官方安全建议](https://docs.soliditylang.org/en/latest/security-considerations.html)
- [OpenZeppelin ReentrancyGuard](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard)
