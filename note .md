# Customer Churn Dashboard: Notes

## Tool used

Power BI Desktop on Windows. Dataset: Telco Customer Churn (Kaggle), file `WA_Fn-UseC_-Telco-Customer-Churn.csv` (7,043 rows, 21 columns). No other dataset was used.

The report has three visible pages and one hidden tooltip page:

- **Churn Dashboard**: KPI cards, four churn-rate charts and three slicers.
- **Segment Analysis**: a Contract x Internet Service matrix and two detail tables (payment method, tenure band), each showing churn rate next to customer count.
- **What-if Scenario**: a slider that estimates the revenue saved if churn drops in the worst segment.
- **Tooltip - Segment Detail** (hidden): the detail window that appears when hovering over a bar.

## Power Query decisions

All cleaning and typing was done in Power Query. No Python or Excel preprocessing was used.

### Applied steps (in order)

| # | Step | What it does |
|---|---|---|
| 1 | Source | Loads the CSV |
| 2 | Promoted Headers | Uses the first row as column names |
| 3 | Set initial data types | Types every column explicitly |
| 4 | Replace blank TotalCharges with 0 | Replaces the missing `TotalCharges` values with 0 |
| 5 | Set TotalCharges to number | Converts `TotalCharges` from text to a decimal number |
| 6 | Rename columns to snake_case | Renames columns (for example `monthly_charges`, `churn`) |
| 7 | Add tenure_band | Creates the tenure bucket |
| 8 | Set tenure_band type | Sets type to text |
| 9 | Add churn_flag | Yes = 1, No = 0 |
| 10 | Set churn_flag type | Sets type to whole number |
| 11 | Add tenure_band_score | Numeric sort key for the bands |
| 12 | Set tenure_band_score type | Sets type to whole number |
| 13 | Q1, Q2, Q3 | Calculates the quartile cutoffs of `monthly_charges` |
| 14 | Add charge_band | Labels each customer with a quartile band |
| 15 | Reorder columns | Moves the columns into a tidier order |
| 16 | Rename columns for display | Final name clean-up |

The order matters because each step depends on the previous one. For example, `TotalCharges` has to be cleaned before it is converted to a number, and the quartile cutoffs are calculated after all typing is finished.

### TotalCharges

11 customers have `tenure = 0`. They are new customers who have not been billed yet, so their `TotalCharges` is blank. The blank values stop the column from being a number.

**Decision: replace the blanks with 0, then set the column type to a number.**

Why:

- These customers have paid nothing so far, so 0 is factually correct, not a guess.
- They are real customers with a real `MonthlyCharges`. Filtering them out would understate Total Customers (7,032 instead of 7,043) and Monthly Revenue at Risk.
- None of the 11 customers churned, so keeping them does not distort the churn rate.

After the change, `TotalCharges` shows 100% valid and 0 errors in the column quality check.

### Data types

Every column has an explicit type: text for the categorical columns and the customer ID, whole number for `SeniorCitizen`, `tenure`, `churn_flag` and `tenure_band_score`, and decimal number for `MonthlyCharges` and `TotalCharges`.

### Added columns

| Column | Logic |
|---|---|
| `churn_flag` | `churn = "Yes"` gives 1, otherwise 0 |
| `tenure_band` | tenure <= 12 gives `0-12`, <= 24 gives `13-24`, <= 48 gives `25-48`, otherwise `49+` |
| `tenure_band_score` | tenure <= 12 gives 1, <= 24 gives 2, <= 48 gives 3, otherwise 4. Used with "Sort by column" so the bands always display in tenure order. Without it, Power BI sorts a chart by the value (the highest churn rate first) |
| `charge_band` | Quartile bands of `monthly_charges`, with cutoffs calculated by `List.Percentile` (Q1 = 35.50, Q2 = 70.35, Q3 = 89.85). Labels: `Q1 - Low`, `Q2 - Mid-Low`, `Q3 - Mid-High`, `Q4 - High`. Each band holds about 25% of customers |

## DAX measures

All measures are explicit measures stored in the `Telco-Customer-Churn` table.

```dax
Total Customers = COUNTROWS('Telco-Customer-Churn')
```
Counts the rows (customers) in the current filter context.

