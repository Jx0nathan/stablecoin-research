# Maple Finance Core V2 — 合约代码深度解析

> 基于真实源码逐行注释，面向想深入理解 DeFi 借贷合约架构的开发者
> 源码仓库：[maple-labs/pool-v2](https://github.com/maple-labs/pool-v2) · [maple-labs/fixed-term-loan-manager](https://github.com/maple-labs/fixed-term-loan-manager)

---

## 目录

1. [整体架构图](#1-整体架构图)
2. [核心合约职责](#2-核心合约职责)
3. [角色系统](#3-角色系统)
4. [流程一：LP 存款（deposit）](#4-流程一lp-存款)
5. [流程二：发放贷款（fund）](#5-流程二发放贷款)
6. [流程三：借款人还款（claim）](#6-流程三借款人还款)
7. [流程四：违约与清算（triggerDefault）](#7-流程四违约与清算)
8. [流程五：LP 赎回（requestRedeem + redeem）](#8-流程五lp-赎回)
9. [关键设计模式深挖](#9-关键设计模式深挖)
10. [Maple V2 vs Aave V3 深度对比](#10-maple-v2-vs-aave-v3-深度对比)
11. [初学者常见疑问](#11-初学者常见疑问)

---

## 1. 整体架构图

### 1.1 合约依赖关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          MapleGlobals（单例）                             │
│   Governor 控制的全局参数中心                                              │
│   - 合约工厂白名单   - 协议费率   - 借款人白名单   - 紧急暂停               │
└────────────────────────────┬────────────────────────────────────────────┘
                              │ globals() 读取参数
          ┌───────────────────┼────────────────────────────┐
          │                   │                            │
          ▼                   ▼                            ▼
┌──────────────────┐ ┌────────────────────┐    ┌─────────────────────┐
│  MaplePool.sol   │ │ MaplePoolManager   │    │   LoanManager.sol   │
│  (ERC-4626 Vault)│◄►│ .sol              │◄──►│   (利息会计核算)     │
│                  │ │                    │    │                     │
│  LP唯一入口       │ │  Delegate控制面板   │    │  issuanceRate       │
│  deposit()       │ │  fund()            │    │  accountedInterest  │
│  requestRedeem() │ │  triggerDefault()  │    │  domainStart/End    │
│  redeem()        │ │  canCall()         │    │  unrealizedLosses   │
└──────────────────┘ └────────┬───────────┘    └─────────────────────┘
                               │
               ┌───────────────┼───────────────┐
               │               │               │
               ▼               ▼               ▼
┌──────────────────┐ ┌──────────────┐ ┌─────────────────┐
│ WithdrawalManager│ │  MapleLoan   │ │PoolDelegateCover│
│  (赎回FIFO队列)  │ │  (单笔贷款)   │ │  (风险缓冲垫)    │
│                  │ │              │ │                 │
│  addShares()     │ │  drawdown()  │ │  Delegate质押   │
│  processExit()   │ │  makePayment │ │  损失时先扣此处  │
└──────────────────┘ └──────────────┘ └─────────────────┘
```

### 1.2 资金流动路径

```
LP 存入 USDC
    │
    ▼
MaplePool (资金池)
    │
    ├──► [出借] ──► MapleLoan 合约 ──► Borrower 提走
    │                   │
    │              还款+利息
    │                   │
    │    ┌──────────────┘
    │    │  _distributeClaimedFunds()
    │    ├──► Pool Delegate wallet  (delegateManagementFee)
    │    ├──► Maple Treasury         (platformManagementFee)
    │    └──► MaplePool              (净利息，LP收益)
    │
    ├──► [外部策略] ──► Aave / Sky 等收益策略
    │
    └──► [赎回] ──► WithdrawalManager ──► LP 取回 USDC
```

---

## 2. 核心合约职责

### 2.1 MaplePool.sol

**定位**：LP 唯一交互的合约，极度精简，复杂逻辑全部委托给 PoolManager。

**实现标准**：ERC-4626 Tokenized Vault

**核心存储变量**（继承自 ERC20）：
```solidity
address public asset;    // 底层资产（USDC 地址）
address public manager;  // PoolManager 地址
uint256 public BOOTSTRAP_MINT;  // 首次铸造防攻击锁定量

// 继承自 ERC20:
mapping(address => uint256) public balanceOf;   // LP shares 余额
uint256 public totalSupply;                      // shares 总量
```

**核心视图函数**（真实源码）：
```solidity
// totalAssets 委托给 PoolManager 计算
function totalAssets() public view returns (uint256 totalAssets_) {
    totalAssets_ = IPoolManagerLike(manager).totalAssets();
}

// 正常情况 shares → assets（不扣除 unrealizedLosses）
function convertToAssets(uint256 shares_) public view returns (uint256 assets_) {
    uint256 totalSupply_ = totalSupply;
    assets_ = totalSupply_ == 0 ? shares_ : (shares_ * totalAssets()) / totalSupply_;
}

// 退出时 shares → assets（扣除 unrealizedLosses，反映实际可拿到的钱）
function convertToExitAssets(uint256 shares_) public view returns (uint256 assets_) {
    uint256 totalSupply_ = totalSupply;
    assets_ = totalSupply_ == 0 ? shares_ : shares_ * (totalAssets() - unrealizedLosses()) / totalSupply_;
}

// unrealizedLosses 也委托给 PoolManager
function unrealizedLosses() public view returns (uint256 unrealizedLosses_) {
    unrealizedLosses_ = IPoolManagerLike(manager).unrealizedLosses();
}
```

**注意**：`convertToAssets` 和 `convertToExitAssets` 的区别很关键：
- `convertToAssets`：用于计算持仓价值（不扣损失）
- `convertToExitAssets`：用于计算实际可赎回金额（扣除未实现损失）

---

### 2.2 MaplePoolManager.sol

**定位**：Pool 的大脑，持有所有管理逻辑。Pool 和 PoolManager 是 1:1 关系。

**核心存储变量**（来自 MaplePoolManagerStorage）：
```solidity
address public pool;              // 对应的 Pool 合约地址
address public poolDelegate;      // 信用经理地址
address public asset;             // 底层资产
address public withdrawalManager; // 赎回管理器
address public poolDelegateCover; // Delegate 风险垫合约
uint256 public liquidityCap;      // 存款上限
uint256 public delegateManagementFeeRate; // Delegate 管理费率
address[] public strategyList;    // 已接入的策略列表（含 LoanManager）
mapping(address => bool) public isStrategy; // 策略白名单
bool public configured;           // 是否完成初始化配置
uint256 public _locked;           // 重入锁
```

**totalAssets 的真实实现**（非常重要）：
```solidity
function totalAssets() public view returns (uint256 totalAssets_) {
    // 1. Pool 合约中的现金余额（未出借的 USDC）
    totalAssets_ = IERC20Like(asset).balanceOf(pool);

    uint256 length_ = strategyList.length;

    // 2. 遍历所有策略，累加各策略管理的资产
    //    - LoanManager: principalOut + accountedInterest + accruedInterest
    //    - External Strategy (Aave/Sky): 在外部协议中的余额
    for (uint256 i_; i_ < length_;) {
        totalAssets_ += IStrategyLike(strategyList[i_]).assetsUnderManagement();
        unchecked { ++i_; }
    }
}
```

**unrealizedLosses 的真实实现**：
```solidity
function unrealizedLosses() public view returns (uint256 unrealizedLosses_) {
    uint256 length_ = strategyList.length;
    for (uint256 i_; i_ < length_;) {
        unrealizedLosses_ += IStrategyLike(strategyList[i_]).unrealizedLosses();
        unchecked { ++i_; }
    }
}
// LoanManager 的 unrealizedLosses 是当前所有已触发违约但未完成清算的本金+利息之和
```

---

### 2.3 LoanManager.sol（FixedTerm）

**定位**：贷款会计核算引擎，负责计算实时利息，报告给 Pool 的 totalAssets。

**关键存储变量**（来自 LoanManagerStorage）：
```solidity
uint256 public issuanceRate;       // 当前每秒净利息速率（PRECISION = 1e30 精度）
uint112 public accountedInterest;  // 已记账利息（在上次 _advanceGlobal 时固化的）
uint48  public domainStart;        // 上次会计更新的时间戳
uint48  public domainEnd;          // 下次必须更新的时间戳（最近到期贷款日期）
uint128 public principalOut;       // 当前出借中的本金总量
uint128 public unrealizedLosses;   // 已触发违约但未完成清算的损失

// 每笔贷款的支付信息
mapping(uint256 => PaymentInfo) public payments;
mapping(address => uint24) public paymentIdOf;  // loan地址 → paymentId

struct PaymentInfo {
    uint24  platformManagementFeeRate; // 协议管理费率
    uint24  delegateManagementFeeRate; // Delegate 管理费率
    uint48  startDate;                 // 本期开始时间
    uint48  paymentDueDate;            // 本期到期时间
    uint128 incomingNetInterest;       // 本期预计净利息
    uint128 refinanceInterest;         // 再融资利息
    uint256 issuanceRate;              // 本笔贷款的每秒净利息速率
}
```

**实时利息计算（assetsUnderManagement）**：
```solidity
function assetsUnderManagement() public view returns (uint256 value_) {
    value_ = principalOut + accountedInterest + accruedInterest();
}

function accruedInterest() public view returns (uint256 accruedInterest_) {
    uint256 issuanceRate_ = issuanceRate;

    if (issuanceRate_ == 0) return 0;

    // 懒惰计算：issuanceRate × 时间差 / PRECISION
    // PRECISION = 1e30，用来保持精度
    accruedInterest_ = (
        _min(block.timestamp, domainEnd) - domainStart
    ) * issuanceRate_ / PRECISION;
}
```

---

## 4. 流程一：LP 存款

### 完整调用链

```
LP 调用 Pool.deposit(assets_, receiver_)
    │
    ├─1─► checkCall("P:deposit") modifier 触发
    │         └─► PoolManager.canCall("P:deposit", LP地址, calldata)
    │                 └─► PoolPermissionManager.hasPermission(pool, LP, "P:deposit")
    │                         检查：LP 是否在允许名单 / 是否通过 KYC
    │                         返回：(true, "") 或 (false, "错误信息")
    │
    ├─2─► previewDeposit(assets_) 计算 shares
    │         └─► convertToShares(assets_)
    │                 = assets_ * totalSupply / totalAssets()
    │                 （如果 totalSupply == 0，1:1 换算）
    │
    ├─3─► _mint(shares_, assets_, receiver_, msg.sender)
    │         ├─► Bootstrap 检查（首次铸造锁定 BOOTSTRAP_MINT 份额到 address(0)）
    │         ├─► _mint(receiver_, shares_)  — ERC20 内部铸造
    │         ├─► emit Deposit(...)
    │         └─► ERC20Helper.transferFrom(asset, caller_, pool, assets_)
    │                 — 把 LP 的 USDC 划入 Pool
    │
    └─► return shares_
```

### 源码注释（真实代码）

```solidity
// ===== MaplePool.sol =====

function deposit(uint256 assets_, address receiver_)
    external
    override
    nonReentrant               // 重入锁：_locked 从 1 → 2 → 1
    checkCall("P:deposit")     // 权限检查 modifier
    returns (uint256 shares_)
{
    // previewDeposit = convertToShares(assets_)
    // 向下取整（EIP-4626 规定：给用户发 shares 时向下取整）
    _mint(shares_ = previewDeposit(assets_), assets_, receiver_, msg.sender);
}

// _mint 内部实现：
function _mint(uint256 shares_, uint256 assets_, address receiver_, address caller_) internal {
    require(receiver_ != address(0), "P:M:ZERO_RECEIVER");
    require(shares_   != uint256(0), "P:M:ZERO_SHARES");
    require(assets_   != uint256(0), "P:M:ZERO_ASSETS");

    // Bootstrap 保护：第一次 mint 时，锁定 BOOTSTRAP_MINT 到 address(0)
    // 防止「share price 操控攻击」（通过早期大额存款操纵价格）
    if (totalSupply == 0 && BOOTSTRAP_MINT != 0) {
        _mint(address(0), BOOTSTRAP_MINT);   // 死锁 shares
        shares_ -= BOOTSTRAP_MINT;           // 用户少拿这部分
    }

    _mint(receiver_, shares_);  // ERC20 标准铸造

    emit Deposit(caller_, receiver_, assets_, shares_);

    // 把 USDC 从 LP 转入 Pool（Pool 已对 PoolManager approve max）
    require(ERC20Helper.transferFrom(asset, caller_, address(this), assets_), "P:M:TRANSFER_FROM");
}
```

### shares 价格计算详解

```
场景：Pool 当前状态
  totalAssets = 1,050,000 USDC（100万本金 + 5万利息累积）
  totalSupply = 1,000,000 shares

新用户存入 10,500 USDC：
  shares = assets * totalSupply / totalAssets
         = 10,500 * 1,000,000 / 1,050,000
         = 10,000 shares

此时 sharePrice = 1,050,000 / 1,000,000 = 1.05 USDC/share
用户存 10,500 USDC，拿到 10,000 shares（每 share 值 1.05 USDC）
```

### checkCall 权限门控详解

```solidity
// ===== MaplePoolManager.sol =====

function canCall(bytes32 functionId_, address caller_, bytes calldata data_)
    external view returns (bool canCall_, string memory errorMessage_)
{
    // 1. 全局暂停检查
    if (IGlobalsLike(globals()).isFunctionPaused(msg.sig))
        return (false, "PM:CC:PAUSED");

    // 2. 解析 calldata 参数（assets, receiver 等）
    uint256[3] memory params_ = _decodeParameters(data_);

    // ... 针对不同函数做不同处理 ...

    // 3. 委托给 PoolPermissionManager 检查权限
    //    支持多种权限模式：
    //    - PUBLIC：任何人
    //    - ALLOWLIST：白名单
    //    - FUNCTION_LEVEL：按函数级别控制
    //    - POOL_LEVEL：按 Pool 级别控制
    if (!IPoolPermissionManagerLike(poolPermissionManager)
            .hasPermission(address(this), caller_, functionId_)) {
        return (false, "PM:CC:NOT_ALLOWED");
    }

    // 4. 容量检查（存款时检查是否超过 liquidityCap）
    if (functionId_ == "P:deposit" || ...) {
        require(totalAssets() + assets_ <= liquidityCap, "PM:CC:LIQCAP");
    }

    return (true, "");
}
```

---

## 5. 流程二：发放贷款

### 完整调用链

```
Borrower 创建 Loan 合约（通过 MapleLoanFactory）
    ↓
Pool Delegate 调用 LoanManager.fund(loanAddress_)
    │
    ├─1─► 安全验证（四重验证）
    │       ├─ globals_.isInstanceOf("FT_LOAN_FACTORY", factory_)  // 工厂在白名单
    │       ├─ factory_.isLoan(loan_)                               // loan 由白名单工厂创建
    │       ├─ globals_.isBorrower(loan_.borrower())               // 借款人在白名单
    │       └─ loan_.paymentsRemaining() != 0                       // 贷款未失效
    │
    ├─2─► _advanceGlobalPaymentAccounting()  // 先把历史利息算清楚
    │
    ├─3─► _skimFundsFromLoan(loan_)          // 清理 Loan 合约残留资金
    │
    ├─4─► IPoolManagerLike(poolManager).requestFunds(loan_, principal_)
    │       └─► Pool 把 USDC 转给 Loan 合约
    │
    ├─5─► IMapleLoanLike(loan_).fundLoan()   // Loan 合约确认收款
    │
    ├─6─► principalOut += principal_          // 记账：出借本金增加
    │
    └─7─► issuanceRate += _queueNextPayment(...)  // 利息开始计算
```

### 源码注释（真实代码）

```solidity
// ===== LoanManager.sol =====

function fund(address loan_) external override nonReentrant whenNotPaused onlyPoolDelegate {
    ILoanFactoryLike  factory_ = ILoanFactoryLike(IMapleLoanLike(loan_).factory());
    IMapleGlobalsLike globals_ = IMapleGlobalsLike(_globals());

    // ① 四重安全验证
    require(globals_.isInstanceOf("FT_LOAN_FACTORY", address(factory_)), "LM:F:INV_LOAN_FACTORY");
    require(factory_.isLoan(loan_),                                      "LM:F:INV_LOAN_INSTANCE");
    require(globals_.isBorrower(IMapleLoanLike(loan_).borrower()),       "LM:F:INV_BORROWER");
    require(IMapleLoanLike(loan_).paymentsRemaining() != 0,              "LM:F:LOAN_INACTIVE");

    // ② 先把历史利息算清楚（时间推进到当前）
    _advanceGlobalPaymentAccounting();

    uint256 principal_ = IMapleLoanLike(loan_).principalRequested();

    // ③ 清理 Loan 合约中可能存在的残留 USDC（防止会计错误）
    _skimFundsFromLoan(loan_);

    // ④ 通过 PoolManager 把 USDC 转给 Loan 合约
    IPoolManagerLike(poolManager).requestFunds(loan_, principal_);

    // ⑤ Loan 合约内部确认收款，记录贷款开始时间
    IMapleLoanLike(loan_).fundLoan();

    // ⑥ 更新出借本金总量
    emit PrincipalOutUpdated(principalOut += _uint128(principal_));

    // ⑦ 计算并更新利率（新贷款的利息从现在开始计入全局 issuanceRate）
    _updateIssuanceParams(
        issuanceRate + _queueNextPayment(loan_, block.timestamp, IMapleLoanLike(loan_).nextPaymentDueDate()),
        accountedInterest
    );
}
```

### _queueNextPayment 深挖（利率如何计算）

```solidity
function _queueNextPayment(
    address loan_,
    uint256 startDate_,
    uint256 nextPaymentDueDate_
) internal returns (uint256 newRate_) {
    // 1. 获取管理费率
    uint256 platformManagementFeeRate_ = IMapleGlobalsLike(_globals()).platformManagementFeeRate(poolManager);
    uint256 delegateManagementFeeRate_ = IPoolManagerLike(poolManager).delegateManagementFeeRate();
    uint256 managementFeeRate_         = platformManagementFeeRate_ + delegateManagementFeeRate_;

    // 如果合计超过 100%，截断（防止费率配置错误导致溢出）
    if (managementFeeRate_ > HUNDRED_PERCENT) {
        delegateManagementFeeRate_ = HUNDRED_PERCENT - platformManagementFeeRate_;
        managementFeeRate_         = HUNDRED_PERCENT;
    }

    // 2. 从 Loan 合约获取本期还款明细
    //    interest_[0] = 正常利息
    //    interest_[1] = 滞纳金（逾期才有）
    //    interest_[2] = 再融资利息
    ( , uint256[3] memory interest_, ) = IMapleLoanLike(loan_).getNextPaymentDetailedBreakdown();

    // 3. 计算净利息速率（扣除管理费后，LP 实际获得的部分）
    //    _getNetInterest(gross, feeRate) = gross * (1 - feeRate / HUNDRED_PERCENT)
    //    newRate_ 单位：USDC per second * PRECISION
    //    PRECISION = 1e30，避免精度损失
    newRate_ = (_getNetInterest(interest_[0], managementFeeRate_) * PRECISION)
               / (nextPaymentDueDate_ - startDate_);

    // 4. 生成 paymentId，加入排序链表（按到期日排序）
    uint256 paymentId_ = paymentIdOf[loan_] = _addPaymentToList(_uint48(nextPaymentDueDate_));

    // 5. 存储本期支付信息
    payments[paymentId_] = PaymentInfo({
        platformManagementFeeRate: _uint24(platformManagementFeeRate_),
        delegateManagementFeeRate: _uint24(delegateManagementFeeRate_),
        startDate:                 _uint48(startDate_),
        paymentDueDate:            _uint48(nextPaymentDueDate_),
        incomingNetInterest:       _uint128(newRate_ * (nextPaymentDueDate_ - startDate_) / PRECISION),
        refinanceInterest:         _uint128(netRefinanceInterest_),
        issuanceRate:              newRate_
    });
}
```

### requestFunds 详解（资金从 Pool 转出的安全机制）

```solidity
// ===== MaplePoolManager.sol =====

function requestFunds(address destination_, uint256 principal_)
    external override whenNotPaused nonReentrant
{
    // 只有已注册的 Strategy（LoanManager）才能调用
    address factory_ = IMapleProxied(msg.sender).factory();
    require(globals_.isInstanceOf("STRATEGY_FACTORY", factory_), "PM:RF:INVALID_FACTORY");
    require(IMapleProxyFactory(factory_).isInstance(msg.sender), "PM:RF:INVALID_INSTANCE");
    require(isStrategy[msg.sender],                              "PM:RF:NOT_STRATEGY");

    require(IERC20Like(pool_).totalSupply() != 0,           "PM:RF:ZERO_SUPPLY");
    require(_hasSufficientCover(address(globals_), asset_), "PM:RF:INSUFFICIENT_COVER");

    // 关键：先记录当前「锁定流动性」（WithdrawalManager 中待处理的赎回需要的资金）
    uint256 lockedLiquidity_ = IWithdrawalManagerLike(withdrawalManager).lockedLiquidity();

    // 执行资金转移：Pool → Loan 合约
    require(ERC20Helper.transferFrom(asset_, pool_, destination_, principal_), "PM:RF:TRANSFER_FAIL");

    // 确保转账后 Pool 中的现金 >= 已锁定的赎回资金（不能动待赎回的钱）
    require(IERC20Like(asset_).balanceOf(pool_) >= lockedLiquidity_, "PM:RF:LOCKED_LIQUIDITY");
}
```

---

## 6. 流程三：借款人还款

### 完整调用链

```
Borrower 调用 MapleLoan.makePayment(principalToReturn_)
    │
    ├─1─► Loan 内部计算本次应付金额
    │       ├─ principal（本次还本金，通常最后一期才全还）
    │       ├─ interest（本期利息）
    │       └─ fees（service fee 等）
    │
    ├─2─► Loan 把资金转给 LoanManager（Loan 的 lender）
    │
    └─3─► LoanManager.claim(principal_, interest_, prevDueDate_, nextDueDate_)
              │
              ├─a─► _advanceGlobalPaymentAccounting()  // 先结清历史
              │
              ├─b─► _distributeClaimedFunds(loan_, principal_, interest_)
              │         ├─► Pool Delegate ← delegateManagementFee
              │         ├─► Maple Treasury ← platformManagementFee
              │         └─► Pool ← 净利息 + 本金
              │
              ├─c─► if principal_ > 0: principalOut -= principal_
              │
              ├─d─► _handlePreviousPaymentAccounting(loan_)
              │         删除旧的 paymentId，更新全局 issuanceRate
              │
              └─e─► _queueNextPayment(loan_, ...) 或 结束贷款
```

### 源码注释（真实代码）

```solidity
// ===== LoanManager.sol =====

function claim(
    uint256 principal_,
    uint256 interest_,
    uint256 previousPaymentDueDate_,
    uint256 nextPaymentDueDate_
) external override nonReentrant whenNotPaused {
    // msg.sender = Loan 合约地址（只有 Loan 能调用）

    // ① 推进全局会计到当前时间（把此前累积的利息全部记账）
    _advanceGlobalPaymentAccounting();

    // ② 分配资金：本金回 Pool，利息按比例分给三方
    _distributeClaimedFunds(msg.sender, principal_, interest_);

    // ③ 本金减少
    if (principal_ != 0) {
        emit PrincipalOutUpdated(principalOut -= _uint128(principal_));
    }

    // ④ 处理上一期支付的会计（获取该贷款旧的 issuanceRate，准备更新）
    uint256 previousRate_ = _handlePreviousPaymentAccounting(msg.sender);

    // ⑤ 如果没有下一期（贷款还清），直接退出
    if (nextPaymentDueDate_ == 0) {
        delete paymentIdOf[msg.sender];
        _updateIssuanceParams(issuanceRate - previousRate_, accountedInterest);
        return;
    }

    // ⑥ 计算下一期利率的开始时间
    //    - 按时还款：startDate = now（从当前时间算）
    //    - 逾期还款：startDate = previousDueDate（从上次到期日算，不能「赖账」）
    uint256 newRate_ = _queueNextPayment(
        msg.sender,
        _min(block.timestamp, previousPaymentDueDate_),  // 关键：逾期时用过去时间
        nextPaymentDueDate_
    );

    // ⑦ 三种情况的 issuanceRate 更新逻辑：
    if (block.timestamp <= previousPaymentDueDate_) {
        // 提前还款：直接更新利率
        _updateIssuanceParams(issuanceRate + newRate_ - previousRate_, accountedInterest);
        return;
    }

    if (block.timestamp <= nextPaymentDueDate_) {
        // 逾期但未超过下一期：计算逾期期间的利息补充
        _updateIssuanceParams(
            issuanceRate + newRate_,
            accountedInterest + _uint112(
                (block.timestamp - previousPaymentDueDate_) * newRate_ / PRECISION
            )
        );
        return;
    }

    // 逾期超过下一期：全部利息已产生，完整记账
    ( uint256 accountedInterestIncrease_, ) = _accountToEndOfPayment(...);
    _updateIssuanceParams(issuanceRate, accountedInterest + _uint112(accountedInterestIncrease_));
}
```

### _distributeClaimedFunds 详解（三方分账）

```solidity
function _distributeClaimedFunds(address loan_, uint256 principal_, uint256 interest_) internal {
    uint256 paymentId_ = paymentIdOf[loan_];
    require(paymentId_ != 0, "LM:DCF:NOT_LOAN");

    // 平台管理费 = 利息 * platformManagementFeeRate / 1e6
    uint256 platformFee_ = interest_ * payments[paymentId_].platformManagementFeeRate / HUNDRED_PERCENT;

    // Delegate 管理费（只有 Pool 有足够 Cover 时才收）
    // 如果 Cover 不足，Delegate 就收不到管理费——进一步激励 Delegate 维护健康池子
    uint256 delegateFee_ = IPoolManagerLike(poolManager).hasSufficientCover()
        ? interest_ * payments[paymentId_].delegateManagementFeeRate / HUNDRED_PERCENT
        : 0;

    // LP 的净利息 = 总利息 - 平台费 - Delegate 费
    uint256 netInterest_ = interest_ - platformFee_ - delegateFee_;

    // 转账：三路同时出
    require(_transfer(fundsAsset_, _pool(),        principal_ + netInterest_), "LM:DCF:TRANSFER_P");
    require(_transfer(fundsAsset_, poolDelegate(), delegateFee_),              "LM:DCF:TRANSFER_PD");
    require(_transfer(fundsAsset_, _treasury(),    platformFee_),              "LM:DCF:TRANSFER_MT");
}
```

### 数字示例

```
贷款参数：
  本金 = 1,000,000 USDC
  年化利率 = 10%
  期限 = 3 个月（按季还息，到期还本）

每季末还款：
  总利息 = 1,000,000 * 10% / 4 = 25,000 USDC
  platformManagementFeeRate = 10% (of interest)
  delegateManagementFeeRate = 15% (of interest)

  platformFee = 25,000 * 10% = 2,500 USDC → Maple Treasury
  delegateFee = 25,000 * 15% = 3,750 USDC → Pool Delegate
  netInterest = 25,000 - 2,500 - 3,750 = 18,750 USDC → Pool（LP 收益）

  LP 实际年化收益率 = 18,750 * 4 / 1,000,000 = 7.5%
```

---

## 7. 流程四：违约与清算

### 完整调用链

```
借款人逾期超过 gracePeriod

Pool Delegate 调用 PoolManager.triggerDefault(loan_, liquidatorFactory_)
    │
    ├─1─► 验证 liquidatorFactory_ 在 Globals 白名单
    │
    ├─2─► ILoanManagerLike(_getLoanManager(loan_)).triggerDefault(loan_, liquidatorFactory_)
    │         │
    │         ├─a─► _advanceGlobalPaymentAccounting()
    │         │
    │         ├─b─► LiquidationInfo 记录损失明细
    │         │       principal, interest, lateInterest, platformFees
    │         │
    │         ├─c─► unrealizedLosses += principal + netInterest  ← sharePrice 立即下降！
    │         │
    │         └─d─► 如果有抵押品：启动 Liquidator 拍卖
    │                   └─► 无抵押或清算立即完成：直接 return losses
    │
    ├─3─► if (liquidationComplete_):
    │         _handleCover(losses_, platformFees_)
    │             ├─► Cover 优先支付 platformFees → Treasury
    │             ├─► Cover 剩余支付 losses → Pool
    │             └─► 如果 Cover 不足：LP 承担剩余损失
    │
    └─4─► 如果清算未完成（有抵押品需要拍卖）：
              emit CollateralLiquidationTriggered
              等待 finishCollateralLiquidation() 调用
```

### 源码注释（真实代码）

```solidity
// ===== MaplePoolManager.sol =====

function triggerDefault(address loan_, address liquidatorFactory_)
    external override whenNotPaused nonReentrant onlyPoolDelegateOrProtocolAdmins
{
    // ① 验证清算合约工厂合法性
    require(
        IGlobalsLike(globals()).isInstanceOf("LIQUIDATOR_FACTORY", liquidatorFactory_),
        "PM:TD:NOT_FACTORY"
    );

    // ② 委托 LoanManager 执行违约处理
    (
        bool    liquidationComplete_,  // 是否立即清算完成（无抵押或已有足够现金）
        uint256 losses_,               // LP 需要承担的损失
        uint256 platformFees_          // 协议费用
    ) = ILoanManagerLike(_getLoanManager(loan_)).triggerDefault(loan_, liquidatorFactory_);

    // ③ 如果有抵押品需要继续拍卖，先返回（等待 finishCollateralLiquidation）
    if (!liquidationComplete_) {
        emit CollateralLiquidationTriggered(loan_);
        return;
    }

    // ④ 清算完成：处理 Cover 赔付
    _handleCover(losses_, platformFees_);

    emit CollateralLiquidationFinished(loan_, losses_);
}

// _handleCover：Cover 赔付逻辑
function _handleCover(uint256 losses_, uint256 platformFees_) internal {
    // 可动用的 Cover 金额（受 maxCoverLiquidationPercent 限制，防止一次性清光）
    uint256 availableCover_ =
        IERC20Like(asset).balanceOf(poolDelegateCover)
        * IGlobalsLike(globals_).maxCoverLiquidationPercent(address(this))
        / HUNDRED_PERCENT;

    // 优先级：先支付协议费，再补偿 LP 损失
    uint256 toTreasury_ = _min(availableCover_,               platformFees_);
    uint256 toPool_     = _min(availableCover_ - toTreasury_, losses_);

    // Cover → Treasury（平台费）
    if (toTreasury_ != 0) {
        IPoolDelegateCoverLike(poolDelegateCover).moveFunds(
            toTreasury_,
            IGlobalsLike(globals_).mapleTreasury()
        );
    }

    // Cover → Pool（补偿 LP 损失）
    if (toPool_ != 0) {
        IPoolDelegateCoverLike(poolDelegateCover).moveFunds(toPool_, pool);
    }

    // 如果 Cover 不足以覆盖所有损失，剩余由 Pool totalAssets 承担
    // → LP 的 sharePrice 永久下降
}
```

### impairLoan（软违约，不同于 triggerDefault）

Maple 还有一个 `impairLoan()` 函数，比 `triggerDefault` 更温和：

```solidity
function impairLoan(address loan_) external override whenNotPaused onlyPoolDelegateOrGovernor {
    // 用于：借款人出现财务困难但尚未正式违约时
    // 效果：立即将贷款标记为「受损」，unrealizedLosses 增加

    uint256 paymentId_ = paymentIdOf[loan_];
    require(paymentId_ != 0, "LM:IL:NOT_LOAN");

    _advanceGlobalPaymentAccounting();

    // 从利息计算列表中移除（停止新增利息）
    _removePaymentFromList(paymentId_);
    _updateIssuanceParams(issuanceRate - payments[paymentId_].issuanceRate, accountedInterest);

    uint256 principal_ = IMapleLoanLike(loan_).principal();

    // 标记损失：本金 + 应计净利息
    liquidationInfo[loan_] = LiquidationInfo({
        triggeredByGovernor: msg.sender == governor(),
        principal:           _uint128(principal_),
        interest:            _uint120(netInterest_),
        ...
    });

    // unrealizedLosses 增加 → sharePrice 立即反映潜在损失
    emit UnrealizedLossesUpdated(unrealizedLosses += _uint128(principal_ + netInterest_));

    // Loan 合约进入 impaired 状态（宽限期开始倒计时）
    IMapleLoanLike(loan_).impairLoan();
}
```

### 损失传导数字示例

```
贷款本金：1,000,000 USDC
Pool Cover：200,000 USDC
Pool totalAssets（违约前）：5,000,000 USDC
Pool totalSupply：4,500,000 shares

Step 1：triggerDefault() 调用
  unrealizedLosses += 1,000,000
  Pool.totalAssets() 在计算时变为：
    = 真实totalAssets - unrealizedLosses
    = 5,000,000 - 1,000,000 = 4,000,000 USDC
  sharePrice = 4,000,000 / 4,500,000 = 0.889 USDC/share（下降！）

Step 2：清算完成，回收 400,000 USDC（抵押品）
  实际损失 = 1,000,000 - 400,000 = 600,000 USDC

Step 3：_handleCover(600,000, platformFees)
  availableCover = 200,000 * 80% = 160,000（假设 maxCoverLiquidationPercent=80%）
  toPool = 160,000（Cover 赔付给 Pool）

Step 4：最终
  Pool 净损失 = 600,000 - 160,000 = 440,000 USDC
  totalAssets = 5,000,000 - 440,000 = 4,560,000 USDC
  sharePrice = 4,560,000 / 4,500,000 = 1.013 USDC/share（仍高于初始值）
  
  → LP 整体仍有盈利，只是少赚了
```

---

## 8. 流程五：LP 赎回

### 完整调用链（两步走）

```
【第一步】LP 调用 Pool.requestRedeem(shares_, owner_)
    │
    ├─1─► checkCall("P:requestRedeem")  // 权限检查
    │
    ├─2─► _requestRedeem(shares_, owner_) 内部函数
    │         │
    │         ├─a─► IPoolManagerLike(manager).getEscrowParams(owner_, shares_)
    │         │       返回：(escrowShares_, destination_=PoolManager地址)
    │         │
    │         ├─b─► _transfer(owner_, PoolManager地址, escrowShares_)
    │         │       LP 的 shares 转移到 PoolManager 保管（失去控制权）
    │         │
    │         └─c─► IPoolManagerLike(manager).requestRedeem(escrowShares_, owner_, sender_)
    │                   └─► PoolManager.requestRedeem():
    │                         ① approve Pool shares 给 WithdrawalManager
    │                         ② IWithdrawalManagerLike(wm).addShares(shares_, owner_)
    │                            ← shares 正式进入赎回队列
    │
    └─► return escrowedShares_

【等待期】Pool Delegate 调用 WithdrawalManager.processRedemptions(sharesToProcess_)
    │
    ├─► FIFO 队列按顺序处理
    ├─► exchangeRate = (totalAssets - unrealizedLosses) / totalSupply
    └─► 自动模式：直接向 LP 转 USDC

【第二步（手动模式）】LP 调用 Pool.redeem(shares_, receiver_, owner_)
    │
    ├─1─► checkCall("P:redeem")
    │
    ├─2─► IPoolManagerLike(manager).processRedeem(shares_, owner_, sender_)
    │         └─► IWithdrawalManagerLike(wm).processExit(shares_, owner_)
    │               返回：(redeemableShares_, resultingAssets_)
    │
    └─3─► _burn(redeemableShares_, resultingAssets_, receiver_, owner_, caller_)
              ├─► burn shares
              └─► Pool → receiver 转 USDC
```

### 源码注释（真实代码）

```solidity
// ===== MaplePool.sol =====

function requestRedeem(uint256 shares_, address owner_)
    external override nonReentrant checkCall("P:requestRedeem")
    returns (uint256 escrowedShares_)
{
    emit RedemptionRequested(
        owner_,
        shares_,
        escrowedShares_ = _requestRedeem(shares_, owner_)
    );
}

function _requestRedeem(uint256 shares_, address owner_) internal returns (uint256 escrowShares_) {
    address destination_;

    // ① 获取托管参数（escrowShares_ = shares_，destination_ = PoolManager）
    ( escrowShares_, destination_ ) = IPoolManagerLike(manager).getEscrowParams(owner_, shares_);

    // ② 如果调用者不是 owner，检查授权并扣减 allowance
    if (msg.sender != owner_) {
        _decreaseAllowance(owner_, msg.sender, escrowShares_);
    }

    // ③ 把 shares 从 owner 转移到 PoolManager（托管）
    if (escrowShares_ != 0 && destination_ != address(0)) {
        _transfer(owner_, destination_, escrowShares_);
    }

    // ④ 通知 PoolManager 和 WithdrawalManager 创建赎回请求
    IPoolManagerLike(manager).requestRedeem(escrowShares_, owner_, msg.sender);
}

// ===== MaplePoolManager.sol =====
function requestRedeem(uint256 shares_, address owner_, address sender_)
    external override whenNotPaused nonReentrant onlyPool
{
    address pool_ = pool;

    // ① approve shares 给 WithdrawalManager（允许 WM 后续 burn 这些 shares）
    require(ERC20Helper.approve(pool_, withdrawalManager, shares_), "PM:RR:APPROVE_FAIL");

    // ② 调用 WithdrawalManager 添加请求进队列
    IWithdrawalManagerLike(withdrawalManager).addShares(shares_, owner_);

    emit RedeemRequested(owner_, shares_);
}
```

### WithdrawalManager 队列机制详解

```
FIFO 队列状态示意（5个请求）：

requestId:  1     2     3     4     5
owner:     Alice  Bob  Carol  Dave  Eve
shares:    1000  2000   500  3000  1500
status:    ✅    ✅    ⏳    ⏳    ⏳
           已处理 已处理 待处理 待处理 待处理

Delegate 调用 processRedemptions(3000 shares 价值的 USDC):
  - Carol: 需要 500 shares × 1.05 = 525 USDC → 处理 ✅
  - Dave: 需要 3000 shares × 1.05 = 3150 USDC，剩余 2475 USDC
    → 部分赎回：2475/3150 * 3000 = 2357 shares 处理 ✅
    → 余下 643 shares 留在队列

交换汇率公式：
  exchangeRate = (totalAssets - unrealizedLosses) / totalSupply

部分赎回公式（最后一个被处理的请求）：
  redeemableShares = min(
    lockedShares,
    lockedShares * availableAssets / requiredAssets
  )
```

### 为什么不能即时赎回（架构层面）

```
Pool 资金分布示意：

总计 10,000,000 USDC（totalAssets）
  ├── Pool 现金余额：1,500,000 USDC    (15%)  ← 可以直接赎回
  ├── Loan A（出借中）：4,000,000 USDC  (40%)  ← 贷款未到期，无法赎回
  ├── Loan B（出借中）：3,000,000 USDC  (30%)  ← 同上
  └── Aave 策略中：1,500,000 USDC      (15%)  ← 需要赎回操作

问题：如果允许即时赎回，先来的 LP 把 1,500,000 现金取完，
      后来的 LP 面对全是出借中的资产，什么都拿不到。

解决方案：FIFO 排队
  → 公平性：先申请先服务
  → Delegate 控制处理节奏：根据贷款还款情况，逐步处理赎回请求
  → lockedLiquidity 保护：新贷款发放时不能动待处理的赎回资金
```

---

## 9. 关键设计模式深挖

### 9.1 ERC-4626 的 round-up / round-down 规则

```solidity
// deposit：用户给 assets，给他 shares → 向下取整（保护协议）
function previewDeposit(uint256 assets_) public view returns (uint256 shares_) {
    shares_ = convertToShares(assets_);  // round down
    // 原因：宁可少给用户一点 shares，也不能超发
}

// mint：用户要 shares，让他付 assets → 向上取整（保护协议）
function previewMint(uint256 shares_) public view returns (uint256 assets_) {
    uint256 totalSupply_ = totalSupply;
    assets_ = totalSupply_ == 0 ? shares_ : _divRoundUp(shares_ * totalAssets(), totalSupply_);
    // 原因：宁可多收用户一点 assets，也不能让协议亏
}

// _divRoundUp 实现：(a + b - 1) / b
function _divRoundUp(uint256 numerator_, uint256 divisor_) internal pure returns (uint256 result_) {
    result_ = (numerator_ + divisor_ - 1) / divisor_;
}
```

### 9.2 重入锁实现（非 OpenZeppelin）

```solidity
// Maple 自己实现的重入锁，原理完全一样但更轻量
uint256 private _locked = 1;

modifier nonReentrant() {
    require(_locked == 1, "P:LOCKED");
    _locked = 2;     // 进入时设为 2
    _;
    _locked = 1;     // 退出时恢复 1
}

// 为什么用 1/2 而不是 false/true？
// 答：Solidity 中写 0 比写非零值更贵（从 0→1 需要 20000 gas，1→2 只需 5000 gas）
// 所以用 1/2 而不是 0/1，节省 gas
```

### 9.3 MapleProxy 可升级代理模式

```
PoolManager 是可升级的（Maple 自己的代理框架，类似 EIP-1967）：

┌──────────────────────────────────────┐
│  Proxy Contract（永久地址）            │
│  - 存储：所有状态变量                  │
│  - 逻辑：delegatecall → Implementation│
└────────────────┬─────────────────────┘
                 │ delegatecall
┌────────────────▼─────────────────────┐
│  Implementation Contract（可替换）     │
│  - 逻辑：所有函数代码                  │
│  - 无存储（变量都在 Proxy 中）          │
└──────────────────────────────────────┘

升级流程（有时间锁保护）：
  1. poolDelegate 调用 scheduleCall("PM:UPGRADE", ...)
  2. 等待 timelock 延迟期
  3. 在执行窗口内调用 upgrade(newVersion_, arguments_)
  4. Factory 切换 Implementation 指针
```

### 9.4 _advanceGlobalPaymentAccounting 的精妙设计

这是整个协议中最复杂的函数，解决了"多笔贷款、不同到期日、懒惰计算"的会计问题：

```
时间轴示意：

贷款A：|--issuanceRate_A--|  (到期日：T1)
贷款B：|--------issuanceRate_B--------|  (到期日：T2)
贷款C：|---issuanceRate_C---|  (到期日：T3，T1 < T2 < T3)

初始状态：
  domainStart = T0
  domainEnd = T1（最近到期）
  issuanceRate = A + B + C

现在时间 = T_now > T3（所有贷款都过期了但还没结算）

_advanceGlobalPaymentAccounting() 执行：

  Loop 1（domain: T0 → T1）：
    accountedInterest += (T1 - T0) * (A + B + C)
    issuanceRate -= A  // A 到期，从总利率移除
    domainStart = T1, domainEnd = T2

  Loop 2（domain: T1 → T2）：
    accountedInterest += (T2 - T1) * (B + C)
    issuanceRate -= B  // B 到期
    domainStart = T2, domainEnd = T3

  Loop 3（domain: T2 → T3）：
    accountedInterest += (T3 - T2) * C
    issuanceRate -= C  // C 到期
    domainStart = T3, domainEnd = T_now（无更多贷款）

  最后：
    accountedInterest += accruedInterest()  // T3 → T_now 的剩余利息
    domainStart = T_now
```

---

## 10. Maple V2 vs Aave V3 深度对比

| 维度 | Aave V3 | Maple V2 |
|------|---------|----------|
| **设计哲学** | 无许可自动机 | 有许可管理机 |
| **利率模型** | 算法曲线（供需驱动，每块更新） | 人工谈判固定利率，lazily 计算 |
| **利率更新** | 每笔交易 `updateState()` | `_advanceGlobalPaymentAccounting()` 懒惰触发 |
| **清算触发** | `liquidationCall()` 任何人调用 | `triggerDefault()` 仅 Delegate 调用 |
| **清算机制** | Health Factor < 1 自动触发 | Delegate 人工判断 + 抵押品拍卖合约 |
| **借款人准入** | 任何钱包，无 KYC | KYC 白名单 + Globals 注册 |
| **存款凭证** | aToken（rebase，余额实时增加） | ERC-4626 shares（价格增加） |
| **赎回机制** | `withdraw()` 即时（有流动性时） | `requestRedeem()` + FIFO 排队 |
| **每笔贷款** | Reserve mapping（全部在 Pool 合约） | 独立合约（`MapleLoan`） |
| **利息流向** | 直接体现在 aToken rebase | `claim()` 回调 → `_distributeClaimedFunds` |
| **风险分层** | 无（所有存款人共担） | Cover（Delegate质押）→ LP（按比例） |
| **代码复杂度** | 利率曲线数学 + 清算激励 | 会计状态机 + 角色权限 |
| **Gas 效率** | 高（无新合约部署） | 低（每笔贷款 deploy 合约） |

---

## 11. 初学者常见疑问

### Q1：convertToAssets 和 convertToExitAssets 的区别？

```solidity
// 普通换算：不扣除 unrealizedLosses
// 用途：显示持仓价值、一般计算
convertToAssets(shares) = shares * totalAssets / totalSupply

// 退出换算：扣除 unrealizedLosses
// 用途：赎回时计算实际可取金额
convertToExitAssets(shares) = shares * (totalAssets - unrealizedLosses) / totalSupply
```

在违约期间，两者会不同：
- `convertToAssets` 显示"乐观值"（假设违约能全部回收）
- `convertToExitAssets` 显示"保守值"（扣除潜在损失）

赎回时用 `convertToExitAssets`，防止先赎回的 LP 占便宜，让后面的 LP 承担所有损失。

### Q2：PRECISION = 1e30 是什么意思？

```
issuanceRate 的单位是：USDC per second * 1e30

为什么要乘 1e30？
  USDC 精度 = 6 位小数（1 USDC = 1e6 units）
  如果贷款期限 90 天 = 7,776,000 秒
  利息 = 100,000 USDC = 100,000 * 1e6 units

  issuanceRate = 100,000 * 1e6 / 7,776,000 = 12.86 USDC/second

  问题：Solidity 没有浮点数，12.86 会被截断为 12！
  
  解决方案：issuanceRate = 100,000 * 1e6 * 1e30 / 7,776,000
           = 12,860,082,304... （保留所有精度）
           
  使用时除以 1e30：accruedInterest = issuanceRate * seconds / 1e30
```

### Q3：Bootstrap Mint 是什么攻击，怎么防？

```
攻击步骤（如果没有 Bootstrap Mint）：
  1. 攻击者在 Pool 刚创建时，存入 1 wei（1e-18 USDC）
  2. 获得 1 share
  3. 然后直接向 Pool 地址转入大量 USDC（不经过 deposit()）
  4. totalAssets 突然变大，但 totalSupply 还是 1 share
  5. sharePrice = 1e24 USDC/share
  6. 正常用户存入 1000 USDC，convertToShares = 1000 / 1e24 ≈ 0 shares
  7. 用户白交钱，shares 被取整为 0

防御机制（BOOTSTRAP_MINT）：
  第一次 mint 时，强制锁定 BOOTSTRAP_MINT 份额到 address(0)
  即使攻击者控制初始状态，也无法将价格拉到让后续用户 shares = 0 的程度
```

### Q4：为什么 Pool Cover 有 maxCoverLiquidationPercent 限制？

```
Cover 不能全部一次性清光的原因：

假设 Delegate 质押了 500,000 USDC 的 Cover

如果允许 100% 清算：
  - 第一笔违约：清光所有 Cover → Cover = 0
  - 第二笔违约：Cover 为零，LP 直接承担所有损失

设置 maxCoverLiquidationPercent = 50%：
  - 每次最多动用 50% 的当前余额
  - 第一笔违约：最多 250,000
  - 第二笔违约：最多 125,000
  - ...（指数衰减）

这样的好处：
  1. 多笔违约时，每次都有 Cover 缓冲，LP 损失更小
  2. Delegate 有动力补充 Cover（否则管理费收不到）
  3. 防止 Delegate 故意违约套走 Pool 资金后 Cover 空了
```

---

*文档基于真实源码（commit: 2026 Q1），仓库：[maple-labs/pool-v2](https://github.com/maple-labs/pool-v2) / [fixed-term-loan-manager](https://github.com/maple-labs/fixed-term-loan-manager)*
