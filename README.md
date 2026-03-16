# BaoClaw

Price protection for Pokemon TCG cards. Put options settled in USDC on Binance Smart Chain.

宝可梦集换式卡牌价格保护协议。看跌期权以 USDC 结算，运行在币安智能链上。

---

**[English](#english)** | **[中文](#中文)**

---

<a id="english"></a>

## What is BaoClaw?

You own a $500 Charizard. You're worried the market tanks. You pay a protocol-computed premium (based on √time × utilization), and if the price drops below $500 within 1-30 days, you claim the difference from the pool. If it doesn't, the pool keeps your premium.

BaoClaw is a **put option protocol** for Pokemon TCG cards. No owner. No governance. No oracle. Prices are crowdsourced by reporters who post bonds and submit prices. Median of N reports = canonical price. Outliers get slashed.

Four roles, one contract:

| Role | What they do | What they pay | What they earn |
|---|---|---|---|
| **Put buyer** | Buys price protection on a card | Premium in USDC + 0.3% of premium in $BAO (burned) | Payout if price drops below floor |
| **Liquidity provider** | Deposits USDC into the pool, receives LP shares | USDC + 0.3% of deposit in $BAO (burned) | Premium income — withdraw more than you deposited |
| **Reporter** | Reports card prices on-chain by scraping PriceCharting, TCGPlayer, eBay Sold | USDC bond per report (1% of price, min 10 USDC) + 0.3% of bond in $BAO (burned) | Bond refunded on next report or after 24h; 10% of put premiums shared as rewards; outlier reporters lose their bond |
| **Seeder** | Bootstraps initial prices without posting bond (contract deployer sets this address) | Nothing (bond-free) | Earns reporter rewards like any other honest reporter |

## Getting Started

BaoClaw runs on [OpenClaw](https://github.com/openclaw) — an AI agent framework. All interactions happen through the `!bao` command in any OpenClaw-enabled chat. No dApp, no website, no MetaMask. Just type commands.

### Step 1 — Authenticate with Privy

When you first use `!bao`, OpenClaw creates an **embedded wallet** for you via [Privy](https://privy.io). You sign in with your email, Google, or Twitter account — Privy generates a wallet on BSC behind the scenes.

No seed phrases. No browser extensions. Your wallet lives in Privy's secure infrastructure and signs transactions on your behalf.

### Step 2 — Get BNB for gas

Your Privy wallet needs a small amount of BNB to pay for transaction fees on BSC. Send BNB to your Privy wallet address from any exchange (Binance, Coinbase, etc.) or another wallet.

To check your wallet address:

```
!bao shares
```

The response will show your wallet address and current holdings.

### Step 3 — Buy $BAO

Every BaoClaw operation (except exercising puts) burns 0.3% of the USDC amount in $BAO tokens. You need $BAO in your wallet before you can buy puts, provide liquidity, or report prices.

Buy $BAO directly through OpenClaw:

```
!bao buy 100
```

This swaps BNB for $BAO via PancakeSwap V2. The agent will quote the BNB cost, add 1% slippage, and confirm before executing.

You can also buy $BAO directly on [PancakeSwap](https://pancakeswap.finance/swap?outputCurrency=0x67777b71A41eebab7D3aD06498D9d48669025873).

$BAO token: [`0x67777b71A41eebab7D3aD06498D9d48669025873`](https://bscscan.com/token/0x67777b71A41eebab7D3aD06498D9d48669025873) (1B supply)

### Step 4 — Get USDC

You need USDC on BSC to pay premiums, provide liquidity, or post reporter bonds. Bridge or purchase USDC from any exchange and send it to your Privy wallet address.

### Step 5 — Use BaoClaw

You're ready. Pick what you want to do:

**Report a card's price (become a reporter):**

```
!bao report sv1-4
```

The agent scrapes PriceCharting, TCGPlayer, and eBay Sold listings, calculates a price, and submits it on-chain with a proportional USDC bond (1% of the reported price, minimum 10 USDC). Your bond is refunded on your next report or after 24 hours — unless your price is an outlier (>10% from the median), in which case your bond is slashed to the pool.

**Protect a card's value:**

```
!bao Charizard ex Scarlet Violet put 14
```

The agent fetches the current median reporter price, calculates the protocol-computed premium (based on √time × utilization), and asks you to confirm. Three transactions are signed automatically: approve $BAO, approve USDC, buy put.

**Provide liquidity and earn premiums:**

```
!bao fund 1000
```

Deposits USDC into the pool. You receive LP shares. As put buyers pay premiums, your shares grow in value.

**Check your positions:**

```
!bao puts              # Your active puts
!bao shares            # Your LP shares and their value
!bao pool              # Pool stats
```

### Complete Onboarding Example

```
> !bao buy 10                          # Buy 10 $BAO with BNB
  "Buy 10 $BAO for ~0.015 BNB. Proceed?"
> yes
  "Bought 10 $BAO for 0.0148 BNB."

> !bao sv1-4 price                     # Check Charizard ex price
  "sv1-4 (Charizard ex): $487.50 USDC (median of 4 reports)"

> !bao report sv1-4                    # Report price (proportional bond)
  "Scraped: PriceCharting $490, TCGPlayer $485, eBay Sold $488. Submitting $488. Bond: 10 USDC (1% of $488, min $10). Proceed?"
> yes
  "Report submitted. Bond: 10 USDC (refundable after 24h or on next report)."

> !bao Charizard ex Scarlet Violet put 14  # Protect at $487.50 for 14 days
  "Premium: $37.42 USDC (protocol-computed). $BAO burn: 0.112 BAO. Proceed?"
> yes
  "Put #0 purchased. Protected at $487.50 for 14 days."

  ... 45 days later, price drops to $300 ...

> !bao report sv1-4                    # Must report before exercising
  "Scraped: PriceCharting $305, TCGPlayer $298, eBay Sold $297. Submitting $300. Bond: 10 USDC (1% of $300, min $10). Proceed?"
> yes
  "Report submitted."

> !bao 0 exercise                      # Claim payout
  "Payout: $487.50 - $300 = $187.50 USDC. Proceed?"
> yes
  "Exercised. $187.50 USDC sent to your wallet."
```

## How It Works

### Buying a Put

```
!bao Charizard ex Scarlet Violet put 14
```

1. Median reporter price for sv1-4 (Charizard ex): **$500 USDC**
2. Premium: **$37.42 USDC** (protocol-computed via √time × utilization) paid to the pool
3. $BAO burn: 0.3% of premium = **0.112 BAO** burned (permanently reduces totalSupply)
4. Floor locked at **$500** for **14 days**

If the price drops to $300 within 14 days:

```
!bao 0 exercise
```

Payout: $500 - $300 = **$200 USDC** from the pool. Net profit after premium: **$162.58**.

If the price stays above $500, the put expires worthless. The pool keeps the $37.42 premium.

**Automatic cleanup:** Expired puts are automatically cleaned during `buyPut` and `withdrawPool` operations (up to 5 per call), so the pool stays healthy without manual intervention. A `cleanExpiredPuts` function also exists for manual batch cleanup.

### Providing Liquidity

Deposit USDC, receive LP shares. Your shares represent proportional ownership of the pool. As premiums accumulate, your shares become worth more. Withdraw anytime.

```
!bao fund 10000        # Deposit $10,000 USDC → receive LP shares
!bao shares            # Check your shares and their value
!bao withdraw 5000     # Burn 5,000 shares → receive USDC
```

**How LP shares work:**

| Event | Pool | Total Shares | Share Price |
|---|---|---|---|
| You deposit $10,000 | $10,000 | 10,000 | $1.00 |
| 5 puts sold at 5% on $500 cards ($125 premium) | $10,125 | 10,000 | **$1.0125** |
| 1 put exercised, $200 payout | $9,925 | 10,000 | $0.9925 |
| 3 more puts expire worthless ($75 premium) | $10,000 | 10,000 | $1.00 |
| 10 more puts expire ($250 premium) | $10,250 | 10,000 | **$1.025** |
| You withdraw all shares | $0 | 0 | — |
| **You get back: $10,250** | | | **+2.5% return** |

The share price is the single number that tells LPs everything: above $1.00 = profit, below $1.00 = loss.

**Why provide liquidity:**

1. **Premium yield.** Most puts expire worthless (~70-80% in traditional options). Every expired put is pure income for the pool.
2. **Withdraw anytime.** No lock-up. Burn your LP shares, get back your proportional cut of the pool.
3. **$BAO burns compound.** Every deposit and every put purchase burns $BAO. More protocol usage = more burns = scarcer $BAO.

**The risk:** If many puts are exercised (card prices crash across the board), payouts drain the pool and share price drops below $1.00. LPs withdraw less than they deposited. This is the underwriting risk — you're the counterparty to every put.

### $BAO Burn

Three operations burn 0.3% of the USDC amount in $BAO tokens (true burn via `burnFrom` — permanently reduces `totalSupply`): `buyPut`, `fundPool`, and `withdrawPool`. Reporting burns 0.3% of the bond amount. Exercising a put does **not** burn — payouts must never be blocked by missing $BAO.

Additionally, **10% of every put premium** is distributed equally to honest fresh reporters as claimable rewards. This incentivizes active price reporting on cards with put markets.

```
More puts bought → more $BAO burned → lower supply → higher $BAO price
                → more premium income → pool grows → more puts available
```

| Action | USDC Amount | $BAO Burned (0.3%) |
|---|---|---|
| Buy put ($500 card, 5% premium) | $25 USDC | 0.075 BAO |
| Buy put ($500 card, 10% premium) | $50 USDC | 0.15 BAO |
| Exercise put ($200 payout) | $200 USDC | None (no burn) |
| Report a price (bond = max(10, 1% of price)) | Bond (scales with price) | 0.3% of bond |
| Deposit $1,000 to pool | $1,000 USDC | 3 BAO |
| Withdraw $1,000 from pool | $1,000 USDC | 3 BAO |

[Buy $BAO on PancakeSwap](https://pancakeswap.finance/swap?outputCurrency=0x67777b71A41eebab7D3aD06498D9d48669025873)

### Crowdsourced Pricing

There is no oracle. Anyone can be a reporter by posting a **USDC bond** (1% of reported price, minimum 10 USDC) and submitting a price for any tracked card. Reporters scrape prices from three verifiable public sources: [PriceCharting](https://www.pricecharting.com/), [TCGPlayer](https://www.tcgplayer.com/), and eBay Sold listings.

**How it works:**

1. A reporter calls `!bao report <cardId>` — the agent scrapes the three sources, computes a price, and submits it on-chain with a proportional USDC bond (1% of reported price, minimum 10 USDC).
2. When **3 or more fresh reports** (within the last 24 hours) exist for a card, the **median** of those reports becomes the canonical price used for puts and exercises.
3. If a reporter's submitted price is an **outlier** (more than 10% away from the median), their bond is **slashed** and sent to the pool.
4. Honest reporters get their bond **refunded** when they submit their next report, or they can claim it back after 24 hours via `!bao claim-bond <cardId>`.

**Why this works (Schelling point):** The truth is the natural convergence point because all three data sources (PriceCharting, TCGPlayer, eBay Sold) are publicly verifiable. Rational reporters have no incentive to lie — submitting an outlier price means losing your bond. You only need to trust that a majority of reporters are honest, which is the economically rational strategy.

If fewer than 3 fresh reports exist for a card, the price is considered stale. No new puts can be bought or exercised with stale prices. No funds are at risk — the contract just pauses until enough fresh reports arrive.

**Tracked cards** (launch set):

| Card ID | Card | Set |
|---|---|---|
| `sv1-4` | Charizard ex | Scarlet & Violet |
| `sv1-195` | Charizard ex (SAR) | Scarlet & Violet |
| `sv3pt5-189` | Charizard ex (151 SAR) | 151 |
| `base1-4` | Charizard | Base Set |
| `neo1-9` | Lugia | Neo Genesis |
| `base1-58` | Pikachu | Base Set |
| `sm115-SV49` | Charizard GX | Hidden Fates |
| `sv2-227` | Iono (SAR) | Paldea Evolved |

## Commands

BaoClaw is an [OpenClaw](https://github.com/openclaw) skill. All interactions happen through the `!bao` command:

```
Put Options:
  !bao <card name> put <days>    Buy price protection (1–30 days)
  !bao <card name> quote <days>  Preview premium before buying
  !bao <card name> price          Check market price + on-chain median (free)
  !bao <putId> exercise          Exercise a put (claim payout)
  !bao puts                      List my active puts

Reporting:
  !bao report <cardId>           Report a card's price (posts proportional USDC bond)
  !bao report-info <cardId>      View current reports and median for a card
  !bao reporters                 List active reporters
  !bao claim-bond <cardId>       Claim back your bond after 24h
  !bao claim-rewards             Claim accumulated reporter rewards

Pool (LP):
  !bao pool                      Pool balance and share price
  !bao fund <amount>             Deposit USDC → receive LP shares
  !bao withdraw <shares>         Burn LP shares → receive USDC
  !bao shares                    Check my LP shares and value

Token:
  !bao buy <amount>              Buy $BAO with BNB via PancakeSwap

  !bao help                      Show commands
```

Card IDs use the Pokemon TCG standard format: `{setCode}-{cardNumber}` (e.g., `sv1-4`, `base1-4`, `neo1-9`).

## Protocol Constants

| Parameter | Value |
|---|---|
| Premium model | √time × utilization kink (protocol-computed) |
| Base rate | 2% |
| Utilization kink | 70% |
| Max utilization | 95% |
| Put duration | 1–30 days |
| Price staleness threshold | 24 hours |
| Report bond | max(10 USDC, 1% of reported price) |
| Min reports for canonical price | 3 |
| Outlier threshold | 10% from median |
| $BAO burn rate | 0.3% of USDC amount on buyPut, fundPool, withdrawPool, report (true burn) |
| Reporter reward | 10% of put premium split equally among honest fresh reporters |
| Chain | Binance Smart Chain (56) |
| Settlement token | USDC |

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    BaoClaw.sol                       │
│                                                     │
│  Immutables (set once, never change):               │
│    usdc        → USDC token on BSC                  │
│    baoToken    → $BAO token (true burn on use)      │
│    seeder      → bootstrap reporter (no bond)       │
│                                                     │
│  Crowdsourced Pricing:                              │
│    reports[cardHash][reporter] → Report struct       │
│    reportCount[cardHash]       → # fresh reports     │
│    medianPrice[cardHash]       → canonical price     │
│    bonds[reporter][cardHash]   → USDC bond            │
│                                                     │
│  Puts:                                              │
│    puts[putId]           → Put struct               │
│    holderPuts[address]   → user's put IDs           │
│                                                     │
│  Pool:                                              │
│    poolBalance           → total USDC available     │
│    totalShares           → total LP shares          │
│    totalExposure         → sum of active put floors  │
│    shares[address]       → LP shares per funder     │
│                                                     │
│  No owner. No admin. No upgrade path. No oracle.    │
└─────────────────────────────────────────────────────┘
                           ▲
                           │ reports prices / buys puts / provides liquidity
                           │
                  ┌────────┴────────┐
                  │  Users (Privy)  │
                  │  via !bao       │
                  │                 │
                  │  Reporters:     │
                  │   PriceCharting │
                  │   TCGPlayer     │
                  │   eBay Sold     │
                  │   → on-chain    │
                  │                 │
                  │  USDC + $BAO    │
                  │  burn           │
                  └─────────────────┘
```

## Trust Assumptions

BaoClaw has **no single trusted entity**. Trust is distributed across crowdsourced reporters.

| Component | Trust required? | Why |
|---|---|---|
| Contract logic | No | Open source, auditable, immutable |
| Pool funds | No | LPs can withdraw proportionally anytime |
| Crowdsourced pricing | **Distributed** | You only need to trust that a majority of reporters are honest. Schelling point: truth is the natural convergence since prices come from 3 verifiable public sources (PriceCharting, TCGPlayer, eBay Sold). Outliers get slashed. |
| $BAO burn | No | True burn via `burnFrom` reduces totalSupply — irreversible |
| Put settlement | No | Math is on-chain: `payout = floor - current` |
| LP shares | No | Proportional math: `payout = shares × poolBalance / totalShares` |

If a majority of reporters collude to post false prices, incorrect payouts could occur. This is mitigated by the economic incentive: colluding reporters must each risk a bond proportional to the reported price (1% of price, min 10 USDC), and honest reporters can always submit correct prices to push the median back toward truth. No funds can be drained — the contract only pays `floor - current`, and the pool can't go negative.

## Development

### Prerequisites

- [Foundry](https://book.getfoundry.sh/getting-started/installation)
- [Node.js](https://nodejs.org/) 18+
- [cast](https://book.getfoundry.sh/reference/cast/) (included with Foundry)

### Build and Test

```bash
forge build
forge test -v
```

115 tests covering: crowdsourced reporter operations, proportional bond posting/refund/slashing, median price calculation, outlier detection, put lifecycle, exercise edge cases, amortized expired put cleanup, true $BAO burn (reduces totalSupply), seeder bootstrapping, reporter rewards (10% of premium), LP share minting/withdrawal/pricing, multi-LP scenarios, pool profit/loss propagation, immutability checks, √time × utilization premium model, variable duration, pool utilization caps, and full lifecycle scenarios.

### Deploy

```bash
cp .env.example .env
# Fill in USDC_ADDRESS, BAO_TOKEN_ADDRESS

source .env
forge script scripts/Deploy.s.sol:DeployBaoClaw \
  --rpc-url $RPC_URL \
  --broadcast \
  --private-key $PRIVATE_KEY
```

### CLI (without OpenClaw)

```bash
source .env

# Report a price
./scripts/baoclaw-cli.sh report sv1-4

# View report info
./scripts/baoclaw-cli.sh report-info sv1-4

# List active reporters
./scripts/baoclaw-cli.sh reporters

# Claim bond back
./scripts/baoclaw-cli.sh claim-bond sv1-4

# Check on-chain median price
./scripts/baoclaw-cli.sh price sv1-4

# Check market price from TCGPlayer (no on-chain interaction)
./scripts/baoclaw-cli.sh market-price "Charizard ex"

# Check if price is fresh
./scripts/baoclaw-cli.sh fresh sv1-4

# Quote a premium (preview before buying)
./scripts/baoclaw-cli.sh quote sv1-4 14

# Quote the reporter bond for a given price
./scripts/baoclaw-cli.sh quote-bond 488

# Approve tokens and buy a put (14 days)
./scripts/baoclaw-cli.sh approve-bao 2.5
./scripts/baoclaw-cli.sh approve-usdc 40
./scripts/baoclaw-cli.sh buy-put sv1-4 14

# Exercise
./scripts/baoclaw-cli.sh exercise 0

# Pool info
./scripts/baoclaw-cli.sh pool

# Provide liquidity
./scripts/baoclaw-cli.sh approve-bao 100
./scripts/baoclaw-cli.sh approve-usdc 1000
./scripts/baoclaw-cli.sh fund 1000

# Check LP position
./scripts/baoclaw-cli.sh shares 0xYourAddress

# Withdraw
./scripts/baoclaw-cli.sh withdraw 500
```

## Project Structure

```
baoclaw/
├── src/
│   └── BaoClaw.sol              # Core contract (puts + LP pool + crowdsourced pricing)
├── test/
│   └── BaoClaw.t.sol            # 115 tests
├── scripts/
│   ├── Deploy.s.sol             # Foundry deploy script
│   └── baoclaw-cli.sh           # CLI wrapper using cast
├── skills/
│   └── baoclaw/
│       └── SKILL.md             # OpenClaw skill definition
├── foundry.toml                 # Solidity 0.8.24, Paris EVM
└── .env.example                 # Environment template
```

## Roadmap

- [x] Put options contract (buy, exercise, expire)
- [x] Crowdsourced reporter pricing (bond + median + slash)
- [x] $BAO burn mechanism (0.3% of premium)
- [x] LP shares with proportional withdrawal
- [x] OpenClaw skill (Privy wallets)
- [x] CLI tooling
- [x] Pool utilization cap — limit total put exposure to a percentage of pool balance
- [x] Variable duration — 1–30 day puts with √time × utilization kink premium
- [ ] Mainnet deployment + audit
- [ ] Expand tracked cards (50+ high-value Pokemon TCG cards)
- [ ] Cross-chain (Base, Arbitrum)

## License

MIT

---

<a id="中文"></a>

# BaoClaw 中文文档

宝可梦集换式卡牌价格保护协议。看跌期权以 USDC 结算，运行在币安智能链（BSC）上。

## BaoClaw 是什么？

你有一张价值 $500 的喷火龙卡。你担心市场下跌。你支付一笔协议计算的权利金（基于 √时间 × 利用率），如果价格在 1-30 天内跌破 $500，你可以从资金池中获得差价补偿。如果没有跌破，资金池保留你的权利金。

BaoClaw 是一个宝可梦卡牌的**看跌期权协议**。无所有者、无治理、无预言机。价格由报价者众包提供：报价者提交保证金并上报价格。N 个报价的中位数 = 标准价格。异常值会被罚没。

四种角色，一个合约：

| 角色 | 做什么 | 支付什么 | 赚取什么 |
|---|---|---|---|
| **期权买方** | 购买卡牌价格保护 | USDC 权利金 + 权利金的 0.3% $BAO（销毁） | 价格下跌时获得补偿 |
| **流动性提供者** | 向资金池存入 USDC，获得 LP 份额 | USDC + 存款的 0.3% $BAO（销毁） | 权利金收入 — 提取时比存入更多 |
| **报价者** | 通过抓取 PriceCharting、TCGPlayer、eBay 已售数据将卡牌价格上链 | 每次报价 USDC 保证金（价格的 1%，最低 10 USDC）+ 保证金的 0.3% $BAO（销毁） | 下次报价时或 24 小时后退还保证金；获得权利金 10% 的奖励；异常报价者的保证金被罚没 |
| **种子报价者** | 无需保证金即可引导初始价格（合约部署者设定） | 无（免保证金） | 与其他诚实报价者一样获得报价奖励 |

## 新手入门

BaoClaw 运行在 [OpenClaw](https://github.com/openclaw) 上 — 一个 AI 代理框架。所有操作通过 `!bao` 命令在任何支持 OpenClaw 的聊天中完成。无需 dApp、无需网站、无需 MetaMask。只需输入命令。

### 第 1 步 — 通过 Privy 认证

首次使用 `!bao` 时，OpenClaw 会通过 [Privy](https://privy.io) 为你创建一个**内嵌钱包**。你可以用邮箱、Google 或 Twitter 账户登录 — Privy 会在后台自动生成一个 BSC 钱包。

无需助记词。无需浏览器插件。你的钱包存储在 Privy 的安全基础设施中，代替你签署交易。

### 第 2 步 — 获取 BNB 作为 Gas 费

你的 Privy 钱包需要少量 BNB 来支付 BSC 上的交易手续费。从任何交易所（币安、Coinbase 等）或其他钱包向你的 Privy 钱包地址发送 BNB。

查看你的钱包地址：

```
!bao shares
```

响应中会显示你的钱包地址和当前持仓。

### 第 3 步 — 购买 $BAO

每次 BaoClaw 操作（行权除外）都会销毁 USDC 金额 0.3% 的 $BAO 代币。在购买期权、提供流动性或报价之前，你的钱包中需要有 $BAO。

通过 OpenClaw 直接购买 $BAO：

```
!bao buy 100
```

这会通过 PancakeSwap V2 将 BNB 兑换为 $BAO。代理会报价 BNB 成本，加上 1% 滑点，并在执行前确认。

你也可以直接在 [PancakeSwap](https://pancakeswap.finance/swap?outputCurrency=0x67777b71A41eebab7D3aD06498D9d48669025873) 上购买 $BAO。

$BAO 代币：[`0x67777b71A41eebab7D3aD06498D9d48669025873`](https://bscscan.com/token/0x67777b71A41eebab7D3aD06498D9d48669025873)（总量 10 亿）

### 第 4 步 — 获取 USDC

你需要 BSC 上的 USDC 来支付权利金、提供流动性或提交报价保证金。从任何交易所购买或跨链 USDC，然后发送到你的 Privy 钱包地址。

### 第 5 步 — 使用 BaoClaw

你已准备就绪。选择你想做的事情：

**报价卡牌价格（成为报价者）：**

```
!bao report sv1-4
```

代理会抓取 PriceCharting、TCGPlayer 和 eBay 已售数据，计算价格，并以按比例 USDC 保证金（报价价格的 1%，最低 10 USDC）提交上链。你的保证金在下次报价时或 24 小时后退还 — 除非你的价格是异常值（偏离中位数 >10%），此时你的保证金将被罚没到资金池。

**保护卡牌价值：**

```
!bao Charizard ex Scarlet Violet put 14
```

代理会获取当前报价者中位数价格，计算协议权利金（基于 √时间 × 利用率）和 $BAO 销毁量，并要求你确认。三笔交易会自动签署：批准 $BAO、批准 USDC、购买期权。

**提供流动性赚取权利金：**

```
!bao fund 1000
```

向资金池存入 USDC。你会收到 LP 份额。随着期权买方支付权利金，你的份额价值会增长。

**查看你的持仓：**

```
!bao puts              # 你的活跃期权
!bao shares            # 你的 LP 份额及其价值
!bao pool              # 资金池统计信息
```

### 完整入门示例

```
> !bao buy 10                          # 用 BNB 购买 10 个 $BAO
  "购买 10 $BAO，约需 0.015 BNB。确认？"
> 确认
  "已购买 10 $BAO，花费 0.0148 BNB。"

> !bao sv1-4 price                     # 查看喷火龙 ex 价格
  "sv1-4（喷火龙 ex）：$487.50 USDC（4 个报价的中位数）"

> !bao report sv1-4                    # 报价（按比例保证金）
  "抓取结果：PriceCharting $490，TCGPlayer $485，eBay 已售 $488。提交 $488。保证金：10 USDC（$488 的 1%，最低 $10）。确认？"
> 确认
  "报价已提交。保证金：10 USDC（24 小时后或下次报价时可退还）。"

> !bao Charizard ex Scarlet Violet put 14  # 以 $487.50 保护 14 天
  "权利金：$37.42 USDC（协议计算）。$BAO 销毁：0.112 BAO。确认？"
> 确认
  "期权 #0 已购买。保护价 $487.50，有效期 14 天。"

  ... 45 天后，价格跌至 $300 ...

> !bao report sv1-4                    # 行权前必须先报价
  "抓取结果：PriceCharting $305，TCGPlayer $298，eBay 已售 $297。提交 $300。保证金：10 USDC（$300 的 1%，最低 $10）。确认？"
> 确认
  "报价已提交。"

> !bao 0 exercise                      # 领取补偿
  "补偿金额：$487.50 - $300 = $187.50 USDC。确认？"
> 确认
  "已行权。$187.50 USDC 已发送到你的钱包。"
```

## 运作原理

### 购买看跌期权

```
!bao Charizard ex Scarlet Violet put 14
```

1. sv1-4（喷火龙 ex）的报价者中位数价格：**$500 USDC**
2. 权利金：**$37.42 USDC**（协议通过 √时间 × 利用率计算）支付给资金池
3. $BAO 销毁：权利金的 0.3% = **0.112 BAO** 真正销毁（减少 totalSupply）
4. 保护价锁定在 **$500**，有效期 **14 天**

如果价格在 14 天内跌至 $300：

```
!bao 0 exercise
```

补偿金额：$500 - $300 = **$200 USDC**（从资金池支付）。扣除权利金后净利润：**$162.58**。

如果价格保持在 $500 以上，期权到期作废。资金池保留 $37.42 权利金。

**自动清理：** 过期的期权会在 `buyPut` 和 `withdrawPool` 操作期间自动清理（每次最多 5 个），因此资金池无需人工干预即可保持健康状态。`cleanExpiredPuts` 函数仍可用于手动批量清理。

### 提供流动性

存入 USDC，获得 LP 份额。你的份额代表对资金池的按比例所有权。随着权利金累积，你的份额价值增加。随时可提取。

```
!bao fund 10000        # 存入 $10,000 USDC → 获得 LP 份额
!bao shares            # 查看你的份额及其价值
!bao withdraw 5000     # 销毁 5,000 份额 → 获得 USDC
```

**LP 份额运作方式：**

| 事件 | 资金池 | 总份额 | 份额价格 |
|---|---|---|---|
| 你存入 $10,000 | $10,000 | 10,000 | $1.00 |
| 售出 5 份 5% 权利金的 $500 卡牌期权（$125 权利金） | $10,125 | 10,000 | **$1.0125** |
| 1 份期权被行权，$200 补偿 | $9,925 | 10,000 | $0.9925 |
| 3 份期权到期作废（$75 权利金） | $10,000 | 10,000 | $1.00 |
| 10 份期权到期（$250 权利金） | $10,250 | 10,000 | **$1.025** |
| 你提取全部份额 | $0 | 0 | — |
| **你取回：$10,250** | | | **+2.5% 回报** |

份额价格是告诉 LP 一切的唯一数字：高于 $1.00 = 盈利，低于 $1.00 = 亏损。

**为什么提供流动性：**

1. **权利金收益。** 大多数期权到期作废（传统期权中约 70-80%）。每份到期的期权都是资金池的纯收入。
2. **随时提取。** 无锁定期。销毁你的 LP 份额，取回你在资金池中的按比例份额。
3. **$BAO 销毁复利效应。** 每次存款和每次购买期权都会销毁 $BAO。更多协议使用 = 更多销毁 = $BAO 更稀缺。

**风险：** 如果大量期权被行权（卡牌价格全面暴跌），补偿金额会消耗资金池，份额价格跌破 $1.00。LP 提取时获得的比存入的少。这是承保风险 — 你是所有期权的对手方。

### $BAO 销毁

三种操作会销毁 USDC 金额 0.3% 的 $BAO 代币（通过 `burnFrom` 真正销毁 — 永久减少 `totalSupply`）：`buyPut`、`fundPool` 和 `withdrawPool`。报价操作销毁保证金金额的 0.3%。行权（exercisePut）**不会**销毁 — 补偿金额绝不能因为缺少 $BAO 而被阻止。

此外，**每笔权利金的 10%** 会平均分配给诚实的新鲜报价者作为可领取的奖励。这激励报价者积极为有期权市场的卡牌报价。

```
更多期权购买 → 更多 $BAO 销毁 → 供应减少 → $BAO 价格上升
            → 更多权利金收入 → 资金池增长 → 更多期权可售
```

| 操作 | USDC 金额 | $BAO 销毁量（0.3%） |
|---|---|---|
| 购买期权（$500 卡牌，5% 权利金） | $25 USDC | 0.075 BAO |
| 购买期权（$500 卡牌，10% 权利金） | $50 USDC | 0.15 BAO |
| 行权（$200 补偿） | $200 USDC | 无（不销毁） |
| 报价（保证金 = max(10, 价格的 1%)） | 保证金（随价格变化） | 保证金的 0.3% |
| 存入 $1,000 到资金池 | $1,000 USDC | 3 BAO |
| 从资金池提取 $1,000 | $1,000 USDC | 3 BAO |

[在 PancakeSwap 上购买 $BAO](https://pancakeswap.finance/swap?outputCurrency=0x67777b71A41eebab7D3aD06498D9d48669025873)

### 众包定价

没有预言机。任何人都可以通过提交 **USDC 保证金**（报价价格的 1%，最低 10 USDC）并上报任何跟踪卡牌的价格来成为报价者。报价者从三个可验证的公开来源抓取价格：[PriceCharting](https://www.pricecharting.com/)、[TCGPlayer](https://www.tcgplayer.com/) 和 eBay 已售数据。

**运作方式：**

1. 报价者调用 `!bao report <卡牌ID>` — 代理抓取三个来源，计算价格，并以按比例 USDC 保证金（报价价格的 1%，最低 10 USDC）提交上链。
2. 当某张卡牌有 **3 个或以上的新鲜报价**（24 小时内）时，这些报价的**中位数**成为用于期权买卖和行权的标准价格。
3. 如果报价者提交的价格是**异常值**（偏离中位数超过 10%），其保证金将被**罚没**并转入资金池。
4. 诚实的报价者在提交下一次报价时会获得保证金**退还**，或者可以在 24 小时后通过 `!bao claim-bond <卡牌ID>` 领回。

**为什么有效（谢林点）：** 真实价格是自然收敛点，因为三个数据源（PriceCharting、TCGPlayer、eBay 已售）都是公开可验证的。理性的报价者没有动机撒谎 — 提交异常价格意味着损失保证金。你只需要信任大多数报价者是诚实的，而这正是经济理性的策略。

如果某张卡牌的新鲜报价少于 3 个，价格被视为过期。使用过期价格无法购买或行权期权。资金不会面临风险 — 合约只是暂停，直到足够的新鲜报价到达。

**跟踪的卡牌**（首发系列）：

| 卡牌 ID | 卡牌 | 系列 |
|---|---|---|
| `sv1-4` | 喷火龙 ex | 朱与紫 |
| `sv1-195` | 喷火龙 ex (SAR) | 朱与紫 |
| `sv3pt5-189` | 喷火龙 ex (151 SAR) | 151 |
| `base1-4` | 喷火龙 | 初始系列 |
| `neo1-9` | 洛奇亚 | Neo Genesis |
| `base1-58` | 皮卡丘 | 初始系列 |
| `sm115-SV49` | 喷火龙 GX | Hidden Fates |
| `sv2-227` | 奇树 (SAR) | Paldea Evolved |

## 命令

BaoClaw 是一个 [OpenClaw](https://github.com/openclaw) 技能。所有交互通过 `!bao` 命令完成：

```
看跌期权：
  !bao <卡牌名称> put <天数>      购买价格保护（1–30 天）
  !bao <卡牌名称> quote <天数>    购买前预览权利金
  !bao <卡牌名称> price           查看市场价格 + 链上中位数（免费）
  !bao <期权ID> exercise         行权（领取补偿）
  !bao puts                      列出我的活跃期权

报价：
  !bao report <卡牌ID>           报价卡牌价格（提交按比例 USDC 保证金）
  !bao report-info <卡牌ID>      查看当前报价和中位数
  !bao reporters                 列出活跃报价者
  !bao claim-bond <卡牌ID>       24 小时后领回保证金
  !bao claim-rewards             领取报价奖励

资金池（LP）：
  !bao pool                      资金池余额和份额价格
  !bao fund <金额>               存入 USDC → 获得 LP 份额
  !bao withdraw <份额数>         销毁 LP 份额 → 获得 USDC
  !bao shares                    查看我的 LP 份额和价值

代币：
  !bao buy <数量>                用 BNB 通过 PancakeSwap 购买 $BAO

  !bao help                      显示命令
```

卡牌 ID 使用宝可梦 TCG 标准格式：`{系列代码}-{卡牌编号}`（例如 `sv1-4`、`base1-4`、`neo1-9`）。

## 协议常量

| 参数 | 值 |
|---|---|
| 权利金模型 | √时间 × 利用率拐点（协议计算） |
| 基础费率 | 2% |
| 利用率拐点 | 70% |
| 最大利用率 | 95% |
| 期权有效期 | 1–30 天 |
| 价格过期阈值 | 24 小时 |
| 报价保证金 | max(10 USDC, 报价价格的 1%) |
| 最低报价数量 | 3 |
| 异常值阈值 | 偏离中位数 10% |
| $BAO 销毁率 | buyPut、fundPool、withdrawPool、report 时 USDC 金额的 0.3%（真正销毁） |
| 报价奖励 | 权利金的 10% 平均分配给诚实的新鲜报价者 |
| 链 | 币安智能链 (56) |
| 结算代币 | USDC |

## 信任假设

BaoClaw **没有单一信任实体**。信任分布在众包报价者中。

| 组件 | 需要信任？ | 原因 |
|---|---|---|
| 合约逻辑 | 否 | 开源、可审计、不可变 |
| 资金池资金 | 否 | LP 可随时按比例提取 |
| 众包定价 | **分布式** | 你只需要信任大多数报价者是诚实的。谢林点：真实价格是自然收敛点，因为价格来自 3 个可验证的公开来源（PriceCharting、TCGPlayer、eBay 已售）。异常值会被罚没。 |
| $BAO 销毁 | 否 | 通过 `burnFrom` 真正销毁减少 totalSupply — 不可逆 |
| 期权结算 | 否 | 计算在链上：`补偿 = 保护价 - 当前价` |
| LP 份额 | 否 | 按比例计算：`补偿 = 份额 × 资金池余额 / 总份额` |

如果大多数报价者串通提交虚假价格，可能导致不正确的补偿。缓解措施：串通的报价者每次报价都需要冒与报价价格成比例的保证金风险（价格的 1%，最低 10 USDC），而诚实的报价者随时可以提交正确价格将中位数拉回真实值。资金不会被掏空 — 合约只支付 `保护价 - 当前价`，资金池不会变为负数。

## 许可证

MIT
