# Approved SQL Query MCP Design Addendum

This addendum defines the required contract for **query inputs, output fields, and approved filters** in the generic MCP query tool and in each query's registry manifest.

## Generic MCP Query Contract

The generic execution tool should support four controlled dimensions:

```text
query_id       -> which approved query to execute
parameters     -> required/optional business inputs
filters        -> approved WHERE-style predicates
output_fields  -> approved result columns to return
```

Suggested logical signature:

```python
execute_query(
    query_id: str,
    parameters: dict,
    filters: list[dict] | None = None,
    output_fields: list[str] | None = None
)
```

Example:

```json
{
  "query_id": "cmbs_risk_score",
  "parameters": {
    "cusips": ["123456AB1", "987654CD2"]
  },
  "filters": [
    {
      "field": "rating",
      "operator": "in",
      "value": ["BBB", "BB"]
    },
    {
      "field": "spread",
      "operator": "gte",
      "value": 150
    }
  ],
  "output_fields": ["cusip", "risk_score", "rating", "spread"]
}
```

The LLM must never provide raw SQL, raw WHERE clauses, column expressions, table names, or other executable SQL fragments.

## 1. Query Parameters / Inputs

`parameters` represent the core inputs required by the approved query, such as CUSIPs, account ID, as-of date, portfolio ID, issuer ID, or a date range.

Example manifest:

```json
{
  "parameters": {
    "cusips": {
      "type": "array",
      "items": {"type": "string"},
      "required": true,
      "minItems": 1,
      "maxItems": 500,
      "description": "One or more CUSIPs to evaluate."
    },
    "as_of_date": {
      "type": "string",
      "format": "date",
      "required": false,
      "description": "Optional as-of date in YYYY-MM-DD format."
    }
  }
}
```

Each parameter definition should support, where applicable: `type`, `required`, `description`, `items`, `enum`, `format`, `minItems`, `maxItems`, `minimum`, `maximum`, `minLength`, `maxLength`, and optional defaults.

The server must reject missing required parameters, unknown parameters, invalid types, invalid formats, and out-of-range values before execution. Parameter values must always be bound through the database driver.

## 2. Output Field Specification

A query may return a superset of columns while users request different subsets. Do not create separate SQL files only because callers want different output columns.

Example request:

```json
{
  "output_fields": ["cusip", "rating", "spread"]
}
```

Manifest requirement:

```json
{
  "allowed_output_fields": [
    "cusip",
    "risk_score",
    "rating",
    "spread",
    "duration",
    "price",
    "market_value",
    "sector",
    "issuer",
    "currency"
  ],
  "default_output_fields": ["cusip", "risk_score"]
}
```

The server must use defaults when `output_fields` is omitted and reject any unapproved field. Prefer server-side projection of an approved result superset. If dynamic SELECT projection is needed for performance, build it only from server-owned field mappings.

## 3. Approved Filters / WHERE Conditions

Users may filter the same approved query in different ways. Do not create a separate SQL file for each filter combination and do not allow raw WHERE clauses.

Example request:

```json
{
  "filters": [
    {
      "field": "rating",
      "operator": "eq",
      "value": "BBB"
    },
    {
      "field": "spread",
      "operator": "gte",
      "value": 150
    }
  ]
}
```

A flat filter list should default to logical `AND`.

Manifest requirement:

```json
{
  "filters": {
    "rating": {
      "type": "string",
      "operators": ["eq", "in"],
      "description": "Filter by security rating."
    },
    "spread": {
      "type": "number",
      "operators": ["gt", "gte", "lt", "lte"],
      "description": "Filter by spread."
    },
    "sector": {
      "type": "string",
      "operators": ["eq", "in"],
      "description": "Filter by sector."
    }
  }
}
```

Recommended operator vocabulary:

```text
eq        =
neq       <>
gt        >
gte       >=
lt        <
lte       <=
in        IN
not_in    NOT IN
is_null   IS NULL
not_null  IS NOT NULL
```

Only support operators required by real use cases.

The server must validate the filter field, operator, value type, list size, and maximum filter count. Logical field names must map to trusted database columns, and all values must be bound parameters.

Example internal mapping:

