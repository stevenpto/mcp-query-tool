---
name: Approved SQL MCP Builder
description: Build or modify a FastMCP server that securely discovers and executes approved parameterized SQL stored in S3 through a dynamic query registry.
---

# Approved SQL MCP Builder

## Purpose

Use this skill when building or modifying an MCP server whose job is to execute only pre-approved, parameterized SQL queries stored in Amazon S3.

The architecture must optimize for LLM use while preserving these requirements:

- The LLM must never submit or generate SQL for execution.
- SQL files live in S3 and are treated as approved artifacts.
- Adding a new approved query must not require a source-code change, compilation, server restart, or application deployment.
- Each query has machine-readable metadata describing when to use it, required parameters, allowed output fields, and the S3 SQL object.
- The MCP tool surface remains stable as the number of registered queries grows.
- The server validates query IDs, parameters, output fields, authorization, and SQL artifact identity before execution.
- Parameter values must be bound through the database driver's parameter mechanism; never concatenate user or model values into SQL.

## Preferred MCP Interface

Expose only these stable MCP tools unless the existing application has a strong reason to do otherwise:

### `find_query`

Purpose: discover approved queries by user intent.

Suggested input:

```json
{
  "intent": "CMBS risk score for securities identified by CUSIP",
  "max_results": 5
}
```

Suggested output:

```json
{
  "matches": [
    {
      "query_id": "cmbs_risk_score",
      "name": "CMBS Risk Score",
      "description": "Returns security-level CMBS risk scores by CUSIP.",
      "required_parameters": ["cusips"],
      "default_output_fields": ["cusip", "risk_score"],
      "resource_uri": "query://cmbs_risk_score"
    }
  ]
}
```

Rules:

- `intent` may be arbitrary natural language.
- Search operates only over registry metadata, never over SQL execution.
- Return a small ranked candidate set, normally 3-5 results.
- Prefer deterministic metadata search first: normalized terms, tags, names, aliases, and descriptions.
- Full-text/BM25 or semantic/vector search may be added later without changing the MCP contract.
- Search scores are retrieval hints only. They are not authorization and must never bypass validation.

### `execute_query`

Purpose: execute exactly one approved registry query.

Suggested input:

```json
{
  "query_id": "cmbs_risk_score",
  "parameters": {
    "cusips": ["123456AB1", "123456AB2"]
  },
  "output_fields": ["cusip", "risk_score"]
}
```

Execution must:

1. Resolve `query_id` by exact lookup in the current registry.
2. Reject unknown, disabled, expired, or unauthorized query IDs.
3. Validate parameters against the registered schema.
4. Reject unknown parameters unless a query explicitly permits them.
5. Validate requested output fields against the query's allow-list.
6. Load the approved SQL object identified by the registry.
7. Verify integrity/version information when configured.
8. Bind values using the database driver's parameter API.
9. Execute using a least-privilege database identity.
10. Return only requested/default approved output fields plus safe metadata.

Never accept a SQL string, table name, column expression, WHERE fragment, ORDER BY expression, or other executable SQL fragment from the LLM.

## MCP Resources

Expose query documentation through one dynamic resource template rather than one hard-coded resource per query.

Conceptual FastMCP resource:

```python
@mcp.resource("query://{query_id}")
def get_query_definition(query_id: str):
    return registry.get_public_definition(query_id)
```

`query://cmbs_risk_score` is a custom URI identifying documentation for that registry entry. It is not SQL syntax.

The resource should expose only safe documentation such as:

```json
{
  "query_id": "cmbs_risk_score",
  "name": "CMBS Risk Score",
  "description": "Returns security-level CMBS risk scores.",
  "use_when": [
    "The user asks for CMBS risk score",
    "The user provides CUSIPs and asks for CMBS risk information"
  ],
  "parameters": {
    "cusips": {
      "type": "array",
      "items": {"type": "string"},
      "required": true,
      "description": "One or more security CUSIPs"
    }
  },
  "allowed_output_fields": [
    "cusip",
    "risk_score",
    "rating",
    "spread",
    "duration"
  ],
  "default_output_fields": ["cusip", "risk_score"]
}
```

Do not expose SQL text through the MCP resource unless explicitly required by the application.

## S3 Registry Layout

Prefer one folder/prefix per approved query:

```text
s3://<bucket>/queries/
  cmbs_risk_score/
    query.sql
    manifest.json

  issuer_exposure/
    query.sql
    manifest.json
```

A new query becomes available by adding a valid `query.sql` and `manifest.json` to S3. The application code must not need to change.

Example `manifest.json`:

