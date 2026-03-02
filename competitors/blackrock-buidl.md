# BlackRock BUIDL — 调研笔记

> 调研时间：2026-03-02
> 官方名称：BlackRock USD Institutional Digital Liquidity Fund
> 链接：https://www.blackrock.com/us/individual/products/333205/

---

## 一句话定位

**全球最大资产管理公司亲自下场做链上货币市场基金，用 Securitize 做技术底座，为机构投资者提供「链上美元流动性」。**

---

## 产品概况

- **底层资产：** 短期美国国债、现金及等价物
- **链：** Ethereum（主链）+ 多链扩展（Aptos、Arbitrum、Optimism、Polygon、Avalanche）
- **Token：** BUIDL（ERC-20）
- **最低投资：** $5,000,000（500 万美元）
- **结算：** T+0（即时）
- **目标收益：** 紧跟联邦基金利率（约 4-5%）
- **AUM：** $5 亿+（2024 年 3 月上线，迅速成为最大链上美债基金）

---

## 商业模式

BlackRock 本身不靠 BUIDL 赚多少管理费，战略意图更重要：

- **抢占机构 RWA 基础设施标准**：BUIDL 成为其他 DeFi 协议的「链上美元储备资产」
- **对 Ondo OUSG 形成降维打击**：OUSG 底层持有的就是 BUIDL，BlackRock 相当于是 Ondo 的上游
- **稳定币竞争**：BUIDL 被用作多个合成稳定币（如 Usual 的 USD0）的底层抵押品

---

## 技术架构

**合作伙伴分工：**

```
BlackRock（资产管理 + 品牌）
  + Securitize（Token 化平台 + KYC/AML + 二级市场）
  + BNY Mellon（基金托管）
  + PwC（审计）
  + Anchorage Digital / Coinbase（数字资产托管）
```

**关键设计：BUIDL 的零 gas 转账**

传统 Token 转账需要 gas，BUIDL 做了特殊设计：基金支付 gas 费，持有人转账零成本。这对机构来说很重要，避免了 gas 费的会计处理麻烦。

**每日 rebase：**
- 每天 UTC 15:00 基金将当日利息以新 BUIDL Token 形式分发给持有人
- Token 数量增加，价格始终锚定 $1

---

## 合规结构

- 注册为美国证券法下的共同基金（Registered Investment Company）
- 走 Regulation D（Rule 506(c)），面向 Qualified Purchaser
- 只对通过 Securitize KYC 的白名单地址开放转账
- 与 Securitize ID 系统深度集成，所有 Token 持有地址均经过 AML/KYC 验证

---

## 生态影响力

BUIDL 不只是产品，更是「链上美元标准资产」的竞争：

- Ondo 的 OUSG 把大部分资金配置到 BUIDL
- Usual Protocol 的 USD0 稳定币以 BUIDL 为底层抵押
- Arca Labs、Superstate 等也在竞相模仿
- 多家 DeFi 借贷协议开始接受 BUIDL 作为抵押品

---

## 对标维度（vs Ondo OUSG）

- 资产管理人：BlackRock（全球第一）vs Ondo（初创）
- 门槛：$5M vs $100K（Ondo 更 accessible）
- 链上扩展：BUIDL 已在 6 条链 vs Ondo 在 3 条链
- DeFi 集成：Ondo 更深（Pendle/Morpho）vs BUIDL 主要在稳定币层
- 技术平台：Securitize vs 自研

---

## 关键启示

1. 机构做 RWA 不必自建技术，找 Securitize 这类平台可以快速上线
2. 链上美债市场的竞争已经白热化，差异化要找细分资产（SGS、公司债、房地产）
3. BlackRock 进场意味着这个市场足够大，但也意味着美债 Token 化这个品类基本被垄断
