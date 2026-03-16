---
name: bao
description: BaoClaw — price protection for Pokemon TCG cards. Put options settled in USDC on BSC with crowdsourced reporter pricing and $BAO burn. Command — !bao <card name> put <days>
version: 9.0.0
user-invocable: true
command-arg-mode: raw
metadata: {"openclaw":{"emoji":"🐾","requires":{"bins":["cast","curl","jq"],"env":["BAOCLAW_ADDRESS","USDC_ADDRESS","BAO_TOKEN_ADDRESS"]},"primaryEnv":"PRIVY_APP_SECRET"}}
---

# BaoClaw — Pokemon TCG Price Protection / 宝可梦卡牌价格保护

You are an agent that executes the `!bao` command. BaoClaw sells put options on Pokemon TCG cards, settled in USDC on Binance Smart Chain. Users protect their cards' value — if the market price drops, they claim the difference.

No owner. No governance. No oracle. Prices are crowdsourced by reporters who post USDC bonds and submit prices scraped from multiple sources. Median of N reports = canonical price. Outliers get slashed.

Buying puts, funding the pool, withdrawing from the pool, and reporting prices each burn 0.3% of the USDC amount in $BAO tokens (true burn via `burnFrom` — reduces totalSupply permanently). Exercising a put does **not** burn — payouts must never be blocked by missing $BAO.

10% of each put premium is distributed equally to honest fresh reporters as rewards (claimable via `claimRewards`). The remaining 90% goes to the insurance pool.

## Card Identification / 卡牌识别

Users specify cards by **natural language name**, not by internal API IDs. The skill resolves names to pokemontcg.io IDs using search, then **always shows the card image and metadata for visual verification** before any action.

### Resolution flow

1. User types a card name (e.g., "Charizard ex Scarlet Violet")
2. Skill searches the pokemontcg.io API: `?q=name:"charizard ex" set.series:"Scarlet & Violet"`
3. API returns matches with images, set info, and TCGPlayer prices
4. Skill shows the card image + name + set + collector number + pokemontcg.io ID
5. User confirms this is the right card before proceeding
6. The on-chain contract uses the pokemontcg.io ID (e.g., `sv1-4`) as the card identifier

### Card image display (MANDATORY)

Before **any** put purchase or price report, you MUST show the card image to the user for visual verification. Fetch the image URL from the pokemontcg.io API response (`images.small` or `images.large` field) and display it inline.

If multiple matches are found, show the top 3-5 results with images and ask the user to pick one.

### Caching

Card metadata (name, set, images, pokemontcg.io ID) is cached locally in `bot/.cache/cards.json` for 30 days. Price data is refreshed every 12 hours. Images are cached in `bot/.cache/images/`. The cache avoids redundant API calls — the pokemontcg.io free tier allows 20K requests/day.

## Language Detection / 语言检测

Detect the user's language from their message. If the user writes in Chinese (Simplified or Traditional), respond entirely in Chinese. If the user writes in English, respond in English. If unclear, default to English.

## Command Format

```
!bao <card name> put <days>       Buy price protection / 购买价格保护
!bao <card name> quote <days>    Quote premium for protection / 查询保护权利金
!bao <card name> price           Check current median price / 查看当前众包价格
!bao <card name> report          Report a card's price (posts 10 USDC bond) / 报价（质押 10 USDC 保证金）
!bao <card name> claim-bond      Reclaim bond after report expires / 领回过期保证金
!bao claim-rewards               Claim accumulated reporter rewards / 领取报价奖励
!bao <putId> exercise            Exercise a put (claim payout) / 行权（领取补偿）
!bao puts                        List my puts / 列出我的期权
!bao pool                        Check pool balance and share price / 查看资金池
!bao fund <amount>               Deposit USDC into the pool / 存入 USDC
!bao withdraw <shares>           Withdraw USDC from the pool / 提取 USDC
!bao shares                      Check my LP shares / 查看我的 LP 份额
!bao buy <amount>                Buy $BAO with BNB via PancakeSwap / 购买 $BAO
!bao search <card name>          Search for a card (show matches with images) / 搜索卡牌
!bao help                        Show available commands / 显示命令
```

Cards are specified by natural language name (e.g., "Charizard ex Scarlet Violet", "Pikachu Base Set", "Lugia Neo Genesis"). The skill resolves names to pokemontcg.io IDs internally. You can also use IDs directly if you know them (e.g., `sv1-4`).

## Price Sources / 价格来源

Reporters should aggregate prices from multiple sources to compute a fair median. The following sources are supported:

| Source | Type | Data |
|---|---|---|
| **TCGPlayer** | pokemontcg.io API | Market price (included in card lookup) |
| **PriceCharting** | Browser scrape | pricecharting.com market price |
| **eBay Sold** | Browser scrape | Median of recent sold listings |
| **Alt.xyz** | Browser scrape | Alt Value from app.alt.xyz/research |
| **Collectr** | Browser scrape | Market price from app.getcollectr.com |
| **Phygitals** | Browser scrape | Marketplace price from phygitals.com |

When reporting, the agent should:
1. Fetch the TCGPlayer price from the pokemontcg.io API (automatic, included in card resolution)
2. Use browser orchestration to scrape 2-3 additional sources
3. Compute the median of all available sources
4. Show the user all source prices before confirming the report

Not all sources will have data for every card. Use whatever is available (minimum 1 source).

## Environment

| Variable | Required | Description |
|---|---|---|
| `BAOCLAW_ADDRESS` | Yes | Deployed BaoClaw contract address |
| `USDC_ADDRESS` | Yes | USDC token address on BSC |
| `BAO_TOKEN_ADDRESS` | Yes | $BAO token: `0x67777b71A41eebab7D3aD06498D9d48669025873` |
| `RPC_URL` | No | BSC RPC (default: `https://bsc-dataseed.binance.org/`) |
| `POKEMONTCG_API_KEY` | No | Free API key from dev.pokemontcg.io (20K req/day). Stored on user's machine. |
| `PRIVY_APP_ID` | For writes | Privy app ID for embedded wallet signing |
| `PRIVY_APP_SECRET` | For writes | Privy app secret |

**Note on API keys:** There is no server involved. The pokemontcg.io API key is stored on the user's own machine as an environment variable. It is a free key (register at dev.pokemontcg.io). Without a key, the API allows 1,000 requests/day; with a key, 20,000/day.

## Wallet: Privy Embedded Wallets

All write transactions are signed via **Privy embedded wallets**. Users authenticate with email or social login — no MetaMask or raw private keys.

### Transaction signing flow

1. User authenticates via Privy (email, Google, Twitter, etc.)
2. Privy creates/retrieves an embedded wallet for the user on BSC
3. The skill submits transactions through Privy's server-side signing API
4. Privy signs and broadcasts to BSC

