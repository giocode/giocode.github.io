---
layout: page
title: "Time Series: trend analysis, smoothing and anomaly detection"
date: 2015-08-10 19:54
comments: true
sharing: true
footer: true
---


## Forecasting

Forecasting is an important tool for making effective decisions, planning efficiently and reducing risks. 

Forecasting is useful in many situations when uncertain quantity or event must and can be predicted. In other words, we must have data that allows to forecast the quantity or event accurately. Then, we need to build a model that can tell the difference between a random fluctuation and a existing pattern in the data. A good forecasting model should capture such patterns with a minimum out-of-sample error. 


### Time series forecasting

Time series forecasting is concerned with only one question. Can we predict future values by looking at past ones? We are not attempting to identify factors that help explain the future. Instead, we want to accurately estimate how the historical data series will continue into the future. Thus, we want to find a functional relationship of the following form: 

$$
Y_{t+1} = f(Y_t, Y_{t-1}, Y_{t-2}, ... , Y_{t-n}, \text{error})
$$

Such relationship is described by a time series model.

### Basic forecasting steps 

There are in general four tasks that must be undertaken in forecasting.

1. **Define the problem.** This is the most difficult task as we need to determine what exactly to forecast, within what time horizon and how frequently. 
2. **Gather data.** It is primordial to find and collect historical data to construct a model with which we can forecast the future. For time series, this can be anything that are observed sequentially over time. 
3. **Building and choosing models.** This step involves choosing and fitting several forecasting models. Each model has parameters that must be fitted based on the known historical data. 
4. **Selecting and using the best model.** We select the model that provides the most accurate predictions on the test data samples, i.e. data that were held out and not used for building the models. With the selected model, we can then predict future values and determine some prediction intervals.


### Time series patterns 

Before building any forecasting model, it often helps to explore time series data by simply plotting them. This help to understand the behavior of the series data and the patterns that exist in the data. There exist four types of time series data patterns: 

1. A **stationary _horizontal_ pattern (H)** exists when the data values clearly fluctuate around a constant mean. 
2. A **_seasonal_ pattern (S)** exists because of the effects of seasonal factors (e.g. beginning of fiscal year, financial reporting quarters, weekends, days of the week, holidays, etc.) Note that such patterns need not to be exactly periodic. 
3. A **_cyclical_ pattern (C)** is present when the data values show a sequence of rises and falls that are not of a fixed period. Typical examples of a cyclical pattern are business cycles that impact the quantity of sales over a longer period of time. 
4. A **_trend_ (T)** is a long-term increase or decrease in the data. For instance, stock prices of high-growth companies usually exhibit a strong upward trend over a long period of time. 

A time series can have many of these four components at the same time. Sometimes, it is more convenient to combine H, C and T as a cycle-trend pattern that represents long-term changes in the time series. As a result, we will think of time series as a function of three components: the trend, the seasonality and the random error:

$$
\text{Time Series} = f(\text{Trend-Cycle}, \text{Seasonality}, \text{error})
$$

Most forecasting models explicitly assume that the random error is always present and try to minimize this error during model fitting. Depending on the actual time series, the functional relationship above can be additive or multiplicative. For example, we have the following with an additive decomposition model: 

$$ 
\text{Time Series} = \text{Trend-Cycle} + \text{Seasonality} + \text{error}
$$

For a multiplicative model, the time series decomposes as follows: 

$$
\text{Time Series} = \text{Trend-Cycle} \times \text{Seasonality} \times \text{error}
$$

In Haskell, we can for instance define a `TimeSeries` type as follows: 

```haskell
data TimeSeries = TimeSeries [TimeStamp] [Float]
type TS = TimeSeries
```

Let's suppose that we can appropriately define addition, substraction and multiplication over `TimeSeries`: 

```haskell
(+) :: TS -> TS -> TS
(-) :: TS -> TS -> TS
(*) :: TS -> TS -> TS
```

Additionally, we need convenience functions for creating lagged version of a time series or for extracting data points that are within a given window of the series:
```haskell
lag :: TS -> Time -> TS 
window :: TS -> Time -> Time -> TS
```


## Time series decomposition

In principle, if we want to further analyze the time series, we can use decomposition methods that extract these individual patterns. Decomposition methods are useful for forecasting as the trend-cycle can be used to extrapolate future values. This is the idea of **smoothing techniques**, which _smooth_ the series by eliminating the random fluctuations. Decomposition method is also needed when we want to detect anomalies such as outliers in the time series. We will cover about this topic later. 


### Basic decomposition method: 

A simple decomposition method consists of the following steps: 

1. The trend-cycle is computed using a Moving Average or LOESS method. 
2. The de-trended series is computed by removing the trend component from the time series. 
3. The seasonality component is extracted by averaging the values of the seasonal points over multiple periods (e.g. the values corresponding to same month for a monthly time series data)
4. We obtain the residual errors by subtracting the previously estimated components from the time series.

