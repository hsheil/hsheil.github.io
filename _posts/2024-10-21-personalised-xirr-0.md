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

The unvarnished truth is that I just didn't want the reckoning that such a comparison would bring. But on the cycle trip I realised that the reckoning is needed. Like training for cycling or running, if you don't measure then what are you improving? In essence, I need a __personalised__ performance report to power my future recommended stock market strategy.

There's little to no Machine Learning in this post, just data crunching and statistics in Python and the use of two beautifully elegant functions [merge_asof](https://pandas.pydata.org/docs/reference/api/pandas.merge_asof.html) and [pyxirr](https://pypi.org/project/pyxirr/) to save writing lots of code.

## XIRR

There are many metrics to measure investment performance, but the most relevant for most portfolios and investors is Extended Internal Rate of Return or XIRR. XIRR returns the internal rate of return for a schedule of cash flows that is not necessarily periodic. That's important when most investments are not stable and periodic - they are made when cash is added or re-invested which is different for everyone.

XIRR will tell you this basic stat for every holding in your portfolio - "your annualised return for this holding is X%", regardless of how long the holding period is, when you bought, how much you bought, or how frequently you bought. It's an incredibly useful investing statistic.

Put another way, if you have dripped £10,000, £100,000 or £1,000,000 into a portfolio over a large timeframe (10 years+) as funds allowed then XIRR will __annualise__ the return on the funds you have invested.

