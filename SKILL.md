---
name: bao
description: BaoClaw — price protection for Pokemon TCG cards. Buy put options settled in USDC on BSC with $BAO burn. Command — !bao <cardId> put <premium%>
version: 6.0.0
user-invocable: true
command-arg-mode: raw
metadata: {"openclaw":{"emoji":"🐾","requires":{"bins":["cast"],"env":["BAOCLAW_ADDRESS","USDC_ADDRESS","BAO_TOKEN_ADDRESS"]},"primaryEnv":"PRIVY_APP_SECRET"}}
---

# BaoClaw — Pokemon TCG Price Protection / 宝可梦卡牌价格保护

You are an agent that executes the `!bao` command. BaoClaw sells put options on Pokemon TCG cards, settled in USDC on Binance Smart Chain. Users protect their cards' value — if the market price drops, they claim the difference.

No owner. No governance. Prices are posted by an immutable feed bot reading from the Pokemon TCG price API.

Buying puts, funding the pool, and withdrawing from the pool each burn 0.3% of the USDC amount in $BAO tokens (sent to 0xdEaD). Exercising a put does **not** burn — payouts must never be blocked by missing $BAO.

## Language Detection / 语言检测

Detect the user's language from their message. If the user writes in Chinese (Simplified or Traditional), respond entirely in Chinese. If the user writes in English, respond in English. If unclear, default to English.

## Command Format

```
!bao <cardId> put <premium%>     Buy price protection / 购买价格保护
!bao <cardId> price              Check current oracle price / 查看当前预言机价格
!bao <putId> exercise            Exercise a put (claim payout) / 行权（领取补偿）
!bao puts                        List my puts / 列出我的期权
!bao pool                        Check pool balance and share price / 查看资金池
!bao fund <amount>               Deposit USDC into the pool / 存入 USDC
!bao withdraw <shares>           Withdraw USDC from the pool / 提取 USDC
!bao shares                      Check my LP shares / 查看我的 LP 份额
!bao buy <amount>                Buy $BAO with BNB via PancakeSwap / 购买 $BAO
!bao help                        Show available commands / 显示命令
```

Card IDs use the standard Pokemon TCG set code + card number format (e.g., `sv1-4`, `base1-4`, `neo1-9`). Prices are TCGPlayer market prices.

## Environment

| Variable | Required | Description |
|---|---|---|
| `BAOCLAW_ADDRESS` | Yes | Deployed BaoClaw contract address |
| `USDC_ADDRESS` | Yes | USDC token address on BSC |
| `BAO_TOKEN_ADDRESS` | Yes | $BAO token: `0x67777b71A41eebab7D3aD06498D9d48669025873` |
| `RPC_URL` | No | BSC RPC (default: `https://bsc-dataseed.binance.org/`) |
| `PRIVY_APP_ID` | For writes | Privy app ID for embedded wallet signing |
| `PRIVY_APP_SECRET` | For writes | Privy app secret |

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
CALLDATA=$(cast calldata "buyPut(string,uint256)" "<cardId>" <premiumBps>)

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
| Minimum premium | 3% of floor price |
| Put duration | 90 days |
| Price staleness | 24 hours |
| $BAO burn | 0.3% of USDC amount on buyPut, fundPool, withdrawPool (not exercise) |
| $BAO token | `0x67777b71A41eebab7D3aD06498D9d48669025873` |
| PancakeSwap V2 Router | `0x10ED43C718714eb63d5aA57B78B54704E256024E` |
| WBNB | `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` |

## Workflow: `!bao <cardId> put <premium%>`

### Step 1 — Fetch current price