### Privy API: Sign and send a transaction

```bash
curl -s -X POST "https://auth.privy.io/api/v1/wallets/${PRIVY_WALLET_ID}/rpc" \
  -H "Authorization: Bearer ${PRIVY_ACCESS_TOKEN}" \
  -H "privy-app-id: ${PRIVY_APP_ID}" \
  -H "Content-Type: application/json" \
  -d '{
    "method": "eth_sendTransaction",
    "params": {
      "transaction": {
        "to": "<contract_address>",
        "data": "<encoded_calldata>",
        "chain_id": 56
      }
    }
  }'
```

### Encoding calldata with cast

For all write operations, encode the calldata with `cast calldata`, then submit via Privy:

```bash
# Encode
CALLDATA=$(cast calldata "buyPut(string,uint256)" "<cardId>" <durationDays>)

# Submit via Privy
curl -s -X POST "https://auth.privy.io/api/v1/wallets/${PRIVY_WALLET_ID}/rpc" \
  -H "Authorization: Bearer ${PRIVY_ACCESS_TOKEN}" \
  -H "privy-app-id: ${PRIVY_APP_ID}" \
  -H "Content-Type: application/json" \
  -d "{
    \"method\": \"eth_sendTransaction\",
    \"params\": {
      \"transaction\": {
        \"to\": \"${BAOCLAW_ADDRESS}\",
        \"data\": \"${CALLDATA}\",
        \"chain_id\": 56
      }
    }
  }"
```

### Getting the user's wallet address

```bash
WALLET_ADDRESS=$(curl -s "https://auth.privy.io/api/v1/wallets/${PRIVY_WALLET_ID}" \
  -H "Authorization: Bearer ${PRIVY_ACCESS_TOKEN}" \
  -H "privy-app-id: ${PRIVY_APP_ID}" | jq -r '.address')
```

## Protocol Constants

| Parameter | Value |
|---|---|
| Premium pricing | √time × utilization kink model (protocol-computed) |
| Base rate | 2% (200 bps) |
| Utilization kink | 70% (optimal) |
| Max utilization | 95% (hard cap) |
| Max put duration | 30 days |
| Price staleness | 24 hours |
| Report bond | max(10 USDC, 1% of reported price) |
| Min reports for valid price | 3 fresh reports |
| Outlier threshold | 10% deviation from median → bond slashed |
| $BAO burn | 0.3% of USDC amount on buyPut, fundPool, withdrawPool, reportPrice (not exercise) |
| $BAO token | `0x67777b71A41eebab7D3aD06498D9d48669025873` |
| PancakeSwap V2 Router | `0x10ED43C718714eb63d5aA57B78B54704E256024E` |
| WBNB | `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` |

## Workflow: `!bao <card name> report`

Report a card's price on-chain. The agent resolves the card name, shows the card image for verification, scrapes multiple price sources, and submits the median with a USDC bond (1% of reported price, minimum 10 USDC).

### Step 1 — Resolve card and show image

Search the pokemontcg.io API by name. **Show the card image and metadata to the user for visual verification.**

```bash
# Search pokemontcg.io API directly
curl -s "https://api.pokemontcg.io/v2/cards?q=name:\"<card name>\"&orderBy=-set.releaseDate&pageSize=5" \
  -H "X-Api-Key: ${POKEMONTCG_API_KEY:-}" | jq '.data[] | {id, name, set: .set.name, series: .set.series, number, images, tcgplayer: .tcgplayer.prices}'
```

The API returns: pokemontcg.io ID, name, set, collector number, image URLs, TCGPlayer prices.

**Display to user (MANDATORY — show image):**

**English:**
```
[CARD IMAGE]
Card: <name> (<id>)
Set: <set name> (<series>)
Collector #: <number>/<total>
Is this the right card?
```

**Chinese:**
```
[卡牌图片]
卡牌：<name>（<id>）
系列：<set name>（<series>）
编号：<number>/<total>
这是正确的卡牌吗？
```

If multiple matches, show top 3-5 with images and ask user to pick.

### Step 2 — Fetch prices from multiple sources

After card confirmation, aggregate prices from available sources:

1. **TCGPlayer** — already included from pokemontcg.io API response
2. **PriceCharting** — use browser orchestration to scrape pricecharting.com
3. **eBay Sold** — use browser orchestration to scrape recent sold listings
4. **Alt.xyz** — use browser orchestration to scrape app.alt.xyz/research
5. **Collectr** — use browser orchestration to scrape app.getcollectr.com
6. **Phygitals** — use browser orchestration to scrape phygitals.com

Compute the median of all available source prices. Minimum 1 source required.

### Step 3 — Confirm with user

Calculate:
- `bond_usdc = max(median * 0.01, 10)`
- `bao_burn = bond_usdc * 0.003`

**English:**
```
[CARD IMAGE]
Reporting price for <name> (<id>):
  TCGPlayer:     $<price1>
  PriceCharting: $<price2>
  eBay Sold:     $<price3>
  Alt.xyz:       $<price4> (or "N/A")
  Collectr:      $<price5> (or "N/A")
  Median: $<median> (<N> sources)
Bond: <bond_usdc> USDC (1% of median, min 10 USDC — fully refunded after 24h)
$BAO burn: <bao_burn> BAO (0.3% of bond — non-refundable)
💰 Net cost: <bao_burn> BAO only (bond is 100% refundable)
💰 You earn: a share of 10% of every put premium sold on this card while your report is fresh
```

**Chinese:**
```
[卡牌图片]
为 <name>（<id>）报价：
  TCGPlayer：     $<price1>
  PriceCharting： $<price2>
  eBay Sold：     $<price3>
  Alt.xyz：       $<price4>（或"无数据"）
  Collectr：      $<price5>（或"无数据"）
  中位数：$<median>（<N> 个来源）
保证金：<bond_usdc> USDC（中位数的 1%，最低 10 USDC — 24 小时后全额退还）
$BAO 销毁：<bao_burn> BAO（保证金的 0.3% — 不可退还）
💰 净成本：仅 <bao_burn> BAO（保证金 100% 可退还）
💰 收益：你的报价有效期间，该卡牌每笔期权权利金的 10% 将分配给你
```

Ask user to confirm.

### Step 4 — Approve and report via Privy

Use the **resolved pokemontcg.io ID** (e.g., `sv1-4`) as the on-chain card identifier.

Compute the bond: `bond = max(median * 0.01, 10)`. Use `quoteBond` to preview:

```bash
PRICE=$(cast to-wei "<median_price>")
BOND=$(cast call "$BAOCLAW_ADDRESS" "quoteBond(uint256)(uint256)" "$PRICE" --rpc-url "$RPC_URL")
```

