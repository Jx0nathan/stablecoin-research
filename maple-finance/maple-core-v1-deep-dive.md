# Maple Finance Core V1 — 合约代码深度解析

> 基于 [maple-labs/maple-core](https://github.com/maple-labs/maple-core) 源码（Solidity 0.6.11），分两个视角深度解析：  
> **① 合约编写技巧** — 值得借鉴的工程实践  
> **② 业务逻辑设计** — DeFi 信贷协议的架构决策

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [合约编写维度：值得学习的工程实践](#2-合约编写维度值得学习的工程实践)
3. [业务逻辑维度：信贷协议的设计决策](#3-业务逻辑维度信贷协议的设计决策)
4. [核心合约逐一拆解](#4-核心合约逐一拆解)
5. [V1 vs V2 演进对比](#5-v1-vs-v2-演进对比)
6. [总结：最值得带走的设计思路](#6-总结最值得带走的设计思路)

---

## 1. 整体架构概览

### 合约地图

```
MapleGlobals (全局配置中枢)
    │
    ├── PoolFactory ──► Pool (流动性池)
    │                    ├── LiquidityLocker  (存放 LP 资金)
    │                    ├── StakeLocker      (存放 Staker 的 BPT，作为违约缓冲)
    │                    └── DebtLocker       (持有 LoanFDT，代表对某笔贷款的债权)
    │
    └── LoanFactory ──► Loan (贷款合约)
                          ├── FundingLocker    (存放贷款资金，待借款人提取)
                          └── CollateralLocker (存放借款人抵押品)
```

### 关键角色

| 角色 | 英文 | 职责 |
|------|------|------|
| 治理者 | Governor | 设置全局参数、白名单管理 |
| 资金池委托人 | Pool Delegate | 管理资金池，决策放款目标 |
| 流动性提供者 | LP (Liquidity Provider) | 向资金池存入稳定币，赚取利息 |
| 质押者 | Staker | 质押 BPT（Balancer Pool Token），为池子承担部分违约损失 |
| 借款人 | Borrower | 机构借款方，可以超抵押或低抵押借款 |

---

## 2. 合约编写维度：值得学习的工程实践

### 2.1 Locker 隔离模式 — 资金安全的最佳实践

**问题**：贷款合约需要暂存大量资金（等待借款人提款的资金、抵押品等），如果全放在主合约里，一旦主合约漏洞，所有资金都暴露。

**Maple 的解法**：每个功能对应一个独立的 Locker 合约：

```solidity
// Loan 构造时自动部署两个 Locker
collateralLocker = ICollateralLockerFactory(_clFactory).newLocker(_collateralAsset);
fundingLocker    = IFundingLockerFactory(_flFactory).newLocker(_liquidityAsset);
```

- `FundingLocker`：存放 LP 借给借款人的资金，借款人 drawdown 之前不会流出
- `CollateralLocker`：存放借款人提供的抵押物
- `LiquidityLocker`：存放 Pool 里的闲置资金
- `StakeLocker`：存放质押者的 BPT，违约时优先被销毁补偿

**核心收益**：
- 每个 Locker 只暴露 `pull()` 和 `drain()` 接口，权限最小化
- 攻击面被隔离 — 即便 Loan 合约有 bug，资金在 Locker 里也需要单独的攻击路径
- 审计更容易 — 每个 Locker 逻辑极简，几十行代码

**可以复用的思路**：在你的 CaaS 卡系统里，不同用途的资金（沉淀资金 / 运营资金 / 准备金）可以用类似思路进行账户隔离，哪怕是链下系统也适用。

---

### 2.2 FDT（Funds Distribution Token）— 按持仓比例分发收益

**ERC-2222 标准实现**，这是 Maple 整套收益分发机制的基础。

```solidity
// BasicFDT 核心逻辑（简化）
int256 internal pointsPerShare;  // 每份额对应的收益点数（WAD 精度）
mapping(address => int256) internal pointsCorrection;  // 个人纠偏值

// 有新收益进来时
function _distributeFunds(uint256 value) internal {
    pointsPerShare += value * POINT_MULTIPLIER / totalSupply;
}

// 某账户可提取的收益
function withdrawableFundsOf(address owner) public view returns (uint256) {
    return uint256(
        int256(pointsPerShare * balanceOf(owner))
        + pointsCorrection[owner]
    ) / POINT_MULTIPLIER;
}
```

**为什么这个设计精妙**：

1. **O(1) 分发**：不需要遍历所有持仓者，收益来了只更新一个全局变量 `pointsPerShare`
2. **pointsCorrection 解决「中途加入」问题**：如果 A 持有 100 份额，产生了 10 USDC 收益，之后 B 买入 100 份额 — B 不应该分到这 10 USDC。通过在 mint/transfer 时调整 `pointsCorrection` 实现精确计算
3. **Loan / Pool / StakeLocker 全部继承 FDT**，同一套机制驱动三层收益分发

```solidity
// 转账时纠正历史收益
function _transfer(address from, address to, uint256 value) internal override {
    super._transfer(from, to, value);
    // 防止 to 拿到转账前已经积累的收益
    pointsCorrection[from] += int256(pointsPerShare * value);
    pointsCorrection[to]   -= int256(pointsPerShare * value);
}
```

**可以直接抄**：任何需要「按持仓比例自动分发收益」的合约场景，ERC-2222 + pointsCorrection 是标准答案。

---

### 2.3 状态机模式 — 明确的生命周期管理

```solidity
// Loan 的状态机
enum State { Ready, Active, Matured, Expired, Liquidated }
State public loanState;

// 每个关键函数先校验状态
function drawdown(uint256 amt) external {
    _isValidState(State.Ready);  // 只有 Ready 状态才能 drawdown
    // ...
    loanState = State.Active;
}

function makePayment() external {
    _isValidState(State.Active);  // 只有 Active 状态才能还款
    // ...
}
```

**好处**：
- 防止函数被错误顺序调用（e.g. 还没 drawdown 就 makePayment）
- 状态转换显式、可审计，每次转换都 emit `LoanStateChanged` 事件
- 比 bool 标志位更清晰，测试也更容易 exhaustive

**Pool 也是三状态**：`Initialized → Finalized → Deactivated`，finalize 前必须满足质押要求才能开放存款。

---

### 2.4 Library 模式减少合约体积

Maple V1 把复杂逻辑抽离到 `LoanLib` 和 `PoolLib`：

```solidity
// Loan.sol 中调用 Library
(amountLiquidated, amountRecovered) = LoanLib.liquidateCollateral(
    collateralAsset, address(liquidityAsset), superFactory, collateralLocker
);
```

为什么要这样：
- Solidity 合约有 24KB 大小限制，Library 可以被 `delegatecall` 调用，不计入主合约大小
- 复用性：`LoanLib` 可以被多个 Loan 实例共用，只部署一次
- 测试隔离：可以单独测试 Library 函数

**具体拆分原则**：
- 纯计算逻辑（collateral 计算、payment 计算）→ Library
- 状态变量相关逻辑 → 主合约
- 跨合约交互逻辑（Uniswap 清算）→ Library

---

### 2.5 `immutable` 的合理使用

```solidity
IERC20 public immutable liquidityAsset;
address public immutable borrower;
address public immutable collateralLocker;
uint256 public immutable apr;
uint256 public immutable termDays;
```

**规律**：合约创建时确定、之后绝不变更的变量全部 `immutable`：
- Gas 更省（直接编译进字节码，不走 storage slot）
- 安全性更高（无法被修改，即使 Governor 权限也不行）
- 清晰地表达「这是合约的不变量」

**反例**（可变的）**：**
```solidity
uint256 public paymentsRemaining;  // 每次还款减少
uint256 public principalOwed;      // 随还款变化
```

---

### 2.6 双阶段 Governor 权限转移

```solidity
function setPendingGovernor(address _pendingGovernor) external isGovernor {
    pendingGovernor = _pendingGovernor;
}

function acceptGovernor() external {
    require(msg.sender == pendingGovernor, "MG:NOT_PENDING_GOV");
    governor        = msg.sender;
    pendingGovernor = address(0);
}
```

**防止的攻击**：如果 Governor 直接 `setGovernor(wrongAddress)`，治理权永久丢失。两阶段确保新地址必须主动接受，防止误操作或地址输错。

这是权限转移的标准模式，任何重要权限转移都应该用这个模式。

---

### 2.7 协议级暂停 vs 合约级暂停的分离

```solidity
// MapleGlobals：协议暂停（GlobalAdmin 控制）
bool public protocolPaused;
function setProtocolPause(bool pause) external {
    require(msg.sender == globalAdmin, "MG:NOT_ADMIN");
}

// Loan.sol：合约级暂停（Borrower 或 LoanAdmin 控制）
// 继承 OpenZeppelin Pausable
function pause() external {
    _isValidBorrowerOrLoanAdmin();
    super._pause();
}
```

**两层暂停的意义**：
- 协议级：整体出现安全事件，一键暂停全部交互（类似熔断器）
- 合约级：某笔贷款出现问题，借款人可以暂停这一笔，不影响其他

```solidity
// 每个关键函数都同时检查两层
function drawdown(uint256 amt) external {
    _whenProtocolNotPaused();  // 协议级检查
    _isValidBorrower();
    // 合约级 Pausable 的 whenNotPaused 通过函数签名隐式检查
```

---

### 2.8 `reclaimERC20` — 防止资金被永久锁死

```solidity
function reclaimERC20(address token) external {
    LoanLib.reclaimERC20(token, address(liquidityAsset), _globals(superFactory));
}

// LoanLib 中：
function reclaimERC20(address token, address liquidityAsset, IMapleGlobals globals) external {
    require(msg.sender == globals.governor(), "L:NOT_GOV");
    require(token != liquidityAsset && token != address(0), "L:INVALID_TOKEN");
    IERC20(token).safeTransfer(msg.sender, IERC20(token).balanceOf(address(this)));
}
```

**场景**：有人误转了不相关的 ERC20 到合约地址，或者某个 DeFi 集成产生了非预期的 token 流入。这个函数允许 Governor 取回，但 **明确禁止取回核心资产**（liquidityAsset），防止 Governor 作恶。

---

## 3. 业务逻辑维度：信贷协议的设计决策

### 3.1 「三明治」风险结构 — 损失吸收顺序

这是 Maple V1 最核心的商业设计：

```
LP 损失顺序（从后至前）：
[最后承担] LP（流动性提供者）
[中间承担] Staker（BPT 质押者）
[最先承担] Pool Delegate（管理人名声 + 部分质押）
```

具体流程（违约时）：
1. 借款人抵押品先通过 Uniswap 清算，尽量弥补欠款
2. 清算后的缺口，由 `StakeLocker` 里的 BPT 销毁来补偿
3. 如果 BPT 还不够，剩余损失才按比例分摊给 LP（通过 `PoolFDT` 的亏损分发机制）

```solidity
// Pool._handleDefault() 中
(uint256 bptsBurned,, uint256 liquidityAssetRecoveredFromBurn) = 
    PoolLib.handleDefault(liquidityAsset, stakeLocker, stakeAsset, defaultSuffered);

// 如果 BPT 燃烧还不够
if (defaultSuffered > liquidityAssetRecoveredFromBurn) {
    poolLosses = poolLosses.add(defaultSuffered - liquidityAssetRecoveredFromBurn);
    updateLossesReceived();  // 把损失分摊给 PoolFDT 持有者（LP）
}
```

**为什么这个设计合理**：
- Staker 主动选择承担更高风险换取更高收益（质押 BPT）
- Pool Delegate 为了保护名誉会尽职做信用评估
- LP 承担最低风险，适合风险偏好低的机构资金

---

### 3.2 `Pool Delegate` 模式 — 半去中心化信贷管理

这是 Maple 区别于 Aave 的核心差异。Aave 是纯算法定价，Maple 引入了 **人工信用评估**：

```solidity
// 只有 Governor 认证的 Pool Delegate 才能创建池子
function setPoolDelegateAllowlist(address delegate, bool valid) external isGovernor {
    isValidPoolDelegate[delegate] = valid;
}

// Pool Delegate 对放款有完全决策权
function fundLoan(address loan, address dlFactory, uint256 amt) external {
    _isValidDelegateAndProtocolNotPaused();
    // ...
}
```

**业务逻辑**：
- Pool Delegate 是持牌的信贷机构（类似基金经理）
- 他们负责对借款人做 KYC、尽调、谈判利率
- 借款人提交 Loan 参数（APR、期限、抵押比例），Pool Delegate 决定是否 fundLoan
- Delegate 赚取 `delegateFee`（利息的一定 bps），Staker 赚取 `stakingFee`

**这个模式的缺点**（V2 改进的原因）：
- 依赖 Pool Delegate 的诚信和能力，中心化风险
- Delegate 作恶可以给高风险借款人放款，LP 承担损失
- V2 引入了更多链上强制约束来降低这个风险

---

### 3.3 可插拔计算器模式 — 利率和费用的灵活性

```solidity
// Loan 创建时注入三个计算器合约
address public immutable repaymentCalc;  // 还款计算
address public immutable lateFeeCalc;    // 逾期费计算
address public immutable premiumCalc;    // 提前还款溢价计算

// 使用时动态调用
(uint256 total, uint256 principal, uint256 interest) = 
    IRepaymentCalc(repaymentCalc).getNextPayment(address(this));
```

**好处**：
- 不同借款人可以使用不同的利率模型（固定利率、阶梯利率等）
- 不需要升级 Loan 合约本身，只需要部署新的 Calc 合约
- 已部署的 Loan 不受新利率模型影响（immutable 绑定）

**Governor 管理白名单**：
```solidity
function setCalc(address calc, bool valid) external isGovernor {
    validCalcs[calc] = valid;  // 防止恶意计算器被注入
}
```

这是**策略模式（Strategy Pattern）**在合约中的经典应用。

---

### 3.4 DebtLocker 的「增量 claim」设计

每个 Pool 对每笔 Loan 都有一个对应的 `DebtLocker`。DebtLocker 的 `claim()` 函数使用增量计算：

```solidity
// DebtLocker.claim() 中
uint256 newInterest  = loan.interestPaid()  - lastInterestPaid;   // 增量
uint256 newPrincipal = loan.principalPaid() - lastPrincipalPaid;  // 增量

// 更新 checkpoint
lastInterestPaid  = loan.interestPaid();
lastPrincipalPaid = loan.principalPaid();
```

**为什么不直接读当前值**：
- Loan 可能有多个 Pool 同时资助（虽然 V1 实际上是 1:1，但设计上支持多 lender）
- 如果多个 DebtLocker 都 claim，每个只应拿走自己对应比例的新增收益
- 通过 LoanFDT 持仓比例计算各自分得的金额

---

### 3.5 `intendToWithdraw` + 冷静期 — 防止挤兑

```solidity
function intendToWithdraw() external {
    require(balanceOf(msg.sender) != uint256(0), "P:ZERO_BAL");
    withdrawCooldown[msg.sender] = block.timestamp;
}

function withdraw(uint256 amt) external {
    (uint256 lpCooldownPeriod, uint256 lpWithdrawWindow) = _globals(superFactory).getLpCooldownParams();
    
    // 必须在冷静期结束后，且在提款窗口内
    require(
        (block.timestamp - (withdrawCooldown[msg.sender] + lpCooldownPeriod)) <= lpWithdrawWindow,
        "P:WITHDRAW_NOT_ALLOWED"
    );
```

**机制**：LP 必须先申请提款，等待 `lpCooldownPeriod`（默认 10 天），然后在 `lpWithdrawWindow`（默认 2 天）内完成提款。

**业务逻辑**：
- 防止所有 LP 同时提款导致资金池枯竭（贷款资金已经放出去了）
- Pool Delegate 有时间提前知道大额提款，可以提前要求借款人还款
- 如果超过提款窗口没有操作，意向自动失效

---

### 3.6 `minLoanEquity` — 防止利益冲突的清算触发机制

```solidity
function canTriggerDefault(...) external view returns (bool) {
    bool pastDefaultGracePeriod = block.timestamp > nextPaymentDue.add(defaultGracePeriod);
    
    // 触发清算的账户必须持有足够比例的 LoanFDT
    return pastDefaultGracePeriod && 
           balance >= ((totalSupply * _globals(superFactory).minLoanEquity()) / 10_000);
}
```

**为什么需要最小持仓要求（默认 20%）**：
- 防止持有少量 LoanFDT 的人恶意触发清算获利（清算价差套利）
- 触发清算的人必须自己也有足够利益在其中，才能触发
- 持有 20% 以上的人大概率是 Pool（通过 DebtLocker 持有），这才是合理的清算触发方

---

### 3.7 `collateralRatio` 动态计算 — 使用预言机而非写死价格

```solidity
function collateralRequiredForDrawdown(uint256 amt) public view returns (uint256) {
    uint256 liquidityAssetPrice = globals.getLatestPrice(address(liquidityAsset));  // Chainlink
    uint256 collateralPrice     = globals.getLatestPrice(address(collateralAsset)); // Chainlink

    uint256 collateralRequiredUSD = wad.mul(liquidityAssetPrice).mul(collateralRatio).div(10_000);
    uint256 collateralRequiredWAD = collateralRequiredUSD.div(collateralPrice);
    
    return collateralRequiredWAD.mul(10 ** collateralAsset.decimals()).div(10 ** 18);
}
```

**关键点**：借款人在 drawdown 时需要根据当前市场价格提交抵押品，不是按创建贷款时的价格。这意味着市场波动会影响所需抵押品数量，而不是使用静态数字。

---

### 3.8 LiquidityCap + allowList — 渐进式开放策略

```solidity
bool public openToPublic;
mapping(address => bool) public allowedLiquidityProviders;

function isDepositAllowed(uint256 depositAmt) public view returns (bool) {
    return (openToPublic || allowedLiquidityProviders[msg.sender]) &&
           _balanceOfLiquidityLocker().add(principalOut).add(depositAmt) <= liquidityCap;
}
```

**业务逻辑**：
- 新池子刚上线时，通常先对 KYC 认证的机构开放（`allowList`）
- 运营稳定后再 `setOpenToPublic(true)` 向所有人开放
- `liquidityCap` 防止池子规模增长过快，超过 Pool Delegate 的管理能力

这是合规优先的产品设计 — 先白名单，后开放。对于 MAS 监管下的 DCS，这个模式可以直接参考。

---

## 4. 核心合约逐一拆解

### 4.1 `MapleGlobals` — 协议的中枢神经

```
职责：
  - 全局参数管理（grace period, fees, cooldown）
  - 白名单管理（assets, factories, delegates, calculators）
  - 预言机注册（asset → Chainlink oracle）
  - 协议暂停开关
  - Uniswap 清算路径配置

权限层级：
  Governor > GlobalAdmin > 普通用户
  Governor：设置参数、白名单
  GlobalAdmin：可暂停协议（紧急情况快速响应）
```

**设计亮点**：Governor 和 GlobalAdmin 是分离的两个角色：
- Governor 掌握参数修改权（高权限，多签）
- GlobalAdmin 只能做紧急暂停（操作快，权限有限）

这个分权设计防止「暂停协议」这个紧急操作受制于多签等待时间。

### 4.2 `Loan` — 贷款的完整生命周期

```
创建 → Ready 状态
  │
  ├── fundingPeriod 内未 drawdown → unwind() → Expired
  │
  └── drawdown() → Active 状态
        │
        ├── makePayment() × N 次
        │
        ├── 最后一次 makePayment() → Matured (贷款正常结束)
        │
        └── 逾期 + 超过 grace period → triggerDefault() → Liquidated
```

**注意**：drawdown 时同时发生：
1. `collateralAsset.safeTransferFrom(borrower, collateralLocker, ...)` — 借款人锁仓抵押品
2. 扣除 `treasuryFee` 和 `investorFee`
3. 剩余资金转给借款人
4. 多余资金（FundingLocker 中 > 借款额 的部分）退回，成为 LP 的 excess 收益

### 4.3 `Pool` — 资金池的多层收益分发

```
deposit()
  → mint PoolFDT
  → 资金进 LiquidityLocker

fundLoan()
  → LiquidityLocker 资金流向 FundingLocker
  → 部署 DebtLocker 持有 LoanFDT

claim()
  → DebtLocker.claim() 取回 LoanFDT 收益
  → 分三路分配：
      poolDelegate  ← delegateFee 比例
      stakeLocker   ← stakingFee 比例
      liquidityLocker ← 剩余（LP 的收益）

withdraw()
  → 冷静期检查
  → burn PoolFDT
  → LiquidityLocker 资金回到 LP
```

### 4.4 `DebtLocker` — 债权凭证的中间层

DebtLocker 是 Pool 对 Loan 持有债权的代理合约：
- 它持有 `LoanFDT`（代表对贷款收益的权利）
- Pool 通过调用 `DebtLocker.claim()` 收回利息和本金
- 违约时，Pool Delegate 调用 `DebtLocker.triggerDefault()`

**为什么需要这一层**？Pool 可能同时资助多笔贷款，每笔对应一个 DebtLocker。DebtLocker 提供了隔离 + 增量 claim 的能力，Pool 不需要直接管理每笔贷款的细节。

---

## 5. V1 vs V2 演进对比

| 维度 | V1 (maple-core) | V2 (pool-v2) |
|------|----------------|--------------|
| 升级机制 | 不可升级，直接部署新版本 | 引入代理模式（MapleProxied） |
| 借款计算 | 分散在 Loan 合约 | 集中在 LoanManager |
| 赎回机制 | 直接 withdraw（冷静期） | 两步：requestRedeem + redeem（按批次处理） |
| 资金流向 | LP → LiquidityLocker → FundingLocker → 借款人 | LP → Pool → LoanManager → 借款人 |
| 抵押品 | 支持超抵押（WBTC/WETH） | 主要做无/低抵押机构贷款 |
| 清算路径 | Uniswap 硬编码 | 更灵活的清算策略 |

---

## 6. 总结：最值得带走的设计思路

### 合约编写层面

| 模式 | 核心价值 | 适用场景 |
|------|----------|----------|
| **Locker 隔离** | 资金安全隔离，最小权限原则 | 任何需要暂存资金的合约 |
| **FDT（ERC-2222）** | O(1) 按持仓比例分收益，无需遍历 | 质押挖矿、利息分发、AUM 分配 |
| **状态机** | 清晰生命周期，防止乱序调用 | 贷款、期权、众筹等有明确阶段的合约 |
| **Library 模式** | 绕过 24KB 限制，逻辑复用 | 大型合约 |
| **immutable** | Gas 优化 + 安全性 | 一次设定绝不变更的变量 |
| **双阶段权限转移** | 防止误操作导致权限丢失 | 所有高权限地址变更 |
| **两层暂停** | 协议级 vs 合约级分离 | 有紧急情况处理需求的协议 |
| **策略模式（Calc）** | 业务逻辑可插拔 | 利率模型、风控规则经常变化的场景 |

### 业务逻辑层面

| 设计 | 核心思路 | 对你的启发 |
|------|----------|------------|
| **三明治风险结构** | 损失按 Staker→LP 顺序吸收 | 稳定币收单系统的风险准备金分层 |
| **Pool Delegate 模式** | 链上合约 + 链下信用评估结合 | 合规场景下不可能完全去信任，要设计好角色权责 |
| **渐进式开放（allowList）** | 白名单先行，验证后开放 | MAS 监管下，新功能先许可名单测试 |
| **冷静期防挤兑** | 大额赎回需要提前申请 | 适用于任何资金池类产品的流动性管理 |
| **minLoanEquity 清算** | 触发清算需要利益相关 | 防止恶意套利的清算机制设计 |
| **可插拔计算器** | 利率模型与贷款逻辑解耦 | 业务参数经常调整时，不要把计算逻辑写死 |

---

> **参考资料**  
> - 源码：https://github.com/maple-labs/maple-core  
> - 审计报告：PeckShield V1.0 / Code Arena April 2021 / Dedaub  
> - ERC-2222 标准：https://github.com/ethereum/EIPs/issues/2222  
> - 上链地址：Etherscan `MapleGlobals` → `0xC234c62c8C09687DFf0d9047e40042cd166F3600`