```dax
Churned Customers = CALCULATE([Total Customers], 'Telco-Customer-Churn'[churn] = "Yes")
```
Same count, but `CALCULATE` changes the filter so that only churned customers are counted.

```dax
Churn Rate % = DIVIDE([Churned Customers], [Total Customers])
```
Churned customers divided by all customers, recalculated for whatever the user selects.

```dax
Avg Monthly Charges = AVERAGE('Telco-Customer-Churn'[monthly_charges])
```
Average monthly bill in the current filter context.

```dax
Monthly Revenue at Risk = CALCULATE(SUM('Telco-Customer-Churn'[monthly_charges]), 'Telco-Customer-Churn'[churn] = "Yes")
```
Total monthly charges of customers who churned.

### Why DIVIDE and not `/`

If a filter combination contains no customers, the denominator is 0. The `/` operator returns an error or infinity, which can break a visual. `DIVIDE` returns a blank instead, so the visual simply stays empty.

### Why churn rate is a measure and not a column

A measure recalculates for every slicer selection and every bar. The average of pre-calculated rates would give a wrong overall figure, because the average of rates is not the overall rate. The measure always divides the churned count by the customer count of the current selection, so it stays correct when sliced.

### Checks

With no filters applied: Total Customers = 7,043, Churn Rate % = 26.5%, Monthly Revenue at Risk = $139.13K, Avg Monthly Charges = $64.76.

## Dashboard

- **KPI cards:** Total Customers, Churn Rate %, Monthly Revenue at Risk, Avg Monthly Charges.
- **Charts (all show churn RATE, not counts):** by Contract, Internet Service, Payment Method and Tenure Band (ordered 0-12 to 49+).
- **Slicers:** Contract, Internet Service, Tenure Band. Clicking a slicer or a bar cross-filters all other visuals and cards. I confirmed that every visual updates.
- **Segment Analysis page:** matrix of Contract x Internet Service with churn rate and customer count in each cell, plus detail tables for payment method and tenure band.

## Bonus features

| Bonus | Status |
|---|---|
| Conditional formatting (churn above average turns red) | Done |
| Matrix of Contract x InternetService with churn rate | Done |
| Tooltip page | Done |
| What-if parameter | Done |
| Publish to Power BI Service | Not done |

### 1. Conditional formatting

```dax
Overall Churn Rate = CALCULATE([Churn Rate %], ALL('Telco-Customer-Churn'))
```
Ignores all filters and always returns the company-wide rate (26.5%).

```dax
Bar Color = IF([Churn Rate %] > [Overall Churn Rate], "#D64550", "#3F51B5")
```
Returns red when a segment is above the overall rate and blue when it is below. The bars use it through "Field value" formatting, so the comparison does not change when a slicer is used. The matrix cells use a rule instead: churn rate at 27% or above gets a red background.

### 2. Matrix

The matrix on the Segment Analysis page puts Contract on the rows, Internet Service on the columns, and shows `Churn Rate %` and `Total Customers` in every cell. Reading rate and volume together is what identifies the worst segment (Month-to-month + Fiber optic: 54.6% churn, 2,128 customers).

### 3. Tooltip page

A hidden page called **Tooltip - Segment Detail** is set up as a tooltip page and linked to the four charts on the main dashboard. When hovering over a bar, a small window shows `Total Customers`, `Churned Customers`, `Churn Rate %` and `Monthly Revenue at Risk` for that bar. For example, hovering over Month-to-month shows 3,875 customers, 1,655 churned and 42.7% churn. The page filters itself by the hovered bar, so no extra logic was needed.

### 4. What-if parameter

A numeric range parameter called `Churn Reduction` (0 to 50, step 5, default 10) drives a slider on the **What-if Scenario** page. Two measures use it:

```dax
Worst Segment Revenue at Risk =
CALCULATE(
    [Monthly Revenue at Risk],
    'Telco-Customer-Churn'[contract] = "Month-to-month",
    'Telco-Customer-Churn'[internet_service] = "Fiber optic"
)
```
Monthly charges of customers who churned in the worst segment. This is fixed at $100,482 (1,162 churned customers), about 72% of the total $139K at risk.