1. **Approve $BAO** for burn (0.3% of bond):

```bash
BAO_BURN=$(echo "$BOND * 3 / 10000" | bc)
CALLDATA=$(cast calldata "approve(address,uint256)" "$BAOCLAW_ADDRESS" "$BAO_BURN")
# Submit via Privy to BAO_TOKEN_ADDRESS
```

2. **Approve USDC** for bond:

```bash
CALLDATA=$(cast calldata "approve(address,uint256)" "$BAOCLAW_ADDRESS" "$BOND")
# Submit via Privy to USDC_ADDRESS
```

3. **Report price**:

```bash
PRICE=$(cast to-wei "<median_price>")
# Use the resolved pokemontcg.io ID, NOT the natural language name
CALLDATA=$(cast calldata "reportPrice(string,uint256)" "<resolved_card_id>" "$PRICE")
# Submit via Privy to BAOCLAW_ADDRESS
```

### Step 5 — Confirm result

- EN: "Price reported: <name> (<id>) @ $<price> (median of <N> sources). Bond: <bond_usdc> USDC locked. If your report is within 10% of the median, your bond is safe. Reclaim after 24h with `!bao <name> claim-bond`."
- ZH: "已报价：<name>（<id>）@ $<price>（<N> 个来源的中位数）。保证金：<bond_usdc> USDC 已锁定。如果你的报价在中位数 10% 以内，保证金安全。24 小时后可通过 `!bao <name> claim-bond` 领回。"

## Workflow: `!bao <card name> claim-bond`

Reclaim a report bond after the report expires (>24h old). Resolve the card name first (same flow).

```bash
# Use the resolved pokemontcg.io ID
CALLDATA=$(cast calldata "claimBond(string)" "<resolved_card_id>")
# Submit via Privy to BAOCLAW_ADDRESS
```

- EN: "Bond reclaimed: <bond_amount> USDC returned."
- ZH: "保证金已领回：<bond_amount> USDC 已退还。"

If bond is still locked (report < 24h old):
- EN: "Bond is still locked. Your report is less than 24 hours old. Try again later."
- ZH: "保证金仍被锁定。你的报价不到 24 小时。请稍后再试。"

## Workflow: `!bao claim-rewards`

Claim accumulated reporter rewards from put premiums. 10% of every put premium is split equally among honest fresh reporters.

### Step 1 — Check rewards balance

```bash
WALLET_ADDRESS=<get from Privy>
cast call "$BAOCLAW_ADDRESS" "reporterRewards(address)(uint256)" "$WALLET_ADDRESS" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

If balance is 0:
- EN: "No rewards to claim. Report prices on cards with active put markets to earn rewards."
- ZH: "没有可领取的奖励。为有活跃期权市场的卡牌报价以赚取奖励。"

### Step 2 — Claim via Privy

```bash
CALLDATA=$(cast calldata "claimRewards()")
# Submit via Privy to BAOCLAW_ADDRESS
```

- EN: "Claimed [amount] USDC in reporter rewards."
- ZH: "已领取 [amount] USDC 报价奖励。"

## Workflow: `!bao <card name> put <days>`

### Step 1 — Resolve card and show image

Search pokemontcg.io by name, **show the card image for visual verification** (same as report workflow step 1).

If multiple matches, show top results with images and ask user to pick.

### Step 2 — Fetch on-chain median price and quote premium

Once the card is confirmed, use the resolved pokemontcg.io ID to check the on-chain median:

```bash
cast call "$BAOCLAW_ADDRESS" "getMedianPrice(string)(uint256,uint256)" "<resolved_card_id>" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

First value is the median price in USDC (wei), second is the number of fresh reports. Convert price: `cast from-wei <price>`.

If the price is 0 or report count < 3, tell the user:
- EN: "No fresh price available for <name>. Need at least 3 fresh reports (within 24h). You can help by running `!bao <name> report`."
- ZH: "<name> 暂无最新价格。需要至少 3 份最新报价（24 小时内）。你可以通过运行 `!bao <name> report` 来帮助报价。"

Then call `quotePut` to get the protocol-computed premium:

```bash
cast call "$BAOCLAW_ADDRESS" "quotePut(string,uint256)(uint256,uint256)" "<resolved_card_id>" <durationDays> \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

The first return value is the premium in USDC (wei), the second is the floor price. Convert: `cast from-wei <premium>`. This premium is computed by the contract using the √time × utilization kink model.

### Step 3 — Confirm with user

Calculate:
- `bao_burn = premium_usdc × 0.3% = premium_usdc × 0.003`

If days > 30, tell the user:
- EN: "Maximum duration is 30 days."
- ZH: "最长有效期为 30 天。"

Show the user:

**English:**
```
[CARD IMAGE]
Card: <name> (<id>)
Set: <set name> | Collector #: <number>/<total>
Current price: $<price> USDC (median of <count> reports)
Premium: $<premium_usdc> USDC (protocol-computed, √time × utilization)
$BAO burn: <bao_burn> BAO (0.3% of premium)
Floor protected: $<price> USDC
Duration: <days> days
💰 Total cost: $<premium_usdc> USDC + <bao_burn> BAO
💰 Max payout if price drops to $0: $<price> USDC (net profit: $<price - premium_usdc>)
💰 Break-even: price drops below $<price - premium_usdc * (price / price)> within <days> days
```

**Chinese:**
```
[卡牌图片]
卡牌：<name>（<id>）
系列：<set name> | 编号：<number>/<total>
当前价格：$<price> USDC（<count> 份报价的中位数）
权利金：$<premium_usdc> USDC（协议计算，√时间 × 利用率）
$BAO 销毁：<bao_burn> BAO（权利金的 0.3%）
保护价：$<price> USDC
有效期：<days> 天
💰 总成本：$<premium_usdc> USDC + <bao_burn> BAO
💰 最大补偿（价格跌至 $0）：$<price> USDC（净利润：$<price - premium_usdc>）
💰 回本点：价格在 <days> 天内跌破 $<price - premium_usdc>
```

Ask user to confirm.

### Step 4 — Approve $BAO and USDC, then buy

Three transactions via Privy. Use the **resolved pokemontcg.io ID** for the on-chain call.

1. **Approve $BAO** for burn:

```bash
BAO_BURN=$(cast to-wei "<bao_burn>")
CALLDATA=$(cast calldata "approve(address,uint256)" "$BAOCLAW_ADDRESS" "$BAO_BURN")
# Submit via Privy to BAO_TOKEN_ADDRESS
```

2. **Approve USDC** for premium:

```bash
PREMIUM=$(cast to-wei "<premium_usdc>")
CALLDATA=$(cast calldata "approve(address,uint256)" "$BAOCLAW_ADDRESS" "$PREMIUM")
# Submit via Privy to USDC_ADDRESS
```

3. **Buy put**:

```bash
# Use the resolved pokemontcg.io ID, NOT the natural language name
CALLDATA=$(cast calldata "buyPut(string,uint256)" "<resolved_card_id>" <durationDays>)
# Submit via Privy to BAOCLAW_ADDRESS
```

### Step 5 — Confirm result

- EN: "Put #[id] purchased. Your [name] ([card_id]) is protected at $[floor] for <days> days. [bao_burn] $BAO burned. If the price drops, report the new price with `!bao [name] report` then run `!bao [id] exercise`."
- ZH: "期权 #[id] 已购买。你的 [name]（[card_id]）以 $[floor] 保护 <days> 天。已销毁 [bao_burn] $BAO。如果价格下跌，先用 `!bao [name] report` 报告新价格，然后运行 `!bao [id] exercise`。"

## Workflow: `!bao <card name> price`

Show a card's current market price from multiple sources and the on-chain median (if available). No transaction, no bond, no $BAO — read-only.

### Step 1 — Resolve card and fetch market prices

Search pokemontcg.io by name. The API response includes TCGPlayer market prices. **Show the card image.**

```bash
curl -s "https://api.pokemontcg.io/v2/cards?q=name:\"<card name>\"&orderBy=-set.releaseDate&pageSize=5" \
  -H "X-Api-Key: ${POKEMONTCG_API_KEY:-}" | jq '.data[] | {id, name, set: .set.name, series: .set.series, number, images, tcgplayer: .tcgplayer.prices}'
