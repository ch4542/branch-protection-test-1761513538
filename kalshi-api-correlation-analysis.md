# Kalshi Market API Correlation Analysis

## Executive Summary

After extensive research, I've identified the **US Gas Price Market** as the optimal Kalshi market for correlation analysis, paired with **two free APIs** (FRED and EIA) that provide complementary data with strong predictive potential. This combination offers:

1. **Moderate volume** with consistent weekly trading activity
2. **Clear settlement source** (AAA data, accessible via FRED)
3. **Multiple upstream predictive variables** (crude oil, refinery utilization, inventories)
4. **Minimal existing exploration** - no detailed correlation strategies found in research

---

## Recommended Market: US Gas Price

### Market Details

| Attribute | Value |
|-----------|-------|
| **Market URL** | https://kalshi.com/markets/kxaaagasm/us-gas-price |
| **Settlement Source** | AAA (American Automobile Association) |
| **Update Frequency** | Weekly (Mondays) |
| **Contract Type** | Binary (Yes/No on price thresholds) |
| **Available Thresholds** | $2.50, $2.70, $2.90, $3.10, $3.30+ per gallon |
| **Trading Volume** | $12,000+ weekly on active markets |

### How Settlement Works

- AAA publishes average regular gas prices daily/weekly
- Kalshi contracts resolve based on whether the average price exceeds specific thresholds
- Contract closes 5 hours before target date with 5-minute settlement window
- Resolution statement: "If average regular gas prices are strictly greater than [Price] on [Date] according to AAA, then the market resolves to Yes"

---

## API #1: FRED (Federal Reserve Economic Data)

### Why FRED?

FRED is the **direct source** for AAA gas price data used by Kalshi for settlement. It also provides comprehensive economic indicators for correlation analysis.

### Registration

1. Visit: https://fred.stlouisfed.org/docs/api/api_key.html
2. Create free account
3. Receive API key instantly

### Key Data Series

#### Primary Series (Settlement Data)

```
Series ID: GASREGW
Name: US Regular All Formulations Gas Price
URL: https://fred.stlouisfed.org/series/GASREGW
Units: Dollars per gallon
Frequency: Weekly (Mondays, 8:00 AM)
Coverage: August 1990 - Present
Source: EIA (via AAA sampling methodology)
```

**API Call Example:**
```
https://api.stlouisfed.org/fred/series/observations?series_id=GASREGW&api_key=YOUR_KEY&file_type=json
```

#### Correlation Series (Upstream Predictors)

| Series ID | Description | Frequency | Correlation Use |
|-----------|-------------|-----------|-----------------|
| **DCOILWTICO** | WTI Crude Oil Price (Cushing, OK) | Daily | 58% of gas price is crude oil cost |
| **DCOILBRENTEU** | Brent Crude Oil Price | Daily | Global crude benchmark |
| **MORTGAGE30US** | 30-Year Mortgage Rate | Weekly | Economic demand indicator |
| **UNRATE** | Unemployment Rate | Monthly | Demand indicator |
| **CPIAUCSL** | Consumer Price Index | Monthly | Inflation correlation |

### API Endpoints

**Base URL:** `https://api.stlouisfed.org/fred/`

| Endpoint | Purpose |
|----------|---------|
| `/series/observations` | Get data values for a series |
| `/series/search` | Find series by keywords |
| `/series` | Get series metadata |
| `/releases` | Get data releases |

### Rate Limits

- **Unauthenticated**: Not available
- **Authenticated**: 120 requests per minute

### Sample API Call

```python
import requests

API_KEY = "your_api_key"
series_id = "GASREGW"

url = f"https://api.stlouisfed.org/fred/series/observations"
params = {
    "series_id": series_id,
    "api_key": API_KEY,
    "file_type": "json",
    "observation_start": "2024-01-01"
}

response = requests.get(url, params=params)
data = response.json()

for obs in data["observations"]:
    print(f"{obs['date']}: ${obs['value']}")
```

---

## API #2: EIA (Energy Information Administration)

### Why EIA?

EIA provides **upstream supply-side data** that predicts future gas prices before they appear in AAA settlement data. This creates a predictive edge.

### Registration

1. Visit: https://www.eia.gov/opendata/
2. Click "Register"
3. Receive API key via email

