# Ontario BPS Energy & GHG Dashboard — Process Log

*The complete, unfiltered build log for this project — every Power Query step, every DAX formula, every dead end and the reasoning behind it. For the short, portfolio-facing version, see [README.md](README.md).*

Power BI dashboard built on Ontario's **Broader Public Sector (BPS) Energy Use and Greenhouse Gas Emissions** dataset (data.ontario.ca), covering municipalities, school boards, universities/colleges, and hospitals reporting under O. Reg. 25/23.

This is a companion project to the PL-300 (Power BI Data Analyst Associate) certification — its purpose is to demonstrate data modeling, Power Query, DAX, and dashboard design skills on a real, messy, regulator-published dataset, not to produce a novel analytical finding. It's the second in a two-project portfolio pair: the first, [Ontario Energy Mix](https://github.com/ksprihar/ontario-energy-mix) (SQL/Python), focused on analytical depth; this one focuses on BI-building competence instead.

**Live report:** https://app.powerbi.com/view?r=eyJrIjoiMmQ1NzVhMWYtNjQ3MS00MzYwLTlhZmEtMWYxODEyYWViMTRkIiwidCI6IjNlYzYzMzRlLWM2MzUtNDM4Yy05YTczLThlM2M0N2JkM2RjZSJ9&pageName=473afd5e19814b8d12b7

## Dataset Selection: BPS vs. EWRB

Two Ontario government energy datasets were considered before settling on BPS.

**EWRB (Energy and Water Reporting and Benchmarking)** — covers all Ontario buildings over 100,000 sq ft — was rejected primarily because it has no Gross Floor Area and no absolute consumption values; every metric is already a pre-computed intensity ratio (per m²/ft²). That limits any DAX built on it to averages/medians/counts — no `SUM`-based measures, % of total, weighted comparisons, or running totals, which defeats the point of a project meant to demonstrate DAX skills. EWRB was also messier in ways BPS isn't: heavy City field inconsistency (case variants, Toronto borough fragmentation) and a property-type taxonomy that expanded from 22 categories (2018) to 51 (2024).

**BPS (Broader Public Sector Energy Use and GHG Emissions)** was chosen instead because it has real GFA and absolute consumption values (GJ, kWh, metric tons CO2e) alongside intensity metrics, unlocking proper DAX (`SUM`, % of total, weighted averages, running totals). Its City field was also considerably cleaner going in — no case-variant duplicates on initial inspection, just the Toronto-borough fold to handle (see Star Schema Build Log).

## Scope

- **Years used: 2021–2023.** The source publishes one file per year from 2011–2024. 2011–2020 isn't a single stable legacy schema either — the column structure shifts across nearly every individual year in that range, not just once between two eras. That's the actual reason it was dropped rather than reconciled: matching one consistent alternate schema onto 2021–2023's ENERGY STAR Portfolio Manager–based schema (58 columns) would have been a reasonable scope call, but reconciling ten different one-off schemas year over year wasn't. 2021–2023 is the one stretch where all three years share an (almost) identical schema, still cover every sector including School Boards, and keep the project timeboxed.
- 2024 is excluded entirely, since that file drops School Board data.

## Data Cleaning Log (`fact_energy_staging`)

Each entry below is a transformation applied in Power Query, in the order performed, with the reasoning behind it.

1. **Standardized the 2021 file's schema to match 2022/2023.** 2021 was missing a `Year` column (it only had `Year Ending`, a date) — added it as a literal value. 2021's diesel column was named `Diesel #2 Use (GJ)` while 2022/2023 use `Diesel Use (GJ)` for the same field — renamed to match. 2021 also had an extra column, `Calculated with new source factors (Yes/No)`, with no equivalent in 2022/2023 — removed. This had to happen before appending, since Power Query's Append only merges cleanly when column names match exactly across all three source queries.

2. **Appended 2021, 2022, and 2023 into one fact table, fact_energy.** Once the schemas matched, all three years could be combined at a Property ID + Year grain instead of living as three separate tables.

