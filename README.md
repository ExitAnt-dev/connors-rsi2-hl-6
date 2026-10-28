# ExitAnt Connors RSI2 — Hyperliquid

Deploy this automated trading bot to **your own Render account** as a background worker. The bot runs from a prebuilt Docker image;
this repository contains **only the Render Blueprint (`render.yaml`)** — no source code. One repository per image tag (`connors-rsi2-hl-1` … `-6`).

The bot trades **Larry Connors RSI2**, **Turtle only** or **Turtle + Connors RSI2 merge** on [Hyperliquid](https://app.hyperliquid.xyz) USDC perpetual futures with
exactly the rules of the TanQuant backtest engine, on the exchange's own candles — daily (the original rule), 4-hour or 1-hour, chosen in the bot;
UTC-aligned, stocks/ETFs only US trading days and, on 4-hour/1-hour, only NYSE regular-session candles — for every indicator. On daily candles, Yahoo
daily candles are used only to complete the 200-day moving average of symbols with fewer than 200 exchange candles (recently listed).
Copy a TanQuant result with **“Copy bot setup”** and paste it into the bot to set everything up at once.

## Deploy to Render

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/ExitAnt-dev/connors-rsi2-hl-6)

```
https://render.com/deploy?repo=https://github.com/ExitAnt-dev/connors-rsi2-hl-6
```

1. Click the button (or open the link) and sign in to Render.
2. Render reads `render.yaml` and proposes **one background worker** with a **1 GB persistent disk** mounted at `/data`
   (settings, books and positions survive restarts and redeploys).
3. Enter the environment variables listed below. They are marked `sync: false`, so Render asks you for them on the deploy screen
   and stores them **only in your own Render account**.
4. Click **Apply** and wait for the first deploy to finish.
5. Open Telegram and send **`/start`** to your bot to begin setup.

## Before you deploy

- **A wallet used only by this bot**, with USDC deposited to Hyperliquid (Perps balance; USDC on Spot is not collateral — move it with Transfer).
  Do not run another bot or trade by hand on the same wallet: Hyperliquid keeps one position per coin, so the positions would mix, and another
  ExitAnt Hyperliquid bot on the same wallet would overwrite this bot's fee approval (see below).
- **The private key of that wallet** (`0x` + 64 hexadecimal characters; the `0x` is optional) — not the wallet address and not an API wallet.
  The key signs orders, the fee approval, the switch to the Manual account type and transfers between your own balances inside the bot; it never leaves the bot, and the bot never writes
  it to logs, files or Telegram. It sits only in your Render environment variable, so keep your Render account secure (2FA).
  **The bot never asks for a key in Telegram** — a message asking for one is not from the bot.
- **Account type: Manual** (the API calls it standard). On a Unified Account or Portfolio Margin account the bot switches it to Manual at the first
  start with the wallet signature and moves the USDC from Spot to Perps — only while no position or order is open (otherwise it tells you to close
  them first). To switch it yourself: on the Hyperliquid trade screen click the **Unified** button above the order form (next to Cross and the
  leverage) → **Account Type** → **Manual** → sign, then move USDC from Spot to Perps with **Transfer** — a Unified Account has no Spot ⇄ Perps
  transfer, only EVM.
- **Builder fee approval = the membership check.** At the first start the bot approves the ExitAnt builder fee for this wallet (up to 0.1% per trade,
  charged by Hyperliquid on each fill) and then confirms it with the ExitAnt server. Only the wallet's public address is sent, never the key.
  Without a confirmed approval nothing is traded.

## Environment variables

| Variable | What to enter |
|---|---|
| `HYPE_WALLET_API` | The **private key** of the trading wallet (`0x` + 64 hexadecimal characters) — not the wallet address |
| `TELEGRAM_BOT_TOKEN` | Bot token issued by [@BotFather](https://t.me/BotFather) |
| `TELEGRAM_CHAT_ID` | Your own Telegram chat ID — the bot ignores every other sender |

Nothing else to configure: the exchange, trading mode and state-file location are built into the image.

## What the bot does

- Decides once per candle of the chosen length — daily at UTC midnight, 4-hour/1-hour right after each candle closes (stocks: session candles only) —
  and sends market orders at once (the backtest's “fill at the close”). On 4-hour/1-hour candles every indicator length is counted in candles, as in TanQuant
  (200-day MA → 200-bar MA).
- Turtle only (bot-only — not a TanQuant Connors strategy): the Turtle side of the merge alone, always long, on one candle length, with no
  Connors entries. In settings you can switch between the three, except that the side with an open position cannot be switched off
  (a Turtle position blocks Connors only, a Connors position blocks Turtle only) — the bot says why and what you can choose instead.
- In the Turtle + Connors merge the Turtle side can run on its own candle length (e.g. Turtle 4-hour, Connors 1-hour) — chosen on the first Turtle
  screen of the wizard and under 🐢 Turtle rules in settings. Keeping both the same is the TanQuant backtest setup; a different pair is a bot-only
  setting that TanQuant cannot backtest (the Turtle channel and its ATR stop come from the Turtle candles, each side decides at its own candle close).
- Margin-saving entry (bot-only, not in the TanQuant backtest): when a new lot enters while the symbol already holds another lot (Turtle long,
  Connors long or Connors short — same direction or not), it uses the set leverage × 2 and ½ of the held lot's entry size; if the exchange still
  refuses for insufficient margin, × 4 and ¼ once more. The leverage goes back to the setting when that lot closes. The wizard, the settings summary,
  the positions screen and the entry notice explain it.
- Trades the main Hyperliquid perps (crypto) and the trade.xyz perps (US stocks, the Nasdaq-100 and S&P 500 index perps for QQQ and SPY, gold, silver)
  that allow **cross margin**; isolated-only markets are not offered. It moves USDC between those two balances **inside your own account** as
  positions need it.
- Places an exchange stop-loss order per symbol (the farthest stop of the open lots) and watches the closer stops and the exit rules itself
  around the clock — **keep the bot running**.
- Keeps one account per symbol (trading pocket + vault), like TanQuant's multi-asset backtest. The Turtle merge moves part of the profit to the
  vault (your account's **Spot USDC**) on a trend exit and reinjects it after a deep drawdown, by the USA Turtle rules.
- Hyperliquid holds **one position per coin**: a symbol runs long *or* short (the “Both” direction is not offered, also when changing settings).
- Re-checks every hour while trading that the fee approval still holds. If it was lowered (for example by another bot on the same wallet), the bot
  approves it again with the wallet signature; if that fails, it stops opening new positions — the stops and exits of held positions keep running —
  and tells you in Telegram.
- Never stops trading because of an exchange answer — a refused or failed request skips that symbol for that cycle and is retried.

## Important

- **Keep the instance count at 1.** Two instances cause a Telegram `getUpdates` conflict.
- Render **background workers have no free plan** — choose Starter or higher.
- **The bot trades live from the first start.** Begin with a small allocation and watch one full cycle on your account before scaling up.
- The smallest order on Hyperliquid is 10 USDC.

⚠️ Futures trading carries a very high risk of loss. Backtest results do not guarantee future returns.

---

Blueprint spec: <https://render.com/docs/blueprint-spec> · Prebuilt images: <https://render.com/docs/deploying-an-image>
