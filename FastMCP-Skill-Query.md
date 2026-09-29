---
name: Integrate Approved SQL Query Tool
description: Add a dynamic S3-backed approved SQL query capability into an existing FastMCP server while preserving the existing architecture, tools, configuration, and deployment model.
---

# Integrate Approved SQL Query Tool

## Goal

Use this skill when modifying an existing FastMCP server to add a secure, dynamic query capability for approved parameterized SQL stored in Amazon S3.

Do not redesign the whole MCP server. Integrate this capability into the existing codebase with the smallest maintainable change set.

The new capability must support:

- approved SQL files stored in S3
- a dynamic S3-backed query registry
- no arbitrary SQL from the LLM
- no code change or redeployment when a new approved query is registered
- discovery of available queries by natural-language intent
- query documentation through dynamic MCP resources
- exact query ID validation before execution
- strict parameter validation
- allowed/default output-field validation
- parameter binding through the database driver
- server-side authorization and audit logging

## First Step: Inspect the Existing Repository

Before changing code, inspect the repository and identify:

1. FastMCP version and dependency versions.
2. The file where `FastMCP` is instantiated.
3. Existing `@mcp.tool()` and `@mcp.resource()` patterns.
4. Existing configuration management.
5. Existing AWS/S3 client abstractions.
6. Existing database connection/execution abstractions.
7. Existing logging, metrics, auth, and dependency-injection patterns.
8. Existing test structure.
9. Existing async/sync conventions.
10. Existing error-handling conventions.

Do not assume a specific FastMCP API until the installed version is confirmed.

Do not replace working infrastructure merely because a different pattern is shown in this skill.

## Integration Principle

Add the query capability as a module inside the existing server.

Prefer this logical separation:

```text
Existing MCP Server
    |
    +-- existing tools/resources
    |
    +-- query capability
         |
         +-- find_query
         +-- execute_query
         +-- query://{query_id}
         |
         +-- QueryRegistry
         +-- S3QueryStore
         +-- QueryExecutor
```

If the existing project already has service/repository layers, fit these components into those layers rather than inventing a parallel architecture.

## Stable MCP Surface

Add two tools:

### `find_query`

Purpose: allow the LLM to discover approved queries from natural-language intent.

Suggested logical signature:

```python
find_query(
    intent: str,
    max_results: int = 5
)
```

Typical result:

```json
{
  "matches": [
    {
      "query_id": "cmbs_risk_score",
      "name": "CMBS Risk Score",
      "description": "Returns CMBS risk scores by CUSIP.",
      "required_parameters": ["cusips"],
      "default_output_fields": ["cusip", "risk_score"],
      "resource_uri": "query://cmbs_risk_score"
    }
  ]
}
```

Rules:

- `intent` may contain arbitrary natural language.
- It must only search registry metadata.
- Never translate `intent` directly into SQL.
- Limit returned candidates.
- Do not automatically execute the highest-scoring match inside `find_query`.

### `execute_query`

Purpose: execute exactly one approved query.

Suggested logical signature:

```python
execute_query(
    query_id: str,
    parameters: dict,
    output_fields: list[str] | None = None
)
```

Execution must:

1. Perform exact lookup of `query_id`.
2. Reject unknown/disabled/unauthorized IDs.
3. Validate parameters.
4. Validate output fields.
5. Resolve the SQL file only from trusted registry metadata.
6. Load SQL from the approved S3 location.
7. Bind parameters through the DB API.
8. Execute with existing project DB infrastructure.
9. Project the response to allowed requested/default fields.
10. Return safe structured results.

Never accept:

- raw SQL
- raw S3 object keys
- raw table names
- raw column expressions
- WHERE fragments
- JOIN fragments
- ORDER BY fragments
- arbitrary executable database expressions

## Dynamic Query Resource

Add one dynamic resource template rather than one hard-coded resource per query.

Conceptual form:

```python
@mcp.resource("query://{query_id}")
def get_query_definition(query_id: str):
    return registry.get_public_definition(query_id)
```

Adapt syntax to the FastMCP version found in the repo.

The resource should expose safe metadata only:

```json
{
  "query_id": "cmbs_risk_score",
  "name": "CMBS Risk Score",
  "description": "Returns CMBS risk scores by CUSIP.",
  "use_when": [
    "User asks for CMBS risk score",
    "User asks for CMBS risk information for one or more CUSIPs"
  ],
  "parameters": {
    "cusips": {
      "type": "array",
      "items": {"type": "string"},
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

Do not expose SQL text unless the existing product explicitly requires it.

## S3 Registry Design

Use S3 as the source of truth for query registration.

Preferred layout:

```text
s3://<approved-query-bucket>/queries/
  cmbs_risk_score/
    query.sql
    manifest.json

  issuer_exposure/
    query.sql
    manifest.json
```

Example manifest:

```json
{
  "schema_version": 1,
  "query_id": "cmbs_risk_score",
  "name": "CMBS Risk Score",
  "description": "Returns CMBS risk scores by CUSIP.",
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
      "items": {"type": "string"},
      "required": true,
      "minItems": 1,
      "maxItems": 500
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

Do not hard-code query definitions in Python.

## Query Registry

Create or integrate a `QueryRegistry` abstraction.

Suggested logical API:

```text
refresh()
search(intent, max_results)
get(query_id)
get_public_definition(query_id)
validate_parameters(query_id, parameters)
validate_output_fields(query_id, output_fields)
```

Responsibilities:

- load manifests from S3
- validate manifests
- cache active definitions
- expose exact lookup
- expose search
- reject malformed entries
- preserve last known-good registry if refresh fails

Do not put all of this logic inside the MCP tool functions.

## Registry Refresh

The server must discover new S3 queries without redeployment or restart.

Recommended initial behavior:

- load registry on startup
- keep it in memory
- refresh using a short TTL, typically 1-5 minutes
- replace cache atomically
- preserve the previous valid cache if refresh fails
- log additions, removals, changes, and failures

If the existing system already uses background refresh, events, SQS, SNS, EventBridge, or cache invalidation, use that existing mechanism instead.

Do not introduce new infrastructure unless needed.

## Search Strategy

Start simple.

Search over:

- query ID
- name
- aliases if present
- tags
- description
- `use_when`

Ranking may include:

1. exact ID/name match
2. exact tag/alias match
3. normalized token overlap
4. description/use_when overlap

Add BM25 or embeddings only if the current query catalog warrants it.

The MCP contract must not change when the internal search strategy changes.

## Parameter Validation

Use a versioned manifest schema.

For every execution:

- reject unknown parameters by default
- enforce required parameters
- enforce types
- enforce string/array length constraints
- enforce numeric ranges where defined
- enforce enum values where defined
- enforce maximum list sizes
- enforce date formats where defined

Never trust LLM-generated arguments without server-side validation.

If the existing project already uses Pydantic, JSON Schema, Marshmallow, or another validator, reuse it where practical.

## SQL Execution

The SQL file itself must be pre-approved and loaded from S3 based only on trusted registry metadata.

Requirements:

- parameter values must be passed through DB-driver bind parameters
- never use string concatenation for parameter values
- never use model-provided SQL fragments
- use existing DB connection pools and retry logic
- respect existing transaction behavior
- prefer read-only DB identity if queries are read-only
- set statement/query timeouts
- enforce result-row/result-size limits
- close cursors/resources correctly
- preserve async patterns if the existing DB layer is async

## Output Fields

Do not create separate SQL files for every output-column combination.

Prefer:

```text
one approved query
    -> approved superset of fields
    -> service projects to requested approved fields
```

Manifest controls:

```json
{
  "allowed_output_fields": [
    "cusip",
    "risk_score",
    "rating"
  ],
  "default_output_fields": [
    "cusip",
    "risk_score"
  ]
}
```

If output fields are omitted, use defaults.

If requested fields contain an unapproved field, reject the request.

Only introduce dynamic SELECT-list construction if performance clearly requires it, and then construct it exclusively from server-owned mappings, never raw LLM values.

## Authorization

Reuse the MCP server's existing authentication/authorization framework.

Authorization must be enforced on the server, not by prompts.

Support query-level authorization if needed, for example:

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

The LLM should never be able to elevate privileges by changing arguments.

## S3 Security

Prefer:

- runtime role has read-only access to approved query prefix
- publishing role is separate from runtime role
- bucket encryption enabled
- S3 versioning enabled where appropriate
- block public access
- restrict `sql_key` to approved prefix
- validate object identity/version if required by the organization

Never allow the model to supply an arbitrary bucket or object key for execution.

## Error Handling

Follow the existing project's error pattern but make errors model-friendly.

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
  "message": "Parameter 'cusips' is required and must be a non-empty array."
}
```

```json
{
  "error": "INVALID_OUTPUT_FIELDS",
  "message": "One or more requested output fields are not allowed."
}
```

Do not return:

- stack traces
- SQL text
- credentials
- DB hostnames unless already considered public/safe
- AWS secrets
- internal exception dumps

## Logging and Audit

Reuse existing logging infrastructure.

Log at minimum:

- caller/user/service identity where available
- query ID
- query version
- success/failure
- duration
- row count
- registry refresh events

Do not log sensitive parameter values unless explicitly permitted.

Prefer logging parameter names and types rather than values.

## Backward Compatibility

Do not break existing MCP tools or resources.

Before finishing:

- verify existing tool names/signatures remain unchanged
- verify existing resource behavior remains unchanged
- avoid renaming shared configuration variables without necessity
- avoid replacing existing AWS/DB clients unless required
- avoid broad formatting/refactoring unrelated modules
- keep changes localized

If a conflict is found between this design and existing project conventions, preserve behavior and adapt the new query capability to fit.

## Suggested File Placement

Use the existing project structure if one exists.

If no clear structure exists, a reasonable addition is:

```text
src/
  existing_server.py

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
```

Tests:

```text
tests/
  query_tool/
    test_registry.py
    test_search.py
    test_manifest_validation.py
    test_execute_query.py
    test_resource.py