```dax
Monthly Revenue Saved =
[Worst Segment Revenue at Risk] * [Churn Reduction Value] / 100
```
Revenue saved per month if churn in that segment falls by the slider percentage. `Annual Revenue Saved` is this value multiplied by 12.

| Churn reduction | Monthly revenue saved | Annual revenue saved |
|---|---|---|
| 10% | $10,048 | $120,578 |
| 15% | $15,072 | $180,868 |
| 20% | $20,096 | $241,157 |
| 50% | $50,241 | $602,892 |

**Assumption:** the percentage is a relative reduction of the customers who would have left, and the customers who are saved keep paying their current monthly charge. It is a simple estimate and does not include the cost of the retention offer, so the real saving would be lower.

The segment is hard-coded in the measure, so the Contract and Internet Service slicers are not used on this page.

### 5. Publish to Power BI Service

Not done. Publishing requires a work or school Microsoft account, which I did not have. The report was built and tested in Power BI Desktop only.

## Key findings

Overall churn is **26.5%** (1,869 of 7,043 customers). Churned customers represent about **$139K of monthly revenue at risk**.

| Segment | Churn rate | Customers | Share of all churn |
|---|---|---|---|
| Month-to-month + Fiber optic | 54.6% | 2,128 | 62.2% |
| Tenure 0-12 months | 47.4% | 2,186 | 55.5% |
| Electronic check | 45.3% | 2,365 | 57.3% |
| Month-to-month (all) | 42.7% | 3,875 | 88.6% |
| Fiber optic (all) | 41.9% | 3,096 | 69.4% |

Segments overlap, so the shares do not add up to 100%.

### The two segments that matter most

1. **Month-to-month + Fiber optic** has the highest churn rate (54.6%) and a large base (2,128 customers, about 30% of all customers). It accounts for 62% of all churn.
2. **Tenure 0-12 months** has 47.4% churn across 2,186 customers, so the first year is the critical period. Churn falls steadily with tenure: 28.7% (13-24), 20.4% (25-48), 9.5% (49+).

### Why volume matters

A segment with a very high rate but only a handful of customers is not actionable. A rate based on so few customers can swing by chance, and even saving every customer in it would barely change overall churn. Both segments above have more than 2,000 customers, so improving them would move the total noticeably. This is why every segment is read as rate and customer count together.

### Other observations

- **Fiber optic is not the root problem, the contract is.** Fiber customers on a two-year contract churn at only 7.2% (429 customers), versus 54.6% on month-to-month.
- The combination **Month-to-month + Fiber optic + Electronic check** covers 1,307 customers with 60.4% churn, more than double the average.
- Payment method is probably a signal of low commitment rather than a cause of churn. This is my interpretation, not something the data proves.

### Limitation

The data shows where churn happens, not why. Price, service quality or competitor offers could explain the Fiber result, and this dataset cannot separate them.

## Recommendations (prioritised)

**1. Move Month-to-month + Fiber optic customers onto 12-month contracts.**
Target: 2,128 customers, 54.6% churn, 62% of all churn. Offer a discount or a free add-on (for example tech support or online security) in exchange for a 1-year contract. This comes first because it has the highest rate and the biggest share of churn. Two-year Fiber customers churn at only 7.2%, which shows how much a contract commitment reduces risk. The what-if page estimates the size of the prize: a 20% reduction in churn in this segment would protect about $20K of monthly revenue (about $241K a year), before the cost of the offer.

**2. Build a first-year onboarding program for new customers (tenure 0-12).**
Target: 2,186 customers, 47.4% churn. Contact new customers during the first 3 months (welcome call, service check, help with setup) and flag those with no add-on services for follow-up. This comes second because the volume is as large as segment 1, and it prevents churn early, before customers settle into month-to-month habits. It also supports recommendation 1: a satisfied customer at month 6 is easier to convert to a yearly contract.

**3. Encourage Electronic check customers to switch to automatic payment.**
Target: 2,365 customers, 45.3% churn, compared with 15-19% for the other payment methods. Offer a small monthly credit for moving to bank transfer or credit card autopay. This comes third because payment method is likely a signal of low commitment rather than the cause, so the effect is less certain than for recommendations 1 and 2. It is still cheap to run and reaches a large group.

