# Reverse convertible pricer (S&P 500)

I built this to understand how a bank prices a simple structured product and how much coupon it can actually offer. It takes real SPX option quotes, prices a 1-year reverse convertible and solves for the fair coupon given the bank's margin.

## Result

With SPX options quoted on 30 Sep 2026 (expiry 30 Sep 2027, spot 7,670.84), a 1-year at-the-money reverse convertible with a 1.5% issuer margin can pay a coupon of about **8.75%**.

That breaks down roughly as 5.11% risk-free interest, plus 5.22% for the put the investor is effectively selling, minus 1.58% that the bank keeps (all compounded to maturity). So the extra yield over the risk-free rate is basically put premium.

<img width="1029" height="586" alt="coupon_vs_vol" src="https://github.com/user-attachments/assets/55346ada-1490-4903-9c16-a114f9568854" />


## How the product works

You invest 100. After one year, if the S&P 500 is at or above its starting level you get 100 plus the coupon. If it's below, you get the index performance (e.g. 80 after a 20% drop) plus the coupon.

That payoff is the same as holding a zero-coupon bond and selling a put, so it has a closed-form value:

```
Payoff = (1 + c) - max(K - S_T, 0) / K
Value  = (1 + c) * exp(-rT) - P/K
```

The bank sells at 100 and keeps a margin m, so setting the value equal to 1 - m gives the coupon:

```
c = (1 - m + P/K) * exp(rT) - 1
```

## Method

**Rates and dividends.** Instead of taking a rate and a dividend yield from another source, I got them from the options themselves using put-call parity. C - P is linear in the strike, so regressing it on K over 32 call-put pairs gives the discount factor and the forward. I got r = 4.99%, q = 0.83% and F = 7,996.18.

A near-perfect R² is expected here since parity is an identity. The more useful check was that the fitted line stays inside the bid-ask spread for all 32 pairs.

A side benefit: the spot I had was the previous day's close, but the pricing input S·exp(-qT) equals D·F, which comes only from the options, so the lag doesn't affect the price. It does make the implied q less reliable.

**Volatility.** Implied vol from mid prices, 17.54% at the product strike (interpolated, since the spot isn't a listed strike). Put and call vols at the same strike agreed within 0.03 vol points, which was my check that r and q were right.

<img width="1029" height="586" alt="smile" src="https://github.com/user-attachments/assets/e46d7820-d605-4872-be3e-9cff90a49598" />


**Check.** Monte Carlo with 1M paths on the full payoff gives 0.98504 ± 0.00015 at the fair coupon, against a target of 0.98500.

## Testing

I built it bottom-up and tested each part before using it in the next one, with synthetic inputs first and market data only at the end. The tests are in the notebook after each block:

- Black-Scholes: Hull's textbook values, put-call parity on 1,000 random inputs, zero-vol limit
- Implied vol: round trip with max error 2.7e-8 over 500 cases, impossible prices return NaN
- Monte Carlo: matches Black-Scholes within the confidence interval, standard error halves with 4x the paths
- Product: MC on the full payoff matches the bond minus put decomposition, and the fair coupon reprices to exactly 1 - m
- Market data: bad quotes filtered out, no simple arbitrage (calls fall and puts rise with the strike)
- Calibration: fit inside bid-ask, put and call vols agree, skew goes the right way
- Result: model put sits between the neighbouring market puts, MC confirms the price

## Limitations

- Constant vol when pricing. The smile is only used to pick the vol at the strike.
- No issuer credit risk. In reality the bond part is discounted at the bank's funding rate, which would push the coupon up.
- No barrier, and most reverse convertibles sold in Europe have one.
- One coupon at maturity instead of periodic payments.

Things I'd like to add: a barrier version, an autocallable (needs Monte Carlo since it's path-dependent), and Greeks.

## Running it

Open `reverse_convertible.ipynb` in Colab and run all cells. It loads the saved snapshot from `data/` and the seeds are fixed, so you should get the same numbers. Library versions are in `requirements.txt`.

Option quotes can't be downloaded again for a past date, which is why the snapshot is in the repo. `create_snapshot()` in Block 5 grabs new data if you want to run it on a different day.