```

Do not force this layout into a mature repository with established conventions.

## Implementation Order

Use this sequence:

1. Inspect repo and FastMCP version.
2. Identify existing S3/DB/config/auth/logging abstractions.
3. Define manifest model/schema.
4. Implement S3 query store.
5. Implement registry + refresh/cache.
6. Implement deterministic search.
7. Implement parameter/output validation.
8. Implement query executor using existing DB layer.
9. Add `find_query`.
10. Add `execute_query`.
11. Add `query://{query_id}` resource.
12. Add tests.
13. Run existing test suite.
14. Fix regressions.
15. Document how to register a new query in S3.

## Required Tests

Add tests proving:

1. Existing MCP tools still work.
2. Registry loads a valid S3 manifest.
3. New query becomes visible after refresh without restart.
4. Malformed manifest is rejected.
5. Unknown query ID is rejected.
6. Disabled query is rejected.
7. Arbitrary S3 key cannot be supplied by the LLM.
8. Missing required parameter is rejected.
9. Unknown parameter is rejected.
10. Invalid parameter type is rejected.
11. Output field outside allow-list is rejected.
12. Default output fields are applied.
13. Query parameters are bound rather than interpolated.
14. SQL file is resolved only through trusted registry metadata.
15. Registry refresh failure preserves previous known-good cache.
16. Query resource returns documentation without exposing SQL.
17. Auth rules are enforced if the existing system supports them.
18. Timeouts/result limits are enforced.
19. Logs do not leak prohibited sensitive values.

Mock S3 and DB calls where appropriate.

## Definition of Done

This integration is complete when:

- the existing MCP server still runs normally
- `find_query` can discover registered S3 queries
- `query://{query_id}` exposes dynamic documentation
- `execute_query` executes only exact registered query IDs
- parameter/output validation is enforced server-side
- SQL is loaded only from approved S3 metadata
- the LLM cannot submit executable SQL
- adding a new valid query folder + manifest to S3 requires no source code change
- the new query becomes available after registry refresh
- tests cover the security boundaries
- documentation explains the S3 registration process

## Do Not Do These Things

Do not:

- create one MCP tool per SQL file
- create one Python resource function per query
- hard-code query IDs in application source
- place the whole catalog in one static `oneOf` schema
- accept arbitrary SQL
- accept arbitrary SQL fragments
- accept arbitrary S3 paths
- rely on prompts for authorization
- create separate SQL files only because callers want different output columns
- restart/redeploy the service merely to register a new query
- rewrite unrelated parts of the existing MCP server

## Final Deliverables When Implementing

When working on the repository, provide:

1. the code changes
2. tests
3. example `manifest.json`
4. example query registration folder structure
5. any configuration additions
6. concise run/test instructions
7. a short explanation of how to add another query without deploying code

Prefer working code over pseudocode.
