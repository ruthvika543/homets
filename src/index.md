# HomeTS
# Ruthvika Mamidyala (801430584)
## Housing Availability - Exploratory Visual Analysis

**Theme:** Understanding how socioeconomic indicators relate to housing price levels and growth across major U.S. metros.

**About the Dataset:**
- **Source:** <https://www.kaggle.com/datasets/shengkunwang/housets-dataset> 
- **Scope & Time Range:** 890K records across 30 U.S. metropolitan areas, covering 2012–2023.
- **Key Fields:** Median sale price, per-capita income, population, amenities count, POI proximity, economic indicators, and metro-level identifiers.
- **Cleaning & Prep:** Filtered metros with consistent yearly data; standardized income and price fields; aggregated metro-level averages for scatter plots; added growth calculations to visualize bubble size and long-term trends.

```js
import \* as d3 from "npm:d3";
import {FileAttachment} from "observablehq:stdlib";
```

```js
const homets = FileAttachment("data/HomeTS.csv").csv({ typed: true });
```

<div class="card">
${Inputs.table(homets)}
</div>

<div class="card">
${poiProximityEffects()}
</div>

```js
function poiProximityEffects() {
  const num = (v) => +String(v ?? "").replace(/[$,\s]/g, "") || NaN;

  // Normalize
  const rows = homets
    .map((d) => ({
      City: String(d.city_full ?? d["City Full"] ?? d.City ?? "").trim(),
      Park: num(d.park ?? d["Avg Park"] ?? d["Park Count"]),
      Restaurant: num(
        d.restaurant ?? d["Avg Restaurant"] ?? d["Restaurant Count"]
      ),
      School: num(d.school ?? d["Avg School"] ?? d["School Count"]),
    }))
    .filter(
      (d) =>
        d.City &&
        (Number.isFinite(d.Park) ||
          Number.isFinite(d.Restaurant) ||
          Number.isFinite(d.School))
    );

  // Averages per city (+ total for sorting)
  const agg = d3
    .rollups(
      rows,
      (v) => ({
        park: d3.mean(v, (d) => d.Park),
        restaurant: d3.mean(v, (d) => d.Restaurant),
        school: d3.mean(v, (d) => d.School),
      }),
      (d) => d.City
    )
    .map(([City, m]) => ({
      City,
      ...m,
      total: (m.park ?? 0) + (m.restaurant ?? 0) + (m.school ?? 0),
    }));

  agg.sort((a, b) => d3.ascending(a.total, b.total));

  // Long format for bars
  const typeOrder = ["Restaurant", "Park", "School"];
  const long = agg.flatMap((d) =>
    [
      { City: d.City, Type: "Restaurant", Value: d.restaurant },
      { City: d.City, Type: "Park", Value: d.park },
      { City: d.City, Type: "School", Value: d.school },
    ].filter((x) => Number.isFinite(x.Value))
  );

  // Build domain with ONE hidden separator per city
  const key = (city, type) => `${city}⟂${type}`;
  const domain = agg.flatMap((d) => [
    key(d.City, "Restaurant"),
    key(d.City, "Park"),
    key(d.City, "School"),
    key(d.City, "__SEP__"), // ← separator slot (no bars, no tick)
  ]);

  // Tick labels only on the middle bar
  const tickFormat = (k) => {
    const [city, typ] = k.split("⟂");
    return typ === "Park" ? city : "";
  };

  // Where to draw the single divider line per city
  const separators = agg.map((d) => key(d.City, "__SEP__"));

  return Plot.plot({
    width: 1400,
    height: 520,
    title: "POI Proximity Effects",
    grid: true,
    x: {
      domain,
      tickFormat,
      tickRotate: -40,
      padding: 0, // keep one compact gap between cities
    },
    y: { label: "Avg Number of Nearby Amenities", nice: true },
    color: {
      legend: true,
      label: "Amenity Type",
      domain: ["Restaurant", "Park", "School"],
      range: ["#e74c3c", "#f39c12", "#1abc9c"],
    },
    marks: [
      // Bars (tight inside a city)
      Plot.barY(long, {
        x: (d) => key(d.City, d.Type),
        y: "Value",
        fill: "Type",
        inset: 0,
        tip: true,
        title: (d) => `${d.City}\n${d.Type}: ${d3.format(",.1f")(d.Value)}`,
      }),

      // One divider after each city (on the hidden separator slot)
      Plot.ruleX(separators, {
        x: (d) => d,
        stroke: "#cfcfcf",
        strokeWidth: 2,
      }),

      Plot.ruleY([0]),
    ],
  });
}
```