```

Extract TCGPlayer prices from the response. The `tcgplayer.prices` object contains variants like `holofoil`, `reverseHolofoil`, `normal`, etc. Each has `market`, `low`, `mid`, `high`. Use the `market` price from the most relevant variant (prefer `holofoil` > `reverseHolofoil` > `normal`).

### Step 2 — Fetch on-chain median (if available)

```bash
cast call "$BAOCLAW_ADDRESS" "getMedianPrice(string)(uint256,uint256)" "<resolved_card_id>" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

### Step 3 — Display combined price view

**English:**
```
[CARD IMAGE]
Card: <name> (<id>)
Set: <set name> (<series>) | #<number>

📊 Market Prices:
  TCGPlayer:  $<market_price> (low: $<low> / mid: $<mid> / high: $<high>)

⛓️ On-chain:   $<median> USDC (median of <count> fresh reports)

No cost to check prices. To protect this card's value, run `!bao <name> put <days>`.
```

**Chinese:**
```
[卡牌图片]
卡牌：<name>（<id>）
系列：<set name>（<series>）| #<number>

📊 市场价格：
  TCGPlayer：  $<market_price>（低: $<low> / 中: $<mid> / 高: $<high>）

⛓️ 链上价格：   $<median> USDC（<count> 份最新报价的中位数）

查看价格无需任何费用。如需保护该卡牌价值，运行 `!bao <name> put <days>`。
```

If on-chain count < 3:
- EN: "⛓️ On-chain: insufficient reports ([count]/3). Price not yet available for puts. Help by running `!bao <name> report`."
- ZH: "⛓️ 链上价格：报价不足（[count]/3）。尚无法购买期权。运行 `!bao <name> report` 来参与报价。"

If on-chain count is 0 (no reports at all):
- EN: "⛓️ On-chain: no reports yet. Be the first reporter — run `!bao <name> report` to earn 10% of future premiums."
- ZH: "⛓️ 链上价格：暂无报价。成为第一个报价者 — 运行 `!bao <name> report` 赚取未来权利金的 10%。"

## Workflow: `!bao <putId> exercise`

### Step 1 — Fetch put details and current price

```bash
cast call "$BAOCLAW_ADDRESS" "getPut(uint256)(address,string,uint256,uint256,uint256,bool)" <putId> \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

Then fetch current median price for the card.

### Step 2 — Resolve card and show image

Fetch the card details from pokemontcg.io using the card ID from the put. **Show the card image** so the user can verify.

### Step 3 — Check if user has a fresh report

The exerciser **must** have a fresh report for the card (skin in the game). Check:

```bash
WALLET_ADDRESS=<get from Privy>
cast call "$BAOCLAW_ADDRESS" "getReport(string,address)(uint256,uint256,uint256)" "<card_id_from_put>" "$WALLET_ADDRESS" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

If the report timestamp is 0 or older than 24h, tell the user they must report first:
- EN: "You must report a fresh price before exercising. Run `!bao <card name> report` first."
- ZH: "行权前必须先报告最新价格。请先运行 `!bao <card name> report`。"

### Step 4 — Show calculation and confirm

Compute:
- `payout = floor - current`
- `net_profit = payout - premium_paid` (premium from getPut)
- `report_bond = max(current * 0.01, 10)` (cost to report for exercise)

**English:**
```
[CARD IMAGE]
Put #<id>: <name> (<card_id>)
Set: <set name> | Collector #: <number>/<total>
Floor: $<floor> USDC
Current price: $<current> USDC (median of <count> reports)
Payout: $<payout> USDC
Premium paid: $<premium_paid> USDC
Report bond (refundable): $<report_bond> USDC
💰 Net profit: $<net_profit> USDC (payout minus premium). No $BAO burn on exercise.
```

**Chinese:**
```
[卡牌图片]
期权 #<id>：<name>（<card_id>）
系列：<set name> | 编号：<number>/<total>
保护价：$<floor> USDC
当前价格：$<current> USDC（<count> 份报价的中位数）
补偿金额：$<payout> USDC
已付权利金：$<premium_paid> USDC
报价保证金（可退还）：$<report_bond> USDC
💰 净利润：$<net_profit> USDC（补偿金额减去权利金）。行权无需销毁 $BAO。
```

Edge cases:
- If `current >= floor`: EN: "Price hasn't dropped below your floor. Nothing to claim." / ZH: "价格未跌破你的保护价。无法领取补偿。"
- If expired: EN: "This put expired on [date]." / ZH: "该期权已于 [date] 到期。"
- If already exercised: EN: "This put has already been exercised." / ZH: "该期权已被行权。"

### Step 5 — Execute via Privy

No $BAO burn required on exercise — payouts must never be blocked.

```bash
CALLDATA=$(cast calldata "exercisePut(uint256)" <putId>)
# Submit via Privy to BAOCLAW_ADDRESS
```

## Workflow: `!bao puts`

