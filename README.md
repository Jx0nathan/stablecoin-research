# RWA Research

Real World Assets (RWA) 调研笔记，记录技术架构、商业逻辑、竞品分析及市场洞察。

## 目录结构

```
rwa-research/
├── competitors/                # 竞品分析
│   ├── ondo/                   # Ondo Finance 深度分析（技术 + 商业 + L1）
│   ├── blackrock-buidl.md      # BlackRock BUIDL（机构美债基金）
│   ├── franklin-templeton-benji.md  # Franklin Templeton BENJI（零售货基）
│   ├── securitize.md           # Securitize（RWA 基础设施平台）
│   ├── centrifuge.md           # Centrifuge（私人信贷 + DeFi）
│   └── maple-finance.md        # Maple Finance（机构信贷借贷）
├── concepts/                   # 核心概念
│   └── rwa-fundamentals.md     # RWA 基础知识
└── architecture/               # 技术架构参考（待补充）
```

## 竞品调研进度

**已完成**

- Ondo Finance — 商业逻辑 + 技术架构（源码级）+ Ondo Chain L1
- BlackRock BUIDL — 机构货币市场基金，Securitize 底座
- Franklin Templeton BENJI — SEC 注册，区块链即登记册，最干净法律结构
- Securitize — RWA 基础设施平台，BlackRock/KKR/Hamilton Lane 合作方
- Centrifuge — DeFi 原生私人信贷，Apollo/Janus Henderson，$2B+ TVL
- Maple Finance — 链上机构信贷市场，Syrup 零售 DeFi 入口

**待调研**

- Backed Finance（瑞士，股票 Token 化）
- OpenEden（新加坡，T-Bill Token 化）
- Superstate（美国，货币市场基金）
- MakerDAO / Sky（DAI 背后的 RWA 资产配置）
- Goldfinch（新兴市场信贷）

## 竞品对比（核心维度）

**资产类型**
- 美债 / 货基：Ondo OUSG、BlackRock BUIDL、Franklin BENJI、Superstate
- 私人信贷：Centrifuge、Maple、Goldfinch
- 股票 / ETF：Ondo Global Markets、Backed Finance
- 基础设施：Securitize

**目标用户**
- 机构（$1M+）：BlackRock BUIDL、Securitize、Centrifuge Prime
- 合规零售：Franklin BENJI（$1 门槛，SEC 注册）
- DeFi 原生：Ondo USDY、Maple Syrup、Centrifuge Pool

**合规路径**
- 正面 SEC 注册：Franklin BENJI（最干净，最慢）
- Reg D/S 豁免：Ondo OUSG/USDY、BlackRock BUIDL
- DeFi 原生（最小化合规）：Centrifuge、Maple

**技术差异化**
- 自研技术栈：Ondo、Franklin Templeton
- 外包给 Securitize：BlackRock BUIDL、KKR、Hamilton Lane
- Protocol 层（开放）：Centrifuge、Maple

## 核心研究结论（持续更新）

**关于竞争格局：**
美债 Token 化已红海（Ondo + BlackRock + Franklin），差异化要找细分资产。

**关于技术路径：**
自研 > Securitize 合作（长期），但第一个产品可考虑合作验证市场。

**关于新加坡机会：**
- 本地债券（SGS / MAS Bills）仍是空白，Ondo 未进入
- SME 贸易融资（Centrifuge 模式）适合本地场景
- 机构 USDC 借贷（Maple 模式）与支付/清算基础设施高度互补

**关于链选择：**
Ethereum 首选（机构信任），Base 次选（低 gas，Coinbase 背书）。不建议自建 L1。