```python
FILTER_FIELDS = {
    "rating": {
        "column": "rating",
        "operators": {
            "eq": "=",
            "in": "IN"
        }
    },
    "spread": {
        "column": "oas_spread",
        "operators": {
            "gt": ">",
            "gte": ">=",
            "lt": "<",
            "lte": "<="
        }
    }
}
```

The LLM sends:

```json
{
  "field": "spread",
  "operator": "gte",
  "value": 150
}
```

The server safely converts that to a trusted predicate such as:

```sql
AND oas_spread >= :filter_1
```

with `filter_1 = 150`.

## 4. Recommended Complete Query Manifest

```json
{
  "schema_version": 1,
  "query_id": "cmbs_risk_score",
  "name": "CMBS Risk Score",
  "description": "Returns security-level CMBS risk data.",
  "use_when": [
    "User asks for CMBS risk information",
    "User asks for risk metrics for one or more CUSIPs"
  ],
  "tags": ["cmbs", "risk", "cusip"],
  "enabled": true,
  "sql_key": "queries/cmbs_risk_score/query.sql",

  "parameters": {
    "cusips": {
      "type": "array",
      "items": {"type": "string"},
      "required": true,
      "minItems": 1,
      "maxItems": 500,
      "description": "CUSIPs to evaluate."
    }
  },

  "filters": {
    "rating": {
      "type": "string",
      "operators": ["eq", "in"]
    },
    "spread": {
      "type": "number",
      "operators": ["gt", "gte", "lt", "lte"]
    },
    "sector": {
      "type": "string",
      "operators": ["eq", "in"]
    }
  },

  "allowed_output_fields": [
    "cusip",
    "risk_score",
    "rating",
    "spread",
    "duration",
    "sector"
  ],

  "default_output_fields": ["cusip", "risk_score"],
  "version": 1
}
```

## 5. LLM-Facing Query Resource

The dynamic resource `query://cmbs_risk_score` should expose enough information for the model to call the generic tool correctly:

```json
{
  "query_id": "cmbs_risk_score",
  "description": "Returns security-level CMBS risk data.",
  "parameters": {
    "cusips": {
      "type": "array",
      "required": true,
      "description": "CUSIPs to evaluate."
    }
  },
  "filters": {
    "rating": {
      "type": "string",
      "operators": ["eq", "in"]
    },
    "spread": {
      "type": "number",
      "operators": ["gt", "gte", "lt", "lte"]
    }
  },
  "allowed_output_fields": [
    "cusip",
    "risk_score",
    "rating",
    "spread",
    "duration",
    "sector"
  ],
  "default_output_fields": ["cusip", "risk_score"]
}
```

The SQL text should remain hidden.

## 6. Example LLM Interaction

User request:

```text
Get rating, spread, and risk score for these CMBS CUSIPs,
only where rating is BBB or BB and spread is at least 150.
```

LLM call:

```json
{
  "query_id": "cmbs_risk_score",
  "parameters": {
    "cusips": ["123456AB1", "987654CD2"]
  },
  "filters": [
    {
      "field": "rating",
      "operator": "in",
      "value": ["BBB", "BB"]
    },
    {
      "field": "spread",
      "operator": "gte",
      "value": 150
    }
  ],
  "output_fields": [
    "cusip",
    "rating",
    "spread",
    "risk_score"
  ]
}
```

The MCP server then validates the query ID, parameters, filters, and output fields, loads the approved SQL from S3, adds only trusted filter predicates, binds values, executes the query, projects approved output fields, and returns results.

## 7. Security Boundary

The model controls only:

```text
approved query ID
approved parameter values
approved filter choices
approved output-field choices
```

The server controls:

```text
SQL files
S3 paths
table names
database columns
operator-to-SQL mappings
authorization
parameter binding
filter construction
output projection
execution limits
```

## 8. Important Design Rule

Do not treat `parameters`, `filters`, and `output_fields` as interchangeable.

```text
parameters
    inputs required or optionally accepted by the business query

filters
    optional predicates applied to the approved result space

output_fields
    approved columns the caller wants returned
```

This separation should be reflected consistently in the MCP tool schema, query manifest, dynamic query resource, validation code, audit logging, and tests.
