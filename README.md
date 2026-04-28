The project allows you to create bots for trading on pump.fun and letsbonk.fun. Its core feature is to snipe new tokens. Besides that, learning examples contain a lot of useful scripts for different types of listeners (new tokens, migrations) and deep dive into calculations required for trading.

Leave your feedback by opening **Issues**.

---

## 🚀 Getting started

### Prerequisites
- Install [uv](https://github.com/astral-sh/uv), a fast Python package manager.

> If Python is already installed, `uv` will detect and use it automatically.

### Installation

#### 1️⃣ Clone the repository
```bash
git clone https://github.com/laddercrawler/trading-bot-solana.git
cd trading-bot-solana
```

#### 2️⃣ Set up a virtual environment
```bash
# Create virtual environment
uv sync

# Activate (Unix/macOS)
source .venv/bin/activate  

# Activate (Windows)
.venv\Scripts\activate
```
> Virtual environments help keep dependencies isolated and prevent conflicts.

#### 3️⃣ Configure the bot
```bash
# Copy example config
cp .env.example .env  # Unix/macOS

# Windows
copy .env.example .env
```
Edit the `.env` file and add your **Solana RPC endpoints** and **private key**.

Edit `.yaml` templates in the `bots/` directory. Each file is a separate instance of a trading bot. Examine its parameters and apply your preferred strategy.

For example, to run the pump.fun bot, set `platform: "pump_fun"`; to run the bonk.fun bot, set `platform: "lets_bonk"`.

#### 4️⃣ Install the bot as a package
```bash
uv pip install -e .
```
> **Why `-e` (editable mode)?** Lets you modify the code without reinstalling the package—useful for development!

### Running the bot

```bash
# Option 1: run as installed package
pump_bot

# Option 2: run directly
uv run src/bot_runner.py
```

> **You're all set! 🎉**

---

## Note on throughput & limits

Solana is an amazing piece of web3 architecture, but it's also very complex to maintain.

All node providers have their own setup recommendations & limits, like method availability, requests per second (RPS), free and paid plan specific limitations and so on. Please make sure you consult the docs of the node provider you are going to use for this bot. Public RPC nodes won't work for heavier use case scenarios like this bot.

### Built-in RPC Rate Limiting

The bot now includes built-in RPC rate limiting to prevent hitting provider limits:

- **Token bucket algorithm**: Smoothly controls request rate while allowing short bursts
- **Configurable max RPS**: Set `max_rps` parameter in `SolanaClient` (defaults to 25 RPS)
- **Automatic retry logic**: Handles 429 (Too Many Requests) errors with exponential backoff
- **Shared session management**: Reuses connections for improved performance

This helps ensure reliable operation within your node provider's rate limits without manual throttling.

## IDLs

The IDLs under [`idl/`](idl/) are vendored from [pump-fun/pump-public-docs](https://github.com/pump-fun/pump-public-docs). To refresh, copy `pump.json`, `pump_amm.json`, `pump_fees.json` from that repo into `pump_fun_idl.json`, `pump_swap_idl.json`, `pump_fees.json` respectively, and reference the upstream commit hash in your commit message.

## Changelog

Quick note on a couple on a few new scripts in `/learning-examples`:

*(this is basically a changelog now)*

## Bonding curve state check

`get_bonding_curve_status.py` — checks the state of the bonding curve associated with a token. When the bonding curve state is completed, the token is migrated to Raydium.

To run:

`uv run learning-examples/bonding-curve-progress/get_bonding_curve_status.py TOKEN_ADDRESS`

## Listening to the Pump AMM migration

When the bonding curve state completes, the liquidity and the token graduate to Pump AMM (PumpSwap).

`listen_logsubscribe.py` — listens to the migration events of the tokens from bonding curves to AMM and prints the signature of the migration, the token address, and the liquidity pool address on Pump AMM.

`listen_blocksubscribe_old_raydium.py` — listens to the migration events of the tokens from bonding curves to AMM and prints the signature of the migration, the token address, and the liquidity pool address on Pump AMM (previously, tokens migrated to Raydium).

Note that it's using the `blockSubscribe` method that not all providers support.

To run:

`uv run learning-examples/listen-migrations/listen_logsubscribe.py`

`uv run learning-examples/listen-migrations/listen_blocksubscribe_old_raydium.py`

**The following two new additions are based on this question [associatedBondingCurve #26](https://github.com/laddercrawler/trading-bot-solana/issues/26)**

You can compute the associatedBondingCurve address following the [Solana docs PDA](https://solana.com/docs/core/pda) description logic. Take the following as input *as seed* (order seems to matter):

- bondingCurve address
- the Solana system token program address: `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`
- the token mint address

And compute against the Solana system associated token account program address: `ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL`.

The implications of this are kinda huge:
* you can now use `logsSubscribe` to snipe the tokens and you are not limited to the `blockSubscribe` method
* see which one is faster
* not every provider supports `blockSubscribe` on lower tier plans or at all, but everyone supports `logsSubscribe`

The following script showcases the implementation.

## Compute associated bonding curve

`compute_associated_bonding_curve.py` — computes the associated bonding curve for a given token.

To run:

`uv run learning-examples/compute_associated_bonding_curve.py` and then enter the token mint address.

## Listen to new tokens

`listen_logsubscribe_abc.py` — listens to new tokens and prints the signature, the token address, the user, the bonding curve address, and the associated bonding curve address using just the `logsSubscribe` method. Basically everything you need for sniping using just `logsSubscribe` (with some [limitations](https://github.com/laddercrawler/trading-bot-solana/issues/87)) and no extra calls like doing `getTransaction` to get the missing data. It's just computed on the fly now.

To run:

`uv run learning-examples/listen-new-tokens/listen_logsubscribe_abc.py`

So now you can run `compare_listeners.py` to see which one is faster.

`uv run learning-examples/listen-new-tokens/compare_listeners.py`

---

# Pump.fun bot development roadmap (March - April 2025, mostly completed)


| Stage | Feature | Comments | Implementation status |
|-------|---------|----------|---------------------|
| **Stage 1: General updates & QoL** | Lib updates | Updating to the latest libraries | ✅ |
| | Error handling | Improving error handling | ✅ |
| | Configurable RPS | Ability to set RPS in the config to match your provider's and plan RPS | ✅ |
| | Dynamic priority fees | Ability to set dynamic priority fees | ✅ |
| | Review & optimize `json`, `jsonParsed`, `base64` | Improve speed and traffic for calls, not just `getBlock`. | ✅ |
| **Stage 2: Bonding curve and migration management** | `logsSubscribe` integration | Integrate `logsSubscribe` instead of `blockSubscribe` for sniping minted tokens into the main bot | ✅ |
| | Dual subscription methods | Keep both `logsSubscribe` & `blockSubscribe` in the main bot for flexibility and adapting to Solana node architecture changes | ✅ |
| | Transaction retries | Do retries instead of cooldown and/or keep the cooldown | ✅ |
| | Bonding curve status tracking | Checking a bonding curve status progress. Predict how soon a token will start the migration process | ✅ |
| | Account closure script | Script to close the associated bonding curve account if the rest of the flow txs fails | ✅ |
| | PumpSwap migration listening | pump_fun migrated to their own DEX — [PumpSwap](https://x.com/pumpdotfun/status/1902762309950292010), so we need to FAFO with that instead of Raydium (and attempt `logSubscribe` implementation) | ✅ |
| **Stage 3: Trading experience** | Take profit/stop loss | Implement take profit, stop loss exit strategies | ✅ |
| | Market cap-based selling | Sell when a specific market cap has been reached | Not started |
| | Copy trading | Enable copy trading functionality | Not started |
| | Token analysis script | Script for basic token analysis (market cap, creator investment, liquidity, token age) | Not started |
| | Archive node integration | Use Solana archive nodes for historical analysis (accounts that consistently print tokens, average mint to raydium time) | Not started |
| | Geyser implementation | Leverage Solana Geyser for real-time data stream processing | ✅ |
| **Stage 4: Minting experience** | Token minting | Ability to mint tokens (based on user request - someone minted 18k tokens) | ✅ |

---