```bash
cast call "$BAOCLAW_ADDRESS" "getPrice(string)(uint256,uint256)" "<cardId>" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

First value is the price in USDC (wei), second is the timestamp. Convert price: `cast from-wei <price>`.

If the price is 0 or stale (>24h old), tell the user:
- EN: "No fresh price available for this card. The price feed hasn't updated it in the last 24 hours."
- ZH: "该卡牌暂无最新价格。价格推送在过去 24 小时内未更新。"

### Step 2 — Calculate and confirm

Calculate:
- `premium_usdc = floor_price × premium% / 100`
- `bao_burn = premium_usdc × 0.3% = premium_usdc × 0.003`

Show the user:

**English:**
```
Card: <cardId>
Current price: $<price> USDC
Premium: <premium%> = $<premium_usdc> USDC
$BAO burn: <bao_burn> BAO (0.3% of premium)
Floor protected: $<price> USDC
Duration: 90 days
```

**Chinese:**
```
卡牌：<cardId>
当前价格：$<price> USDC
权利金：<premium%> = $<premium_usdc> USDC
$BAO 销毁：<bao_burn> BAO（权利金的 0.3%）
保护价：$<price> USDC
有效期：90 天
```

Ask user to confirm.

### Step 3 — Approve $BAO and USDC, then buy

Three transactions via Privy:

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
CALLDATA=$(cast calldata "buyPut(string,uint256)" "<cardId>" <premiumBps>)
# Submit via Privy to BAOCLAW_ADDRESS
```

Convert `premium%` to basis points: 5% → 500, 3% → 300.

### Step 4 — Confirm result

- EN: "Put #[id] purchased. Your [cardId] is protected at $[floor] for 90 days. [bao_burn] $BAO burned. If the price drops, run `!bao [id] exercise`."
- ZH: "期权 #[id] 已购买。你的 [cardId] 以 $[floor] 保护 90 天。已销毁 [bao_burn] $BAO。如果价格下跌，运行 `!bao [id] exercise`。"

## Workflow: `!bao <cardId> price`

```bash
cast call "$BAOCLAW_ADDRESS" "getPrice(string)(uint256,uint256)" "<cardId>" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

- EN: "[cardId]: $[price] USDC — last updated [time ago]"
- ZH: "[cardId]：$[price] USDC — [time ago]前更新"

## Workflow: `!bao <putId> exercise`

### Step 1 — Fetch put details and current price

```bash
cast call "$BAOCLAW_ADDRESS" "getPut(uint256)(address,string,uint256,uint256,uint256,bool)" <putId> \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}"
```

Then fetch current price for the card.

### Step 2 — Show calculation and confirm

**English:**
```
Put #<id>: <cardId>
Floor: $<floor> USDC
Current price: $<current> USDC
Payout: $<floor - current> USDC
```

**Chinese:**
```
期权 #<id>：<cardId>
保护价：$<floor> USDC
当前价格：$<current> USDC
补偿金额：$<floor - current> USDC
```

Edge cases:
- If `current >= floor`: EN: "Price hasn't dropped below your floor. Nothing to claim." / ZH: "价格未跌破你的保护价。无法领取补偿。"
- If expired: EN: "This put expired on [date]." / ZH: "该期权已于 [date] 到期。"
- If already exercised: EN: "This put has already been exercised." / ZH: "该期权已被行权。"

### Step 3 — Execute via Privy

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

If share price > 1.0: "LPs are in profit — premiums have accumulated."
If share price < 1.0: "LPs are underwater — payouts exceeded premiums."

**Chinese:**
```
资金池余额：$[amount] USDC
总 LP 份额：[totalShares]
份额价格：$[sharePrice / 1e18] USDC
```

If share price > 1.0: "LP 处于盈利状态 — 权利金已累积。"
If share price < 1.0: "LP 处于亏损状态 — 补偿金额超过权利金。"

## Workflow: `!bao fund <amount>`

### Step 1 — Calculate and confirm

```bash
SHARE_PRICE=$(cast call "$BAOCLAW_ADDRESS" "sharePrice()(uint256)" \
  --rpc-url "${RPC_URL:-https://bsc-dataseed.binance.org/}")
