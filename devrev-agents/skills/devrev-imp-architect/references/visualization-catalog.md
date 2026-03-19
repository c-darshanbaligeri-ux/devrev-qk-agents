# Visualization Catalog

Complete JSON patterns for every DevRev visualization type. Copy and modify these patterns — do NOT improvise.

---

## 1. Metric (single value)

Shows one number. Cannot display ID types (use display_name/display_id as text instead).

### Normal Metric
```json
"visualization": {
    "metric": {
        "y": [{
            "label": "Total Tickets",
            "reference_name": "source_name.measure_name"
        }]
    },
    "type": "metric"
}
```

### PVP Metric (period-vs-period comparison)
Compares selected time frame to the same period before it. Only works when a date filter is applied.
```json
"visualization": {
    "metric": {
        "pvp": {
            "enabled": true,
            "invert_correlation_color": true
        },
        "y": [{
            "label": "Ticket Count",
            "reference_name": "source_name.measure_name"
        }]
    },
    "type": "metric"
}
```

---

## 2. Column (vertical bar chart)

### Normal Column
```json
"visualization": {
    "column": {
        "is_stacked": true,
        "x": [{
            "label": "Owner",
            "reference_name": "source_name.dimension_name"
        }],
        "y": [{
            "label": "Count",
            "reference_name": "source_name.measure_name"
        }]
    },
    "type": "column"
}
```

### Stacked Column (+1 dimension for color split)
Add a second item to the x array for the categorical split:
```json
"visualization": {
    "column": {
        "is_stacked": true,
        "x": [
            {
                "label": "Owner",
                "reference_name": "source_name.dimension_name"
            },
            {
                "label": "Stage",
                "reference_name": "source_name.dimension_2_name"
            }
        ],
        "y": [{
            "label": "Amount",
            "reference_name": "source_name.measure_name"
        }]
    },
    "type": "column"
}
```

### Non-Stacked Column (+1 dimension)
Same as stacked but `"is_stacked": false` — shows side-by-side bars.

### Column with Colors
```json
"x": [
    {
        "label": "Date",
        "reference_name": "source_name.record_date"
    },
    {
        "color": {
            "key_lookup": [
                { "key": "breached", "value": "chart-category-6-base" },
                { "key": "warning", "value": "chart-category-2-base" },
                { "key": "paused", "value": "chart-category-7-base" }
            ],
            "type": "key_lookup"
        },
        "label": "SLA Stage",
        "reference_name": "source_name.sla_stage"
    }
]
```

---

## 3. Bar (horizontal bar chart)

Same structure as column. Use for long labels that read better horizontally.

### Normal Bar
```json
"visualization": {
    "bar": {
        "is_stacked": false,
        "x": [{
            "label": "Service",
            "reference_name": "source_name.dimension_name"
        }],
        "y": [{
            "label": "Total",
            "reference_name": "source_name.measure_name"
        }]
    },
    "type": "bar"
}
```

### Stacked Bar (+1 dimension)
```json
"visualization": {
    "bar": {
        "is_stacked": true,
        "x": [
            {
                "label": "Stage",
                "reference_name": "source_name.dim1"
            },
            {
                "label": "Forecast Category",
                "reference_name": "source_name.dim2"
            }
        ],
        "y": [{
            "label": "Amount",
            "reference_name": "source_name.measure"
        }]
    },
    "type": "bar"
}
```

---

## 4. Line

For trends over time. ALWAYS add `order_by` ascending on the time dimension.

### Normal Line
```json
"visualization": {
    "line": {
        "is_stacked": false,
        "x": [{
            "label": "Created Week",
            "reference_name": "source_name.date_dimension"
        }],
        "y": [{
            "label": "Issues",
            "reference_name": "source_name.measure_name"
        }]
    },
    "type": "line"
}
```

### Multiple Lines (+1 dimension for categorical split)
Second x item splits into one line per category:
```json
"visualization": {
    "line": {
        "is_stacked": false,
        "x": [
            {
                "label": "Date",
                "reference_name": "source_name.date_dim"
            },
            {
                "label": "Priority",
                "reference_name": "source_name.priority"
            }
        ],
        "y": [{
            "label": "Count",
            "reference_name": "source_name.measure"
        }]
    },
    "type": "line"
}
```

### Multiple Lines (+1 measure for multi-series)
Multiple y items show multiple lines on the same chart:
```json
"y": [
    {
        "label": "Created",
        "reference_name": "source_name.created_count"
    },
    {
        "label": "Resolved",
        "reference_name": "source_name.resolved_count"
    }
]
```

### Line/Bar Design Attributes
Add to individual x/y items or the chart object:

| Attribute | Where | Effect |
|-----------|-------|--------|
| `"is_spline": true` | In x or y item | Curved line |
| `"is_hidden": true` | In y item | Hidden by default (toggle on) |
| `"is_area_filled": true` | In y item | Fill area under line |
| `"is_stacked": true` | In line/bar object | Stack items |
| `"show_data_labels": true` | In line/bar object | Show numbers on points/bars |

---

## 5. Table