3. **Removed unwanted columns from fact_energy**, including:
   - `Number of Buildings` — dropped entirely. Testing showed this field is unreliable: genuinely huge multi-building campuses (e.g., University of Toronto St. George Campus, which spans dozens of buildings) report a value of `1`, the same as an actual single small building. Since it doesn't reliably reflect true building count, it isn't safe to build any measure on.
   - `Custom Property ID 1/2/3 - Name/Value` (three pairs) — dropped. These don't hold genuine custom identifiers; inspection showed they're repurposed to store values (Organization, SubSector, Weekly Average Hours) that already exist in their own dedicated columns elsewhere in the table.
   - `Portfolio Manager Parent Property ID` and `Parent Property Name` — dropped. Parent/child property relationships (e.g., a hospital's multiple wings sharing a parent record) are being ignored entirely for this project, per an earlier explicit scope decision.
   - `Year Ending` — dropped. Redundant once a proper `Year` column exists on all three years (this was the field 2021 originally used in place of `Year`, before step 1's fix).
   - `Electricity Use - Grid Purchase (kWh)` and `Natural Gas Use (therms)` — dropped, keeping only their GJ-denominated equivalents. Keeps every energy column in one consistent unit instead of a mix of kWh/therms/GJ.
   - `Site EUI (GJ/m²)`, `Site EUI (ekWh/sqft)`, `Source EUI (GJ/m²)`, `Source EUI (ekWh/sqft)`, `Total (Location-Based) GHG Emissions Intensity (kgCO2e/m²)` — dropped. These pre-computed intensity figures are instead built as DAX measures (`DIVIDE(Energy, GFA)`), so that nulling a bad GFA value automatically produces a blank intensity everywhere, rather than leaving stale pre-computed ratios sitting in the fact table.
   - `Weather Normalized Site/Source Energy Use` and `Weather Normalized Site/Source EUI` (GJ/m² and ekWh/sqft variants — 6 columns total) — dropped. These were 70–76% "Not Available" across the dataset, too incomplete to build reliable measures on.
   - `Drinking Water Treatment & Distribution - Average Flow` and `Wastewater Treatment Plant - Average Influent Flow` — dropped. ~98% "Not Available" (only real for the small number of properties that are actual water/wastewater plants), a completely different unit (m³/day) than the rest of the fact table, and not used by any planned page.
   - `Address 2` — dropped. 100% "Not Available" across the dataset.
   - `List of All Property Use Types (GFA) (m²)` — dropped. A property can have multiple use types listed here, but the analysis is scoped to stay focused on each property's single primary use rather than expanding into a full use-type breakdown per property.
   - `Report Submission Date`, `Data Quality Checker Run?`, `Data Quality Checker - Date Run` — dropped. Internal Portfolio Manager audit/administrative metadata with no analytical value for the dashboard.

4. **Replaced "Not Available" with 0 in the individual energy-type columns** (Natural Gas, Propane, the Fuel Oil variants, District Heating/Cooling, etc.). For a specific fuel type, "Not Available" means the property simply doesn't use that fuel — a true zero, not a missing value.

5. **Replaced "Not Available" with null in Site Energy Use, Source Energy Use, and GHG Emissions.** Unlike an individual fuel type, "Not Available" here means the property didn't report that summary figure at all — genuinely unknown, not zero. Treating it as null (rather than 0) prevents incompletely-reported rows from artificially dragging down average-based measures.

6. **Replaced "Postsecondary" with "Post-Secondary" in the Sector column.** 2021 used "Postsecondary" while 2022/2023 used "Post-Secondary" for the same category — left unfixed, this would have split one real sector into two in any Sector-based visual, slicer, or measure.

7. **Trimmed all columns.** Removes leading/trailing whitespace (confirmed present in City and Sector values — e.g., "Toronto " vs. "Toronto") that would otherwise silently fragment identical categories into separate groups.

8. **Changed column types appropriately.** Ensures numeric fields (energy, GHG, GFA) are stored as actual numbers and dates as actual dates rather than text — necessary for correct aggregation and DAX. This had to happen *after* steps 4–5, since Power Query can't convert text like "Not Available" directly into a numeric type.

9. **Filtered out Non-BPS sector rows.** The "Non-BPS" sector only appears in the 2022 and 2023 files (~62 rows/year each, absent from 2021). Since this project's whole scope is regulated Broader Public Sector reporting, keeping a sector that's inconsistently present across years and technically outside the regulation would muddy Sector-level comparisons without adding real analytical value.

10. **Capitalized each word in the City column** (trim already applied in step 7; this adds title-casing, e.g., "OAKVILLE" → "Oakville"). This is the general-purpose city cleanup applied at the fact_energy level; several more targeted city fixes happen later on `dim_building`, driven by problems only surfaced once the Map page was built — see Star Schema Build Log and Known Data Quality Issues.

11. **Removed 3 City of London properties entirely** (City Hall — Property ID 24322103, Centennial Hall — 24322100, J Allyn Taylor Building — 24322184), 9 rows total across 2021–2023. All three share a District Steam Use figure roughly 900–1,000x too large every year; City Hall alone accounted for ~19–20% of the entire dataset's total GHG emissions each year, which is obviously not real for a single office building. Filtered on Portfolio Manager Property ID rather than Organization/Property Name, since property names like "City Hall" aren't unique across municipalities in this dataset — filtering on the numeric ID guarantees only these exact 3 properties are removed.

12. **Combined Fuel Oil #1/#2/#4/#5&6, Diesel, and Kerosene into one `Oil Use (GJ)` column**, then deleted the six source columns. All six are variants of the same underlying fuel category — refined petroleum liquid fuel used for on-site combustion — and each was individually 96–100% "Not Available." Consolidating turns six near-empty columns into one usable one without losing any real signal. Propane was deliberately left out of this bucket — it's a gas (LPG), a genuinely different fuel class from liquid fuel oil, so grouping it in would be a category error even though it's similarly sparse.

13. **Added a fallback-corrected `Site Energy Use (GJ)` column, replacing the original.** For rows where the source Site Energy Use was null, this recomputes it as the sum of the remaining retained energy columns — recovering a real number for rows where the summary figure was blank but the individual components were reported. If that computed sum is itself 0, it's treated as null rather than a genuine zero, since a real building can't plausibly report literally no energy use across every category — a computed zero means nothing was reported at all, not that consumption was truly nil. The M logic:
    ```m
    let
        computed = [Electricity Use - Grid Purchase (GJ)] + [Natural Gas Use (GJ)] + [Propane Use (GJ)]
            + [District Steam Use (GJ)] + [District Hot Water Use (GJ)] + [District Chilled Water Use (GJ)]
            + [Wood Use (GJ)] + [Oil Use (GJ)]
    in
        if [Site Energy Use (GJ)] <> null then [Site Energy Use (GJ)]
        else if computed = 0 then null
        else computed
    ```

14. **Added an `Adjusted GFA` column**, nulling out `Property GFA - Self-Reported (m²)` for three outlier patterns found during investigation:
    - GFA > 30,000 m² paired with a resulting energy intensity under 0.1 GJ/m² — implausibly large GFA (e.g., Holland Bloorview Kids Rehabilitation Hospital at 3,340,200 m², golf courses and pools reporting hundreds of thousands of m²).
    - GFA ≤ 1 m² — a placeholder value used for infrastructure assets (streetlights, traffic lights, emergency sirens) that don't have a real floor area at all.
    - Resulting energy intensity > 100 GJ/m², regardless of GFA size — catches the remaining implausible cases that don't fit a small-GFA pattern (e.g., Allan A Lamport Stadium at EUI 4,991–6,321, Township of Matachewan's Water Tower at 2,447.8, Jonathan Ashbridge Park at 435.8, Bond Park at ~342). **100 was chosen as a clean, round cutoff sitting inside a natural gap, not a precisely derived number** — checking the four property types where this problem concentrates (Drinking Water Treatment & Distribution, Wastewater Treatment Plant, Other - Utility, Other - Public Services) showed their 99th-percentile EUI consistently landing around 100–150, while every legitimate building type sampled throughout this project (offices, schools, hospitals, university campuses) topped out around 4–5, with the single highest real case around 8–10. 100 sits comfortably in the gap between those two clusters; 90, 110, or 120 would have worked equivalently — the point was landing in the right neighborhood, not the exact digit.

    The M logic:
    ```m
    if [Property GFA - Self-Reported (m²)] > 30000 and [Site Energy Use (GJ)] <> null
        and ([Site Energy Use (GJ)] / [Property GFA - Self-Reported (m²)]) < 0.1 then null
    else if [Property GFA - Self-Reported (m²)] <= 1 then null
    else if [Site Energy Use (GJ)] <> null
        and ([Site Energy Use (GJ)] / [Property GFA - Self-Reported (m²)]) > 100 then null
    else [Property GFA - Self-Reported (m²)]
    ```

    In all three cases the raw Site Energy Use/GHG figures are left untouched — they look legitimate — only the area figure (and anything calculated per-area from it) is nulled. EUI itself is calculated as a DAX measure (`DIVIDE(Energy, Adjusted GFA)`) rather than carried as a static column, so a null Adjusted GFA automatically produces a blank EUI everywhere instead of needing separate cleanup.

    **Verified before closing this out:** with the rule applied, zero rows exceed EUI 100. About 1,090 rows (2.9% of all rows with a computable EUI) still sit in the 10–100 range — 90% of those are concentrated in the same four infrastructure property types (where EUI was never a fully meaningful metric to begin with, since energy for pumps/treatment plants is driven by motor size and volume processed, not floor area), and 98% belong to the Municipal sector specifically. Public Hospital, School Boards, and Post-Secondary each have only 5–6 rows total in this band. **Decision: stop here.** The remaining imprecision is small, concentrated in property types where area-based intensity was always a soft metric, and essentially absent from the sectors where "energy efficiency per building" is the meaningful comparison — GFA and EUI are ready to use for analysis as of this point.

15. **Reordered columns** for readability:
    ```m
    = Table.ReorderColumns(#"Added adjusted GFA column", {"Year", "Sector", "Subsector", "Organization",
        "Property Name", "Primary Property Type - Self Selected", "Portfolio Manager Property ID", "Address",
        "City", "Postal Code", "Property GFA - Self-Reported (m²)", "Adjusted GFA",
        "Electricity Use - Grid Purchase (GJ)", "Natural Gas Use (GJ)", "Oil Use (GJ)", "Propane Use (GJ)",
        "District Steam Use (GJ)", "District Hot Water Use (GJ)", "District Chilled Water Use (GJ)",
        "Wood Use (GJ)", "Site Energy Use (GJ)", "Source Energy Use (GJ)",
        "Total (Location-Based) GHG Emissions (Metric Tons CO2e)", "Largest Property Use Type"})
    ```

## Star Schema Build Log

**Query architecture.** The original `fact_energy` query (Data Cleaning Log steps 1–15) was renamed to `fact_energy_staging` and its **Enable Load** turned off — it's no longer a table in the model, just a shared building block. This was necessary because `dim_building` and `dim_year` both need columns (Organization, Sector, City, Postal Code, etc.) that the real fact table shouldn't carry once the star schema is normalized. A new `fact_energy` query references `fact_energy_staging` and adds one more step — removing the descriptive columns now owned by `dim_building` — leaving just keys (`Portfolio Manager Property ID`, `Year`) and measures (GFA, Adjusted GFA, the energy columns, Site/Source Energy, GHG). Because Power Query references are live pointers to a query's *current* output, not a frozen snapshot, `dim_building` and `dim_year` reference `fact_energy_staging` directly rather than the trimmed `fact_energy` — otherwise they'd break the moment `fact_energy` dropped the columns they depend on.

**`dim_building`** (referenced off `fact_energy_staging`):
1. Reference `fact_energy_staging`, remove all columns except `Portfolio Manager Property ID`, `Year`, `Property Name`, `Organization`, `Sector`, `Subsector`, `Primary Property Type - Self Selected`, `Address`, `City`, `Postal Code`, `Largest Property Use Type`.
2. Group by `Portfolio Manager Property ID`, taking the max-`Year` row per group, then expand back into columns:
    ```m
    = Table.Group(#"Removed Other Columns", {"Portfolio Manager Property ID"},
        {{"Latest", each Table.Max(_, "Year"), type record}})
    = Table.ExpandRecordColumn(#"Grouped Rows", "Latest",
        {"Year", "Property Name", "Organization", "Sector", "Subsector",
         "Primary Property Type - Self Selected", "Address", "City", "Postal Code", "Largest Property Use Type"},
        {"Year", "Property Name", "Organization", "Sector", "Subsector",
         "Primary Property Type - Self Selected", "Address", "City", "Postal Code", "Largest Property Use Type"})
    ```
    This resolves the one-property-can-appear-up-to-3-times problem by keeping each property's most recent reported attributes, rather than relying on sort order (which Power Query doesn't reliably preserve through a `Table.Distinct` without `Table.Buffer`). Verified there are no duplicate (Property ID, Year) rows, so `Table.Max`'s tie-breaking behavior is a non-issue here.