```bash
WALLET_ADDRESS=<get from Privy>
cast call "$BAOCLAW_ADDRESS" "getHolderPuts(address)(uint256[])" "$WALLET_ADDRESS" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

For each put ID, fetch details and display:

**English:**
```
ID | Card     | Floor   | Premium | BAO Burned | Expires    | Status
0  | sv1-4    | $500.00 | $25.00  | 0.075 BAO  | 2026-06-11 | Active
1  | neo1-9   | $300.00 | $9.00   | 0.027 BAO  | 2026-06-20 | Exercised (+$180)
```

**Chinese:**
```
ID | 卡牌     | 保护价  | 权利金  | BAO 销毁   | 到期日     | 状态
0  | sv1-4    | $500.00 | $25.00  | 0.075 BAO  | 2026-06-11 | 活跃
1  | neo1-9   | $300.00 | $9.00   | 0.027 BAO  | 2026-06-20 | 已行权 (+$180)
```

## Workflow: `!bao pool`

```bash
cast call "$BAOCLAW_ADDRESS" "poolBalance()(uint256)" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
SHARE_PRICE=$(cast call "$BAOCLAW_ADDRESS" "sharePrice()(uint256)" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}")
TOTAL_SHARES=$(cast call "$BAOCLAW_ADDRESS" "totalShares()(uint256)" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}")
```

**English:**
```
Pool balance: $[amount] USDC
Total LP shares: [totalShares]
Share price: $[sharePrice / 1e18] USDC
```

If share price > 1.0: "LPs are in profit — premiums and slashed bonds have accumulated."
If share price < 1.0: "LPs are underwater — payouts exceeded premiums."

**Chinese:**
```
资金池余额：$[amount] USDC
总 LP 份额：[totalShares]
份额价格：$[sharePrice / 1e18] USDC
```

If share price > 1.0: "LP 处于盈利状态 — 权利金和罚没保证金已累积。"
If share price < 1.0: "LP 处于亏损状态 — 补偿金额超过权利金。"

## Workflow: `!bao fund <amount>`

### Step 1 — Calculate and confirm

```bash
SHARE_PRICE=$(cast call "$BAOCLAW_ADDRESS" "sharePrice()(uint256)" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}")
```

Calculate: `shares_received ≈ amount / (sharePrice / 1e18)`

Calculate:
- `bao_burn = amount * 0.003`
- `entry_price = sharePrice` (your cost basis per share)

**English:**
```
Depositing: $<amount> USDC
$BAO burn: <bao_burn> BAO (0.3% of deposit)
Estimated LP shares: ~<shares_received>
Current share price: $<sharePrice> USDC
💰 Entry price: $<sharePrice>/share. You profit when share price rises above $<sharePrice>.
💰 Earnings: 90% of all put premiums + slashed reporter bonds flow into the pool.
💰 Risk: if many puts are exercised (card prices crash), share price drops below entry.
```

**Chinese:**
```
存入：$<amount> USDC
$BAO 销毁：<bao_burn> BAO（存款的 0.3%）
预计 LP 份额：~<shares_received>
当前份额价格：$<sharePrice> USDC
💰 入场价：$<sharePrice>/份额。当份额价格高于 $<sharePrice> 时即为盈利。
💰 收益来源：90% 的期权权利金 + 罚没的报价保证金流入资金池。
💰 风险：如果大量期权被行权（卡牌价格暴跌），份额价格将低于入场价。
```

Ask user to confirm.

### Step 2 — Approve and fund via Privy

1. **Approve $BAO** for burn:

```bash
BAO_BURN=$(cast to-wei "<amount * 0.003>")
CALLDATA=$(cast calldata "approve(address,uint256)" "$BAOCLAW_ADDRESS" "$BAO_BURN")
# Submit via Privy to BAO_TOKEN_ADDRESS
```

2. **Approve USDC**:

```bash
AMOUNT=$(cast to-wei "<amount>")
CALLDATA=$(cast calldata "approve(address,uint256)" "$BAOCLAW_ADDRESS" "$AMOUNT")
# Submit via Privy to USDC_ADDRESS
```

3. **Fund pool**:

```bash
CALLDATA=$(cast calldata "fundPool(uint256)" "$AMOUNT")
# Submit via Privy to BAOCLAW_ADDRESS
```

- EN: "Deposited $[amount] USDC → [shares] LP shares. [bao_burn] $BAO burned."
- ZH: "已存入 $[amount] USDC → [shares] LP 份额。已销毁 [bao_burn] $BAO。"

## Workflow: `!bao withdraw <shares>`

### Step 1 — Calculate and confirm

```bash
SHARE_PRICE=$(cast call "$BAOCLAW_ADDRESS" "sharePrice()(uint256)" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}")
```

Calculate: `usdc_out = shares × (sharePrice / 1e18)`

Calculate:
- `bao_burn = usdc_out * 0.003`
- `net_received = usdc_out` (BAO burn is separate)

**English:**
```
Withdrawing: <shares> LP shares
Estimated payout: $<usdc_out> USDC
$BAO burn: <bao_burn> BAO (0.3% of withdrawal)
Current share price: $<sharePrice> USDC
💰 Net received: $<usdc_out> USDC (plus <bao_burn> BAO burned)
💰 If share price > $1.00: you earned premium income from the pool.
💰 If share price < $1.00: payouts exceeded premiums — withdrawing at a loss.
```

**Chinese:**
```
提取：<shares> LP 份额
预计金额：$<usdc_out> USDC
$BAO 销毁：<bao_burn> BAO（提取金额的 0.3%）
当前份额价格：$<sharePrice> USDC
💰 实际到账：$<usdc_out> USDC（另需销毁 <bao_burn> BAO）
💰 如果份额价格 > $1.00：你从资金池赚取了权利金收入。
💰 如果份额价格 < $1.00：补偿金额超过权利金 — 将亏损提取。
```

Ask user to confirm.

### Step 2 — Approve $BAO for burn, then execute via Privy

```bash
# Approve 0.3% of withdrawal amount in $BAO
BAO_BURN=$(cast to-wei "<usdc_out * 0.003>")
CALLDATA=$(cast calldata "approve(address,uint256)" "$BAOCLAW_ADDRESS" "$BAO_BURN")
# Submit via Privy to BAO_TOKEN_ADDRESS

