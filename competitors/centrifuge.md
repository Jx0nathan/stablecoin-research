# Centrifuge — 调研笔记

> 调研时间：2026-03-02
> 官网：https://centrifuge.io
> 文档：https://docs.centrifuge.io

---

## 一句话定位

**DeFi 原生的 RWA 协议层，把真实世界的信贷资产（发票、贷款、ABS）Token 化后接入 MakerDAO/Aave/Morpho，$2B+ TVL，Apollo/Janus Henderson 背书。**

---

## 产品概况

- **TVL：** $2B+（RWA 赛道最大之一）
- **客户：** Apollo Global、Janus Henderson、S&P Dow Jones Indices 等机构
- **DeFi 集成：** Sky（MakerDAO）、Aave Horizon、Morpho、Pendle
- **链：** Centrifuge Chain（自有，基于 Substrate）+ Ethereum + 7 条链
- **Token 标准：** ERC-4626 + ERC-7540（异步赎回）
- **审计：** 21+ 次审计

---

## 核心产品：Centrifuge Pool

**资产发行方（Issuer）视角：**

```
真实世界资产
（发票、贷款、ABS、基金份额）
    ↓
创建 Centrifuge Pool
    ↓
资产 Token 化（每笔资产 = 1 个 NFT）
    ↓
NFT 存入 Pool，换取流动性
    ↓
DeFi 用户提供 USDC 流动性
```

**DeFi 用户视角：**

```
存入 USDC
    ↓
获得 Pool Token（Junior / Senior Tranche）
    ↓
赚取比普通 DeFi 更高的固定收益
```

---

## 分级（Tranche）结构

这是 Centrifuge 最重要的技术设计，直接从传统 ABS 借鉴：

```
Senior Tranche（高级）
  → 优先还款，风险最低
  → 收益率：~5-8%
  → 面向机构 / 保守投资者

Junior Tranche（劣后）
  → 最后还款，风险最高（首亏）
  → 收益率：~10-20%
  → 面向高风险偏好 / 发行方自持（信用增强）
```

发行方通常自持 Junior Tranche，作为对 Senior 投资者的信用背书。

---

## ERC-7540：异步赎回的技术解决方案

Centrifuge 主导了 ERC-7540 标准（ERC-4626 的异步扩展），解决了 RWA 的核心流动性问题：

**问题：** RWA 的底层资产不能即时变现（贷款有锁定期、发票有账期），但 DeFi 用户期望随时赎回。

**ERC-7540 方案：**
```
用户发起赎回请求（异步）
    ↓
请求进入队列，等待底层资产到期
    ↓
资产到期后，用户 claim USDC
    ↓
用户等待期间，可以在二级市场出售 pending 状态的 Token
```

这比简单的 T+1 更复杂，但更真实地反映了 RWA 的流动性特征。

---

## 与机构合作模式（Centrifuge Prime）

**Apollo 合作案例：**

Apollo 通过 Centrifuge 把私信贷基金份额 Token 化，接入 DeFi 流动性：

```
Apollo 私信贷基金（链下，$500B AUM）
    ↓ Centrifuge Pool
链上 Token（aToken）
    ↓
Aave Horizon（借贷抵押品）
```

机构提供资产，Centrifuge 提供技术 + DeFi 分发渠道，双方都获益。

---

## 技术架构

```
Centrifuge Chain（Substrate 自有链）
  → 资产 NFT 登记
  → Pool 逻辑 / NAV 计算
  → 合规检查

Centrifuge Liquidity Pools（EVM 侧）
  → ERC-4626 / ERC-7540 接口
  → 接入 DeFi 协议

跨链消息（Wormhole / LayerZero）
  → Centrifuge Chain ↔ Ethereum 双向通信
```

---

## 竞争定位

Centrifuge 和 Ondo 是**完全不同的赛道**：

- Ondo → 标准化、流动性高的公开市场资产（美债、美股）
- Centrifuge → 非标准化、流动性低的私人信贷资产（发票、贷款、ABS）

两者的 DeFi 集成路径也不同：
- Ondo → Token 直接作为抵押品（高流动性）
- Centrifuge → Pool 提供 Senior/Junior Tranche，通过分级承担信用风险

---

## 关键数据（2025 年）

- 处理资产总额：$2B+
- 活跃 Pool 数量：60+
- 最大 Pool：Blocktower（信用基金）、New Silver（房地产过桥贷款）
- 接入 MakerDAO 借贷额度：$2.2 亿（最高时）

---

## 对你的项目的启示

**新加坡场景下，Centrifuge 模式比 Ondo 模式更适合你：**

原因：
- 新加坡有大量中小企业贸易融资需求（发票、信用证）
- SGX 上有大量房地产信托（REITs），可以做 REIT 份额 Token 化
- MAS 对私人信贷有完整的监管框架（Prescribed Capital Market Products）

**具体方向建议：**
- 新加坡 SME 贸易融资 Token 化（发票 → Pool → DeFi 流动性）
- 新加坡 REITs 的链上分级（Senior/Junior Tranche）
- 和 SGX 合作，把 SGX 上市产品的链上接口做出来

这比直接复制 Ondo 的美债路线更有本地差异化，也更契合你的支付/清算基础设施。