3. Uppercased `Postal Code`.
4. Replaced the letter `O` with the digit `0` in `Postal Code`, as a blanket replace with no ambiguity — Canada Post never uses the letters D, F, I, O, Q, or U in any real postal code, so every `O` present is guaranteed to be a mistyped zero. This alone recovers most of the malformed-FSA cases found during investigation (`NOG`→`N0G`, `NOC`→`N0C`, `NOH`→`N0H`, `NOB`→`N0B`, `NOL`→`N0L`, etc. — see breakdown below).
5. Replaced the space character with nothing in `Postal Code`, to handle the mix of "space after 3 characters" and "no space" formats present across the source data.
6. Validated the full postal code against the Canadian format (Letter-Digit-Letter-Digit-Letter-Digit) and set anything that doesn't match to `null`:
    ```m
    = Table.TransformColumns(#"Replaced Value1", {{"Postal Code", each
        let
            isValid = Text.Length(_) = 6
                and List.Contains({"A".."Z"}, Text.At(_,0))
                and List.Contains({"0".."9"}, Text.At(_,1))
                and List.Contains({"A".."Z"}, Text.At(_,2))
                and List.Contains({"0".."9"}, Text.At(_,3))
                and List.Contains({"A".."Z"}, Text.At(_,4))
                and List.Contains({"0".."9"}, Text.At(_,5))
        in if isValid then _ else null
    , type text}})
    ```
    This catches the genuine junk placeholders (`UNK`, `N/A`, `VAR`, `-`, `1`, `L75`, etc.) that the `O`→`0` fix can't recover, since they were never real postal codes to begin with.