In Haskell, a decomposition method takes an input time series and return its three components:
```haskell
decompose :: TS -> (TS, TS, TS)
decompose timeseries = (trendcycle, seasonality, residual)
```

Next, let's see these specific techniques in more details. 


### Seasonal effects adjustment

We have seen that a decomposition method fully decomposes the series. Sometimes, we may just need to adjust time series to seasonal effects. In that case, we only need to remove its seasonality component. For example, the seasonal variation may be irrelevant and even lead to false conclusions to our analysis objectives. For an additive model, the _seasonally adjusted series_ is obtained after removing the seasonal pattern: 

$$
\text{Seasonally Adjusted Series} = \text{Time Series} - \text{Seasonality} = \text{Trend-Cycle} + \text{error}
$$

For that, we can define function that decomposes a time series into its seasonality component and a seasonally adjusted series: 

```haskell
seasonAdjust :: TS -> (TS, TS)
seasonAdjust ts = (seas, adjusted)
	where adjusted = ts - seas
		  (_, seas, _) = decompose ts
```

### Moving averages 

The Trend-Cycle component can be estimated by _smoothing_ the seasonally adjusted series. In other words, we reduce the random error fluctuations after removing the seasonality. The easiest way to do this is with a simple _moving average_. In the following, let's consider several moving average methods. 

#### Simple moving averages

Moving average is based on the idea that values that are nearby in time are likely to be close. Hence, taking an average of neighboring points of each data point will yield a good estimate of the trend at that point. Local averaging smooths out the randomness in the data. There are two design parameters to consider: 

1. How many data points do we include in each average? 
2. How much weight do we put for each neighboring point involved in the average? 

The simplest MA smoother averages an odd number of observations around each data point and put equal weights to them. For instance, a MA smoother of order 3 centered at a time `t` is: 

$$
T_t = \frac{1}{3} (Y_{t-1} + Y_{t} + Y_{t+1})
$$

Then, the estimate at time $t+1$ is obtained by dropping the oldest observation $Y_{t-1}$ and by including the next observation $Y_{t-2}$. Hence, the term _moving average_ describes this procedure. 

> The number of points involved in the average, or the order of the MA, affects the smoothness of the estimate. More points lead to a smoother trend. The drawback of MA is that it is impossible to estimate the trend-cycle close to the beginning and end of the series. The higher the order, the more points we miss at the front and back of the series.

### Double moving averages: 

Moving averages can be combined by smoothing an already smoothed series. For example, a 3 x 3 MA is a 3 MA of a 3 MA. When each 3 MA uses equal weights, the result is equivalent to a 5-period weighted moving average with the weigths 0.111, 0.222, 0.333, 0.222 and 0.111.


### Weighted moving averages: 

In general, a weighted MA with order `k` can be written as: 

$$
T_t = \sum_{j=-\frac{k-1}{2}}^{\frac{k-1}{2}} w_j Y_{t+j}
$$

### LOESS smoothing 

Instead of using MA, we can use a local linear regression technique to determine the trend. Precisely, we can fit a straight line through the observations near each data point. Then, the estimate of the trend at each point is given by each fitted line. For the data point at time $t$, the estimated trend-cycle is given by: 

$$
T_t = a + bt 
$$

where the parameters $a$ and $b$ are determined for each point at time $t$ by minimizing the sum of squared error: 

$$
\sum{j=-\frac{k-1}{2}}^{\frac{k-1}{2}} w_j(Y_{t+j} - a - b(t+j))^2
$$

> Note that the LOESS is more computationally intensive than MA methods since a different value of $a$ and $b$ needs to be computed for every value of $t$. For that, a different line is fitted at each data point using least squares. The number of points used in the local regression should not be too large to avoid underfitting and not too small to produce the desired smoothing. 

### STL decomposition 

The STL decomposition method literally stands for "Seasonal-Trend decomposition procedure based on Loess". It iteratively apply a LOESS smoother to give a decomposition that is robust against outliers. Another benefit of STL is that it has a flexibility to handle seasonality with abritrary length. The seasonality length just needs to be greater than one, which is usually satisfied in most applications. 

STL consists of the following two recursive procedures: 

1. **Inner loop.** In each iteration of the inner loop, the seasonal and trend-cycle components are updated once. 
2. **Outer loop.** An iteration of outer loop consists of one or two iterations of the inner loop. At the end of each outer loop iteration, outliers are detected. Future iterations of the inner loop puts smaller weights to these outliers in the LOESS regression.

More details about the operations performed in these two loops are described below. 

## Further reading: 

- Makridakis, S., S. Wheelwright, R. Hyndman, and Y. Chang. Forecasting Methods and Applications. 3rd ed. New York: John Wiley & Sons, 1998.
- Cleveland, R.B, Cleveland, W.S, Mcrae, J.E, and Terpenning, I. STL: A Seasonal-Trend decomposition procedure based on loess. Journal of Official Statistics, 6(1):3–73, 1990.
