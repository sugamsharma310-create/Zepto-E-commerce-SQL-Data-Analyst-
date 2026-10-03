
# Zepto Inventory & Pricing Analysis –SQL  Project Report

## 1. Objective
Analyse a Zepto product catalogue with SQL to understand pricing, discounting, stock availability and inventory value, and to surface insights an inventory or category manager could act on.

## 2. Data overview
- **Source file:** `zepto_v2.csv` – 3,732 rows, 9 columns, 14 categories, no NULLs.
- **Price unit:** `mrp` and `discountedSellingPrice` are stored in **paise**; they were divided by 100 to get rupees.
- **Stock flag:** `outOfStock` is fully consistent with `availableQuantity` (every out-of-stock row has quantity 0 and vice versa).
- **Discount field:** `discountPercent` matches the value recomputed from MRP and selling price (within rounding) for all rows.

| Metric | Value |
|---|---|
| SKUs (after cleaning) | 3,731 |
| Out of stock | 453 (12.1 %) |
| Average discount | 7.62 % |
| Products with 0 % discount | ~31 % |
| Average MRP | ₹156.84 |
| Max discount | 51 % |

## 3. Data cleaning
| Issue | Action |
|---|---|
| 1 row with `mrp = 0` and selling price 0 (*Cherry Blossom Liquid Shoe Polish*) | Deleted |
| Prices in paise | Converted to rupees |
| 4 rows with `weightInGms = 0` (Maybelline Kajal) | Not removed; excluded from Q6 by the ≥ 100 g filter |
| 2,051 repeated product names | Expected – the same product appears in multiple pack sizes/categories; `DISTINCT` used in queries |

## 4. Findings

### Q1 – Highest discounts
Three Dukes Waffy wafer flavours lead at **51 %** (MRP ₹45), followed by RRO Ricotta / Cheddar / Mozzarella, Moi Soi Sichuan Chilli Oil and Epigamia Fruit Yogurts at **50 %**. Deep discounts concentrate in dairy and short-shelf-life items, which suggests clearance or promotional pricing.

### Q2 – High-MRP products out of stock (MRP > ₹300)
Only four distinct products: **Patanjali Cow's Ghee (₹565)**, **MamyPoko Pants XL (₹399)**, **Aashirvaad Atta with Multigrains (₹315)** and **Everest Kashmiri Lal Chilli Powder (₹310)**. All are high-demand household staples, so stock-outs likely mean lost revenue and are good candidates for priority replenishment.

### Q3 – Estimated inventory value by category
Top categories as labelled: Cooking Essentials and Munchies (₹3.37 lakh each), then Paan Corner and Personal Care (₹2.71 lakh each). Fruits & Vegetables is lowest (₹10.8 k). *See the data-quality caveat – these are inflated by duplicate blocks.*

![Inventory value](images/q3_inventory_value.png)

### Q4 – Premium, barely discounted products
39 distinct products have MRP > ₹500 and discount < 10 %. They are dominated by **jar-packed cooking oils** (Dhara, Fortune, Saffola – ₹925–₹1,250, mostly 0–1 % discount). Price-sensitive bulk staples with almost no discount are a margin opportunity but also a competitive risk.

### Q5 – Categories with highest average discount
| Rank | Category | Avg discount |
|---|---|---|
| 1 | Fruits & Vegetables | 15.46 % |
| 2 | Meats, Fish & Eggs | 11.03 % |
| 3–5 | Ice Cream & Desserts / Packaged Food / Chocolates & Candies | 8.32 % (identical – duplicated data) |

Perishables are discounted most aggressively; Home & Cleaning is lowest (5.70 %).

![Average discount](images/q5_avg_discount.png)

### Q6 – Price per gram (≥ 100 g)
Cheapest: onion, iodised salt (~₹0.02/g). Most expensive: **L'Oréal hair colour (₹3.60/g)** and **Indulekha Bhringa Hair Oil (₹3.67/g)**. Staples are priced near commodity levels while personal-care items carry the highest value density.

### Q7 – Weight classes (unique SKUs)
| Class | SKUs |
|---|---|
| Low (< 1 kg) | 1,613 |
| Medium (1–5 kg) | 147 |
| Bulk (≥ 5 kg) | 22 |

The catalogue is overwhelmingly small-pack, consistent with quick-commerce convenience shopping.

![Weight classes](images/q7_weight_class.png)

### Q8 – Inventory weight by category
Cooking Essentials and Munchies each hold ~1,405 kg of stock, far above other categories (Fruits & Vegetables ≈ 92 kg, Meats ≈ 48 kg). Heavy categories need more storage and handling capacity.

### Additional view – stock-out rate
Biscuits have the highest out-of-stock rate (28.6 %), followed by Beverages and Dairy, Bread & Batter (21.7 % each).

![Out of stock by category](images/oos_by_category.png)

## 5. Data-quality caveat: duplicated category blocks
Comparing categories shows that several are **exact row-for-row copies** differing only in the category label:

- Cooking Essentials = Munchies (514 rows each)
- Paan Corner = Personal Care (344 each)
- Ice Cream & Desserts = Chocolates & Candies = Packaged Food (388 each)
- Dairy, Bread & Batter = Beverages (129 each)

Ignoring the category column, only **1,800 of 3,731 rows are unique**. Product names confirm the mismatch (e.g. Maggi noodles and Tata Salt under *Munchies*; toothpaste and hand sanitizer under *Paan Corner*). Consequences:

- Q3, Q5 and Q8 show tied values for these categories and overstate totals (inventory value ≈ ₹22.4 lakh as labelled vs ≈ ₹10.6 lakh on unique rows).
- Overall metrics such as out-of-stock rate (12.1 %) and average discount (7.7 %) are almost unaffected, because duplicates are proportional.

**Recommendation:** treat the `Category` column as unreliable in this file; obtain the source data or re-map categories before using category-level numbers in decisions.

## 6. SQL review and suggested improvements
| Item | Suggestion |
|---|---|
| Q3 label | The query computes `price × available quantity`, i.e. **inventory value at selling price**, not revenue. Rename to `inventory_value`, and use `ORDER BY ... DESC`. |
| Q8 ordering | Same – add `DESC` to show the largest first. |
| Q6 comment/filter | Comment says "above 100 g" but the filter is `>= 100`. Align them. |
| Q6 rounding | `ROUND(...,2)` makes many products tie at ₹0.02/g. Use 3–4 decimals. |
| Q1 wording | "Best value" is ranked by discount % only; add a tie-breaker such as MRP or ₹ saved. |
| Cleaning | Also delete or flag `weightInGms = 0` rows; repeat-running the UPDATE divides prices by 100 again – wrap cleaning in a transaction or add a new column instead. |
| Deduplication | Add a `SELECT DISTINCT` view that ignores `category` to avoid duplicate counting. |

## 7. Recommendations
1. Prioritise replenishment of the four high-MRP out-of-stock staples.
2. Review Biscuits, Beverages and Dairy availability (stock-out rates of 22–29 %).
3. Review discount strategy: ~30 % of products have no discount while perishables are discounted heavily; test modest promotions on high-MRP oils.
4. Fix the category mapping before any category-level planning.

## 8. Conclusion
The catalogue is dominated by small, lightly discounted packs with a ~12 % stock-out rate. The SQL workflow cleanly covers exploration, cleaning and eight business questions, but category-level conclusions depend on correcting the duplicated category labels in the dataset.