7. Renamed `City` to `Old City`, then added a new `City` column applying the Toronto-borough fold:
    ```m
    = Table.AddColumn(#"Renamed Columns", "City", each
        if List.Contains({"Scarborough", "North York", "Etobicoke", "East York", "York"}, [Old City])
        then "Toronto" else [Old City])
    ```
    Safe as an exact match because fact_energy already trimmed and title-cased City upstream, so casing is guaranteed to match the hardcoded borough list. `Old City` removed afterward.
8. Added `FSA` as the first 3 characters of `Postal Code` (null-guarded):
    ```m
    = Table.AddColumn(#"Removed Columns", "FSA", each
        if [Postal Code] is null then null else Text.Start([Postal Code], 3), type text)
    ```
    Because `Postal Code` was already validated to the full 6-character pattern, any non-null `FSA` is automatically well-formed.
9. Added constant `Province` ("ON") and `Country` ("Canada") columns — needed later for the Map page's location hierarchy.
10. Reordered columns to bring `FSA` and `City` next to `Postal Code`.

**More city cleanup, found later while building the Map page** (see Known Data Quality Issues and DAX & Report Build Log for the geocoding story that surfaced these):
- `1` and `-` junk placeholder values in `City` (3 rows) replaced with an explicit `"Unknown"` label rather than left null, so those buildings stay selectable in the City slicer instead of disappearing behind a `(Blank)` entry:
    ```m
    = Table.TransformColumns(#"Replaced Value4", {{"City", each
        if _ = "1" or _ = "-" then "Unknown" else _, type text}})
    ```
