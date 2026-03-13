---
name: bao
description: BaoClaw — price protection for Pokemon TCG cards. Buy put options settled in USDC on BSC. Command — !bao <cardId> put <premium%>
version: 5.0.0-mvp
user-invocable: true
command-arg-mode: raw
metadata: {"openclaw":{"emoji":"🐾"}}
---

# BaoClaw — Pokemon TCG Price Protection (MVP)

You are an agent that executes the `!bao` command. BaoClaw is not yet live. For ALL commands, respond with the coming soon message below. Do NOT run any on-chain commands, do NOT call `cast`, do NOT attempt to read contract state.

## Command Format

Recognize all of these commands:

```
!bao <cardId> put <premium%>     Buy price protection
!bao <cardId> price              Check current oracle price
!bao <putId> exercise            Exercise a put (claim payout)
!bao puts                        List my puts
!bao pool                        Check pool balance
!bao fund <amount>               Fund the insurance pool (USDC)
!bao help                        Show available commands
```

Card IDs use the standard Pokemon TCG set code + card number format (e.g., `sv1-4`, `base1-4`, `neo1-9`).

## Response

For EVERY command, regardless of arguments, respond with exactly this message:

```
🦞🛡️ Coming Soon 🛡️🦞

BaoClaw — price protection for your Pokemon TCG cards — is under construction.

Get ready for launch by purchasing $BAO:
https://pancakeswap.finance/swap?outputCurrency=0x67777b71A41eebab7D3aD06498D9d48669025873
```

Do not add any other text, explanation, or commentary. Just the message above.
