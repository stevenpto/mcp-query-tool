# Approved SQL Query MCP Server

A FastMCP server for safely discovering and executing **pre-approved, parameterized SQL queries stored in Amazon S3**.

The server is designed for LLM/agent use while keeping SQL execution tightly controlled:

- The LLM never writes or submits SQL.
- Only registered SQL files in S3 can be executed.
- Query parameters are validated and bound through the database driver.
- New queries can be added without changing or redeploying application code.
- Query documentation and metadata are dynamically discovered from the S3 registry.

---

## Overview

This MCP server exposes a small, stable interface to an evolving catalog of approved SQL queries.

The LLM interacts with the server through:

- `find_query` — finds approved queries based on natural-language intent.
- `execute_query` — executes an exact registered query with validated parameters.
- `query://{query_id}` — MCP resource exposing documentation for a registered query.

The query catalog itself is stored in S3.

```text
                     LLM / Agent
                         |
               +---------+---------+
               |                   |
               v                   v
          find_query          execute_query
               |                   |
               +---------+---------+
                         |
                         v
                  Query Registry
                         |
                         v
                        S3
               +-------------------+
               | manifest.json     |
               | query.sql         |
               +-------------------+
                         |
                         v
                      Database
```

---

## Design Goals

The server is built around the following principles:

1. **No arbitrary SQL**
   - The LLM cannot submit raw SQL.
   - SQL is loaded only from approved S3 registry entries.

2. **Stable MCP contract**
   - MCP tools remain unchanged as the query catalog grows.
   - Adding a new query does not require adding a new MCP tool.

3. **Dynamic registration**
   - New SQL files and manifests can be added to S3.
   - The registry refreshes automatically.
   - No application code release is required.

4. **Strict validation**
   - Query IDs must exist in the active registry.
   - Parameters are validated against the query manifest.
   - Output fields are restricted to an allow-list.

5. **LLM-friendly discovery**
   - The LLM can search queries using natural-language intent.
   - Query metadata describes when each query should be used.

---

## MCP Interface

### `find_query`

Searches the approved query registry using natural-language intent.

Example request:

```json
{
  "intent": "CMBS risk score for securities identified by CUSIP",
  "max_results": 5
}
```

Example response:

```json
{
  "matches": [
    {
      "query_id": "cmbs_risk_score",
      "name": "CMBS Risk Score",
      "description": "Returns CMBS risk scores by CUSIP.",
      "required_parameters": [
        "cusips"
      ],
      "default_output_fields": [
        "cusip",
        "risk_score"
      ],
      "resource_uri": "query://cmbs_risk_score"
    }
  ]
}
```

`find_query` searches only registry metadata. It does not execute SQL.

---

### `execute_query`

Executes exactly one approved query.

Example request:

```json
{
  "query_id": "cmbs_risk_score",
  "parameters": {
    "cusips": [
      "123456AB1",
      "123456AB2"
    ]
  },
  "output_fields": [
    "cusip",
    "risk_score"
  ]
}
```

Before execution, the server validates:

- the query ID exists
- the query is enabled
- the caller is authorized
- required parameters are present
- parameter types and limits are valid
- requested output fields are allowed

The server then loads the approved SQL file from S3, binds the parameters through the database driver, executes the query, and returns only the requested approved fields.

---

## MCP Query Resources

Each registered query is exposed through a dynamic MCP resource.

Example:

```text
query://cmbs_risk_score
```

The resource returns documentation similar to:

```json
{
  "query_id": "cmbs_risk_score",
  "name": "CMBS Risk Score",
  "description": "Returns CMBS risk scores by CUSIP.",
  "use_when": [
    "User asks for CMBS risk score",
    "User wants CMBS risk information for one or more CUSIPs"
  ],
  "parameters": {
    "cusips": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "required": true
    }
  },
  "allowed_output_fields": [
    "cusip",
    "risk_score",
    "rating",
    "spread",
    "duration"
  ],
  "default_output_fields": [
    "cusip",
    "risk_score"
  ]
}
```

The SQL text itself is not exposed to the LLM.

---

## S3 Query Registry

Each approved query is stored under its own S3 prefix.

Example:

```text
s3://<bucket>/queries/
  cmbs_risk_score/
    query.sql
    manifest.json

  issuer_exposure/
    query.sql
    manifest.json
```

### Example `manifest.json`

```json
{
  "schema_version": 1,
  "query_id": "cmbs_risk_score",
  "name": "CMBS Risk Score",
  "description": "Returns security-level CMBS risk scores.",
  "use_when": [
    "User asks for CMBS risk score",
    "User asks for CMBS risk information by CUSIP"
  ],
  "tags": [
    "cmbs",
    "risk",
    "risk score",
    "cusip"
  ],
  "enabled": true,
  "sql_key": "queries/cmbs_risk_score/query.sql",
  "parameters": {
    "cusips": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "required": true,
      "minItems": 1,
      "maxItems": 500,
      "description": "CUSIPs to evaluate"
    }
  },
  "allowed_output_fields": [
    "cusip",
    "risk_score",
    "rating",
    "spread",
    "duration"
  ],
  "default_output_fields": [
    "cusip",
    "risk_score"
  ],
  "version": 1
}
```

---

## Adding a New Query

Adding a new query does **not** require changing the MCP server code.

Create:

```text
queries/<query_id>/
  query.sql
  manifest.json
```

For example:

```text
queries/issuer_exposure/
  query.sql
  manifest.json
```

Upload both files to the approved S3 query prefix.

After the registry refreshes, the new query becomes available through:

```text
find_query(...)
```

and:

```text
query://issuer_exposure
```

It can then be executed through:

```text
execute_query(
    query_id="issuer_exposure",
    ...
)
```

No FastMCP tool, Python function, build, or application deployment is required.

---

## Example Query

### `query.sql`

```sql
SELECT
    cusip,
    risk_score,
    rating,
    spread,
    duration
FROM cmbs_security_risk
WHERE cusip IN (:cusips)
```

Parameter syntax will depend on the configured database driver.

The application must always use driver-supported parameter binding rather than string interpolation.

---

## Registry Refresh

The server keeps the active registry in memory and periodically refreshes it from S3.

Recommended behavior:

- load registry at startup
- refresh on a configurable TTL
- atomically replace the active cache
- keep the previous known-good registry if refresh fails
- log new, changed, removed, and invalid registry entries

Typical refresh interval:

```text
1–5 minutes
```

This can later be replaced or supplemented with S3 event-driven invalidation if needed.

---

## Query Search

`find_query` searches registry metadata, not SQL.

Searchable fields may include:

- `query_id`
- `name`
- `description`
- `tags`
- aliases
- `use_when`

The initial implementation should favor deterministic matching such as:

1. exact query ID/name match
2. tag or alias match
3. token overlap
4. description and `use_when` matching

Full-text search, BM25, or embeddings can be added later without changing the MCP tool interface.

---

## Output Field Handling

Separate SQL files are not required for every combination of requested columns.

The preferred pattern is:

```text
approved query
    |
    v
approved superset of output fields
    |
    v
server-side projection
    |
    v
requested approved fields
```

For example:

```json
{
  "query_id": "cmbs_risk_score",
  "parameters": {
    "cusips": [
      "123456AB1"
    ]
  },
  "output_fields": [
    "cusip",
    "risk_score"
  ]
}
```

If `output_fields` is omitted, `default_output_fields` from the manifest is used.

---

## Security Model

This server treats query registration and execution as security boundaries.

### SQL Safety

The LLM cannot provide:

- raw SQL
- table names
- column expressions
- `WHERE` clauses
- `JOIN` clauses
- `ORDER BY` clauses
- arbitrary S3 object paths

Only SQL referenced by a valid registry entry can execute.

### Parameter Safety

All runtime values are:

1. validated against the query manifest
2. passed through the database driver's bind-parameter mechanism

Parameter values must never be concatenated into SQL strings.

### S3 Safety

Recommended controls:

- read-only runtime IAM role
- approved query prefix only
- separate publisher role
- S3 encryption
- S3 versioning where appropriate
- block public access
- optional object hash/version verification

### Database Safety

Recommended controls:

- least-privilege database identity
- read-only access where possible
- statement/query timeout
- maximum result rows
- maximum response size
- connection pooling
- controlled retries