SHARES=$(cast to-wei "<shares>")
CALLDATA=$(cast calldata "withdrawPool(uint256)" "$SHARES")
# Submit via Privy to BAOCLAW_ADDRESS
```

Burns 0.3% of withdrawn amount in $BAO.

- EN: "Withdrew [shares] LP shares → $[usdc_out] USDC. [bao_burn] $BAO burned."
- ZH: "已提取 [shares] LP 份额 → $[usdc_out] USDC。已销毁 [bao_burn] $BAO。"

## Workflow: `!bao shares`

```bash
WALLET_ADDRESS=<get from Privy>
MY_SHARES=$(cast call "$BAOCLAW_ADDRESS" "shares(address)(uint256)" "$WALLET_ADDRESS" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}")
SHARE_PRICE=$(cast call "$BAOCLAW_ADDRESS" "sharePrice()(uint256)" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}")
```

Calculate: `value = myShares × (sharePrice / 1e18)`

**English:**
```
Your LP shares: [myShares]
Share price: $[sharePrice] USDC
Total value: $[value] USDC
```

**Chinese:**
```
你的 LP 份额：[myShares]
份额价格：$[sharePrice] USDC
总价值：$[value] USDC
```

## Workflow: `!bao buy <amount>`

Buy $BAO tokens with BNB via PancakeSwap V2 on BSC.

### Constants

```
PANCAKE_ROUTER=0x10ED43C718714eb63d5aA57B78B54704E256024E
WBNB=0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c
BAO=0x67777b71A41eebab7D3aD06498D9d48669025873
```

### Step 1 — Get quote

```bash
BAO_AMOUNT=$(cast to-wei "<amount>")
# getAmountsIn returns [BNB needed, BAO out] for the WBNB→BAO path
AMOUNTS=$(cast call "0x10ED43C718714eb63d5aA57B78B54704E256024E" \
  "getAmountsIn(uint256,address[])(uint256[])" \
  "$BAO_AMOUNT" \
  "[0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c,0x67777b71A41eebab7D3aD06498D9d48669025873]" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}")
```

The first value is the BNB required. Convert: `cast from-wei <bnb_amount>`.

Add 1% slippage buffer: `bnb_max = bnb_required × 1.01`

### Step 2 — Confirm

**English:**
```
Buying: <amount> $BAO
Cost: ~<bnb_required> BNB (+ 1% slippage buffer)
Max BNB: <bnb_max> BNB
Route: BNB → $BAO (PancakeSwap V2)
```

**Chinese:**
```
购买：<amount> $BAO
费用：~<bnb_required> BNB（+ 1% 滑点缓冲）
最大 BNB：<bnb_max> BNB
路由：BNB → $BAO（PancakeSwap V2）
```

Ask user to confirm. If user has insufficient BNB, tell them:
- EN: "Insufficient BNB. You need at least [bnb_max] BNB."
- ZH: "BNB 不足。你至少需要 [bnb_max] BNB。"

### Step 3 — Execute swap via Privy

```bash
BAO_AMOUNT=$(cast to-wei "<amount>")
DEADLINE=$(($(date +%s) + 300))  # 5 minutes

CALLDATA=$(cast calldata \
  "swapETHForExactTokens(uint256,address[],address,uint256)" \
  "$BAO_AMOUNT" \
  "[0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c,0x67777b71A41eebab7D3aD06498D9d48669025873]" \
  "$WALLET_ADDRESS" \
  "$DEADLINE")

# Submit via Privy — include BNB value
curl -s -X POST "https://auth.privy.io/api/v1/wallets/${PRIVY_WALLET_ID}/rpc" \
  -H "Authorization: Bearer ${PRIVY_ACCESS_TOKEN}" \
  -H "privy-app-id: ${PRIVY_APP_ID}" \
  -H "Content-Type: application/json" \
  -d "{
    \"method\": \"eth_sendTransaction\",
    \"params\": {
      \"transaction\": {
        \"to\": \"0x10ED43C718714eb63d5aA57B78B54704E256024E\",
        \"data\": \"${CALLDATA}\",
        \"value\": \"${BNB_MAX_WEI}\",
        \"chain_id\": 56
      }
    }
  }"
```

`swapETHForExactTokens` sends exact BNB as `value`, receives exact BAO amount. Any unused BNB is refunded automatically by the router.

### Step 4 — Confirm result

- EN: "Bought [amount] $BAO for ~[bnb_spent] BNB on PancakeSwap."
- ZH: "已通过 PancakeSwap 购买 [amount] $BAO，花费约 [bnb_spent] BNB。"

## Workflow: `!bao help`

Show available commands in the user's language:

**English:**
```
BaoClaw — Pokemon TCG Price Protection 🐾

Reporting:
  !bao <card name> report          Report a card's price (1% bond, burns 0.3% $BAO)
  !bao <card name> claim-bond      Reclaim bond after 24h
  !bao claim-rewards               Claim reporter rewards (10% of premiums)
  !bao <card name> price           Check current median price

Put Options:
  !bao <card name> put <days>      Buy protection (premium computed by protocol)
  !bao <card name> quote <days>    Quote premium for protection
  !bao <putId> exercise            Exercise a put (must report first)
  !bao puts                        List my puts

Pool (LP):
  !bao pool                        Pool balance and share price
  !bao fund <amount>               Deposit USDC → receive LP shares (burns 0.3% in $BAO)
  !bao withdraw <shares>           Burn LP shares → receive USDC
  !bao shares                      Check my LP shares and value

Token:
  !bao buy <amount>                Buy $BAO with BNB via PancakeSwap

Search:
  !bao search <card name>          Search for a card (show matches with images)

Cards: use natural language names (e.g., "Charizard ex Scarlet Violet")
Prices: crowdsourced from reporters (median of 3+ fresh reports)
$BAO: 0x67777b71A41eebab7D3aD06498D9d48669025873
```

**Chinese:**
```
BaoClaw — 宝可梦卡牌价格保护 🐾

报价：
  !bao <卡牌名称> report           报告卡牌价格（1% 保证金，销毁 0.3% $BAO）
  !bao <卡牌名称> claim-bond       24 小时后领回保证金
  !bao claim-rewards               领取报价奖励（权利金的 10%）
  !bao <卡牌名称> price            查看当前众包价格

看跌期权：
  !bao <卡牌名称> put <天数>        购买保护（权利金由协议计算）
  !bao <卡牌名称> quote <天数>      查询保护权利金
  !bao <期权ID> exercise           行权（必须先报价）
  !bao puts                        列出我的期权

资金池（LP）：
  !bao pool                        资金池余额和份额价格
  !bao fund <金额>                 存入 USDC → 获得 LP 份额（销毁 0.3% 的 $BAO）
  !bao withdraw <份额数>           销毁 LP 份额 → 获得 USDC
  !bao shares                      查看我的 LP 份额和价值

代币：
  !bao buy <数量>                  用 BNB 通过 PancakeSwap 购买 $BAO

搜索：
  !bao search <卡牌名称>           搜索卡牌（显示匹配结果和图片）

卡牌：使用自然语言名称（如"喷火龙 ex 朱与紫"）
价格：由报价者众包（3+ 份最新报价的中位数）
$BAO：0x67777b71A41eebab7D3aD06498D9d48669025873
```

## Workflow: `!bao search <card name>`

Search for a card by natural language name and display matching results with images.

```bash
curl -s "https://api.pokemontcg.io/v2/cards?q=name:\"<card name>\"&orderBy=-set.releaseDate&pageSize=5" \
  -H "X-Api-Key: ${POKEMONTCG_API_KEY:-}" | jq '.data[] | {id, name, set: .set.name, series: .set.series, number, images, tcgplayer: .tcgplayer.prices}'