- `"City of "` and `"Town of "` prefixes stripped from City values (replaced with nothing). `"Township of "` was deliberately **left as-is** — there are too many distinct townships to safely tell which references genuinely need the prefix removed versus which would collide with another place name if stripped, so this one was left rather than risk a silent collision.
- A stray side effect of Power Query's "Capitalize Each Word": any city containing an apostrophe-S (e.g., `King'S`) came out with the S capitalized. Fixed with a targeted replace back to lowercase `'s`.

**`dim_year`** (referenced off `fact_energy_staging`):
1. Reference `fact_energy_staging`, remove all columns except `Year`.
2. Remove duplicate rows — leaves the 3 distinct years (2021, 2022, 2023).

Kept as a live reference off `fact_energy_staging` rather than a hardcoded list of 3 numbers. The trade-off: referencing means the entire staging pipeline re-runs on every refresh just to produce 3 rows, since Power Query doesn't cache a shared referenced query's output across separate branches. A hardcoded `#table({"Year"}, {{2021}, {2022}, {2023}})` would avoid that, at the cost of needing a manual edit if the year range ever changes. Deferred rather than changed — revisit only if refresh performance actually becomes a problem.

**`FSA_Lookup` — planned, then retired.** The original plan was a separate lookup table applying a majority-vote-per-FSA city correction: group by FSA, take whichever city name has the most rows within that FSA, and use it as the canonical city for every property in that FSA.

Investigating this surfaced a real, structural problem, not just an edge case. Majority vote is only valid when the different spellings within an FSA are the *same real place* — it silently breaks when an FSA legitimately spans *multiple distinct real towns*, common for rural Ontario FSAs. Example: FSA `N0G` genuinely covers Wingham, Blyth, Belgrave, and other small towns; Wingham simply has the most reporting properties. Majority-vote would have overwritten every other town's real name with "Wingham" — that doesn't fix an error, it manufactures one. A scan confirmed this isn't rare: 283 of 537 FSAs (over half) still had more than one distinct city name after basic trim/case cleanup.

**Decision: drop the general majority-vote correction entirely.** Keep only the Toronto-borough fold, since that's a known administrative fact, not a "which spelling wins" guess. Cost: a rare genuine typo that survives trim/title-case would go uncorrected — judged an acceptable, honest trade-off against silently erasing real towns' names.

**Malformed postal codes found during this investigation (156 rows, 0.36% of the dataset):** `N0G` written as `NOG` (30 rows), `UNK` (24), `N0C` written as `NOC` (21), `N/A` (16), `L75` (15), `VAR` (6), `N0H` written as `NOH` (6), `N0B` written as `NOB` (4), plus a handful of others (`KOM`, `LOG`, `STR`, `6J3`, `-`, `1`, `ONT`, etc. — 22 distinct malformed values total).

**`fact_energy`** (final): sourced from `fact_energy_staging`, with all the descriptive columns now owned by `dim_building` removed — leaving keys and measures only. `Source Energy Use (GJ)` and `Largest Property Use Type` were dropped in a final cleanup pass once the dashboard build was complete — both turned out unused: Source Energy Use never had a page built against it, and `Largest Property Use Type` is near-identical to `Primary Property Type - Self Selected`, which was the column actually used throughout.

**Measure Table.** A dedicated table created from a blank query (one column, one dummy row) to hold every DAX measure in the model, rather than scattering measures across the physical tables they reference. The dummy column was deleted directly in the main Power BI window once the first real measure existed on the table.

## Known Data Quality Issues

- **Portfolio Manager Property ID is not perfectly stable across years — mainly due to a one-time City of Toronto migration.** Checked every (Organization, Property Name) combination across 2021–2023: 1,950 of 17,011 (about 11%) have more than one distinct Property ID somewhere in the range. Of those, 162 are genuine same-year coexistence — multiple distinct sub-properties sharing one generic name (e.g., several different pump stations all named "Pumpstation"). The remaining 1,788 are true one-ID-per-year churn, and 1,751 of those (97.9%) belong to a single organization, City of Toronto, all following the same pattern — a 2021 ID replaced by a different ID that then stays stable across 2022 and 2023. This looks like a one-time, organization-wide Portfolio Manager re-registration, not scattered noise. **Decision: build `dim_building` on Property ID as planned, rather than attempt manual reconciliation.** The (Organization, Property Name) matching above is exact, not fuzzy, and precise enough to detect and quantify the churn pattern — but it isn't a guaranteed unique real-world identifier, which is exactly what the 162 same-year coexistence cases prove: the same organization can legitimately have multiple distinct properties sharing one generic name at the same time. Using that same key to actually merge individual rows — treating a 2021 ID and its apparent 2022 replacement as definitely the same building — would be right for the vast majority of the 1,751 Toronto cases, but there's no independent confirmation for any single one of them, and a wrong merge would silently combine two genuinely different buildings into one. That's a worse failure mode than the alternative: leaving them as distinct rows, which just modestly inflates a single, fully-explained count rather than risking an invisible error.
- **Adjusted GFA / EUI cleanup resolved** (Data Cleaning Log step 14). A blanket exclusion by property type was considered and rejected — each of the four suspect property types is 78–93% legitimate data; excluding them wholesale would have thrown out far more good data than it fixed.
- **Decision: keep GFA and EUI in the analysis** rather than rely on Site/Source Energy Use totals alone — this was the original reason BPS was chosen over EWRB, EUI is needed for fair per-building/per-sector efficiency comparison, and the Property Drillthrough gauges compare a property against a GFA-based target.
- **City cleanup is fully resolved**, via the Postal Code validation chain, the Toronto-borough fold, City of/Town of prefix stripping, the apostrophe-S fix, and the `1`/`-` → `"Unknown"` replacement (all in Star Schema Build Log above) — not via the originally planned `FSA_Lookup` majority-vote table, which was retired once that general approach turned out to be unsafe.
- Sector/Subsector snowflake was flattened by design: Sector and Subsector live as attribute columns on dim_building rather than as separate linked dimension tables, since the hierarchy is small and fixed.
- Parent/child property relationships are dropped entirely — out of scope.

