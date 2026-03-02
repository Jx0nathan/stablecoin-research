# Franklin Templeton BENJI — 调研笔记

> 调研时间：2026-03-02
> 基金名称：Franklin OnChain U.S. Government Money Fund（FOBXX）
> App：https://www.benji.investments

---

## 一句话定位

**传统资产管理巨头（$1.6 万亿 AUM）最早上链的货币市场基金，区块链仅作为「股份登记册」而非结算层，监管资质最完整。**

---

## 产品概况

- **底层资产：** 美国政府债券、国库券、回购协议
- **Token：** BENJI（1 BENJI = 1 基金份额）
- **链：** Stellar（主链，最早）、Polygon（2023 年扩展）、Arbitrum（2024）、Base（2024）
- **最低投资：** $1（面向零售用户，无门槛）
- **管理费：** 0.20% / 年
- **AUM：** $7 亿+（2024 年，全球最大链上货币市场基金之一）
- **上线时间：** 2021 年（最早上链的传统资管机构）

---

## 核心创新：区块链作为股份登记册

Franklin Templeton 的设计思路与 Ondo 完全不同：

```
传统基金份额登记（Transfer Agent）
    ↓ 替换为
区块链上的 Token（BENJI）
    ↓
SEC 批准：区块链记录具有法律效力
```

BENJI 不是「表示基金份额的 Token」，BENJI **就是**基金份额本身，区块链就是官方股份登记册（SEC 批准）。

这是整个 RWA 行业最干净的法律结构——没有 SPV、没有复杂的债务凭证，链上 Token 直接等于法律上的基金份额。

---

## 与 Ondo OUSG 的本质区别

- OUSG → LP 份额的「链上表示」，底层还是传统基金登记系统
- BENJI → 区块链**就是**登记系统，份额只存在于链上

---

## 合规策略（最激进的一家）

Franklin Templeton 的策略是正面突破 SEC 监管，而不是用豁免绕开：

- 2021 年获得 SEC 批准，成为**首个使用公链记录基金份额的美国注册基金**
- 不走 Reg D/Reg S 豁免，直接作为注册共同基金面向美国零售用户
- 向 SEC 提交了比特币现货 ETF 申请（也是第一批）
- 在 SEC 开放 crypto 的新政策下获益最大的传统机构之一

---

## 技术架构

```
Franklin Templeton（资产管理 + 直接开发）
  + Stellar（主链，低费用高吞吐）
  + Polygon/Arbitrum/Base（EVM 扩展）
  + Franklin Templeton 自建链上 Transfer Agent
```

**不依赖第三方 Token 化平台（Securitize 等）**，自建整套技术，研发投入大但控制权完整。

---

## Token 特性

- 每日 rebase（类似 USDY）：利息以新 BENJI Token 形式发放
- 可通过 app.benji.investments 直接申购/赎回
- 不支持钱包间自由转账（需要通过 Franklin Templeton 系统）
- 暂不接入 DeFi（与 BlackRock BUIDL 一样，合规考虑）

---

## 选择 Stellar 的原因

- Stellar 交易费极低（约 $0.00001）
- 有内置的 KYC/AML 功能（Stellar Account Flags）
- 2021 年时 EVM 还没有成熟的合规基础设施
- 后来扩展到 Polygon/EVM 才有了 DeFi 可能性

---

## 竞争位置

```
覆盖面：Franklin BENJI（零售，$1 门槛）
                ↕
机构垄断：BlackRock BUIDL（机构，$5M 门槛）
                ↕
DeFi 原生：Ondo USDY（DeFi，无门槛）
```

Franklin 是唯一正面走 SEC 注册、面向美国零售的玩家，这个监管资质是其他竞争对手短期内无法复制的。

---

## 关键启示

1. SEC 正式注册的基金路线，是最干净也最费时的合规路径
2. Stellar 的低费用在 L2 普及后优势已经缩小
3. 自建技术 vs 找 Securitize 是个重要的 build vs buy 决策
4. 不接入 DeFi 是合规选择，不是技术限制——这个取舍在你的产品设计中也需要考虑