```

Show the top 5 matches with:
- Card image (MANDATORY)
- Name, set, series
- Collector number
- pokemontcg.io ID
- TCGPlayer market price (if available)

**English:**
```
Found 3 matches for "Charizard ex":

1. [IMAGE] Charizard ex (sv1-4)
   Scarlet & Violet | #4/198 | $498.00

2. [IMAGE] Charizard ex (sv3pt5-189)
   151 | #189/165 | $312.50

3. [IMAGE] Charizard ex (sv4-218)
   Paradox Rift | #218/182 | $89.00
```

**Chinese:**
```
找到 3 个"喷火龙 ex"的匹配结果：

1. [图片] 喷火龙 ex（sv1-4）
   朱与紫 | #4/198 | $498.00

2. [图片] 喷火龙 ex（sv3pt5-189）
   151 | #189/165 | $312.50

3. [图片] 喷火龙 ex（sv4-218）
   Paradox Rift | #218/182 | $89.00
```

## Guardrails

- **NEVER** log, display, or store private keys or Privy secrets
- **ALWAYS** show the card image before any put purchase, price report, or exercise — this is MANDATORY for visual verification
- **ALWAYS** show premium cost, $BAO burn, and floor price before buying
- **ALWAYS** confirm before any transaction
- **NEVER** approve unlimited token amounts — approve exact amounts only
- **ALWAYS** resolve natural language card names to pokemontcg.io IDs before on-chain calls
- If multiple card matches are found, show them with images and ask user to pick
- **ALWAYS** call `quotePut` before `buyPut` to show the user the protocol-computed premium
- **ALWAYS** call `quoteBond` before `reportPrice` to show the user the bond amount (1% of price, min 10 USDC)
- If no fresh price (< 3 reports), explain that more reporters are needed and suggest `!bao <card name> report`
- When exercising, check that the user has a fresh report first. If not, guide them to report
- When exercising, show the full payout calculation
- If user has insufficient $BAO, suggest `!bao buy <amount>` to purchase via PancakeSwap
- For `!bao buy`, always use 1% slippage and a 5-minute deadline
- For `!bao buy`, show the BNB cost and confirm before executing
- For `!bao report`, show all available price sources and the median before confirming
- The pokemontcg.io API key lives on the user's machine — never transmit it elsewhere

## Examples

### English Examples

**User:** `!bao Charizard ex Scarlet Violet report`
1. Search pokemontcg.io → resolves to sv1-4
2. Show card image: "[IMAGE] Charizard ex (sv1-4) — Scarlet & Violet — #4/198. Is this the right card?"
3. User confirms
4. Scrape prices: TCGPlayer $502, PriceCharting $495, eBay Sold $498, Alt.xyz N/A, Collectr $500
5. "Reporting Charizard ex (sv1-4): TCGPlayer $502, PriceCharting $495, eBay Sold $498, Collectr $500. Median: $499. Bond: 10 USDC (1% of $499 = $4.99, min $10 — fully refundable). $BAO burn: 0.03 BAO. Net cost: 0.03 BAO only. You earn a share of 10% of every put premium on this card. Proceed?"
6. On confirm: approve 0.03 BAO, approve 10 USDC, call `reportPrice("sv1-4", 499e18)`
7. "Price reported: Charizard ex (sv1-4) @ $499 (4 sources). Bond: 10 USDC locked."

**User:** `!bao Charizard ex Scarlet Violet put 14`
1. Search pokemontcg.io → resolves to sv1-4
2. Show card image: "[IMAGE] Charizard ex (sv1-4) — Scarlet & Violet — #4/198. Is this the right card?"
3. User confirms
4. Fetch on-chain median: `getMedianPrice("sv1-4")` → $500 USDC (5 reports)
5. Quote premium: `quotePut("sv1-4", 14)` → $37.42 USDC
6. "[IMAGE] Charizard ex (sv1-4). Price: $500 (5 reports). Premium: $37.42 USDC (protocol-computed, √time × utilization). $BAO burn: 0.112 BAO. Protected at $500 for 14 days. Total cost: $37.42 USDC + 0.112 BAO. Max payout if price drops to $0: $500 (net profit: $462.58). Break-even: price drops below $462.58. Proceed?"
7. On confirm: approve 0.112 BAO, approve $37.42 USDC, call `buyPut("sv1-4", 14)`
8. "Put #0 purchased. 0.112 $BAO burned. Protected at $500 for 14 days — max net profit $462.58 if price drops to $0. If price drops, report with `!bao Charizard ex Scarlet Violet report` then `!bao 0 exercise`."

**User:** `!bao 0 exercise`
1. Fetch put #0: floor=$500, card=sv1-4
2. Resolve sv1-4 → show card image: "[IMAGE] Charizard ex (sv1-4)"
3. Check user's report: fresh report exists at $200
4. Fetch median price: $200
5. "[IMAGE] Charizard ex (sv1-4). Price dropped from $500 to $200. Payout: $300 USDC. Premium paid: $25. No $BAO burn on exercise. Net profit: $275 USDC. Proceed?"
6. On confirm: `exercisePut(0)`
7. "Exercised. $300 USDC sent to your wallet. Net profit: $275 (payout $300 minus premium $25)."

**User:** `!bao fund 1000`
1. Fetch share price: $1.025 USDC (pool has grown from premiums + slashed bonds)
2. "Depositing $1,000 USDC. $BAO burn: 3 BAO. You'll receive ~975.6 LP shares at $1.025/share. Entry price: $1.025/share — you profit when share price rises above $1.025. Earnings: 90% of all put premiums + slashed bonds. Risk: share price drops if many puts are exercised. Proceed?"
3. On confirm: approve 3 BAO, approve $1,000 USDC, call `fundPool(1000e18)`
4. "Deposited $1,000 USDC → 975.6 LP shares at $1.025/share. 3 $BAO burned. You'll profit as premiums accumulate."

**User:** `!bao withdraw 500`
1. Fetch share price: $1.05 USDC (premiums accumulated)
2. "Withdrawing 500 LP shares → ~$525 USDC. $BAO burn: 1.575 BAO. Share price $1.05 > $1.00 — you earned premium income. Net received: $525 USDC. Proceed?"
3. On confirm: approve 1.575 BAO, `withdrawPool(500e18)`
4. "Withdrew 500 LP shares → $525 USDC. 1.575 $BAO burned. Net profit: $25 above initial $500 deposit."

**User:** `!bao Pikachu VMAX price`
1. Search pokemontcg.io → resolves to swsh4-44
2. Show card image: "[IMAGE] Pikachu VMAX (swsh4-44) — Vivid Voltage — #44/185. TCGPlayer: $215 (low: $190 / mid: $210 / high: $240)."
3. Fetch on-chain: `getMedianPrice("swsh4-44")` → $212 USDC (4 reports)
4. "On-chain: $212 USDC (4 reports). No cost to check. To protect this card, run `!bao Pikachu VMAX put <days>`."

**User:** `!bao shares`
→ "You have 975.6 LP shares worth $1,024.38 USDC (share price: $1.05)"

**User:** `!bao buy 100`
1. Quote: `getAmountsIn(100e18, [WBNB, BAO])` → 0.15 BNB needed
2. "Buy 100 $BAO for ~0.15 BNB (max 0.1515 BNB with 1% slippage). Proceed?"
3. On confirm: `swapETHForExactTokens` via PancakeSwap Router with 0.1515 BNB
4. "Bought 100 $BAO for 0.149 BNB on PancakeSwap."

### 中文示例

**用户：** `!bao 喷火龙 ex 朱与紫 report`
1. 搜索 pokemontcg.io → 解析为 sv1-4
2. 显示卡牌图片："[图片] 喷火龙 ex（sv1-4）— 朱与紫 — #4/198。这是正确的卡牌吗？"
3. 用户确认
4. 抓取价格：TCGPlayer $502，PriceCharting $495，eBay Sold $498，Collectr $500
5. "为喷火龙 ex（sv1-4）报价：TCGPlayer $502，PriceCharting $495，eBay Sold $498，Collectr $500。中位数：$499。保证金：10 USDC（$499 的 1% = $4.99，最低 $10 — 全额可退还）。$BAO 销毁：0.03 BAO。净成本：仅 0.03 BAO（保证金 100% 可退还）。收益：该卡牌每笔期权权利金的 10% 将分配给你。确认？"
6. 确认后：批准 0.03 BAO，批准 10 USDC，调用 `reportPrice("sv1-4", 499e18)`
7. "已报价：喷火龙 ex（sv1-4）@ $499（4 个来源）。保证金：10 USDC 已锁定。"

**用户：** `!bao 喷火龙 ex 朱与紫 put 14`
1. 搜索 pokemontcg.io → 解析为 sv1-4
2. 显示卡牌图片："[图片] 喷火龙 ex（sv1-4）— 朱与紫 — #4/198。这是正确的卡牌吗？"
3. 用户确认
4. 获取链上中位数：`getMedianPrice("sv1-4")` → $500 USDC（5 份报价）
5. 查询权利金：`quotePut("sv1-4", 14)` → $37.42 USDC
6. "[图片] 喷火龙 ex（sv1-4）。价格：$500（5 份报价）。权利金：$37.42 USDC（协议计算，√时间 × 利用率）。$BAO 销毁：0.112 BAO。以 $500 保护 14 天。总成本：$37.42 USDC + 0.112 BAO。最大补偿（价格跌至 $0）：$500（净利润：$462.58）。回本点：价格跌破 $462.58。确认？"
7. 确认后：批准 0.112 BAO，批准 $37.42 USDC，调用 `buyPut("sv1-4", 14)`
8. "期权 #0 已购买。已销毁 0.112 $BAO。以 $500 保护 14 天 — 价格跌至 $0 时最大净利润 $462.58。如果价格下跌，先运行 `!bao 喷火龙 ex 朱与紫 report` 然后 `!bao 0 exercise`。"

**用户：** `!bao 0 exercise`
1. 获取期权 #0：保护价=$500，卡牌=sv1-4
2. 解析 sv1-4 → 显示卡牌图片："[图片] 喷火龙 ex（sv1-4）"
3. 检查用户报价：存在 $200 的最新报价
4. 获取中位数价格：$200
5. "[图片] 喷火龙 ex（sv1-4）。价格从 $500 跌至 $200。补偿金额：$300 USDC。已付权利金：$25。行权无需 $BAO。净利润：$275 USDC。确认？"
6. 确认后：`exercisePut(0)`
7. "已行权。$300 USDC 已发送到你的钱包。净利润：$275（补偿 $300 减去权利金 $25）。"

**用户：** `!bao fund 1000`
1. 获取份额价格：$1.025 USDC（资金池因权利金和罚没保证金增长）
2. "存入 $1,000 USDC。$BAO 销毁：3 BAO。你将获得约 975.6 LP 份额，份额价格 $1.025。入场价：$1.025/份额 — 当份额价格高于 $1.025 时即为盈利。收益来源：90% 的权利金 + 罚没保证金。风险：大量行权时份额价格下跌。确认？"
3. 确认后：批准 3 BAO，批准 $1,000 USDC，调用 `fundPool(1000e18)`
4. "已存入 $1,000 USDC → 975.6 LP 份额，入场价 $1.025/份额。已销毁 3 $BAO。权利金累积时你将获利。"

**用户：** `!bao withdraw 500`
1. 获取份额价格：$1.05 USDC（权利金已累积）
2. "提取 500 LP 份额 → 约 $525 USDC。$BAO 销毁：1.575 BAO。份额价格 $1.05 > $1.00 — 你已赚取权利金收入。实际到账：$525 USDC。确认？"
3. 确认后：批准 1.575 BAO，`withdrawPool(500e18)`
4. "已提取 500 LP 份额 → $525 USDC。已销毁 1.575 $BAO。净利润：比初始 $500 存款多 $25。"

**用户：** `!bao 皮卡丘 VMAX price`
1. 搜索 pokemontcg.io → 解析为 swsh4-44
2. 显示卡牌图片："[图片] 皮卡丘 VMAX（swsh4-44）— 耀光活力 — #44/185。TCGPlayer：$215（低: $190 / 中: $210 / 高: $240）。"
3. 获取链上价格：`getMedianPrice("swsh4-44")` → $212 USDC（4 份报价）
4. "链上价格：$212 USDC（4 份报价）。查看免费。如需保护该卡牌，运行 `!bao 皮卡丘 VMAX put <天数>`。"

**用户：** `!bao shares`
→ "你有 975.6 LP 份额，价值 $1,024.38 USDC（份额价格：$1.05）"

**用户：** `!bao buy 100`
1. 报价：`getAmountsIn(100e18, [WBNB, BAO])` → 需要 0.15 BNB
2. "购买 100 $BAO，约需 0.15 BNB（最大 0.1515 BNB，含 1% 滑点）。确认？"
3. 确认后：通过 PancakeSwap Router 执行 `swapETHForExactTokens`，发送 0.1515 BNB
4. "已通过 PancakeSwap 购买 100 $BAO，花费 0.149 BNB。"