## Model

`dim_building[Portfolio Manager Property ID]` → `fact_energy[Portfolio Manager Property ID]`, `dim_year[Year]` → `fact_energy[Year]`. Star schema, single-direction relationships throughout.

**Core measures** (Measure Table):
```dax
Total Site Energy Use (GJ) = SUM(fact_energy[Site Energy Use (GJ)])
-- same pattern repeated for Total GHG Emissions (Tons CO2), Total Electricity Use (GJ),
-- Total Gas Use (GJ), Total Oil Use (GJ), Total Propane Use (GJ), Total Wood Use (GJ)

Total Unique Properties = DISTINCTCOUNT(fact_energy[Portfolio Manager Property ID])

Median EUI (GJ/m2) =
MEDIANX(fact_energy, DIVIDE(fact_energy[Site Energy Use (GJ)], fact_energy[Adjusted GFA], BLANK()))
-- Median, not average: unusually high EUI outliers in the data would skew a straight average badly.

GHG Share % =
VAR AllEmissions = CALCULATE([Total GHG Emissions (Tons CO2)], ALL(fact_energy))
RETURN DIVIDE([Total GHG Emissions (Tons CO2)], AllEmissions, BLANK())
```

## Dashboard Design (7 pages)

Every page (except Property Drillthrough, a true drillthrough target) carries a sidebar of icon nav buttons — Exec Dashboard, Map, Property Explore. Property Drillthrough is deliberately excluded from the sidebar, since there's no sensible reason for a user to navigate there manually rather than arriving through an actual drillthrough click.

### Exec Dashboard

Ontario logo, a 4-metric KPI row (Total Unique Properties, Total Site Energy Use, Median EUI, Total GHG Emissions), and a hidden filter panel (Year button slicer, City list slicer with search) toggled via bookmark, alongside a Clear Filters bookmark that resets the page to its data-wise default.

An `Exec Parameter` field parameter (GHG Emissions / Site Energy Use) drives:
- A **Year line chart** — genuinely thin right now with only 3 data points (2021–2023); this is a data-availability limit, not a design flaw, and the chart becomes more useful as Ontario publishes more years under this reporting schema.
- A **Sector → Subsector drill-down bar chart**, with a custom Tooltip Sector page (below).
- A **Top 10 properties table**, rankable by whichever metric is selected, drillthrough-enabled on click.

A **Median EUI by Property Type** bar chart sits below, static (not tied to `Exec Parameter`), filtered to the Top 25 property types by property count — with a custom Tooltip Type page. This started as a line-and-column combo chart (Median EUI as columns, Count of Properties as a line) but was later simplified to a plain bar chart showing Median EUI alone, for a cleaner read. Count of Properties isn't plotted anymore, but it was worth keeping visible somewhere, so it now lives only in the Tooltip Type page alongside the full property type name.

**Two report-wide bugs surfaced and fixed on this page, both worth knowing about:**

1. **Field-Parameter-driven line/marker colors reverting to Power BI's default blue on Service publish.** Three attempts:
   - First: a per-series color override keyed to one specific measure's identity. Broke the instant the field parameter switched metrics — no matching rule existed for the other measure, so it fell back to default blue.
   - Second: a hardcoded color column added to the field-parameter table, with the line's color bound via Conditional Formatting → Field value against that column. This fixed the *line* but not the marker/hover-point or the tooltip's color swatch — which turned out to be a genuinely separate Power BI format property (`dataPoint` fill vs. `lineStyles` stroke) that doesn't inherit the line's color at all.
   - **Actual fix:** temporarily disable forced single-select on the metric slicer so every metric renders as its own series simultaneously, manually color each one to the project's black, then re-enable forced single-select. This bakes a permanent, explicitly-named color entry for every possible metric directly into the visual. Nothing is left to fall back to a theme default for, regardless of which single metric ends up selected — no DAX, no parameter-table color column, no dependency on conditional formatting behaving consistently between Desktop and Service. The helper color columns from the second attempt were deleted once this was in place.
2. **Default (non-custom) tooltips using Power BI's stock light styling instead of the project's dark palette.** Fixed natively via the Format pane's Properties tab — no custom theme JSON required.