```json
{
  "schema_version": 1,
  "query_id": "cmbs_risk_score",
  "name": "CMBS Risk Score",
  "description": "Returns security-level CMBS risk scores.",
  "use_when": [
    "User asks for CMBS risk score",
    "User wants CMBS risk information by CUSIP"
  ],
  "tags": ["cmbs", "risk", "risk score", "cusip"],
  "enabled": true,
  "sql_key": "queries/cmbs_risk_score/query.sql",
  "parameters": {
    "cusips": {
      "type": "array",
      "items": {"type": "string"},
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
  "default_output_fields": ["cusip", "risk_score"],
  "version": 1
}
```

If authorization is query-specific, include metadata such as roles/groups or an authorization policy reference. Do not rely on LLM instructions for authorization.

## Registry Service

Implement a registry abstraction between FastMCP and S3.

Suggested responsibilities:

```text
QueryRegistry
  refresh()
  search(intent, max_results)
  get(query_id)
  get_public_definition(query_id)
  validate_parameters(query_id, parameters)
  validate_output_fields(query_id, output_fields)
```

The MCP handlers should not directly enumerate S3 objects or implement validation logic themselves. Keep that logic in the registry/service layer.

### Refresh behavior

The registry must support adding queries without deployment or restart.

Recommended initial implementation:

- Load manifests from S3 into an in-memory cache.
- Refresh on a short TTL, such as 1-5 minutes.
- Use atomic cache replacement so requests never observe a partially refreshed registry.
- Keep the previous known-good registry if refresh fails.
- Log registry additions, removals, version changes, validation failures, and refresh failures.
- Optional later optimization: S3 event notification/SQS invalidation instead of or in addition to TTL refresh.

Do not require process restart for registry changes.

## Query Registration Validation

Treat registration as a security boundary.

Before a manifest becomes active:

- Validate the manifest against a versioned manifest schema.
- Ensure `query_id` is unique and follows a restrictive naming pattern.
- Ensure `sql_key` remains under the approved S3 prefix.
- Ensure the SQL object exists.
- Optionally verify S3 object version ID, ETag, SHA-256, signing metadata, or approval metadata.
- Ensure parameter definitions use supported types.
- Ensure allowed/default output fields are valid.
- Reject malformed entries and keep them out of the active registry.
- Never silently activate a partially valid query.

Prefer a separate approval/publishing process for writing to the approved S3 prefix. The runtime MCP role should normally have read-only S3 permissions.

## Output Field Strategy

Do not create separate SQL files for every combination of output columns.

Prefer:

1. One approved SQL file per business operation.
2. The SQL returns the approved superset of fields.
3. `allowed_output_fields` defines what callers may receive.
4. `default_output_fields` defines the normal response.
5. The MCP service projects the returned rows to requested approved fields.

Example:

```json
{
  "query_id": "cmbs_risk_score",
  "parameters": {"cusips": ["123456AB1"]},
  "output_fields": ["cusip", "risk_score"]
}
```

If result size or expensive expressions make selecting all fields materially inefficient, support a controlled select-list builder only from server-owned mappings. Never accept raw column names or SQL expressions directly from the LLM.

Create separate query files when business logic, joins, calculations, security semantics, or database access patterns are meaningfully different—not merely because a caller wants different columns.

## LLM Interaction Pattern

For a request such as:

> Get CMBS risk scores for these CUSIPs.

The intended flow is:

```text
User request
   ↓
find_query("CMBS risk scores by CUSIP")
   ↓
candidate: cmbs_risk_score
   ↓
(optional) read query://cmbs_risk_score
   ↓
execute_query(
  query_id="cmbs_risk_score",
  parameters={"cusips": [...]},
  output_fields=["cusip", "risk_score"]
)
```

The LLM may use the resource when it needs detailed parameter or output documentation. `find_query` should return enough metadata that straightforward requests can often go directly from discovery to execution.

The LLM must never invent a query ID and assume it exists. Execution is authoritative: only exact IDs present in the active registry are accepted.

## Security Requirements

Apply defense in depth:

- No arbitrary SQL input.
- No arbitrary S3 object key input from the LLM.
- Exact query-ID lookup only.
- Strict manifest schema validation.
- Strict runtime parameter validation.
- Bound database parameters only.
- No interpolation of parameter values into SQL.
- Output-field allow-listing.
- Row/result size limits.
- Query timeout.
- Database statement timeout where supported.
- Least-privilege DB credentials, preferably read-only if the service is read-only.
- Least-privilege S3 role scoped to the approved prefix.
- Encryption in transit and at rest.
- Structured audit logging: caller, query_id, query version, parameter names/types, execution status, duration, row count.
- Avoid logging sensitive parameter values unless explicitly approved.
- Authorization is enforced server-side and is independent of the LLM.
- Do not trust manifest data until validated.
- Consider blocking multiple SQL statements if the database/driver does not already enforce this.
- Consider validating SQL during registration against policy (for example read-only SELECT-only rules) if appropriate for the system.

