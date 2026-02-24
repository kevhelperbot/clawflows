---
name: token-acquisition
emoji: 🪙
description: Price OTC token deals from foundations. Gathers supply breakdown, fetches market data, calculates extractable value, and recommends offer prices.
author: kevbot
schedule: manual
---

# Token Acquisition Pricing

Evaluate OTC token acquisition deals from foundations/protocols. This workflow gathers the required inputs, runs the pricing analysis, validates data sources, and delivers a clear buy/pass recommendation.

## 1. Gather Token Information

Ask the user for the token they're evaluating:

**Questions to ask:**
- "What token are you looking at?" (name or symbol)

Then search for it on CoinGecko to confirm:
- Verify the correct token (many tokens have similar names)
- Get the CoinGecko ID
- Confirm current price and market cap

**If ambiguous:** Show the user the top matches and ask them to confirm which one.

## 2. Gather Supply Breakdown

Ask the user for the supply matrix. This is critical data that must come from the foundation.

**Questions to ask:**
```
What's the supply breakdown? I need 4 numbers (as percentages):

Foundation owned:
  - VESTED (unlocked): ___%
  - UNVESTED (locked): ___%

Non-Foundation owned:
  - VESTED (unlocked): ___%  
  - UNVESTED (locked): ___%
```

**Validation:**
- Numbers should add up to ~100%
- If they don't, ask the user to verify or proceed with warning

## 3. Run Pricing Analysis

Execute the token pricer script:

```bash
cd ~/clawd/projects/otc-token-pricer
source venv/bin/activate
python3 token_pricer.py
```

Feed in:
- Token selection
- Supply breakdown numbers

Capture the full output including:
- Perps OI from each exchange
- Data source URLs
- Extractable value calculation
- Pricing recommendations

## 4. Validate Data Sources

**Critical step:** Review the data sources output and check for issues.

### Check for ticker mismatches:
- Did any exchanges return "NOT LISTED" that should have the token?
- Common issues:
  - Symbol mismatch (e.g., "U" vs "UNION" vs "UNIONUSDT")
  - Pair format (e.g., "BTCUSDT" vs "BTC-USDT-SWAP")

### If mismatches detected:
1. Search the exchange directly for the correct ticker
2. Offer to re-run with corrected symbols
3. Note any manual corrections made

### Verify reasonable values:
- Does the OI seem reasonable for this token's market cap?
- Is the price close to what CoinGecko shows?
- Any obvious data errors?

## 5. Analyze the Deal

Based on the pricing output, evaluate:

### Key Metrics to Highlight:
- **Free Float / OI Ratio:** Higher = harder to exit = worse deal
- **Sellable Divisor:** What the script calculated
- **Total Extractable:** The realistic value you can pull out
- **Foundation Value vs Extractable:** The gap matters

### Red Flags:
- 🚨 No perps available (can't hedge)
- 🚨 Extractable < 10% of foundation value
- ⚠️ Limited exchange coverage
- ⚠️ Very high free float / OI ratio (>50x)
- ⚠️ Only one exchange has perps

### Green Flags:
- ✅ Multiple perps venues (can hedge)
- ✅ Extractable > 25% of foundation value
- ✅ Listed on major exchanges
- ✅ Reasonable free float / OI ratio (<20x)

## 6. Deliver Summary

Present a concise deal summary using this format:

```
## [TOKEN] Acquisition Analysis

### Quick Take
[ONE SENTENCE: Good deal / Marginal / Pass]

### The Numbers
| Metric | Value |
|--------|-------|
| Market Price | $X.XX |
| Foundation Allocation | XX% (XXB tokens) |
| Foundation Value (market) | $X.XM |
| Total Extractable | $XXK |
| Great Price | $XXK |
| Acceptable Price | $XXK |

### Why This Price
- Free Float / OI Ratio: XXx
- Sellable Divisor: ÷XX
- Perps available: [exchanges]

### Risk Assessment
[List key risks and green flags]

### Recommendation
[CLEAR ACTION: Offer $XX-$XX / Pass / Need more info]

### Data Sources Verified
[List which sources were checked and any corrections made]
```

## 7. Handle Edge Cases

### If no perps anywhere:
- Warn strongly: "Cannot hedge this position"
- Recommend much deeper discount or pass
- Ask if user wants to proceed anyway

### If user disagrees with data:
- Offer to manually override specific values
- Re-run analysis with corrected inputs
- Document what was changed and why

### If supply doesn't add to 100%:
- Ask user to clarify (treasury? burned? etc.)
- Proceed with warning if user confirms

## Notes

**Script location:** `~/clawd/projects/otc-token-pricer/token_pricer.py`

**Methodology:**
1. Perps extractable = 1/3 of total OI
2. Sellable extractable = free float ÷ dynamic divisor (based on free float/OI ratio)
3. Total extractable = perps + sellable
4. Great price = 50% of extractable
5. Acceptable price = 60% of extractable
6. Max price = 1.5x extractable

**Divisor formula:** `divisor = 2 + (free_float/OI ratio) / 4`
- Clamped between 2 and 50
- Higher ratio = harder to sell = higher divisor