---

## Authorization

Authorization must be enforced by the MCP server.

Query manifests may optionally include authorization metadata such as:

```json
{
  "allowed_roles": [
    "structured_products",
    "risk"
  ]
}
```

or:

```json
{
  "authorization_policy": "cmbs-risk-read"
}
```

The LLM must never be responsible for deciding whether a caller is authorized.

---

## Auditing

Recommended audit fields:

- caller identity
- query ID
- query version
- execution timestamp
- execution duration
- success/failure
- row count
- parameter names/types
- registry version

Avoid logging sensitive parameter values unless explicitly required and approved.

---

## Error Responses

Errors should be structured and safe for LLM consumption.

### Unknown query

```json
{
  "error": "UNKNOWN_QUERY",
  "message": "The requested query_id is not registered."
}
```

### Invalid parameters

```json
{
  "error": "INVALID_PARAMETERS",
  "message": "Parameter 'cusips' is required and must contain 1-500 strings.",
  "query_id": "cmbs_risk_score"
}
```

### Invalid output field

```json
{
  "error": "INVALID_OUTPUT_FIELDS",
  "message": "One or more requested output fields are not allowed."
}
```

Do not return SQL text, stack traces, credentials, connection strings, or sensitive infrastructure details.

---

## Suggested Project Structure

Adapt this layout to the existing project conventions.

```text
src/
  mcp_server.py

  query_tool/
    __init__.py
    models.py
    registry.py
    search.py
    manifest_validator.py
    s3_store.py
    executor.py
    result_projection.py
    mcp_handlers.py

tests/
  query_tool/
    test_registry.py
    test_search.py
    test_manifest_validation.py
    test_execute_query.py
    test_resource.py
```

---

## Configuration

Exact environment variable names may vary by deployment.

Typical configuration:

```text
QUERY_S3_BUCKET=<approved-query-bucket>
QUERY_S3_PREFIX=queries/
QUERY_REGISTRY_REFRESH_SECONDS=300

DB_HOST=...
DB_PORT=...
DB_NAME=...
DB_USER=...
DB_SECRET_ARN=...

QUERY_MAX_RESULTS=5
QUERY_MAX_ROWS=10000
QUERY_TIMEOUT_SECONDS=30
```

Prefer the repository's existing configuration and secrets-management patterns.

---

## Local Development

Install dependencies using the project's existing package manager.

Example:

```bash
pip install -r requirements.txt
```

or:

```bash
poetry install
```

or:

```bash
uv sync
```

Configure AWS credentials using the project's normal development workflow.

For example:

```bash
aws sts get-caller-identity
```

Then start the FastMCP server using the project's configured entry point.

Example only:

```bash
python -m src.mcp_server
```

Refer to the actual project configuration for the authoritative startup command.

---

## Testing

Run the repository's normal test suite.

Example:

```bash
pytest
```

Important test cases include:

- valid query registration
- dynamic registry refresh
- unknown query rejection
- disabled query rejection
- malformed manifest rejection
- invalid parameter rejection
- unapproved output-field rejection
- SQL parameter binding
- authorization enforcement
- S3 path restrictions
- query timeouts
- result-size limits
- last-known-good registry behavior after refresh failure

---

## Example LLM Flow

User request:

```text
Get the CMBS risk score for these CUSIPs.
```

The LLM first discovers the appropriate approved query:

```text
find_query(
  intent="CMBS risk score by CUSIP"
)
```

Result:

```text
cmbs_risk_score
```

If additional documentation is needed, the LLM can read:

```text
query://cmbs_risk_score
```

The LLM then executes:

```json
{
  "query_id": "cmbs_risk_score",
  "parameters": {
    "cusips": [
      "123456AB1",
      "123456AB2"
    ]
  },
  "output_fields": [
    "cusip",
    "risk_score"
  ]
}
```

At no point does the LLM generate SQL.

---

## Key Architectural Rule

The MCP server code defines **how queries are discovered, validated, authorized, and executed**.

S3 defines **which queries currently exist**.

```text
Code changes:
    change platform behavior

S3 registry changes:
    add / modify approved queries
```

This separation allows the query catalog to evolve without continuously releasing new MCP server code.