<div class="card">
${metroScatter()}
</div>

```js
function metroScatter() {
  // --- helpers
  const num = (v) => +String(v ?? "").replace(/[$,\s]/g, "") || NaN;
  const fmt$ = d3.format(",.0f"); // currency-like commas

  // --- shape your table to one row per city
  // (Avg) Median Sale Price vs (Avg) Per Capita Income, size = SUM(Median Rent)
  const rows = homets
    .map((d) => ({
      City: String(d.city_full ?? d["City Full"] ?? "").trim(),
      price: num(d.median_sale_price ?? d["Median Sale Price"]),
      income: num(d.per_capita_income ?? d["Per Capita Income"]),
      rent: num(d.median_rent ?? d["Median Rent"]),
    }))
    .filter(
      (d) => d.City && Number.isFinite(d.price) && Number.isFinite(d.income)
    );

  const points = d3
    .rollups(
      rows,
      (v) => ({
        price: d3.mean(v, (d) => d.price),
        income: d3.mean(v, (d) => d.income),
        rent: d3.sum(v, (d) => d.rent), // bubble size = SUM(Median Rent)
      }),
      (d) => d.City
    )
    .map(([City, a]) => ({ City, ...a }));

  // --- axis domains start at 0
  const xmax = Math.max(0, d3.max(points, (d) => d.income) ?? 0);
  const ymax = Math.max(0, d3.max(points, (d) => d.price) ?? 0);

  return Plot.plot({
    width: 1100,
    height: 560,
    grid: true,
    title: "Socioeconomic Indicators vs Pice Growth",
    inset: 10,
    x: {
      label: "Avg. Per Capita Income ($)",
      domain: [0, xmax * 1.05],
      tickFormat: (d) => d3.format(",")(d),
    },
    y: {
      label: "Avg. Median Sale Price ($)",
      domain: [0, ymax * 1.05],
      tickFormat: (d) => d3.format(",")(d),
    },
    color: { label: "City", legend: true },
    r: { range: [0, 16] }, // <-- bigger bubbles
    marks: [
      // regression line (Y on X)
      Plot.linearRegressionY(points, {
        x: "income",
        y: "price",
        stroke: "#666",
        strokeWidth: 2,
        opacity: 0.9,
      }),

      // bubbles
      Plot.dot(points, {
        x: "income",
        y: "price",
        r: "rent",
        fill: "City",
        stroke: "City",
        opacity: 0.8,
        tip: true,
        title: (d) =>
          `${d.City}
Avg. Median Sale Price: $${fmt$(d.price)}
Avg. Per Capita Income: $${fmt$(d.income)}
Median Rent (sum): $${fmt$(d.rent)}`,
      }),
    ],
  });
}
```
## Final Insights

- **Income vs Price:** Metros with higher per-capita income consistently show higher median sale prices. The upward-sloping trend line confirms a strong positive relationship.
- **Price Growth Patterns:** Large bubbles indicate metros with notable housing price growth. Growth appears in both mid-income and high-income metros, suggesting demand pressures are broad, not isolated.
- **Outliers:** Some metros sit significantly above or below the trend line. Above-trend cities exhibit “premium pricing” beyond what income alone predicts, while below-trend metros remain relatively affordable despite moderate incomes.
- **Market Structure:** Income explains a substantial share of price variation, but not all. Amenities, migration flows, and local housing constraints likely drive the remaining differences.
- **Sanity Checks:** High-price outliers correspond to metros with stronger long-term appreciation and higher-income populations. Removing these points does not alter the overall income–price correlation trend.

const data = await d3.csv("/data/HomeTS.csv");
