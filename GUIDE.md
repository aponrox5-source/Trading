# Step-by-step guide (first to last)

You trade manually. The app only reads market data, builds candles, and shows signals.

## 1. Install (10 minutes)

Needs Python 3.10 or newer.

```bash
unzip binary-signal-engine.zip && cd binary-signal-engine
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m pytest -q tests          # about 1-2 minutes, all tests should pass
```

If a test fails, stop and send me the output before going further.

## 2. Get your forex data key (OANDA, free)

1. Create a free **fxTrade Practice** account at oanda.com.
2. In the account portal open **Manage API Access** and generate a token.
3. Note your account ID (looks like `101-001-1234567-001`).
4. `cp .env.example .env` and fill in:
   ```
   FOREX_PROVIDER=oanda
   OANDA_TOKEN=your_token
   OANDA_ACCOUNT=your_account_id
   OANDA_ENV=practice
   FOREX_SYMBOLS=EURUSD,GBPUSD,USDJPY,AUDUSD,USDCAD
   PAYOUT=0.85                     # your broker's real payout
   ```
   Prefer Twelve Data? Set `FOREX_PROVIDER=twelvedata` and `TWELVEDATA_KEY=...`. Its candles are only as accurate as the poll rate, so OANDA is better.

## 3. (Optional) Connect Pocket Option with your SSID

1. `pip install BinaryOptionsToolsV2`
2. In Chrome, log in to Pocket Option (use a **demo** session first), press F12, open **Network**, filter **WS**, click the websocket connection, open **Messages**.
3. Find the message that starts with `42["auth",` and copy the whole message.
4. Put it in `.env` inside single quotes:
   ```
   POCKETOPTION_SSID='42["auth",{"session":"...","isDemo":1,"uid":123,"platform":2}]'
   POCKETOPTION_SYMBOLS=EURUSD,GBPUSD,USDJPY
   ```
5. Test it: `python -m app.cli check-source pocketoption`. It must print 5 candles, then live ticks, and say the read-only guard is OK. If it fails, send me the printed error (the app hides your SSID in errors).

Rules: use real-market symbols (`EURUSD`), not `_otc` ones. Never share the SSID or the `.env` file. The SSID stops working when you log out of the broker. Quotex, IQ Option and Binomo have no verified connector yet: export candles as CSV (time, open, high, low, close) and run
`python -m app.cli import-csv --group quotex --symbol EURUSD --tf 1m --file candles.csv`.

## 4. Collect history

```bash
python -m app.cli collect --group forex --tf 1m --count 50000      # about 5 weeks per pair
```

More data is better: aim for 100,000+ rows in total across pairs. For a broker group, history per request is short, so the app also saves every live 1m candle while it runs. Leave it running for days, then train.

## 5. Train and read the verdict

```bash
python -m app.cli train --group forex --tf 1m
python -m app.cli train --group forex --tf 5m
```

Read the report at the end:

- **breakeven** is the win rate you need after payout (54.1% at 85%).
- **out-of-fold** and **hold-out** are results on data the model never trained on.
- **VALIDATED** means both are profitable after payout. Only then does the app show signals.
- **NO VALIDATED EDGE / NO EDGE FOUND** means it found nothing reliable. Add more data and pairs and retrain. Do not switch on `ALLOW_UNVALIDATED=1` and trade real money on it.

## 6. Run the dashboard

```bash
python -m app.cli serve        # open http://127.0.0.1:8000
```

You get the live chart with the forming candle, a countdown to candle close, the model status, the current signal, and a history table with automatic win/loss results.

## 7. Trade a signal manually

1. A signal appears right after a candle closes (about 0.1 s later). It says CALL or PUT, the confidence, and an entry window.
2. Enter on the pair in your broker at the **start of the next candle**, with expiry equal to the timeframe (1m or 5m).
3. The dashboard scores the signal as win, loss or tie (trade candle close vs its open). Your own entry delay will change your real result slightly.
4. Use a fixed small stake. Never double after a loss.

## 8. Why signals do not repaint

- A signal is created only from a **closed** candle, once, and never edited. Only the result is added later.
- The test suite proves it: adding future candles never changes an earlier signal, and replaying the same data gives identical signals.
- No signal is shown if the data is stale (older than `MAX_SIGNAL_AGE_S`, default 20 s) or has a hole in the last 60 candles. This is what prevents bad signals after a reconnect.
- Signals appear automatically for every pair you listed. Nothing to click.

## 9. Keep it honest over time

- Retrain weekly: repeat steps 4 and 5.
- Watch the **Live results** panel. If the live win rate is below the break-even rate after 200+ signals, stop and retrain with more data.
- Run 2-4 weeks on a demo account before using real money.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Status says `reconnecting (...)` | Read the message. 401 means a wrong key, otherwise it retries by itself. |
| `not trained` in Model status | Run step 5 for that group and timeframe, then restart `serve`. |
| No signals for a long time | Normal if the model is strict or the market is quiet. Check the model is VALIDATED. |
| Chart empty | Wait for history to load (status must say `live`). |
| Pocket Option connect fails | Re-copy a fresh SSID (step 3) and run `check-source` again. |
