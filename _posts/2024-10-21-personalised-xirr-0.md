---
layout: post
title:  "Personalised XIRR and investing - part 1"
date:   2024-10-21 17:39:00 +0100
categories: investing
---

# Motivation - why should you care about XIRR?

Last week I cycled to the [RecSys conference in Bari](https://recsys.acm.org/recsys24/) from Sardinia. Like all cycle tours with hours to be spent in the saddle, there are plenty of kilometres to mull over things. RecSys is about all things Recommender Systems - the algorithms that power most of what we see on our phones / the web - what to watch on Netflix or listen to on Spotify, recommended content on Instagram or TikTok, what to buy on Amazon.. the list is endless. Every app you use has a recommender engine built into it. And recommenders work by being __personal__ - they filter huge piles of content to find just the 0.001% that will appeal to you.

<figure>
    <img src="/images/italy-24.png" alt="Sardinia - Italy 2024" />
    <figcaption>The route from Cagliari to Bari - 1,003 km, 8,500m elevation, 1 puncture and 1 dog chase.</figcaption>
</figure>


For years, I've mostly ignored the published performance of various stock market indices - the NASDAQ, S&P 500, FTSE 100 or a "just buy the world" all-world ETF like [VWRP](https://www.vanguardinvestor.co.uk/investments/vanguard-ftse-all-world-ucits-etf-usd-accumulating/overview) / [VWRL](https://www.vanguardinvestor.co.uk/investments/vanguard-ftse-all-world-ucits-etf-usd-distributing/portfolio-data) - telling myself that my investing style is better and anyway, the aggregate timing of purchases / sales would skew any comparison.

The truth though is that I just didn't want the remorseless reckoning that such a comparison would bring. But on the cycle trip I realised that the reckoning is needed. Like training for cycling or running, if you don't measure then what are you improving? In essence, I needed my own __personalised__ performance report to power my future recommended stock market strategy.

There's little to no Machine Learning in this post, just data crunching and statistics in Python and the use of two beautifully elegant functions [merge_asof](https://pandas.pydata.org/docs/reference/api/pandas.merge_asof.html) and [pyxirr](https://pypi.org/project/pyxirr/) to save writing lots of code.

## XIRR

There are many metrics to measure investment performance, but the most relevant for most portfolios and investors is Extended Internal Rate of Return or XIRR. XIRR returns the internal rate of return for a schedule of cash flows that is not necessarily periodic. That's important when most investments are not stable and periodic - they are made when cash is added or re-invested which is different for everyone.

XIRR will tell you this basic stat for every holding in your portfolio - "your annualised return for this holding is X%", regardless of how long the holding period is, when you bought, how much you bought, or how frequently you bought. It's an incredibly useful investing statistic.

Put another way, if you have dripped £10,000, £100,000 or £1,000,000 into a portfolio over a large timeframe (10 years+) as funds allowed then XIRR will __annualise__ the return on the funds you have invested.

XIRR is remorseless - it cuts through all the noise and marketing and gives one, easy to digest number that sums up investment performance. For a portfolio, it can be calculated both per stock holding and as a total weighted number. Thus it shows overall performance and also the big winners and losers behind the total number (if you're a stock picker).

What is unacceptable to me is that virtually __no online platform I use in the UK actually reports XIRR__! They either focus on total return (disregarding length of time holding an investment) or capital growth (ignoring income via dividends which is very important for the UK indices). I want to know the exact XIRR for each investment in my portfolio and the total XIRR for the portfolio overall.

In summary, knowing your XIRR is key to improving future performance. Think of it as VO2 max for running.

In addition, what I really want is the ability to run multiple __ghost or virtual__ portfolios alongside a real portfolio. In essence to answer the question - "here's what you could have had".

Here's a short example to illustrate how XIRR can compress capex and income together for a stock holding into one number that still clearly conveys investment performance.

| Date       | Amount       | Comment                                                   |
| ---------- | ------------ | --------------------------------------------------------- |
| 14/07/2021 | \-£10,000.00 | Purchase                                                  |
| 17/11/2021 | \-£5,000.00  | Purchase                                                  |
| 14/01/2022 | £544.40      | Dividend income                                           |
| 27/01/2022 | \-£1,000.00  | Purchase                                                  |
| 22/04/2022 | \-£1,500.00  | Purchase                                                  |
| 09/08/2022 | £499.50      | Dividend income                                           |
| 20/12/2022 | £217.20      | Dividend income                                           |
| 06/02/2023 | £868.70      | Dividend income                                           |
| 07/08/2023 | £417.00      | Dividend income                                           |
| 13/02/2024 | £868.70      | Dividend income                                           |
| 21/10/2024 | £44,324.50   | Value if sold on 21/10                                    |
|            |              |                                                           |
| __XIRR__       | __40.93%__       | __Extended Internal Rate of Return from 14/7/21 to 21/10/24__ |

Table 1. XIRR example.

Read on to see how to calculate your own personalised XIRR and run virtual portfolios..

# The approach

The plan of attack is pretty simple:

1. Load my own data into a data structure suitable for calculating XIRR.
3. Write or acquire an XIRR function.
2. Load the reference data into a mirror data structure for the same reason.
4. For every buy and sell transaction I made in the real portfolio, make a twin matching transaction in the virtual portfolio.
4. For every distribution that the virtual portfolio makes, add it to the portfolio so that dividends contribute to the virtual XIRR.
4. Run and report the analysis.

All the code is written in Python with heavy use of the excellent [pandas](https://pandas.pydata.org) library. Our core data structure will be the pandas [DataFrame](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html).

# The data

We need two sets of data to compare - in this case my own transactions and a canonical dataset modelling the target index. I chose the FTSE 100 as the initial comparison, but as we'll see later, any index can be plugged in pretty easily.

Getting at my own data is pretty easy - it's all in [Moneywiz](https://www.wiz.money). It's a solid piece of personal finance software, albeit more focused on things like budgeting and tagging that I don't use. Moneywiz updates prices dynamically (using the Yahoo Finance API I'm pretty sure) and also has good UK bank coverage via Saltedge, when other competitors are more US-centric.

If you are a coder and want to slice and dice your own data as per this article, then the single-best thing about Moneywiz is the ability to take a snapshot export. Inside this export zip file is a SQLite database containing some pretty gnarly tables for internal use only. The table we need is ZSYNCOBJECT and through trial and error here are the SQL queries to get buy / sell transactions as well as income (dividends, special returns):


{% highlight python %}
  buys_sells_query = f"""
  SELECT ZACCOUNT2, ZINVESTMENTHOLDING, ZFEE2, ZDATE1,ZAMOUNT1,
  ZNUMBEROFSHARES1, ZPRICEPERSHARE1,ZSYMBOL1,ZDESC2
  FROM ZSYNCOBJECT
  WHERE ZACCOUNT2 IN ({include_accounts_list})
  AND ZSYMBOL1 IN ({include_tickers_list})
  ORDER BY ZDATE1 ASC
  """
#=> This SQL query retrieves all of the buy and sell transactions.
{% endhighlight %}


And to get the income transactions, same table but different columns:

{% highlight python %}
  income_query = f"""
  SELECT ZDATE1, ZACCOUNT2, ZINVESTMENTSYMBOL, ZAMOUNT1, ZDESC2
  FROM ZSYNCOBJECT
  WHERE ZINVESTMENTSYMBOL IS NOT NULL
  AND ZACCOUNT2 IN ({include_accounts_list})
  AND ZINVESTMENTSYMBOL IN ({include_tickers_list}) COLLATE NOCASE
  ORDER BY ZDATE1 ASC
  """
#=> This SQL query retrieves income  - dividends, return of capital etc.
{% endhighlight %}

# The reference data

Vanguard is the 800-pound gorilla of index and ETF funds in the US and have established a significant footprint in the UK. Handily for this post, not only does Vanguard have an ETF which tracks the FTSE 100 - [VUKE](https://www.vanguardinvestor.co.uk/investments/vanguard-ftse-100-ucits-etf-gbp-distributing/overview) but they also have released all of the daily prices for this ETF in CSV format.

<figure>
    <img src="/images/vuke-prices.png" alt="Vanguard VUKE daily prices" />
    <figcaption>Vanguard VUKE daily prices since 23rd May 2012 and download button.</figcaption>
</figure>

<figure>
    <img src="/images/vuke-income.png" alt="Vanguard VUKE income distributions" />
    <figcaption>Vanguard VUKE quarterly income distributions and download button.</figcaption>
</figure>


Vanguard update the files daily with the latest price so there's no point linking directly to them as the links will become invalid quickly.

# The XIRR function

Now we need to write or acquire an XIRR implementation. It would actually be quite gnarly code to write. We would need to use [Newton's method](https://en.wikipedia.org/wiki/Newton%27s_method). In numerical analysis, Newton's method is an iterative technique used to find roots (or zeros) of a real-valued function. In this case, it's used to find the interest rate (XIRR) that makes the NPV (Net Present Value) of the cashflows equal to zero. Luckily, someone has already done this for us, so once we have the DataFrame populated with 'Date' and 'Amount' columns, we simply call:

{% highlight python %}
  try:
      # xirr only wants these two columns
      df = df[["Date", "Amount"]]
      xirr_val = xirr(df)
      if xirr_val is None:
          log.error("Problem in XIRR calc 1")
          # eyeball / save out the data
          # print(df)
          df.to_csv("pak.csv", index=False)
      log.info(f"{xirr_val=}")
  except InvalidPaymentsError as ipe:
      log.error("Problem in XIRR calc 2", ipe)
{% endhighlight %}

# The read-across, aka virtual portfolios

Given two dataframes A and B, how can I match up transactions made in A with those made in B? A key part of the answer is using the [merge_asof](https://pandas.pydata.org/docs/reference/api/pandas.merge_asof.html) built-in function on the pandas DataFrame. It performs an elegant merging of a column from B onto A using __key distance__. If we order our 'Date' column correctly, we get the complicated merge operation for free!

Below is quite a long code listing - this code is in the middle of a refactoring and I may well split it out into its own post. For now, it serves to show the use of the merge_asof function to do some heavy lifting in the analysis.


{% highlight python %}
def run_comparison():
    # We only need the real buys+sells for the ghost / virtual analysis, not income
    df_buys_sells, _ = get_all_data()
    df_buys_sells["Date"] = pd.to_datetime(df_buys_sells["Date"])

    # Load the VUKE distribution data
    df_vuke_income = pd.read_excel(
        "/Users/hsheil/Downloads/Distribution History - Vanguard FTSE 100 UCITS ETF (GBP) Distributing - 22_08_2024.xlsx",
        skiprows=5,
    )
    df_vuke_income = preprocess_vuke_income(df_vuke_income)

    df_vuke = pd.read_excel(
        "/Users/hsheil/Downloads/Historical Prices - Vanguard FTSE 100 UCITS ETF (GBP) Distributing - 22_08_2024.xlsx",
        skiprows=8,
    )

    # Ball-park for the txns we made before VUKE started trading
    # Values derived from https://investing.thisismoney.co.uk/historic-prices/UKX?from=2010-08-01&to=2012-08-01
    df_vuke = add_out_of_band_txns(df_vuke)
    df_vuke = preprocess_vuke(df_vuke)

    # assign vuke price on actual purchase date to the df, using merge_asof to get the closest match
    df_buys_sells = pd.merge_asof(
        df_buys_sells, df_vuke[["Date", "vuke_price"]], on="Date", direction="nearest"
    )
    # Make sure our shizzle is correct
    assert len(df_buys_sells.loc[df_buys_sells.vuke_price.isna()]) == 0

    # Multiply by -1 here to negate the negative sign of Amount (- means a purchase)
    df_buys_sells["vuke_shares"] = (
        df_buys_sells["Amount"] / -df_buys_sells["vuke_price"]
    )
    # Use cumsum to get a rolling balance of all VUKE shares over time as we buy and sell
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
    cap_profit = num_vuke * latest_price
    log.info(f"{num_vuke=}, {latest_price=}, {cap_profit=}")
    sell_all = {"Date": today, "Amount": cap_profit}
    cap_invested = df_virtual_buys_sells.Amount.sum()
    df_virtual_buys_sells.loc[len(df_virtual_buys_sells)] = sell_all
    # XIRR on capital only
    xirr_val = xirr(df_virtual_buys_sells[["Date", "Amount"]])
    log.info(f"XIRR capital only is {xirr_val}")
    cap_return = cap_invested + cap_profit
    tr_pc = (cap_profit + cap_invested) / -cap_invested * 100
    log.info(
        f"Return (cap only) on £{cap_invested:.2f} is £{cap_profit:.2f}:: {cap_return:.2f} ({tr_pc:.2f}%)"
    )

    total_income = df_vuke_income["Amount"].sum()
    total_profit = cap_profit + total_income
    tr_pc = (total_profit + cap_invested) / -cap_invested * 100
    log.info(
        f"Return (cap+inc) on £{cap_invested:.2f} is £{total_profit:.2f} ({tr_pc:.2f}%)"
    )

    # Now merge buys/sells + income and get an overall XIRR including income
    df_merged = pd.merge(
        df_vuke_income[["Date", "Amount"]],
        df_virtual_buys_sells[["Date", "Amount"]],
        how="outer",
        on="Date",
    )
    df_merged["Amount"] = df_merged["Amount_x"].fillna(df_merged["Amount_y"])

    log.info(
        f"{df_vuke_income.shape=}, {df_virtual_buys_sells.shape=}, {df_merged.shape=}"
    )
{% endhighlight %}


# Putting it all together so far

This first post gets the base XIRR analysis up and running, motivating the reader as to why it is work worth doing. Part two will dig into relative performance (real vs portfolio) and replace the FTSE 100 data with S&P 500 data.

The complete code will be published on [Github](https://github.com/hsheil) once it's been refactored. For now, this post just serves to show the main concepts.