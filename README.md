# S&P 500 Cross-Sectional Long-Short Strategy (in progress)

The idea of the strategy is to rank all point-in-time constituents of the S&P 500 index by their predicted 5-day forward returns and, with a given cadence, buy the top 10% of the rank and sell the bottom 10% of the rank. If the rank predictions are accurate enough, the signal is stable, and the top-to-bottom spread is large enough to cover costs and commissions in any market conditions, the strategy can be profitable. The current version focuses on the ranking signal itself, and the portfilo construction and backtest is not implemented yet.

## Repository

The project is split into two notebooks:
- `S&P 500 Strategy Features.ipynb` contains data collection, explorating, preprocessing and feature engineering.
- `S&P 500 Strategy Model.ipynb` contains feature analysis, walk-forward cross-validation split, model training and evaluation.

A detailed discussion of all steps and decisions is provided in the notebooks.

## Data

The project uses open source data from Yahoo Finance (1998-2026). The index composition is taken from the file "S&P 500 Historical Components & Changes (Updated).csv" from the `fja05680/sp500` repository. It contains all changes of the index composition. For research and model training, we use index data from 2000 to 2026. For feature engineering, we need up to 1 year of margin in price histories, so we download split- and dividend-adjusted OHLCV data starting from 1998.

We clean the price data from invalid OHLC values and frozen quotes (zero volume and no price movements throughout the day), and exclude tickers with long gaps in their history (e.g., ticker reuse) or suspicious extreme returns. The final universe contains about 600 tickers.

The data does not contain history for companies that were delisted for any reason. This introduces **survivorship bias**, as we account only for companies that have survived to this day.

## Assumptions

Throughout the model, we use the next-open-to-open convention: a signal is obtained at the end of the day using the "Close" price, and a trading decision can be executed only the next morning at the "Open" price. The label is the cross-sectional rank of the 5-day forward return on each date, and the rebalancing cadence is also 5 trading days. In the current version of the model, transaction costs and commissions are excluded.

## Features

All 18 features use only past data and are converted into cross-sectional ranks on each date.

- **Momentum:** 12-, 6- and 3-month return with the last 21 days excluded so that mean reversion effects do not distort the signal, plus a risk-adjusted 12-month momentum.
- **Reversal:** 21- and 5-day reversal calculated as returns over a lookback window.
- **Volatility:** 21- and 63-day realised volatility calculated both as a standard deviation of log-returns and using the Garman-Klass formula, plus the ratio of 5-day to 63-day volatility, which reflects the current level of a stock's volatility compared to its normal value.
- **Market Model & Risk Decomposition:** 63- and 252-day β and idiosyncratic volatility from a rolling regression of a stock's daily log-returns on the market's (SPY) daily log-returns: r<sub>i</sub> = α + β · r<sub>market</sub> + ε. They split the overall realised volatility into a systematic (market) part and a stock-specific part.

- **Tail Risk:** 21-day skewness of daily log-returns and the maximum daily return over the last 21 days. Both reflect how "lottery-like" a stock is, as such stocks tend to underperform afterwards.

## Validation

A period from 2024 to 2026 is reserved as a holdout period. It is not used for research or making any decisions and is used only for the final model evaluation.

To estimate the stability of the model behaviour throughout the research period (2000-2023), we use walk-forward cross-validation with yearly test folds (2005-2023) and purging between training and testing periods. Purging removes training labels that overlap with the testing period to prevent information leakage.

After the features are constructed, we analyse the predictive power of single-factor models using the Information Coefficient (IC) as a metric.

## Models and Preliminary Results

Two models are built to predict the cross-sectional ranks: a linear Ridge as a baseline and LightGBM. We evaluate them by the daily Rank IC and the top-minus-bottom decile spread. Since 5-day labels overlap, t-statistics are calculated with the Newey-West correction.

| Model | Rank IC t-stat | Decile spread t-stat |
|---|---|---|
| Ridge | 1.8 | 0.6 |
| LightGBM | 4.1 | 2.9 |

The difference in mean Rank IC between the two models is not significant yet (t = 1.47), so the main advantage of LightGBM is in identifying the extreme deciles rather than in the overall ranking quality. These results are preliminary and cover only the research period.

## Next Steps

- Market regime and volume features.
- Hyperparameter tuning with Optuna (nested cross-validation).
- Final evaluation on the holdout period.
- Portfolio construction with asset allocation and backtest with transaction costs.
