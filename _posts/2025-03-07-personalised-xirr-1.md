---
layout: post
title:  "Personalised XIRR and investing - part 2 (or how I learned to love pandas.merge_asof)"
date:   2025-03-07 09:39:00 +0100
categories: investing
excerpt: XIRR Part Two - comparing to virtual / dream team portfolios.
---

# Recap from Part One

Part one of this two-part series motivated why you (as an investor) should be interested in XIRR (eXtended Internal Rate of Return) for all your investment portfolios. XIRR elegantly handles irregular cash flows in / out and provides an easy to understand, accurate reflection of investing performance - both at the individual stock and portfolio level. XIRR consider _cash_ C invested for _time_ T, when most other metrics just consider _cash_ C which is much less useful.

In [Part One]({% post_url 2024-10-21-personalised-xirr-0 %}), we built a proof of concept system that collects and formats investing transactions (buys, sells, dividends) so that the XIRR could be calculated. Just this functionality by itself has substantial value - as noted then, most investing platforms in the UK do not calculate or provide XIRR.

But Part One was ultimately just a stepping stone to Part Two - this article. Now we ask the hard question: could I have done better? Can I compare my individual decisions over time to simply investing in a passive tracker for the FTSE 100, S&P 500 or an All-World Tracker?


# The reference data

Let's focus on a "my portfolio" <> FTSE 100 comparison - the main index for the UK with about 70% of revenue derived from outside the UK. Large caps in the index include AstraZeneca, Rolls Royce, Shell, BP, the miners etc. The FTSE 100 is not exciting like the NASDAQ but it is respectable and a major benchmark for UK investors. It is usually worth about 4-5% of the total world equity (the USA is worth about 70% currently).