## Error Contract

Return structured, model-friendly errors.

Examples:

```json
{
  "error": "UNKNOWN_QUERY",
  "message": "The requested query_id is not registered."
}
```

```json
{
  "error": "INVALID_PARAMETERS",
  "message": "Parameter 'cusips' is required and must contain 1-500 strings.",
  "query_id": "cmbs_risk_score"
}
```

```json
{
  "error": "INVALID_OUTPUT_FIELDS",
  "message": "Requested field 'internal_model_blob' is not allowed.",
  "allowed_output_fields": ["cusip", "risk_score", "rating", "spread", "duration"]
}
```

Do not return stack traces, SQL text, credentials, DB connection strings, or sensitive infrastructure details to the model.

## Implementation Guidance for FastMCP

When modifying an existing repository:

1. Inspect the installed FastMCP version and existing server structure before choosing decorators/APIs.
2. Reuse the project's configuration, logging, dependency-injection, database, AWS, and testing patterns.
3. Keep tool signatures stable.
4. Put S3 access behind a repository/client abstraction.
5. Put search, manifest validation, and registry refresh behind `QueryRegistry`.
6. Put SQL execution behind a dedicated executor/service.
7. Keep MCP handlers thin.
8. Add tests before or alongside implementation.

Suggested structure:

```text
src/
  mcp_server.py
  registry/
    models.py
    query_registry.py
    manifest_validator.py
    search.py
  storage/
    s3_query_store.py
  execution/
    query_executor.py
    parameter_validation.py
    result_projection.py
  config.py
tests/
  test_registry.py
  test_search.py
  test_execute_query.py
  test_manifest_validation.py
```

Adapt this structure to the existing codebase rather than restructuring a mature project unnecessarily.

## Search Behavior

Start simple and deterministic.

A reasonable initial ranking can combine:

- exact query ID/name match
- tag match
- alias match
- token overlap with `description`
- token overlap with `use_when`

Normalize case and punctuation. Use domain aliases where useful.

Example concepts:

```text
CMBS → commercial mortgage-backed securities
risk number → risk score
security identifier → CUSIP
```

Add BM25 or embeddings only when registry size or language variability makes them useful. Search technology can change internally without changing the MCP tool contract.

## Tests / Acceptance Criteria

The implementation is complete only when tests demonstrate:

1. A registered S3 query can be discovered through `find_query`.
2. A new S3 manifest/query becomes discoverable after registry refresh without code change or process restart.
3. An unknown `query_id` cannot execute.
4. An LLM-supplied S3 key cannot execute.
5. Missing/invalid parameters are rejected before database execution.
6. Extra parameters are rejected unless explicitly allowed.
7. Requested output fields outside the allow-list are rejected.
8. Default output fields are used when none are requested.
9. Database parameters are bound rather than interpolated.
10. A malformed manifest never enters the active registry.
11. Registry refresh failure leaves the last known-good registry operational.
12. Disabled queries cannot execute.
13. Query execution respects timeout/result-size controls.
14. Authorization is enforced server-side.
15. SQL text is not exposed through normal MCP responses/resources.
16. Logs contain query ID/version and execution metadata without leaking disallowed sensitive values.

## Example End-to-End Registration

To add a query named `issuer_exposure`, an authorized publisher uploads:

```text
queries/issuer_exposure/query.sql
queries/issuer_exposure/manifest.json
```

No FastMCP source code is changed.

After the next registry refresh:

```text
find_query("issuer exposure")
```

can return `issuer_exposure`, and:

```text
query://issuer_exposure
```

can expose its public definition.

`execute_query` can then execute it only if the exact registered ID, parameters, authorization, and output fields pass validation.

## What Not to Build

Do not:

- add one FastMCP tool per SQL file
- add one hard-coded resource function per SQL file
- put every query into a giant static `oneOf` tool schema
- accept raw SQL from the model
- accept arbitrary table/column/WHERE/JOIN/ORDER BY fragments
- trust an LLM-selected S3 path
- create one SQL file per possible output-column subset
- require a server deployment simply to register a new approved query

## Build Objective

When asked to implement this design in a codebase, produce a maintainable FastMCP service in which the MCP surface is stable and the approved query catalog is data-driven from S3. Favor simple, testable components and secure defaults. Preserve existing application conventions unless they conflict with the security requirements above.