**Also tried and dropped:** a decomposition tree (Analyze: selected metric; ExplainBy: City → Organization → Property Name) was built, then dropped in favor of the current Year-chart + Median-EUI-chart layout, which fit the available width better. The hidden visual was later deleted outright rather than left on the page.

### Map

Bubble map (Azure Maps visual), bubble size driven by a `Map Parameter` field parameter with 7 metrics (Site Energy, GHG, Electricity, Gas, Oil, Propane, Wood). Sidebar has the same filter-toggle/Clear Filters pattern as Exec Dashboard; the slicer panel here holds Year, Sector, and Subsector.

**Geocoding.** Bare `City` text geocodes ambiguously against the whole world — Ontario has small towns sharing names with far more prominent places elsewhere. Fixed in stages:
1. Built a location hierarchy on dim_building: Country → Province → City → Postal Code → Address.
2. That alone wasn't enough — some locations still resolved outside Ontario/Canada. Added a calculated column, `Location = [City] & ", ON, Canada"`, giving the geocoder one unambiguous string instead of relying on it to combine hierarchy levels correctly at query time.
3. Even with both in place, 3 towns still resolved wrong: `Colchester N Twp`, `Vermillion Bay`, and `Fort Eire` (a misspelling of Fort Erie). Fixed via targeted Replace Values: `Colchester N Twp` → `Colchester North`, `Vermillion` → `Vermilion`, `Eire` → `Erie`. After this, everything resolved inside Ontario.

**Top N filtering.** A calculated table drives how many locations show on the map:
```dax
Top N Selector = DATATABLE(
    "Label", STRING, "N", INTEGER, "Sort Order", INTEGER,
    {{"Top 10", 10, 1}, {"Top 20", 20, 2}, {"Show All", 999999, 3}}
)
Selected Top N = SELECTEDVALUE('Top N Selector'[N], 20)   -- 20 is default

Location Rank = RANKX(ALLSELECTED(dim_building[Location]), [Selected Map Metric], , DESC, Dense)
Show Place = IF([Location Rank] <= [Selected Top N], 1, 0)
```
`Show Place = 1` is applied as a visual-level filter on the map.

**A ranking-mechanics bug, found and fixed:** Wood energy use only has non-zero values for 3 properties in the entire dataset. With `Dense` ranking, every property past those 3 ties at the next rank — so no matter whether Top 10, Top 20, or Show All was selected, the map showed *every* location whenever Wood was the active metric, since dense ranking collapsed everything past rank 3 into one tied group under the "Top 20" cutoff. Not a map bug — a ranking-mechanics edge case specific to metrics with a long run of ties. Fixed with an additional visual-level filter: `[Selected Map Metric] <> 0`, which correctly narrows Wood down to just its 3 real locations regardless of the Top N selection.

A spotlight card block shows the single top-ranked property and its value for the selected metric (Property Name set as Summarized + Category in the drillthrough settings, so this card also supports drillthrough). A custom Tooltip City page provides richer hover context than the default.

### Property Drillthrough

Drillthrough on Portfolio Manager Property ID + Property Name. Three identity cards at top-left, each a plain `SELECTEDVALUE` with a fallback for the (structurally impossible in practice, but defensive) multi-selection case:
```dax
Card Property = SELECTEDVALUE(dim_building[Property Name], "Multiple Properties")
-- same pattern for Organization, City
```

Three gauges across the top compare the selected property against a target derived from the median for its own property type:
```dax
Target EUI (GJ/m2) =
VAR CurrentUseType = SELECTEDVALUE(dim_building[Primary Property Type - Self Selected], BLANK())
RETURN CALCULATE([Median EUI (GJ/m2)], FILTER(ALL(dim_building),
    dim_building[Primary Property Type - Self Selected] = CurrentUseType))

Target Site Energy Use (GJ) = SUM(fact_energy[Adjusted GFA]) * [Target EUI (GJ/m2)]

Target Emissions (Tons CO2) =
VAR CurrentUseType = SELECTEDVALUE(dim_building[Primary Property Type - Self Selected], BLANK())
VAR MedianEmissionIntensity = CALCULATE(
    MEDIANX(fact_energy, DIVIDE(fact_energy[Total (Location-Based) GHG Emissions (Metric Tons CO2e)],
        fact_energy[Adjusted GFA], BLANK())),
    FILTER(ALL(dim_building), dim_building[Primary Property Type - Self Selected] = CurrentUseType))
RETURN SUM(fact_energy[Adjusted GFA]) * MedianEmissionIntensity
```

Below the gauges: two column charts (Total GHG by year, Total Site Energy by year for the selected property), then a `Building Parameter` metric slicer (Electricity, Gas, Oil, Propane, Wood) driving a by-year line chart — same 3-data-point caveat as the Exec Dashboard's Year chart.