### Rate Limits

- 5 requests per second
- 10,000 requests per day

### Key Data Series

#### Crude Oil Inventories (Leading Indicator)

```
Series ID: PET.WCESTUS1.W
Name: Weekly U.S. Ending Stocks excluding SPR of Crude Oil
Units: Thousand Barrels
Frequency: Weekly (Wednesday release)
URL: https://www.eia.gov/dnav/pet/hist/LeafHandler.ashx?n=PET&s=WCESTUS1&f=W
```

**Why it matters:** Inventory changes predict supply constraints → price changes

#### Refinery Utilization (Supply Indicator)

```
Series ID: PET.WPULEUS3.W
Name: Weekly U.S. Percent Utilization of Refinery Operable Capacity
Units: Percent
Frequency: Weekly
```

**Why it matters:** Low utilization = reduced supply = higher prices

#### Motor Gasoline Stocks (Direct Indicator)

```
Series ID: PET.WGTSTUS1.W
Name: Weekly U.S. Ending Stocks of Total Gasoline
Units: Thousand Barrels
Frequency: Weekly
```

**Why it matters:** Gasoline inventory directly impacts retail prices

### API v2 Structure

**Base URL:** `https://api.eia.gov/v2/`

**Petroleum Route:** `https://api.eia.gov/v2/petroleum/`

### Sample API Call

```python
import requests

API_KEY = "your_api_key"

# Get weekly crude oil inventories
url = "https://api.eia.gov/v2/petroleum/stoc/wstk/data/"
params = {
    "api_key": API_KEY,
    "frequency": "weekly",
    "data[0]": "value",
    "facets[product][]": "EPC0",
    "sort[0][column]": "period",
    "sort[0][direction]": "desc",
    "length": 52  # Last year of data
}

response = requests.get(url, params=params)
data = response.json()
```

### Legacy Series ID Translation

For v1 series IDs, use the translation endpoint:
```
https://api.eia.gov/v2/seriesid/PET.WCESTUS1.W?api_key=YOUR_KEY
```

---

## Correlation Methodology

### Hypothesis: Upstream Indicators Predict Gas Prices

Gas prices are influenced by a chain of factors with predictable lag times:

```
Crude Oil Prices (2-3 week lag)
         ↓
Refinery Utilization (1-2 week lag)
         ↓
Gasoline Inventories (0-1 week lag)
         ↓
Retail Gas Prices (Settlement)
```

### Key Correlations to Test

| Variable | Expected Correlation | Lag Time |
|----------|---------------------|----------|
| WTI Crude Price | Strong Positive | 2-3 weeks |
| Crude Inventories | Strong Negative | 1-2 weeks |
| Gasoline Inventories | Strong Negative | 0-1 week |
| Refinery Utilization | Moderate Positive | 1-2 weeks |

### Seasonal Factors

| Season | Factor | Impact on Prices |
|--------|--------|-----------------|
| **Spring (Mar-May)** | Summer blend switchover | ↑ Prices rise |
| **Summer (Jun-Aug)** | Peak driving demand | ↑ High prices |
| **Fall (Sep-Nov)** | Winter blend switchover | ↓ Prices fall |
| **Winter (Dec-Feb)** | Low demand | ↓ Low prices |

### External Event Factors

- **Gulf hurricanes** (Aug-Oct) → Refinery disruptions → Price spikes
- **OPEC decisions** → Supply changes → 2-4 week price impact
- **Geopolitical events** → Crude oil volatility → Gas price follow

---

## Implementation Strategy

### Step 1: Data Collection

```python
# Collect weekly data from both APIs
datasets = {
    "gas_price": "GASREGW",           # FRED - Settlement source
    "crude_oil": "DCOILWTICO",        # FRED - Primary driver
    "crude_inventories": "WCESTUS1",  # EIA - Supply indicator
    "gas_inventories": "WGTSTUS1",    # EIA - Direct indicator
    "refinery_util": "WPULEUS3"       # EIA - Production capacity
}
```

### Step 2: Feature Engineering

1. **Calculate week-over-week changes** for all variables
2. **Create lagged features** (1-week, 2-week, 3-week lags)
3. **Add seasonal indicators** (month, quarter, summer/winter blend)
4. **Include event flags** (hurricane season, OPEC meeting dates)

