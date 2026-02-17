---
name: STAtlas DA
assetId: bdebbdd8-b396-45d5-aa4b-a0efbf564022
type: page
---

# Statistical Model

```sql model_runs
SELECT model_run_id, max(published_date) as published_date
FROM _statistical__model
GROUP BY model_run_id
HAVING published_date IS NOT NULL
ORDER BY published_date DESC
```

{% dropdown
    id="run_filter"
    data="model_runs"
    value_column="model_run_id"
    label_column="published_date"
    title="Select Model Run"
    select_first=true
/%}

{% toggle
    id="avg_toggle"
    label="Average across date"
/%}

```sql model_data
SELECT dd as delivery_date, variable, avg(value) as value
FROM (
    SELECT
        IF({{avg_toggle}}, formatDateTime(toDate(assumeNotNull(delivery_date)), '%m-%d'), formatDateTime(assumeNotNull(delivery_date), '%m-%d %H:%i')) as dd,
        variable,
        value
    FROM _statistical__model
    WHERE variable IS NOT NULL
      AND {{run_filter.filter}}
      AND delivery_date IS NOT NULL
) sub
GROUP BY dd, variable
ORDER BY dd, variable
```

{% dropdown
    id="variable_filter"
    data="model_data"
    value_column="variable"
    title="Select Variable(s)"
    multiple=true
/%}

```sql weekend_holidays
WITH weekend_days AS (
    SELECT DISTINCT toDate(assumeNotNull(delivery_date)) as dt
    FROM _statistical__model
    WHERE variable = 'Weekend_Holiday'
      AND value = 1
      AND delivery_date IS NOT NULL
      AND {{run_filter.filter}}
),
grouped AS (
    SELECT dt,
        dt - ROW_NUMBER() OVER (ORDER BY dt) as grp
    FROM weekend_days
)
SELECT
    IF({{avg_toggle}}, formatDateTime(min(dt), '%m-%d'), formatDateTime(min(dt), '%m-%d') || ' 00:00') as start_date,
    IF({{avg_toggle}}, formatDateTime(max(dt), '%m-%d'), formatDateTime(max(dt), '%m-%d') || ' 23:00') as end_date
FROM grouped
GROUP BY grp
ORDER BY start_date
```

{% line_chart
    data="model_data"
    x="delivery_date"
    y="value"
    series="variable"
    filters=["variable_filter"]
    title="Statistical Model Output"
    x_axis_options={
        label_rotate=-35
    }
    line_options={
        step="end"
    }
%}
    {% reference_area
        data="weekend_holidays"
        x_min="start_date"
        x_max="end_date"
        color="grey"
        area_options={
            opacity=0.5
        }
    /%}
{% /line_chart %}

{% table
    data="model_data"
    filters=["variable_filter"]
    title="Model Data"
    search=true
    page_size=20
%}
    {% dimension value="delivery_date" sort="asc" /%}
    {% pivot value="variable" /%}
    {% measure value="avg(value)" title="Value" /%}
{% /table %}