### Normal Table
```json
"visualization": {
    "table": {
        "columns": [
            {
                "label": "Issue",
                "reference_name": "source_name.id",
                "order": 1
            },
            {
                "label": "Title",
                "reference_name": "source_name.title",
                "order": 2
            },
            {
                "label": "Owner",
                "reference_name": "source_name.owned_by_id",
                "order": 3
            },
            {
                "label": "Count",
                "reference_name": "source_name.count"
            }
        ]
    },
    "type": "table"
}
```

### Table with GroupBy
Add `"is_groupable": true` in the DIMENSION's ui (not in visualization):
```json
{
  "devrev_schema": {
    "field_type": "id",
    "name": "owned_by_id",
    "ui": {
      "display_name": "Owner",
      "is_groupable": true
    }
  }
}
```

### Table Column Attributes

| Attribute | Type | Effect |
|-----------|------|--------|
| `"is_hidden": true` | bool | Hidden by default |
| `"max_width": 200` | int | Max column pixel width |
| `"min_width": 100` | int | Min column pixel width |
| `"order": 1` | int | Column order |

---

## 6. Donut

```json
"visualization": {
    "donut": {
        "x": [{
            "label": "Category",
            "reference_name": "source_name.dimension_name"
        }],
        "y": {
            "label": "Count",
            "reference_name": "source_name.measure_name"
        }
    },
    "type": "donut"
}
```

NOTE: `y` is a single object (not an array) for donut/pie.

---

## 7. Pie

Same structure as donut:
```json
"visualization": {
    "pie": {
        "x": [{
            "label": "Category",
            "reference_name": "source_name.dimension_name"
        }],
        "y": {
            "label": "Count",
            "reference_name": "source_name.measure_name"
        }
    },
    "type": "pie"
}
```

---

## 8. Packed Bubble

```json
"visualization": {
    "packed_bubble": {
        "x": [{
            "label": "Owner",
            "reference_name": "source_name.dimension_name"
        }],
        "y": [{
            "label": "Series 1",
            "reference_name": "source_name.measure_name"
        }]
    },
    "type": "packed_bubble"
}
```

---

## 9. Heatmap

Always: 2 dimensions (x, y) + 1 measure (z).

```json
"visualization": {
    "heatmap": {
        "x": {
            "label": "Created Date",
            "reference_name": "source_name.created_date"
        },
        "y": {
            "label": "Owner",
            "reference_name": "source_name.owned_by_id"
        },
        "z": {
            "label": "Count",
            "reference_name": "source_name.measure_name"
        }
    },
    "type": "heatmap"
}
```

NOTE: x, y, z are single objects (not arrays) for heatmap.

### Heatmap with Static Color
```json
"z": {
    "color": {
        "static": "chart-category-3-base",
        "type": "static"
    },
    "label": "Count",
    "reference_name": "source_name.measure"
}
```

---

## Colors

### Key Lookup (per-value colors)
Assign a specific color to each value of a dimension:
```json
"color": {
    "key_lookup": [
        { "key": "breached", "value": "chart-category-6-base" },
        { "key": "warning", "value": "chart-category-2-base" },
        { "key": "paused", "value": "chart-category-7-base" }
    ],
    "type": "key_lookup"
}
```

### Static (one color for the measure)
```json
"color": {
    "static": "chart-category-1-base",
    "type": "static"
}
```

### Color Catalogue
```
chart-category-1-base       chart-category-1-lighter
chart-category-2-base       chart-category-2-lighter
chart-category-3-base
chart-category-4-base
chart-category-5-base
chart-category-6-base
chart-category-7-base
chart-category-8-base
chart-category-9-base
chart-neutrals-base         chart-neutrals-lighter
chart-alert-base            chart-alert-lighter
chart-success-base          chart-success-lighter
chart-warn-base             chart-warn-lighter
```

---

## Drill-Through

Link a chart element to another dashboard for deep-dive:
```json
"drill_throughs": [{
    "dashboard": "don:data:dvrv-us-1:devo/0:dashboard/1U5KEFvaCV",
    "label": "drill"
}]
```

Add inside any x or y item in the visualization.

---

## Dashboard Layout

Dashboard URL: `https://app.devrev.ai/<slug>/dashboard?dashboardId=<ID>`

Grid: 12 columns wide. Position each widget with x, y, width, height.

```json
{
  "widgets": [
    { "widget_id": "<W1_ID>", "reference_id": "metrics_row" },
    { "widget_id": "<W2_ID>", "reference_id": "trend_chart" },
    { "widget_id": "<W3_ID>", "reference_id": "detail_table" }
  ],
  "layout": [
    { "reference_id": "metrics_row", "position": { "x": 0, "y": 0, "width": 12, "height": 2 } },
    { "reference_id": "trend_chart", "position": { "x": 0, "y": 2, "width": 6, "height": 4 } },
    { "reference_id": "detail_table", "position": { "x": 6, "y": 2, "width": 6, "height": 6 } }
  ]
}
```

Common layout patterns:
- Metric cards row: width=3 each, height=2, across the top (y=0)
- Charts: width=6 (half), height=4
- Tables: width=12 (full), height=6
- Tabs: use `tabs` array to split widgets across tabs