Vanguard is the 800-pound gorilla of index and ETF funds in the US and have established a significant footprint in the UK. Not only does Vanguard have an ETF which tracks the FTSE 100 - [VUKE](https://www.vanguard.co.uk/professional/product/etf/equity/9509/ftse-100-ucits-etf-gbp-distributing) but they also release nightly drops of daily prices and quarterly distributions for this ETF in XLSX format.

This data is crucial for this post - when formatted correctly, it will form "DataFrame B" to act as the comparison to "DataFrame A" that we constructed in [Part One]({% post_url 2024-10-21-personalised-xirr-0 %}).

Lastly, VUKE is also competitive from a charges standpoint - a real source of returns erosion for actively-managed funds. the Ongoing Charge or OCF for VUKE is 0.09%.

<figure>
    <img src="/images/vuke-prices.png" alt="Vanguard VUKE daily prices" />
    <figcaption>Figure 1. Vanguard VUKE daily prices since 23rd May 2012 and download button.</figcaption>
</figure>

<figure>
    <img src="/images/vuke-income.png" alt="Vanguard VUKE income distributions" />
    <figcaption>Figure 2. Vanguard VUKE quarterly income distributions and download button.</figcaption>
</figure>


Vanguard update the files nightly with the latest price so there's no point linking directly to them as the links will become invalid quickly. You can access the latest files [here](https://www.vanguard.co.uk/professional/product/etf/equity/9509/ftse-100-ucits-etf-gbp-distributing).

# The ghost, aka virtual portfolios

Let's define a ghost or virtual portfolio as one that exists in an alternate world but mirrors exactly the cashflows made in the real world portfolio. The key difference is that in the ghost portfolio, we redirect the cash into a different investment to check our investment performance. Put another way, a virtual portfolio is simply an A/B test for your current investment strategy.

# Let's get into the code

Given two dataframes A and B, where A holds my personal investment transactions and B contains a reference target - how can I correctly match up transactions made in A with those made in B to enable a direct comparison? A key part of the answer is using the [merge_asof](https://pandas.pydata.org/docs/reference/api/pandas.merge_asof.html) built-in function on the pandas DataFrame. It performs an elegant merging of a column from B onto A using __key distance__. If we order our 'Date' column correctly in both Dataframes, we get the complicated merge operation for free!

Here is a partial code listing showing exactly how to do this - it shows how to use the _merge_asof_ function to do some heavy lifting in the analysis. As a rough outline, this code:

1. Uses the _DataConnectorFactory_ code written in Part One to load Dataframe A (containing the real-world investments).
2. Loads the VUKE data released by Vanguards for daily prices and quarterly distributions.
3. Creates Dataframe B as a clone of Dataframe A by using a key Pandas function - _merge_asof_ to emulate the real-world buys and sells.
4. Creates a rolling position counter using Pandas _cumsum_ as a precursor to calculating dividends payable per quarter.
5. Calculates the dividends.
6. Calculates the XIRR for capital only (this is very interesting for the FTSE 100 - see below).
7. Calculates the XIRR for capital + distributions.

{% highlight python %}
    # We only need the real buys+sells for the ghost / virtual analysis, not income
    connector = DataConnectorFactory.get_connector("moneywiz")
    df_buys_sells, _ = connector.get_data()
    df_buys_sells["Date"] = pd.to_datetime(df_buys_sells["Date"])

    # Load the VUKE distribution data using read_excel
    df_vuke_income = pd.read_excel(
        "./data/Distribution History - Vanguard FTSE 100 UCITS ETF (GBP) Distributing - 07_03_2025.xlsx",
        # Messy format from Vanguard - skip first 5 rows
        skiprows=5,
    )
    # See what we're dealing with
    log.info(f"{df_vuke_income.columns=}")
    # preprocess_vuke_income mainly converts strs to dates using pd.to_datetime
    # and processes e.g. "£27.33" to 27.33
    df_vuke_income = preprocess_vuke_income(df_vuke_income)

    # Load the VUKE price data using read_excel
    df_vuke = pd.read_excel(
        "./data/Historical Prices - Vanguard FTSE 100 UCITS ETF (GBP) Distributing - 07_03_2025.xlsx",
        # More messiness - skip first 8 rows
        skiprows=8,
    )
    # See what we're dealing with
    log.info(f"{df_vuke.columns=}")

    # Add in day pricing for VUKE before VUKE started trading
    # Values derived from https://investing.thisismoney.co.uk/historic-prices/UKX?from=2010-08-01&to=2012-08-01
    df_vuke = add_out_of_band_txns(df_vuke)
    # preprocess_vuke is an analogue of preprocess_vuke_income
    df_vuke = preprocess_vuke(df_vuke)

    # Assign VUKE price on actual purchase date to the df, using merge_asof to get the closest match
    # This single line is the beating heart of this code!..
    df_buys_sells = pd.merge_asof(
        df_buys_sells, df_vuke[["Date", "vuke_price"]], on="Date", direction="nearest"
    )
    # Make sure our shizzle so far is correct
    assert len(df_buys_sells.loc[df_buys_sells.vuke_price.isna()]) == 0

    # Multiply by -1 here to negate the negative sign of Amount (- means a purchase)
    df_buys_sells["vuke_shares"] = (
        df_buys_sells["Amount"] / -df_buys_sells["vuke_price"]
    )
    # Now use cumsum to get a rolling balance of all VUKE shares over time as we buy and sell
    df_buys_sells["vuke_position"] = df_buys_sells["vuke_shares"].cumsum()
    df_virtual_buys_sells = df_buys_sells[
        ["Date", "Amount", "vuke_position", "vuke_shares", "vuke_price"]
    ]

    # merge_asof again to merge real holdings onto the distributions dataframe
    # so we can see how much dividends to "earn" at each Vanguard distribution date
    df_vuke_income = pd.merge_asof(
        df_vuke_income,
        df_virtual_buys_sells[["Date", "vuke_position"]],
        on="Date",
        direction="nearest",
    )
    df_vuke_income["Amount"] = (
        df_vuke_income["vuke_position"] * df_vuke_income["amount_per_unit"]
    )
    # Add in the balancing txn if we sold everything today
    num_vuke = df_virtual_buys_sells.loc[
        len(df_virtual_buys_sells) - 1, "vuke_position"
    ]
    # Use the latest Vanguard price to add a final balancing transaction
    # where we liquidate the entire virtual platform so that the virtual XIRR is correct
    latest_price = df_vuke.loc[0, "vuke_price"]
    capital = num_vuke * latest_price
    log.info(f"{num_vuke=}, {latest_price=}, {capital=}")
    sell_all = {"Date": today, "Amount": capital}
    cap_invested = df_virtual_buys_sells.Amount.sum()
    # Actually set the final balancing txn
    df_virtual_buys_sells.loc[len(df_virtual_buys_sells)] = sell_all
    # Let's calc XIRR on capital only (ignoring dividends / distributions)
    xirr_val = xirr(df_virtual_buys_sells[["Date", "Amount"]])
    log.info(f"XIRR on capital only is {xirr_val*100:.2f}%")
    cap_return = cap_invested + capital
    # Let's see what the return (cap only) is
    tr_pc = (capital + cap_invested) / -cap_invested * 100
    log.info(f"Ghost portfolio: capital return is {tr_pc:.2f}%")

    # Now let's consider distributions (income) along with capital
    total_income = df_vuke_income["Amount"].sum()
    total_profit = capital + total_income
    tr_pc = (total_profit + cap_invested) / -cap_invested * 100
    log.info(f"Ghost portfolio: capital+income return is {tr_pc:.2f}%")

    # Now merge buys/sells + income and get an overall XIRR including income
    df_merged = pd.merge(
        df_vuke_income[["Date", "Amount"]],
        df_virtual_buys_sells[["Date", "Amount"]],
        how="outer",
        on="Date",
    )
    # The merge op will create _x and _y - combine these back into one col
    df_merged["Amount"] = df_merged["Amount_x"].fillna(df_merged["Amount_y"])

    # Sanity-check - see if 0-vals are causing xirr() issues
    before = len(df_merged)
    df_merged = df_merged[df_merged.Amount != 0]
    after = len(df_merged)
    if after < before:
        log.info(f"{before - after} 0-val txns removed")

    try:
        df_merged = df_merged[["Date", "Amount"]]
        xirr_val = xirr(df_merged)
        if xirr_val is None:
            log.error("Problem in XIRR calc 2")
            df_merged.to_csv("debug_xirr.csv", index=False)
        log.info(f"XIRR for ghost portfolio is {xirr_val*100:.2f}%")
    except InvalidPaymentsError as ipe:
        log.error("Problem in XIRR calc 1", ipe)
        df_merged.to_csv("debug_xirr.csv", index=False)

{% endhighlight %}


# Output

Here's some elided output:
```
07-03-2025 21:26:14,XIRR on capital only is 3.97%
07-03-2025 21:26:14,Ghost portfolio: capital return is 16.92%
07-03-2025 21:26:14,Ghost portfolio: capital+income return is 32.12%
07-03-2025 21:26:14,XIRR for ghost portfolio is 10.16%
```

So for the FTSE 100 starting in 2012, the XIRR on capital alone is just about 4%. We can see that dividend distributions form a significant part of the total return generated and the overall XIRR for this ghost portfolio is 10.16% when both capital returns and dividends are combined.

# Findings / Analysis

How does this theoretical return compare with the actual return generated by the real portfolio? As of today, the real TR is 9.1%, so the FTSE 100 ghost portfolio outperforms my actively-managed portfolio by 1.06% or 11.65% relative.

That's a big difference and in hindsight the reason is obvious. Investing in the FTSE 100 index is statistically safer with the risk spread out over 100 constituents. I've re-run the analysis for different time periods and there are date ranges when I outperform the index, but inevitably, inexorably the index will always [win out](https://www.ig.com/uk/trading-strategies/what-are-the-average-returns-of-the-ftse-100--230511) in the end as the time horizon extends.

We only need to play with the data a little bit and knock out two of my worst performers to beat the index but that is just pointless navel gazing and this is the beauty of XIRR. At the time, I didn't treat my two worst investments (both down over 90% - one exposed to UK government decisions, one to the retail property market during COVID) any differently to my two best investments (both up over 600%, both defence / aerospace) so there's no point airbrushing them out now either.

I use the code from [Part One]({% post_url 2024-10-21-personalised-xirr-0 %}) regularly. That code, and the ghost comparison code in this post, really helped me to understand my investing performance for what I consider to be a lower-risk side of my portfolios (private investing in startups is *much* more risky but also much more lucrative).

Lastly, a comparison to an S&P 500 tracker or a world-wide tracker like [VWRP](https://www.vanguard.co.uk/professional/product/etf/equity/9679/ftse-all-world-ucits-etf-usd-accumulating) (which is 70% US anyway) will show an even greater disparity in investment performance but truthfully I do look at the FTSE 100 a lot. I've considered the US markets to be frothy for a long time now and [Shiller / CAPE ratios](https://www.multpl.com/shiller-pe) indicate that a reversion to mean is on the cards but you cannot time the market..

# Summary 

In summary, in part one we calculated XIRR at both the individual ticker level and portfolio level. In Part Two (this article), we extended the XIRR analysis to perform a comparative analysis vs a major benchmark index - the FTSE 100 in the UK. Conducting this analysis enforces an investing discipline that otherwise wouldn't exist. It's amazing to think that investment platforms do not provide XIRR numbers routinely - they are a key part of understanding (and thus improving) your personalised investment performance.



