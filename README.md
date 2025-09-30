# README

## Overview

This repository contains an exploratory data analysis (EDA) project focused on regional sales data. The goal is to analyze and visualize sales performance across various regions, identify trends, evaluate profitability, and support strategic decision-making.

## Table of Contents

1. [Setup](#setup)
2. [Load Data](#load-data)
3. [Data Dictionary](#data-dictionary)
4. [Data Preparation & Joins](#data-preparation--joins)
5. [KPI Definitions](#kpi-definitions)
6. [Regional Performance](#regional-performance-revenue-profit-margin)
7. [YoY Trend by Region](#yoy-trend-by-region)
8. [Channel Profitability](#channel-profitability)
9. [Seasonality (Monthly)](#seasonality-monthly)
10. [Top Products by Profit](#top-products-by-profit)
11. [Average Profit per Product](#average-profit-per-product)
12. [Profits by Year](#profits-by-year)
13. [States by Profit](#states-by-profit)

## Setup

Ensure you have the required libraries installed. You can install them using the following command:

```bash
pip install pandas numpy matplotlib
```

### Example Code

```python
import os
from pathlib import Path

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.ticker import FuncFormatter

filepath = Path("/content/Regional Sales Dataset.xlsx")
```

## Load Data

Load the relevant sheets from the Excel file into pandas DataFrames:

```python
sales_orders = pd.read_excel(filepath, sheet_name="Sales Orders", parse_dates=["OrderDate"])
customers = pd.read_excel(filepath, sheet_name="Customers")
regions = pd.read_excel(filepath, sheet_name="Regions")
state_regions = pd.read_excel(filepath, sheet_name="State Regions", skiprows=1, names=["state_code","state","us_region"])
products = pd.read_excel(filepath, sheet_name="Products")
budgets = pd.read_excel(filepath, sheet_name="2017 Budgets")
```

You can display the first few rows of each DataFrame to verify the data loaded correctly:

```python
display(sales_orders.head())
display(regions.head())
display(state_regions.head())
display(products.head())
```

## Data Dictionary

### Sales Orders

- **OrderNumber**: Unique order ID
- **OrderDate**: Date of order
- **Customer Name Index**: Foreign key to Customers sheet (index)
- **Channel**: Type of sale (Wholesale / Distributor / Export)
- **Currency Code**: Currency type (USD)
- **Delivery Region Index**: Foreign key to Regions ID
- **Product Description Index**: Foreign key to Products Index
- **Order Quantity**: Number of items ordered
- **Unit Price**: Price per item
- **Line Total**: Total price for the line item
- **Total Unit Cost**: Cost per item

### Regions

- **id**: Unique identifier
- **name**: Name of the region
- **state_code**: State abbreviation
- **state**: Full name of the state
- **latitude**: Latitude of the region
- **longitude**: Longitude of the region

### State Regions

- **state_code**: State abbreviation
- **state**: Full name of the state
- **us_region**: U.S. region (West, South, Midwest, Northeast)

### Products

- **Index**: Unique product identifier
- **Product Name**: Name of the product

## Data Preparation & Joins

Join the necessary DataFrames to create a comprehensive sales dataset:

```python
sales_regions = sales_orders.merge(
    regions[['id', 'state_code', 'state']],
    left_on='Delivery Region Index', right_on='id', how='left'
)
sales_regions = sales_regions.merge(
    state_regions[['state_code', 'us_region']],
    on='state_code', how='left'
)
sales_full = sales_regions.merge(
    products.rename(columns={'Index': 'ProductIndex'}),
    left_on='Product Description Index', right_on='ProductIndex',
    how='left'
)

# Add date components
sales_full['Year'] = sales_full['OrderDate'].dt.year
sales_full['Month'] = sales_full['OrderDate'].dt.month
sales_full['Quarter'] = sales_full['OrderDate'].dt.to_period('Q').astype(str)

print("Date range:", sales_full['OrderDate'].min(), "-", sales_full['OrderDate'].max())
print("US Region counts:")
display(sales_full['us_region'].value_counts(dropna=False).to_frame('counts'))
```

## KPI Definitions

Define key performance indicators (KPIs) for analysis:

```python
sales_full['Revenue'] = sales_full['Line Total']
sales_full['UnitCost'] = sales_full['Total Unit Cost']
sales_full['Profit'] = (sales_full['Unit Price'] - sales_full['UnitCost']) * sales_full['Order Quantity']
sales_full['MarginPct'] = np.where(
    sales_full['Revenue'] > 0,
    sales_full['Profit'] / sales_full['Revenue'],
    np.nan
)
```

## Regional Performance (Revenue, Profit, Margin)

Analyze and visualize revenue, profit, and margin by region:

```python
by_region = sales_full.groupby('us_region', dropna=False).agg(
    Revenue=('Revenue', 'sum'),
    Profit=('Profit', 'sum')
).reset_index()
by_region['Margin %'] = (by_region['Profit'] / by_region['Revenue'] * 100).round(2)

display(by_region.sort_values('Revenue', ascending=False))
```

```python
# Visualize
plt.figure(figsize=(8, 5))
x = np.arange(len(by_region['us_region']))
width = 0.35
plt.bar(x - width / 2, by_region['Revenue'])
plt.bar(x + width / 2, by_region['Profit'])
plt.xticks(x, by_region['us_region'])
plt.ylabel("USD")
plt.title("Revenue vs Profit by US Region (2014–2018)")
plt.gca().yaxis.set_major_formatter(FuncFormatter(currency_fmt))
plt.legend(['Revenue', 'Profit'])
plt.tight_layout()
plt.savefig("outputs/charts/regional_performance.png", dpi=200)
plt.show()
```

## YoY Trend by Region

Visualize the year-over-year revenue trend by region:

```python
by_region_year = sales_full.groupby(['Year', 'us_region']).agg(
    Revenue=('Revenue', 'sum'),
    Profit=('Profit', 'sum')
).reset_index()

plt.figure(figsize=(8, 5))
for region in by_region_year['us_region'].dropna().unique():
    subset = by_region_year[by_region_year['us_region'] == region]
    plt.plot(subset['Year'], subset['Revenue'], marker='o', label=region)

plt.xlabel("Year")
plt.ylabel("Revenue (USD)")
plt.title("YoY Revenue by US Region")
plt.gca().yaxis.set_major_formatter(FuncFormatter(currency_fmt))
plt.legend()
plt.tight_layout()
plt.savefig("outputs/charts/yoy_revenue_by_region.png", dpi=200)
plt.show()
```

## Channel Profitability

Analyze and visualize profitability by sales channel:

```python
by_channel = sales_full.groupby('Channel').agg(
    Revenue=('Revenue', 'sum'),
    Profit=('Profit', 'sum')
).reset_index()
by_channel['Margin %'] = (by_channel['Profit'] / by_channel['Revenue'] * 100).round(2)

plt.figure(figsize=(8, 5))
x = np.arange(len(by_channel['Channel']))
width = 0.35
plt.bar(x - width / 2, by_channel['Revenue'])
plt.bar(x + width / 2, by_channel['Profit'])
plt.xticks(x, by_channel['Channel'])
plt.ylabel("USD")
plt.title("Revenue vs Profit by Channel")
plt.gca().yaxis.set_major_formatter(FuncFormatter(currency_fmt))
plt.legend(['Revenue', 'Profit'])
plt.tight_layout()
plt.savefig("outputs/charts/channel_profitability.png", dpi=200)
plt.show()
```

## Seasonality (Monthly)

Analyze monthly sales trends to identify seasonal patterns:

```python
by_month = sales_full.assign(MonthNum=sales_full['OrderDate'].dt.month).groupby('MonthNum').agg(
    Revenue=('Revenue', 'sum'),
    Profit=('Profit', 'sum')
).reset_index()

month_name_map = {1: 'Jan', 2: 'Feb', 3: 'Mar', 4: 'Apr', 5: 'May', 6: 'Jun', 7: 'Jul', 8: 'Aug', 9: 'Sep', 10: 'Oct', 11: 'Nov', 12: 'Dec'}
by_month['Month'] = by_month['MonthNum'].map(month_name_map)

plt.figure(figsize=(8, 5))
plt.plot(by_month['MonthNum'], by_month['Revenue'], marker='o')
plt.xticks(range(1, 13), [month_name_map[m] for m in range(1, 13)])
plt.ylabel("Revenue (USD)")
plt.title("Seasonality: Total Revenue by Month (2014–2018)")
plt.gca().yaxis.set_major_formatter(FuncFormatter(currency_fmt))
plt.tight_layout()
plt.savefig("outputs/charts/seasonality_by_month.png", dpi=200)
plt.show()
```

## Top Products by Profit

Identify and visualize the top products based on profit:

```python
top_products = sales_full.groupby('Product Name').agg(
    Revenue=('Revenue', 'sum'),
    Profit=('Profit', 'sum')
).reset_index()
top_products['Margin %'] = (top_products['Profit'] / top_products['Revenue'] * 100).round(2)

top10_products = top_products.sort_values('Profit', ascending=False).head(10).reset_index(drop=True)

plt.figure(figsize=(8, 5))
plt.barh(top10_products['Product Name'], top10_products['Profit'])
plt.xlabel("Profit (USD)")
plt.title("Top 10 Products by Profit")
plt.gca().xaxis.set_major_formatter(FuncFormatter(currency_fmt))
plt.gca().invert_yaxis()
plt.tight_layout()
plt.savefig("outputs/charts/top_products_by_profit.png", dpi=200)
plt.show()
```

## Average Profit per Product

Calculate and visualize average profit per order line for the top products:

```python
avg_profit_by_product = (
    sales_full.groupby('Product Name')
    .agg(AvgProfit=('Profit', 'mean'), Orders=('Profit', 'size'))
    .reset_index()
    .sort_values('AvgProfit', ascending=False)
    .head(10)
)

plt.figure(figsize=(9, 6))
plt.barh(avg_profit_by_product['Product Name'], avg_profit_by_product['AvgProfit'])
plt.xlabel("Average Profit per Order (USD)")
plt.title("Top 10 Products by Average Profit (per order line)")
plt.gca().xaxis.set_major_formatter(FuncFormatter(currency_fmt))
plt.gca().invert_yaxis()
plt.tight_layout()
plt.savefig("outputs/charts/top10_products_by_avg_profit.png", dpi=200)
plt.show()
```

## Profits by Year

Aggregate and visualize profits by year:

```python
profit_by_year = (
    sales_full.groupby(sales_full['OrderDate'].dt.year)
    .agg(Profit=('Profit', 'sum'))
    .reset_index()
    .rename(columns={'OrderDate': 'Year'})
)

plt.figure(figsize=(8, 5))
plt.bar(profit_by_year['Year'], profit_by_year['Profit'])
plt.xlabel("Year")
plt.ylabel("Profit (USD)")
plt.title("Profit by Year")
plt.gca().yaxis.set_major_formatter(FuncFormatter(currency_fmt))
plt.tight_layout()
plt.show()
```

## States by Profit

Visualize the top states based on profit:

```python
top_states_profit = (
    sales_full.groupby('state')
    .agg(Profit=('Profit', 'sum'))
    .reset_index()
    .sort_values('Profit', ascending=False)
    .head(10)
)

plt.figure(figsize=(9, 6))
plt.barh(top_states_profit['state'], top_states_profit['Profit'])
plt.xlabel("Total Profit (USD)")
plt.title("Top 10 States by Profit")
plt.gca().xaxis.set_major_formatter(FuncFormatter(currency_fmt))
plt.gca().invert_yaxis()
plt.tight_layout()
plt.savefig("outputs/charts/top10_states_by_profit.png", dpi=200)
plt.show()
```