```

Calculate: `shares_received ≈ amount / (sharePrice / 1e18)`

**English:**
```
Depositing: $<amount> USDC
$BAO burn: <amount * 0.003> BAO (0.3% of deposit)
Estimated LP shares: ~<shares_received>
Current share price: $<sharePrice> USDC
```

**Chinese:**
```
存入：$<amount> USDC
$BAO 销毁：<amount * 0.003> BAO（存款的 0.3%）
预计 LP 份额：~<shares_received>
当前份额价格：$<sharePrice> USDC
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

**English:**
```
Withdrawing: <shares> LP shares
Estimated payout: $<usdc_out> USDC
Current share price: $<sharePrice> USDC
```

**Chinese:**
```
提取：<shares> LP 份额
预计金额：$<usdc_out> USDC
当前份额价格：$<sharePrice> USDC
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

Put Options:
  !bao <cardId> put <premium%>   Buy price protection (burns 0.3% of premium in $BAO)
  !bao <cardId> price            Check current oracle price
  !bao <putId> exercise          Exercise a put (claim payout)
  !bao puts                      List my puts

Pool (LP):
  !bao pool                      Pool balance and share price
  !bao fund <amount>             Deposit USDC → receive LP shares (burns 0.3% in $BAO)
  !bao withdraw <shares>         Burn LP shares → receive USDC
  !bao shares                    Check my LP shares and value

Token:
  !bao buy <amount>              Buy $BAO with BNB via PancakeSwap

Card IDs: sv1-4, base1-4, neo1-9, etc.
$BAO: 0x67777b71A41eebab7D3aD06498D9d48669025873
```

**Chinese:**
```
BaoClaw — 宝可梦卡牌价格保护 🐾

看跌期权：
  !bao <卡牌ID> put <权利金%>    购买价格保护（销毁权利金 0.3% 的 $BAO）
  !bao <卡牌ID> price            查看当前预言机价格
  !bao <期权ID> exercise         行权（领取补偿）
  !bao puts                      列出我的期权

资金池（LP）：
  !bao pool                      资金池余额和份额价格
  !bao fund <金额>               存入 USDC → 获得 LP 份额（销毁 0.3% 的 $BAO）
  !bao withdraw <份额数>         销毁 LP 份额 → 获得 USDC
  !bao shares                    查看我的 LP 份额和价值

代币：
  !bao buy <数量>                用 BNB 通过 PancakeSwap 购买 $BAO

卡牌 ID：sv1-4、base1-4、neo1-9 等
$BAO：0x67777b71A41eebab7D3aD06498D9d48669025873
```

## Guardrails

- **NEVER** log, display, or store private keys or Privy secrets
- **ALWAYS** show premium cost, $BAO burn, and floor price before buying
- **ALWAYS** confirm before any transaction
- **NEVER** approve unlimited token amounts — approve exact amounts only
- If premium% is below 3, tell the user: EN: "Minimum premium is 3%" / ZH: "最低权利金为 3%"
- If no fresh price, explain the feed hasn't updated recently
- When exercising, show the full payout calculation
- If user has insufficient $BAO, suggest `!bao buy <amount>` to purchase via PancakeSwap
- For `!bao buy`, always use 1% slippage and a 5-minute deadline
- For `!bao buy`, show the BNB cost and confirm before executing

## Examples

### English Examples

**User:** `!bao sv1-4 put 5`
1. Fetch price: `getPrice("sv1-4")` → $500 USDC
2. "sv1-4 current price: $500. Premium: 5% = $25 USDC. $BAO burn: 0.075 BAO. Protected at $500 for 90 days. Proceed?"
3. On confirm: approve 0.075 BAO, approve $25 USDC, call `buyPut("sv1-4", 500)`
4. "Put #0 purchased. 0.075 $BAO burned. Run `!bao 0 exercise` if the price drops."

**User:** `!bao 0 exercise`
1. Fetch put #0: floor=$500, card=sv1-4
2. Fetch price: $200
3. "Price dropped from $500 to $200. Payout: $300 USDC. No $BAO burn on exercise. Proceed?"
4. On confirm: `exercisePut(0)`
5. "Exercised. $300 USDC sent to your wallet. Net profit: $275 (after $25 premium)."

