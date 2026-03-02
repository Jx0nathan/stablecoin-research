# Securitize — 调研笔记

> 调研时间：2026-03-02
> 官网：https://securitize.io
> 定位：RWA Token 化基础设施平台

---

## 一句话定位

**RWA 赛道的「AWS」，不自己发资产，专门为机构提供 Token 化的技术栈 + 合规基础设施，服务 BlackRock、KKR、Hamilton Lane 等头部资管机构。**

---

## 商业模式

B2B SaaS + 服务费，不靠 AUM 管理费：

```
收入来源：
  → Token 化平台服务费（按项目收）
  → KYC/AML 服务费（Securitize ID）
  → 二级市场交易撮合费（Securitize Markets）
  → 托管服务费（Securitize Trust Company）
```

Securitize 的定位是卖铲子，不挖矿。

---

## 产品矩阵

**Securitize Platform（发行平台）**
- Token 发行全流程：合规设计 → 合约部署 → 投资者管理
- 支持多链：Ethereum、Polygon、Avalanche、Aptos 等
- 内置 CAP Table 管理、股东通讯、分红分配

**Securitize ID（合规身份）**
- 通用 KYC/AML 身份凭证，做过一次可跨产品复用
- 与 BlackRock BUIDL、Hamilton Lane、KKR 的 Token 产品共用同一套 KYC
- 相当于链上「合规护照」

**Securitize Markets（二级市场）**
- SEC 注册的另类交易系统（ATS）
- 为 Token 化证券提供合规二级流动性
- 解决 RWA 流动性问题的关键一环

**Securitize Trust Company**
- 持牌信托公司，提供数字资产托管
- 监管背书，机构合规要求必备

---

## 核心客户

- **BlackRock BUIDL** — 最大的合作背书，BUIDL 的整个技术 + KYC 基础设施
- **KKR** — 私募股权基金 Token 化（K-PRIME）
- **Hamilton Lane** — 私募市场基金 Token 化
- **Bain Capital** — 私募基金份额 Token 化
- **Apollo Global** — 通过 Securitize 发行信贷产品

---

## 技术架构特点

**DS Token 协议（Digital Security Token）**

Securitize 自研的合规 Token 标准，ERC-20 超集：

```solidity
// 核心合规功能
interface IDSToken {
    function transfer(...) → 检查双方合规状态
    function isWhitelisted(address) → KYC 验证
    function freeze(address) → 临时冻结
    function seize(address, uint256) → 监管强制转移（法院命令执行）
    function forceTransfer(...) → 企业行动（如并购）
}
```

特有的 `seize` 功能支持法院命令强制执行，这是传统证券的法律要求，普通 ERC-20 没有。

**合规检查流程：**

```
每次 transfer
  → 查 Securitize Compliance Service
  → 检查 from/to 是否在白名单
  → 检查转账是否违反持有限制（如 Reg D 锁定期）
  → 检查额度限制
  → 通过 → 执行 transfer
```

---

## 合规资质（护城河）

- **SEC Transfer Agent 执照** — 可以合法记录证券所有权变更
- **SEC Registered Investment Adviser** — 可以提供投资建议
- **FINRA Broker-Dealer（Securitize Markets）** — 可以合法撮合证券交易
- **OCC 信托执照（Securitize Trust）** — 可以托管数字资产

这四张牌照组合，是 Securitize 真正的护城河，竞争对手拿到同样资质至少需要 3-5 年。

---

## 与 Ondo 的关系

不是竞争，是上下游：

```
Securitize（基础设施层）
    ↓
BlackRock BUIDL（Token 化产品）
    ↓
Ondo OUSG（持有 BUIDL 的 DeFi 接口层）
    ↓
DeFi 用户（Pendle/Morpho 的 Ondo 用户）
```

Ondo 是 Securitize + BlackRock 的下游分发渠道之一。

---

## 对你的项目的启示

新加坡的 Securitize 等价物 → **Hashstacs / iSTOX**

- iSTOX（现在的 ADDX）：MAS 持牌，新加坡本地 RWA 平台，Temasek 参投
- 如果你不想自建合规技术栈，和 ADDX 这类平台合作是最快路径
- 但合作意味着让渡控制权和收入分成，长期战略价值有限

**自建 vs 合作判断：**
- 第一个产品 → 考虑和 ADDX 合作，验证市场
- 有了 PMF → 自建技术栈，掌控完整链条