### Step 3: Correlation Analysis

```python
import pandas as pd
from scipy.stats import pearsonr

# Calculate correlations with different lags
for lag in [0, 1, 2, 3]:
    corr, pvalue = pearsonr(
        df['crude_oil'].shift(lag),
        df['gas_price']
    )
    print(f"Crude oil (lag {lag}): r={corr:.3f}, p={pvalue:.4f}")
```

### Step 4: Trading Signal Generation

```python
# Example: If crude inventories drop significantly, expect gas prices to rise
if inventory_change < -2_000_000:  # 2M barrel decrease
    signal = "BUY YES on higher threshold contracts"
elif inventory_change > 3_000_000:  # 3M barrel increase
    signal = "BUY NO on higher threshold contracts"
```

---

## Why This Opportunity is Underexplored

### Existing Research Gaps

1. **TSA Markets** - Already has detailed trading bot (Jacob Ferraiolo's blog)
2. **Weather Markets** - Multiple GitHub projects exist
3. **CPI/Fed Markets** - Well-documented in academic papers

### Gas Price Market Advantages

1. **No detailed correlation strategies found** in research
2. **Multiple free data sources** available
3. **Clear causal relationships** between upstream data and prices
4. **Predictable seasonal patterns** for baseline models
5. **Weekly frequency** allows regular trading opportunities

---

## Data Access Summary

### API Registration Links

| API | Registration URL | Key Delivery |
|-----|-----------------|--------------|
| FRED | https://fred.stlouisfed.org/docs/api/api_key.html | Instant |
| EIA | https://www.eia.gov/opendata/ | Email |

### Data Formats

Both APIs return data in:
- **JSON** (recommended for programmatic access)
- **XML**
- **CSV** (EIA bulk download)

### Update Schedule

| Data Source | Update Day | Update Time |
|-------------|-----------|-------------|
| AAA Gas Prices | Monday | Throughout day |
| FRED GASREGW | Monday | 9:00 AM CST |
| EIA Weekly Petroleum | Wednesday | 10:30 AM ET |

---

## Alternative Market: Arctic Sea Ice Extent

If you want an even more novel opportunity with less competition, consider:

### Market Details

| Attribute | Value |
|-----------|-------|
| **Market URL** | https://kalshi.com/markets/kxarcticicemin/arctic-sea-ice-min-extent |
| **Settlement Source** | NSIDC (National Snow and Ice Data Center) |
| **Frequency** | Annual (September minimum) |
| **Data Coverage** | 1978 - Present |

### Free API

```
Source: NSIDC
URL: https://noaadata.apps.nsidc.org/NOAA/G02135/
Format: CSV, GeoTIFF, ASCII
Documentation: https://nsidc.org/data/g02135
```

### Why It's Novel

- **Zero existing correlation analysis** found in research
- **Clear scientific predictors** (ocean temperatures, atmospheric patterns)
- **Long historical record** for backtesting (45+ years)
- **Single annual event** with months of lead time for analysis

---

## Conclusion

The **US Gas Price Market** with **FRED + EIA APIs** represents the optimal combination for your correlation model because:

1. **Underexplored** - No detailed correlation strategies exist
2. **Multiple free APIs** - Both direct settlement data and upstream predictors
3. **Proven causality** - Economic relationships are well-established
4. **Regular frequency** - Weekly trading opportunities
5. **Predictable patterns** - Seasonal and event-driven factors

The combination of FRED's settlement source data (GASREGW) with EIA's upstream supply indicators (crude inventories, refinery utilization, gasoline stocks) creates a multi-factor prediction model that can identify mispriced contracts before the market adjusts.

---

## Quick Start Checklist

- [ ] Register for FRED API key
- [ ] Register for EIA API key
- [ ] Download historical GASREGW data (5+ years)
- [ ] Download historical crude oil inventory data
- [ ] Download historical refinery utilization data
- [ ] Calculate correlations with various lag times
- [ ] Identify strongest predictive relationships
- [ ] Backtest trading signals against Kalshi historical data
- [ ] Implement automated data collection
- [ ] Deploy correlation model for live trading signals