XIRR is remorseless - it cuts through all the noise and marketing and gives one, easy to digest number that sums up investment performance. For a portfolio, it can be calculated both per stock holding and as a total weighted number. Thus it shows overall performance and also the big winners and losers behind the total number (if you're a stock picker).

What is unacceptable to me is that virtually __no online platform I use in the UK actually reports XIRR__! They either focus on total return (disregarding length of time holding an investment) or capital growth (ignoring income via dividends which is very important for the UK indices). I want to know the exact XIRR for each investment in my portfolio and the total XIRR for the portfolio overall.

In summary, knowing your XIRR is key to improving future performance. Think of it as VO2 max for running.

Both Microsoft Excel and Google Sheets have an XIRR function readily available and I _could_ just put my data in there and go at it. But I've got over 2,600 transactions to check and also I want to do more..

What I really want is the ability to run multiple __virtual__ portfolios alongside a real portfolio. In essence to answer the question - "here's what you could have had if you had tracked the S&P 500 or FTSE 100 with the same money".

Here's a short example to illustrate how XIRR can compress capital and income together for a stock holding into one number that still clearly conveys investment performance.

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

1. (Part 1) Load my own data into a data structure suitable for calculating XIRR.
3. (Part 1) Write or acquire an XIRR function.
4. (Part 1) Run and report the analysis.
2. (Part 2) Load the reference data into a mirror data structure for the same reason.
4. (Part 2) For every buy and sell transaction I made in the real portfolio, make a twin matching transaction in the virtual portfolio.
4. (Part 2) For every distribution that the virtual portfolio makes, add it to the portfolio so that dividends contribute to the virtual XIRR.

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

I used a simple factory design pattern to model the Moneywiz connector and support other datasources in the future:

{% highlight python %}
# The DataConnector class models how the core df_prices and df_income dataframes
# should be built as well as a way to get the current open positions.
# A working impl. for Moneywiz is provided.
class DataConnector:
    def get_data(self):
        raise NotImplementedError("Subclasses should implement this!")

    def get_open_positions(self):
        raise NotImplementedError("Subclasses should implement this!")

    def get_txns(ticker):
        raise NotImplementedError("Subclasses should implement this!")

{% endhighlight %}


# The XIRR function

Now we need to write or acquire an XIRR implementation. It would actually be quite gnarly code to write. We would need to use [Newton's method](https://en.wikipedia.org/wiki/Newton%27s_method). In numerical analysis, Newton's method is an iterative technique used to find roots (or zeros) of a real-valued function. In this case, it's used to find the interest rate (XIRR) that makes the NPV (Net Present Value) of the cashflows equal to zero. Luckily, someone has already done this for us via [PyXIRR](https://github.com/Anexen/pyxirr) "Rust-powered collection of financial functions", so once we have the DataFrame populated with 'Date' and 'Amount' columns, we simply call:

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

Note the native support for DataFrames by the xirr() function - nice!

# Putting it all together for Part One

Below is the main driving function that loads and preprocesses the transaction data, sets up the reporting DataFrame and loops over each ticker in turn calculating XIRR over both capital growth (or loss) and income.

The code should come across as both simple and readable. The more complicated parts in fact relate to an early effort to add UK CGT (Capital Gains Tax) exposure given the impending budget of doom at the end of October 2024.



{% highlight python %}
def run_xirr(args):
    # Stores the stats (as a dict) for each individual stock as we loop over them.
    # Converted to a DF at the end and is the main printed output of this code.
    results = []
    # Holds the txns for each individual stock as we loop over them
    # Merged at the end into one large DF to calc overall portfolio XIRR
    all_dfs = []

    connector = DataConnectorFactory.get_connector(args.source)

    df_buys_sells, df_income = connector.get_data()
    df_buys_sells, df_income = pre_process_actual(df_buys_sells, df_income)

    df_open = connector.get_open_positions()
    df_open = pre_process_open(df_open)

    s_tickers = df_buys_sells.ZSYMBOL1.value_counts()

    # The main loop over each position held in the portfolio
    for ticker, _ in s_tickers.items():
        if ticker in print_list:
            log.info(f"Processing {ticker}")
        capital, income, taxable_ratio, share_count, df = connector.get_txns(ticker)
        if ticker in print_list:
            log.info(f"{ticker}, {taxable_ratio=}, {capital=}")
            print(
                f"{df[['Date', 'Amount', 'taxable_shares', 'nontaxable_shares', 'taxable']]}"
            )

        curr_val = add_open_position(ticker, share_count, df_open, df)
        # crude calculation of tax payable..
        if curr_val > 0:
            taxable = (curr_val + capital) * taxable_ratio
        else:
            # TODO handle shares that have been sold
            taxable = 1
        # Add total curr_val to capital to get capital position
        capital += curr_val
        if ticker in print_list:
            # Some rudimentary support for calculating CGT exposure
            log.info(f"{ticker}, {curr_val=}")
            log.info(f"{ticker}, {taxable=}")
            log.info(f"{ticker}, {taxable_ratio=}")
        overall = df.Amount.sum()

        df = df.sort_values(by="Date")
        df = df.reset_index(drop=True)
        if ticker in print_list:
            log.info(f"{df=}")

        # My Moneywiz instance holds a couple of Vodafone divi payments
        # *before* any purchase for the date range in question - remove these
        df = check_rogue(df, ticker)

        if ticker in print_list:
            log.info(f"=====DF GOING INTO XIRR for {ticker} =====")
            log.info(f"{df[['Date', 'Amount']]=}")
            df.to_csv(f"{ticker}.csv", index=False)
        try:
            curr_xirr = xirr(df[["Date", "Amount"]])
        except InvalidPaymentsError:
            log.error(f"Problem in XIRR calc for {ticker}")
            log.error(f"{df=}")
        if curr_xirr is None:
            log.error(f"Problem in XIRR calc for {ticker}")
            log.error(f"{df=}")

        # Use the sign to distinguish between monies invested (-)
        # and income generated (+)
        df.loc[df["Amount"] < 0, "invested"] = df["Amount"]
        df.loc[df["Amount"] > 0, "income"] = df["Amount"]

        all_dfs.append(df)
        # Work out how many years since the very first purchase
        # of the current ticker
        years = (today - df.Date.iloc[0]).days / 365
        if ticker in print_list:
            log.info(f"{taxable=}")
        results.append(
            {
                "Ticker": ticker,
                "XIRR": curr_xirr * 100,
                "Curr value": curr_val,
                "Overall": overall,
                "Capital": capital,
                "CGT": taxable,
                "Dividends": income,
                "Years": years,
                "Cost": float(df.invested.sum()),
                "Income": float(df.income.sum()),
            }
        )

    df_results = pd.DataFrame(results)
    # TR = Total Return
    df_results["TR"] = -(df_results.Income + df_results.Cost) / df_results.Cost
    # These cols are no longer needed
    df_results = df_results.drop(columns=["Cost", "Income"])

    # Let the user specify which col to sort by on the cmdline
    df_results = df_results.sort_values(by=args.order)
    # Use print instead of log so we get col headers aligned correctly
    print(f"{df_results}")

    # See who is still to be covered
    # covered = set(df_results.Ticker)
    # missing = current_tickers.difference(covered)
    # if len(missing) > 0:
    #     log.info(f"Still to add in, {missing}")

    df_all = pd.concat(all_dfs)
    log.info(f"{df_all.shape=}")
    # This is the portfolio-level XIRR
    overall_xirr = xirr(df_all[["Date", "Amount"]])

    # Check for unused txns
    df_buy_sells_unused = df_buys_sells[df_buys_sells.used == 0]
    if len(df_buy_sells_unused) > 0:
        log.info(f"{len(df_buy_sells_unused)} unused buy/sell txns")
    df_income_unused = df_income[df_income.used == 0]
    if len(df_income_unused) > 0:
        log.info(f"{len(df_income_unused)} unused income txns / {len(df_income)} total")
        log.info(f"{df_income_unused.ZINVESTMENTSYMBOL.value_counts()}")
    log.info(f"Overall XIRR, {overall_xirr:.5f}")
    log.info(f"Indicative CGT exposure, {df_results.CGT.sum():.2f}")

{% endhighlight %}


# Conclusion

This first post gets the base XIRR analysis up and running, motivating the reader as to why it is well worth doing. Part two will dig into relative performance by supporting virtual portfolios that track the real portfolio for indices like the S&P 500, FTSE 100 etc.

The complete code will be published on [Github](https://github.com/hsheil) once it's been refactored. For now, this post just serves to show the main concepts.