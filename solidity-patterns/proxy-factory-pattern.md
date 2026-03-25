# Proxy Factory 模式 — 从入门到精通

> 基于 Maple Finance `maple-proxy-factory` 真实源码逐层拆解
> 学完之后，你可以直接复用这套代码做自己的可升级合约工厂

---

## 目录

1. [为什么需要 Proxy Factory](#1-为什么需要-proxy-factory)
2. [核心概念：三个角色](#2-核心概念三个角色)
3. [Proxy 合约：最底层的魔法](#3-proxy-合约最底层的魔法)
4. [ProxyFactory：工厂如何造合约](#4-proxyfactory工厂如何造合约)
5. [_newInstance 完整拆解](#5-_newinstance-完整拆解)
6. [MapleProxyFactory：加上权限控制](#6-mapleproxyfactory加上权限控制)
7. [升级流程：_upgradeInstance](#7-升级流程_upgradeinstance)
8. [CREATE2 与地址预测](#8-create2-与地址预测)
9. [我能直接抄吗？怎么抄？](#9-我能直接抄吗怎么抄)
10. [完整可用的简化版实现](#10-完整可用的简化版实现)

---

## 1. 为什么需要 Proxy Factory

### 场景一：你有一个协议，需要创建很多相同类型的合约

```
Maple 的情况：
  每个 Pool 对应一个 PoolManager
  每个 Loan 是一个独立合约
  可能同时存在 100 个 PoolManager 实例

问题：
  100 个 PoolManager → 每个都 deploy 一次 → 100 份相同的字节码上链
  费用极高，浪费 gas
```

**解决方案：Proxy 模式**

```
只 deploy 一份逻辑代码（Implementation）
每个"实例"只是一个极小的 Proxy 合约（约 50 行）
Proxy 把所有调用转发给同一份逻辑代码
每个 Proxy 有自己独立的存储（状态变量）
```

### 场景二：你需要升级合约逻辑，但又不能改合约地址

```
传统合约（不可升级）：
  发现 bug → 只能 deploy 新合约 → 用户需要迁移到新地址 → 痛苦

Proxy 模式：
  发现 bug → deploy 新的 Implementation → 把 Proxy 指向新地址
  用户地址不变，逻辑已更新
```

### 场景三：你需要工厂来统一管理所有实例

```
没有 Factory：
  任何人都能 deploy 假冒的 PoolManager
  LoanManager 无法验证"这是不是正规的 PoolManager"

有 Factory + 白名单：
  isInstance[address] = true  ← 只有 Factory 创建的才在这里
  其他合约可以调用 factory.isInstance(addr) 验证合法性
```

---

## 2. 核心概念：三个角色

```
┌─────────────────────────────────────────────────────┐
│                   ProxyFactory                       │
│  - 注册 Implementation 版本                          │
│  - 创建 Proxy 实例                                   │
│  - 管理升级路径                                       │
│  - 维护 isInstance 白名单                            │
└─────────────────┬───────────────────────────────────┘
                  │ 创建
        ┌─────────▼──────────┐
        │    Proxy 合约       │   ← 用户实际交互的地址（永远不变）
        │  - 存储所有状态      │
        │  - 无任何业务逻辑    │
        │  - 所有调用 delegatecall → Implementation
        └─────────┬──────────┘
                  │ delegatecall
        ┌─────────▼──────────┐
        │  Implementation    │   ← 只有逻辑代码，无状态
        │  (PoolManager v1)  │
        │  - 包含所有函数     │
        │  - 可以被替换升级   │
        └────────────────────┘
```

**关键点：状态在 Proxy，逻辑在 Implementation**

`delegatecall` 的魔法：在 Implementation 的代码中，`msg.sender`、`address(this)` 以及所有存储读写，全都作用在 **Proxy** 上，不是 Implementation 上。

---

## 3. Proxy 合约：最底层的魔法

```solidity
// contracts/Proxy.sol（真实源码，精简注释）

contract Proxy is SlotManipulatable {

    // EIP-1967 标准存储槽
    // 用特定的 slot 存地址，防止和 Implementation 的状态变量冲突
    bytes32 private constant FACTORY_SLOT =
        bytes32(0x7a45a402e4cb6e08ebc196f20f66d5d30e67285a2a8aa80503fa409e727a4af1);
    // = keccak256('eip1967.proxy.factory') - 1

    bytes32 private constant IMPLEMENTATION_SLOT =
        bytes32(0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc);
    // = keccak256('eip1967.proxy.implementation') - 1

    constructor(address factory_, address implementation_) {
        // 1. 存储 factory 地址
        _setSlotValue(FACTORY_SLOT, bytes32(uint256(uint160(factory_))));

        // 2. 确定 implementation 地址
        //    - 如果传了 implementation → 直接用
        //    - 如果传了 address(0) → 从 factory 获取默认版本
        address implementation = implementation_ == address(0)
            ? IDefaultImplementationBeacon(factory_).defaultImplementation()
            : implementation_;

        require(implementation != address(0));

        _setSlotValue(IMPLEMENTATION_SLOT, bytes32(uint256(uint160(implementation))));
    }

    // 所有调用（除了 constructor）都会走这里
    fallback() payable external virtual {
        // 从存储槽读取 implementation 地址
        bytes32 implementation = _getSlotValue(IMPLEMENTATION_SLOT);

        // 验证 implementation 是一个合约
        require(address(uint160(uint256(implementation))).code.length != uint256(0));

        assembly {
            // 1. 把调用的 calldata 复制到内存
            calldatacopy(0, 0, calldatasize())

            // 2. delegatecall 到 implementation
            //    关键：在 implementation 的代码中，存储读写作用在当前合约（Proxy）上
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)

            // 3. 复制返回数据
            returndatacopy(0, 0, returndatasize())

            // 4. 根据结果决定 return 还是 revert
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

### 为什么用 EIP-1967 的特殊存储槽？

```
问题：Implementation 合约里也有状态变量，它们也占存储槽
     如果 factory/implementation 地址和这些变量用了相同的 slot，就会数据冲突

解决方案：EIP-1967 规定用特定的 keccak256 哈希值作为 slot
          这些 slot 极不可能和正常变量冲突（正常变量 slot 0, 1, 2, 3...）

IMPLEMENTATION_SLOT = keccak256('eip1967.proxy.implementation') - 1
                    = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
```

---

## 4. ProxyFactory：工厂如何造合约

```solidity
// contracts/ProxyFactory.sol（核心数据结构）

abstract contract ProxyFactory {

    // 版本号 → Implementation 地址
    mapping(uint256 => address) internal _implementationOf;
    // { 1 => 0xImpl_v1, 2 => 0xImpl_v2 }

    // Implementation 地址 → 版本号（反向查询）
    mapping(address => uint256) internal _versionOf;
    // { 0xImpl_v1 => 1, 0xImpl_v2 => 2 }

    // 升级路径 migrator：从版本A升级到版本B时，用哪个 migrator 合约
    // 当 fromVersion == toVersion 时，migrator 就是 initializer（初始化器）
    mapping(uint256 => mapping(uint256 => address)) internal _migratorForPath;
    // { 1 => { 1 => 0xInitializer_v1 } }  ← 初始化
    // { 1 => { 2 => 0xMigrator_v1_v2 } }  ← 升级
}
```

### 注册流程（在 createInstance 之前必须先做）

```
Step 1：registerImplementation(version=1, impl=0xImpl_v1, initializer=0xInit_v1)
        → _implementationOf[1] = 0xImpl_v1
        → _versionOf[0xImpl_v1] = 1
        → _migratorForPath[1][1] = 0xInit_v1  （初始化器）

Step 2：setDefaultVersion(version=1)
        → defaultVersion = 1

Step 3：（可选）enableUpgradePath(from=1, to=2, migrator=0xMig_v1_v2)
        → _migratorForPath[1][2] = 0xMig_v1_v2
        → upgradeEnabledForPath[1][2] = true
```

---

## 5. _newInstance 完整拆解

Maple 实现了两个重载版本的 `_newInstance`：

### 版本 A：使用 CREATE（地址由 nonce 决定）

```solidity
function _newInstance(uint256 version_, bytes memory arguments_)
    internal virtual returns (bool success_, address proxy_)
{
    // 1. 从版本号获取 Implementation 地址
    address implementation = _implementationOf[version_];

    // 如果版本未注册，直接返回失败
    if (implementation == address(0)) return (false, address(0));

    // 2. 用 CREATE 部署新的 Proxy 合约
    //    传入：factory地址（this）+ implementation地址
    proxy_ = address(new Proxy(address(this), implementation));

    // 3. 初始化这个 Proxy
    success_ = _initializeInstance(proxy_, version_, arguments_);
}
```

### 版本 B：使用 CREATE2（地址由 salt 决定，可预测）⭐ Maple 实际使用的

```solidity
function _newInstance(bytes memory arguments_, bytes32 salt_)
    internal virtual returns (bool success_, address proxy_)
{
    // 1. 用 CREATE2 部署 Proxy
    //    注意：传入 implementation = address(0)，让 Proxy 从 factory 获取默认版本
    proxy_ = address(new Proxy{ salt: salt_ }(address(this), address(0)));

    // 2. 从 Proxy 查询它实际用了哪个 implementation
    ( , address implementation ) = _getImplementationOfProxy(proxy_);

    // 3. 查询这个 implementation 的版本号
    uint256 version = _versionOf[implementation];

    // 4. 版本必须非零（已注册）且初始化成功
    success_ = (version != uint256(0)) && _initializeInstance(proxy_, version, arguments_);
}
```

### _initializeInstance：初始化过程

```solidity
function _initializeInstance(address proxy_, uint256 version_, bytes memory arguments_)
    private returns (bool success_)
{
    // 获取初始化器（fromVersion == toVersion 的 migrator）
    address initializer = _migratorForPath[version_][version_];

    // 如果没有初始化器：只要没有传入参数，就算成功
    if (initializer == address(0)) return arguments_.length == uint256(0);

    // 调用 Proxy 的 migrate 函数，Proxy 会 delegatecall 到 initializer
    // 初始化参数在 arguments_ 里
    ( success_, ) = proxy_.call(
        abi.encodeWithSelector(IProxied.migrate.selector, initializer, arguments_)
    );
}
```

### _getImplementationOfProxy：查询 Proxy 当前的 implementation

```solidity
function _getImplementationOfProxy(address proxy_)
    private view returns (bool success_, address implementation_)
{
    bytes memory returnData;
    // staticcall 调用 proxy 的 implementation() 函数
    // Proxy 会从 IMPLEMENTATION_SLOT 读取地址返回
    ( success_, returnData ) = proxy_.staticcall(
        abi.encodeWithSelector(IProxied.implementation.selector)
    );
    implementation_ = abi.decode(returnData, (address));
}
```

### 完整流程图

```
MapleProxyFactory.createInstance(arguments_, salt_)
    │
    ├─1─► keccak256(arguments_ + salt_) → 最终 salt
    │
    ├─2─► _newInstance(arguments_, finalSalt)
    │         │
    │         ├─a─► new Proxy{salt: finalSalt}(factory=this, impl=address(0))
    │         │         │
    │         │         └─► Proxy constructor:
    │         │               - 存 factory 地址到 FACTORY_SLOT
    │         │               - 调用 factory.defaultImplementation() 获取 impl
    │         │               - 存 impl 地址到 IMPLEMENTATION_SLOT
    │         │
    │         ├─b─► _getImplementationOfProxy(proxy)
    │         │         └─► staticcall proxy.implementation()
    │         │             返回：0xImpl_v1
    │         │
    │         ├─c─► _versionOf[0xImpl_v1] = 1（版本号）
    │         │
    │         └─d─► _initializeInstance(proxy, version=1, arguments_)
    │                   └─► proxy.call(migrate(initializer, arguments_))
    │                           └─► Proxy delegatecall → Initializer
    │                               （在 Proxy 的存储空间里初始化状态变量）
    │
    ├─3─► isInstance[proxy] = true  ← 登记白名单
    │
    └─4─► emit InstanceDeployed(version, proxy, arguments_)
```

---

## 6. MapleProxyFactory：加上权限控制

`MapleProxyFactory` 继承 `ProxyFactory`，增加了：

```solidity
contract MapleProxyFactory is IMapleProxyFactory, ProxyFactory {

    address public mapleGlobals;   // 全局配置合约
    uint256 public defaultVersion; // 当前默认版本

    mapping(address => bool) public isInstance;  // 实例白名单
    mapping(uint256 => mapping(uint256 => bool)) public upgradeEnabledForPath; // 升级路径白名单

    // 权限修饰符：只有 Governor 能调用
    modifier onlyGovernor() {
        require(msg.sender == IMapleGlobalsLike(mapleGlobals).governor(), "MPF:NOT_GOVERNOR");
        _;
    }

    // 权限修饰符：协议未暂停才能操作
    modifier whenProtocolNotPaused() {
        require(!IMapleGlobalsLike(mapleGlobals).protocolPaused(), "MPF:PROTOCOL_PAUSED");
        _;
    }

    // Governor 才能注册新 Implementation
    function registerImplementation(uint256 version_, address impl_, address initializer_)
        public onlyGovernor { ... }

    // Governor 才能开启升级路径
    function enableUpgradePath(uint256 from_, uint256 to_, address migrator_)
        public onlyGovernor { ... }

    // 任何人都能创建实例（但协议不能暂停）
    function createInstance(bytes calldata arguments_, bytes32 salt_)
        public whenProtocolNotPaused returns (address instance_)
    {
        bool success;
        // 注意：salt 是 keccak256(arguments_ + 用户传入的salt)
        // 这确保不同参数不会产生地址冲突
        ( success, instance_ ) = _newInstance(
            arguments_,
            keccak256(abi.encodePacked(arguments_, salt_))
        );
        require(success, "MPF:CI:FAILED");

        isInstance[instance_] = true;

        emit InstanceDeployed(defaultVersion, instance_, arguments_);
    }
}
```

---

## 7. 升级流程：_upgradeInstance

```solidity
// 实例自己发起升级（Instance 调用 Factory 的这个函数）
function upgradeInstance(uint256 toVersion_, bytes calldata arguments_)
    public whenProtocolNotPaused
{
    // msg.sender 就是要升级的实例（Proxy）
    uint256 fromVersion = _versionOf[IMapleProxied(msg.sender).implementation()];

    // 检查升级路径是否被允许
    require(upgradeEnabledForPath[fromVersion][toVersion_], "MPF:UI:NOT_ALLOWED");

    emit InstanceUpgraded(msg.sender, fromVersion, toVersion_, arguments_);

    require(_upgradeInstance(msg.sender, toVersion_, arguments_), "MPF:UI:FAILED");
}

// 内部升级逻辑
function _upgradeInstance(address proxy_, uint256 toVersion_, bytes memory arguments_)
    internal virtual returns (bool success_)
{
    address toImplementation = _implementationOf[toVersion_];
    if (toImplementation == address(0)) return false;

    // 1. 获取当前 implementation
    ( success_, address fromImplementation ) = _getImplementationOfProxy(proxy_);
    if (!success_) return false;

    // 2. 设置新的 implementation（切换逻辑代码）
    ( success_, ) = proxy_.call(
        abi.encodeWithSelector(IProxied.setImplementation.selector, toImplementation)
    );
    if (!success_) return false;

    // 3. 运行 migrator（数据迁移）
    address migrator = _migratorForPath[_versionOf[fromImplementation]][toVersion_];

    // 如果没有 migrator，只允许无参数升级
    if (migrator == address(0)) return arguments_.length == uint256(0);

    // 用新版本的逻辑，在 Proxy 的存储空间里跑数据迁移
    ( success_, ) = proxy_.call(
        abi.encodeWithSelector(IProxied.migrate.selector, migrator, arguments_)
    );
}
```

### 升级流程图

```
实例（Proxy）调用 factory.upgradeInstance(toVersion=2, args)
    │
    ├─1─► 检查：upgradeEnabledForPath[1][2] == true？
    │
    ├─2─► _upgradeInstance(proxy, toVersion=2, args)
    │         │
    │         ├─a─► 获取当前 impl：0xImpl_v1
    │         │
    │         ├─b─► proxy.setImplementation(0xImpl_v2)
    │         │         └─► Proxy 的 IMPLEMENTATION_SLOT 更新为 0xImpl_v2
    │         │             从此所有调用走新逻辑
    │         │
    │         └─c─► proxy.migrate(migrator_v1_v2, args)
    │                   └─► Proxy delegatecall → Migrator
    │                       在 Proxy 存储里执行数据迁移
    │                       （比如：重命名字段、添加新字段的默认值）
    │
    └─3─► 升级完成，Proxy 地址不变，逻辑已更新
```

---

## 8. CREATE2 与地址预测

```solidity
// 预测某个实例的地址（不需要实际部署）
function getInstanceAddress(bytes calldata arguments_, bytes32 salt_)
    public view returns (address instanceAddress_)
{
    return _getDeterministicProxyAddress(
        keccak256(abi.encodePacked(arguments_, salt_))
    );
}

// CREATE2 地址计算公式（EIP）
function _getDeterministicProxyAddress(bytes32 salt_)
    internal virtual view returns (address)
{
    return address(uint160(uint256(keccak256(abi.encodePacked(
        bytes1(0xff),          // CREATE2 标识符
        address(this),         // 部署者（factory）
        salt_,                 // 盐值
        keccak256(abi.encodePacked(  // Proxy 合约的 initcode hash
            type(Proxy).creationCode,
            abi.encode(address(this), address(0))
        ))
    )))));
}
```

**CREATE2 的价值**：
```
部署前就能知道合约地址
→ 其他合约可以提前引用这个地址
→ 用户可以提前向这个地址充值
→ 测试时可以预测地址，简化 mock
```

---

## 9. 我能直接抄吗？怎么抄？

**可以直接用，但要理解以下几点：**

### ✅ 可以直接复用的部分

```
contracts/Proxy.sol              ← 几乎不用改，直接用
contracts/ProxyFactory.sol       ← 核心逻辑，直接用
contracts/ProxiedInternals.sol   ← Implementation 合约继承这个
contracts/SlotManipulatable.sol  ← 底层工具，直接用
```

### ⚠️ 需要自己实现的部分

```
1. 你自己的 Factory（类似 MapleProxyFactory）
   → 加上你自己的权限控制（不用 MapleGlobals，换成你自己的 owner）
   → 决定谁能 createInstance、谁能 registerImplementation

2. 你自己的 Implementation 合约
   → 继承 ProxiedInternals
   → 实现你的业务逻辑

3. 你自己的 Initializer 合约
   → 每个版本一个
   → 负责初始化 Proxy 的存储变量

4. （可选）Migrator 合约
   → 版本升级时的数据迁移逻辑
```

### ❌ 不要改的部分

```
- IMPLEMENTATION_SLOT 和 FACTORY_SLOT 的值（EIP-1967 标准）
- Proxy.sol 的 fallback 函数（这是核心）
- _initializeInstance 的调用方式
```

---

## 10. 完整可用的简化版实现

如果你不需要 Maple 的权限复杂度，以下是一个**可直接使用的简化版本**：

### 10.1 你的 Implementation（业务合约）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import { ProxiedInternals } from "maple-proxy-factory/contracts/ProxiedInternals.sol";

contract MyPoolManager is ProxiedInternals {

    // 状态变量（存储在 Proxy 里，不在这里）
    address public pool;
    address public owner;
    uint256 public version;

    // 必须实现这两个函数（来自 IProxied 接口）
    function migrate(address migrator_, bytes calldata arguments_) external override {
        require(msg.sender == _factory(), "Not factory");
        require(_migrate(migrator_, arguments_), "Migration failed");
    }

    function setImplementation(address implementation_) external override {
        require(msg.sender == _factory(), "Not factory");
        require(_setImplementation(implementation_), "Set impl failed");
    }

    function implementation() external view override returns (address) {
        return _implementation();
    }

    // 你的业务函数
    function initialize(address pool_, address owner_) external {
        require(pool == address(0), "Already initialized");
        pool  = pool_;
        owner = owner_;
    }
}
```

### 10.2 你的 Initializer

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

// Initializer 是一个无状态合约，逻辑在 Proxy 上下文中运行（delegatecall）
contract MyPoolManagerInitializer {

    // 这里的 pool/owner 引用的是 Proxy 的存储，不是这个合约的
    address public pool;
    address public owner;

    function initialize(bytes calldata arguments_) external {
        ( address pool_, address owner_ ) = abi.decode(arguments_, (address, address));

        pool  = pool_;
        owner = owner_;
    }

    // fallback：当 Proxy delegatecall 过来时，解析 arguments_ 并初始化
    fallback() external {
        ( address pool_, address owner_ ) = abi.decode(msg.data, (address, address));
        pool  = pool_;
        owner = owner_;
    }
}
```

### 10.3 你的 Factory

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import { ProxyFactory } from "maple-proxy-factory/contracts/ProxyFactory.sol";

contract MyFactory is ProxyFactory {

    address public owner;
    uint256 public defaultVersion;
    mapping(address => bool) public isInstance;

    constructor(address owner_) {
        owner = owner_;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    // 注册新版本的 Implementation
    function registerImplementation(
        uint256 version_,
        address implementation_,
        address initializer_
    ) external onlyOwner {
        require(_registerImplementation(version_, implementation_), "Failed");
        require(_registerMigrator(version_, version_, initializer_), "Failed");
    }

    // 设置默认版本
    function setDefaultVersion(uint256 version_) external onlyOwner {
        defaultVersion = version_;
    }

    // 开启升级路径
    function enableUpgradePath(
        uint256 from_,
        uint256 to_,
        address migrator_
    ) external onlyOwner {
        require(_registerMigrator(from_, to_, migrator_), "Failed");
    }

    // 返回默认 Implementation（Proxy 构造函数会调用这个）
    function defaultImplementation() external view returns (address) {
        return _implementationOf[defaultVersion];
    }

    // 创建新实例
    function createInstance(bytes calldata arguments_, bytes32 salt_)
        external returns (address instance_)
    {
        bool success;
        ( success, instance_ ) = _newInstance(
            arguments_,
            keccak256(abi.encodePacked(arguments_, salt_))
        );
        require(success, "Create failed");

        isInstance[instance_] = true;
    }

    // 实例升级自己的版本
    function upgradeInstance(uint256 toVersion_, bytes calldata arguments_) external {
        require(isInstance[msg.sender], "Not instance");
        require(_upgradeInstance(msg.sender, toVersion_, arguments_), "Upgrade failed");
    }
}
```

### 10.4 部署和使用流程

```javascript
// 部署顺序
const impl_v1    = await deploy("MyPoolManager");
const initializer = await deploy("MyPoolManagerInitializer");
const factory    = await deploy("MyFactory", [ownerAddress]);

// 注册版本
await factory.registerImplementation(1, impl_v1.address, initializer.address);
await factory.setDefaultVersion(1);

// 创建实例
const args = abi.encode(["address", "address"], [poolAddress, ownerAddress]);
const salt = ethers.utils.randomBytes(32);
const tx   = await factory.createInstance(args, salt);

// 从事件获取新实例地址
const instanceAddress = tx.events[0].args.instance;
const poolManager = await ethers.getContractAt("MyPoolManager", instanceAddress);

// 调用业务函数
await poolManager.initialize(poolAddress, ownerAddress);
console.log(await poolManager.pool()); // → poolAddress

// 升级到 v2
const impl_v2  = await deploy("MyPoolManager_v2");
const migrator = await deploy("MyMigrator_v1_v2");
await factory.registerImplementation(2, impl_v2.address, initializer_v2.address);
await factory.enableUpgradePath(1, 2, migrator.address);

// 由实例自己发起升级
const poolManagerAsSelf = poolManager.connect(instanceSigner);
await poolManagerAsSelf.upgradeToVersion(2, migrationArgs);
```

---

## 关键知识点总结

| 概念 | 一句话说明 |
|------|----------|
| Proxy | 只存数据，所有调用转发给 Implementation |
| Implementation | 只有逻辑，无状态，可以被替换 |
| delegatecall | 借用别人的代码，但在自己的存储空间里运行 |
| EIP-1967 | 规定 implementation/factory 地址存在哪个 slot，防止冲突 |
| CREATE2 | 部署前可以预测合约地址 |
| Initializer | 第一次创建时跑的初始化逻辑 |
| Migrator | 升级时跑的数据迁移逻辑 |
| isInstance | 工厂白名单，验证合约是否由工厂创建 |

---

*基于 [maple-labs/maple-proxy-factory](https://github.com/maple-labs/maple-proxy-factory) 和 [maple-labs/proxy-factory](https://github.com/maple-labs/proxy-factory) 真实源码*