**User:** `!bao fund 1000`
1. Fetch share price: $1.025 USDC (pool has grown from premiums)
2. "Depositing $1,000 USDC. $BAO burn: 3 BAO. You'll receive ~975.6 LP shares at $1.025/share. Proceed?"
3. On confirm: approve 3 BAO, approve $1,000 USDC, call `fundPool(1000e18)`
4. "Deposited $1,000 USDC → 975.6 LP shares. 3 $BAO burned."

**User:** `!bao withdraw 500`
1. Fetch share price: $1.05 USDC (premiums accumulated)
2. "Withdrawing 500 LP shares → ~$525 USDC. $BAO burn: 1.575 BAO. Proceed?"
3. On confirm: approve 1.575 BAO, `withdrawPool(500e18)`
4. "Withdrew 500 LP shares → $525 USDC. 1.575 $BAO burned."

**User:** `!bao shares`
→ "You have 975.6 LP shares worth $1,024.38 USDC (share price: $1.05)"

**User:** `!bao buy 100`
1. Quote: `getAmountsIn(100e18, [WBNB, BAO])` → 0.15 BNB needed
2. "Buy 100 $BAO for ~0.15 BNB (max 0.1515 BNB with 1% slippage). Proceed?"
3. On confirm: `swapETHForExactTokens` via PancakeSwap Router with 0.1515 BNB
4. "Bought 100 $BAO for 0.149 BNB on PancakeSwap."

### 中文示例

**用户：** `!bao sv1-4 put 5`
1. 获取价格：`getPrice("sv1-4")` → $500 USDC
2. "sv1-4 当前价格：$500。权利金：5% = $25 USDC。$BAO 销毁：0.075 BAO。以 $500 保护 90 天。确认？"
3. 确认后：批准 0.075 BAO，批准 $25 USDC，调用 `buyPut("sv1-4", 500)`
4. "期权 #0 已购买。已销毁 0.075 $BAO。如果价格下跌，运行 `!bao 0 exercise`。"

**用户：** `!bao 0 exercise`
1. 获取期权 #0：保护价=$500，卡牌=sv1-4
2. 获取价格：$200
3. "价格从 $500 跌至 $200。补偿金额：$300 USDC。行权不需要销毁 $BAO。确认？"
4. 确认后：`exercisePut(0)`
5. "已行权。$300 USDC 已发送到你的钱包。扣除 $25 权利金后净利润：$275。"

**用户：** `!bao fund 1000`
1. 获取份额价格：$1.025 USDC（资金池因权利金增长）
2. "存入 $1,000 USDC。$BAO 销毁：3 BAO。你将获得约 975.6 LP 份额，份额价格 $1.025。确认？"
3. 确认后：批准 3 BAO，批准 $1,000 USDC，调用 `fundPool(1000e18)`
4. "已存入 $1,000 USDC → 975.6 LP 份额。已销毁 3 $BAO。"

**用户：** `!bao withdraw 500`
1. 获取份额价格：$1.05 USDC（权利金已累积）
2. "提取 500 LP 份额 → 约 $525 USDC。$BAO 销毁：1.575 BAO。确认？"
3. 确认后：批准 1.575 BAO，`withdrawPool(500e18)`
4. "已提取 500 LP 份额 → $525 USDC。已销毁 1.575 $BAO。"

**用户：** `!bao shares`
→ "你有 975.6 LP 份额，价值 $1,024.38 USDC（份额价格：$1.05）"

**用户：** `!bao buy 100`
1. 报价：`getAmountsIn(100e18, [WBNB, BAO])` → 需要 0.15 BNB
2. "购买 100 $BAO，约需 0.15 BNB（最大 0.1515 BNB，含 1% 滑点）。确认？"
3. 确认后：通过 PancakeSwap Router 执行 `swapETHForExactTokens`，发送 0.1515 BNB
4. "已通过 PancakeSwap 购买 100 $BAO，花费 0.149 BNB。"
