---
name: kuru-trading-skills
description: Trade and inspect balances for a predetermined allowlist of Kuru tokens on Monad mainnet. Use when a user asks an agent to buy an allowed token with USDC, sell an allowed token into USDC, swap USDC to or from an allowed token through Kuru Flow, get a quote, list supported tokens, troubleshoot a Kuru trade, or check an allowed token balance with RPC.
---

# Kuru Trading Skills

## Agent Workflow

Use `scripts/kuru-trade.ts` for every Kuru trade, quote, token-list, and balance operation. Do not hand-build Kuru transactions, do not trade tokens outside `scripts/tokens.ts`, and do not invent default amounts.

Before running a live buy/sell, confirm the user supplied:

- an allowed token symbol/name/address/alias
- an explicit amount
- a clear side: buy means USDC -> token, sell means token -> USDC

If any piece is missing, ask for it instead of guessing.

## Commands

Run commands from the skill root:

```bash
bun run trade help
bun run trade tokens
bun run trade buy <token> <usdcAmount>
bun run trade sell <token> <tokenAmount>
bun run trade prepare buy <token> <usdcAmount> --from <address>
bun run trade prepare sell <token> <tokenAmount> --from <address>
bun run trade quote buy <token> <usdcAmount>
bun run trade quote sell <token> <tokenAmount>
bun run trade balance <token>
```

Add `--json` to any command when another program or agent needs machine-readable output.

## Intent Mapping

- "buy X", "buy X token", "swap USDC to X": run `bun run trade buy <X> <USDC amount>` only after the amount is explicit.
- "sell X", "swap X to USDC": run `bun run trade sell <X> <token amount>` only after the amount is explicit.
- "prepare buying X" or "return unsigned transactions": run `bun run trade prepare buy <X> <USDC amount> --from <wallet> --json`.
- "prepare selling X": run `bun run trade prepare sell <X> <token amount> --from <wallet> --json`.
- "quote buying X" or "how much X for Y USDC": run `bun run trade quote buy <X> <USDC amount>`.
- "quote selling X": run `bun run trade quote sell <X> <token amount>`.
- "what's my X balance": run `bun run trade balance <X>`.
- "what tokens can I trade": run `bun run trade tokens`.

Live `buy` and `sell` commands auto-broadcast when token and amount are explicit. Use `quote` when the user asks for an estimate or when the request is not an explicit instruction to trade.

Use `prepare` when signing happens outside this skill. Prepare mode returns ordered unsigned transaction requests. If allowance is insufficient, the response includes an approval transaction first, followed by the swap. The external signer should submit them in order and wait for the approval to mine before submitting the swap.

## Runtime

Required environment for live `buy`, `sell`, `quote`, and `balance` commands:

```bash
export KURU_PRIVATE_KEY="0x..."
export MONAD_RPC_URL="https://..."
```

Prepare mode does not require `KURU_PRIVATE_KEY` when `--from` or `KURU_WALLET_ADDRESS` is provided:

```bash
export MONAD_RPC_URL="https://..."
export KURU_WALLET_ADDRESS="0x..."
```

Optional environment:

```bash
export KURU_SLIPPAGE_BPS="50"
export KURU_AUTO_SLIPPAGE="true"
export KURU_REFERRER_ADDRESS="0x..."
export KURU_REFERRER_FEE_BPS="25"
export KURU_DEBUG_RESPONSE="false"
```

Programmatic callers can inject a viem-compatible wallet client instead of using `KURU_PRIVATE_KEY` by calling `createRuntimeWithWalletClient(config, walletClient, { walletAddress })`. The wallet client must expose `sendTransaction`; the wallet address can come from the explicit option, `walletClient.account`, `walletClient.getAddresses()[0]`, or `KURU_WALLET_ADDRESS`.

## Agent Responses

For a successful trade, summarize the receipt:

```text
Submitted Kuru buy: 10 USDC -> quoted 424.72 WMON.
Approval tx: 0x...
Swap tx: 0x...
```

If there is no approval tx, omit that line. For quotes, include route, input, estimated output, and slippage bps. For balances, include wallet, token, and human balance.

For prepared trades, summarize the wallet, route, input, quoted output, and ordered transaction count. Do not describe a prepared response as submitted or signed.

For unsupported or ambiguous tokens, say the token is not currently tradable by this skill and list the allowed symbols from `bun run trade tokens`. Do not suggest editing the registry unless the user asks to add support.

For missing amount, ask for the amount in the correct unit:

```text
How much USDC should I use to buy WMON?
```

For RPC/API failures, report the short error and a useful next step. If Kuru response shape looks wrong, rerun the same quote with `KURU_DEBUG_RESPONSE=true` before changing code.

## References

Read `references/kuru-flow.md` when updating Kuru API behavior, contract addresses, or transaction-building assumptions.