**"Explore other properties" — the one genuinely hard problem this build ran into.** The original plan was a single Property Drillthrough page with a button that cleared the drillthrough's own filter and revealed browsing slicers, all on the same page. That does not work, under any configuration: Power BI bookmarks cannot capture or override a Drillthrough-type filter's value. Confirmed directly — pulling two bookmarks meant to represent different states (filters shown vs. hidden) apart and comparing their filter entries for the drillthrough field showed them to be identical and valueless, meaning neither bookmark actually touched the underlying filter at all. This is a structural Power BI limitation (drillthrough filters are navigation-context, not report-state), not a configuration mistake.

The fix: split into two pages. Property Drillthrough stays a true drillthrough (the best interaction for "jump straight to this one property from anywhere in the report"). **Property Explore** is a full clone of the Drillthrough page — same gauges, same charts, same identity cards — with one difference: the "Explore" button is replaced by three dropdown slicers (City, Organization, Property) with search, positioned bottom-left, plus a Reset Filters button. Since Property Explore is never touched by a live drillthrough filter, its slicers are ordinary and fully bookmark-controllable.

The "Explore other properties" button on Property Drillthrough is bound not to a plain Page Navigation action, but to a bookmark captured on Property Explore with every slicer deselected. Applying a bookmark both navigates to whatever page it was captured on *and* forcibly resets every captured visual to that state — guaranteeing a clean landing regardless of what was left selected there before. The same bookmark backs the in-page Reset Filters button on Property Explore itself.

One nuance surfaced during review: the `Building Parameter` metric slicer on Property Explore is a Field Parameter, not an ordinary categorical slicer — it's single-select by nature, and "nothing selected" isn't a valid state the way it is for City/Org/Property (where no selection just means "show everything"). So the reset bookmark forces this slicer to a specific entry (the first in the parameter table) rather than leaving it blank — not a workaround, but the only coherent choice, since the gauges/charts downstream need `SELECTEDVALUE` to resolve to *something*.

A permanent, generic sidebar shortcut to Property Explore also exists (separate from the "Explore" button), deliberately *not* wired to the reset bookmark — intentional, so a user who navigates away and wants to return to Property Explore mid-browse gets back to it exactly as they left it, rather than being reset every time.

**Sibling buildings — built, then dropped.** An earlier design (`dim_building_peers`, a disconnected table related to `fact_energy` to show other properties under the same organization) was actually implemented, then dropped once the two-page Explore pattern above proved to be the more useful and more general solution to the same underlying need.

**What-If GHG modeling — dropped.** The original plan was a GHG trend chart with a What-If slider modeling reduced use of "unclean" fuels. Cut because Total GHG Emissions is a single reported figure with no per-fuel breakdown in the data, so modeling "reduce Gas Use by X%" would need external, undocumented emission factors per fuel — and Ontario's electricity grid is unusually clean (nuclear/hydro-heavy), so any naive proportional assumption (a GJ of electricity ≈ a GJ of gas) would be actively misleading. District Steam/Hot Water/Chilled Water are effectively impossible to attribute an honest factor to at all. Replaced with the plain GHG-by-year column chart described above.

### Property Explore

A complete clone of Property Drillthrough (see above for the shared build) — the only difference is the Explore button being replaced by City/Organization/Property dropdown slicers and a Reset Filters button.

### Tooltip City

`Tooltip City = SELECTEDVALUE(dim_building[City], "Multiple Cities")`, a column chart of the selected Map metric by year, and a 5-card KPI block (Total Unique Properties, Total Site Energy, Median EUI, GHG, GHG Share %).

### Tooltip Sector

A clone of Tooltip City, adapted for the Sector → Subsector drill-down on the Exec Dashboard's Sector chart:
```dax
Tooltip Sector = SELECTEDVALUE(dim_building[Subsector], SELECTEDVALUE(dim_building[Sector], "Multiple Sectors"))
```
Its by-year chart uses `Exec Parameter` rather than `Map Parameter`.

A DAX lesson learned building this: `ISINSCOPE` was tried first to detect whether the visual was currently drilled to Subsector or sitting at Sector — it didn't work, because `ISINSCOPE` only reads the current grouping level *within the same visual doing the drilling*. A separate tooltip page's card has no hierarchy structure of its own to be "in scope" of, so it never resolved correctly. `SELECTEDVALUE`'s built-in fallback argument works instead, because it reads the plain filter context that *is* passed through to a tooltip page regardless of which visual triggered it — functionally equivalent to an explicit `HASONEVALUE` check, just more compact.

### Tooltip Type

The simplest of the three: selected property type name, property count, and Median EUI. Supports the Exec Dashboard's Median EUI chart.

This tooltip page predates the Median EUI chart's simplification to a plain bar chart. While that chart was still a line-and-column combo (Median EUI as columns, Count of Properties as a line), Power BI's native combo-chart tooltip showed *different* content depending on whether you were hovering the bar or the line marker for the same category — a real, long-documented limitation, not a configuration mistake. A custom Report Page tooltip sidestepped this, since it's bound to the whole visual rather than per-series. Once the chart was simplified to a plain bar (Median EUI alone, Count of Properties dropped from the visual entirely), the original bar-vs-line mismatch stopped being relevant — but the custom tooltip stayed, since it's now the only place Count of Properties shows up at all, alongside the full property type name the crowded axis labels (25 property types in one chart width) don't have room for.
