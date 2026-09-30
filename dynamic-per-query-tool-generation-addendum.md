# Skill Addendum: Config-Driven Dynamic MCP Tool Exposure

## Purpose

Extend the existing Approved SQL Query MCP design so the FIBData(S3)-backed query registry remains the authoritative execution catalog, while a separate **tool exposure configuration** controls which registered queries are exposed as MCP tools and how those tools are presented to the LLM.

This is an optional presentation layer only.

Do not replace or weaken the existing registry, validation, authorization, filter, output-field, or execution architecture.

The query registry remains the source of truth for what is allowed to execute.

The tool exposure config controls what is visible to the LLM.

---

## Core Principle

Keep these concerns separate:

```text
Query Registry
    defines what is allowed to execute

Tool Exposure Config
    defines what is exposed to the LLM

Dynamic Tool Factory
    converts exposure config + registry metadata into MCP tools

QueryExecutor
    enforces the registry at runtime
```

The dependency direction must remain:

```text
FIBdata(S3) query.sql + manifest.json
        ->
QueryRegistry
        ->
Tool Exposure Config
        ->
Dynamic Tool Factory
        ->
MCP tools
```

The MCP tool layer must never become the authoritative execution contract.

---

## Supported Exposure Modes

```text
QUERY_TOOL_EXPOSURE_MODE=generic
```

Allowed values:

```text
generic
config_driven
hybrid
```

### `generic`

Expose only the stable generic interface:

```text
find_query
execute_query
query://{query_id}
```

### `config_driven`

Expose only MCP tools explicitly enabled in the tool exposure configuration.

### `hybrid`

Expose both the generic interface and config-driven generated tools.

Preserve backward compatibility.

---

# 1. Query Registry Remains Authoritative

Each query remains registered using its approved query definition stored in FIBData (e.g. /fermi folder).

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

The query manifest defines the maximum allowed capability.

Example:

```json
{
  "schema_version": 1,
  "query_id": "cmbs_risk_score",
  "name": "CMBS Risk Score",
  "description": "Returns CMBS risk scores by CUSIP.",
  "enabled": true,
  "sql_key": "queries/cmbs_risk_score/query.sql",

  "parameters": {
    "cusips": {
      "type": "array",
      "items": {"type": "string"},
      "required": true,
      "description": "CUSIPs to evaluate."
    },
    "as_of_date": {
      "type": "string",
      "format": "date",
      "required": false
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

  "default_output_fields": [
    "cusip",
    "risk_score"
  ],

  "version": 1
}
```

The registry controls:

```text
SQL artifact
parameters
parameter types
allowed filters
allowed operators
allowed output fields
authorization
enabled status
query version
execution policy
```

---

# 2. Add a Separate Tool Exposure Config

Do not automatically create one MCP tool for every registered query.

Instead, maintain a separate configuration that explicitly selects which registered queries should be exposed as MCP tools.

Example:

```yaml
schema_version: 1

tools:

  - tool_name: cmbs_risk_score
    query_id: cmbs_risk_score
    enabled: true

    description: >
      Get CMBS security risk scores for one or more CUSIPs.

    use_when:
      - user asks for CMBS risk score
      - user asks for CMBS risk information by CUSIP

    exposed_parameters:
      - cusips
      - as_of_date

    exposed_filters:
      - rating
      - spread

    exposed_output_fields:
      - cusip
      - risk_score
      - rating
      - spread

    default_output_fields:
      - cusip
      - risk_score

    tags:
      - cmbs
      - risk
      - cusip

  - tool_name: issuer_exposure
    query_id: issuer_exposure
    enabled: true

    description: >
      Return issuer exposure for a portfolio.

    exposed_parameters:
      - portfolio_id
      - as_of_date
```

The tool exposure config may contain auxiliary LLM-facing metadata such as:

- tool name
- tool description
- `use_when`
- tags
- examples
- categories
- aliases
- exposed parameter subset
- exposed filter subset
- exposed output-field subset
- default output fields
- optional display/help text
- optional deprecation metadata
- optional visibility metadata

This config is the LLM-facing presentation contract.

---

# 3. Registry Capability vs Tool Exposure

The tool config may only reduce or specialize capabilities already allowed by the query registry.

It must never expand them.

Safe rule:

```text
Query Registry = maximum allowed capability

Tool Exposure Config = subset/presentation of that capability
```

Example:

If the registry allows:

```json
{
  "allowed_output_fields": [
    "cusip",
    "risk_score",
    "rating"
  ]
}
```

then the tool config may expose:

```yaml
exposed_output_fields:
  - cusip
  - risk_score
```

but must reject:

```yaml
exposed_output_fields:
  - internal_model_value
```

Likewise, if the registry allows:

```text
rating -> eq, in
```

the tool config cannot introduce:

```text
rating -> contains
```

All config must be validated against the registry before tools become active.

---

# 4. Config Storage

The tool exposure config should be externalized so it can change without a code release.

Preferred options:

```text
Configuration file in FIBData (same folder /fermi as the query registry) object
```

A simple starting point:

```text
s3://<bucket>/mcp-config/query-tools.yaml
```

Do not hard-code the exposed tool list in Python.

---

# 5. Dynamic Tool Factory

Implement a dedicated component such as:

```text
DynamicToolFactory
```

Responsibilities:

```text
load desired tool config
validate config against QueryRegistry
build tool name
build description
build query-specific input schema
register tool
update tool
unregister tool
sync desired tool set
```

Suggested component boundary:

```text
ToolExposureConfigLoader
    ->
ToolExposureValidator
    ->
ToolSchemaBuilder
    ->
DynamicToolFactory
```

Do not place this logic inside `QueryExecutor`.

---

# 6. Generated Tool Contract

For this config entry:

```yaml
tool_name: cmbs_risk_score
query_id: cmbs_risk_score

exposed_parameters:
  - cusips
  - as_of_date

exposed_filters:
  - rating
  - spread

exposed_output_fields:
  - cusip
  - risk_score
  - rating
  - spread
```

generate a tool conceptually like:

```python
cmbs_risk_score(
    cusips: list[str],
    as_of_date: str | None = None,
    filters: list[Filter] | None = None,
    output_fields: list[str] | None = None
)
```

The generated handler must bind permanently to:

```text
query_id = cmbs_risk_score
```

The LLM must not supply or override `query_id`.

The handler must delegate execution to the same central `QueryExecutor`.

---

# 7. Parameter Schema Generation

Generate exposed parameter fields from:

```text
tool exposure config
        +
query registry manifest
```

The config selects which parameters are visible.

The registry defines their actual types, constraints, and validation rules.

Example:

```yaml
exposed_parameters:
  - cusips
  - as_of_date
```

Registry metadata:

```json
{
  "cusips": {
    "type": "array",
    "items": {"type": "string"},
    "required": true
  },

  "as_of_date": {
    "type": "string",
    "format": "date",
    "required": false
  }
}
```

Generated schema should be query-specific and strongly typed where practical.

Runtime validation remains mandatory.

---

# 8. Filter Schema Generation

Do not generate one tool per filter combination.

One generated tool supports all filters explicitly exposed by config.

Example config:

```yaml
exposed_filters:
  - rating
  - spread
```

Registry:

```json
{
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
  }
}
```

Generated tool may expose only:

```text
rating
spread
```

and omit:

```text
sector
```

Preferred request format:

```json
{
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
  ]
}
```

All filter fields and operators must still be validated through the registry.

---

# 9. Output Field Schema Generation

The config may expose only a subset of registry-approved output fields.

Example:

```yaml
exposed_output_fields:
  - cusip
  - risk_score
  - rating
  - spread

default_output_fields:
  - cusip
  - risk_score
```

The generated tool schema should make these output choices visible to the LLM.

Runtime execution must validate:

```text
requested field
    is in exposed_output_fields
    AND
requested field
    is in registry.allowed_output_fields
```

If no output fields are requested, use the config default if valid; otherwise fall back to the registry default.

---

# 10. Auxiliary LLM Metadata

The tool exposure config may add LLM-facing metadata not required for SQL execution.

Examples:

```yaml
description: >
  Returns CMBS security risk scores for one or more CUSIPs.

use_when:
  - user asks for CMBS risk score
  - user asks for CMBS credit risk information

examples:
  - "Get risk score for these CUSIPs"
  - "Show CMBS rating and spread for these securities"

aliases:
  - cmbs risk
  - mortgage bond risk

category: structured_products

tags:
  - cmbs
  - risk
  - cusip
```

These fields may improve LLM tool selection and documentation.

They must not override registry security or execution constraints.

---

# 11. Config Refresh Without Deployment

Updating the tool exposure config must change the MCP tool set without requiring a source-code release.

Recommended flow:

```text
update config in FIBData (AWS s3)
        ->
config refresh
        ->
validate config
        ->
compare desired tool set to active tool set
        ->
add / update / remove tools
```

Use:

```text
TTL polling
```

as the simplest initial implementation.

Recommended initial interval:

```text
1-5 minutes
```

Optional later optimization:

```text
S3 event
    ->
SQS/EventBridge
    ->
config invalidation/refresh
```

Do not require process restart unless the installed FastMCP version cannot safely update the active tool registry.

---

# 12. Tool Set Reconciliation

Maintain:

```text
desired tool set
active tool set
```

On each config refresh:

```text
new config entry
    -> register new MCP tool

removed config entry
    -> unregister/disable MCP tool

enabled false
    -> unregister/disable MCP tool

description/schema/config changed
    -> rebuild/update tool

unchanged entry
    -> do nothing
```

Use a stable fingerprint such as:

```text
tool config hash
+
query manifest version
```

to determine whether a tool needs rebuilding.

---

# 13. MCP Tool-List Changes

If the installed FastMCP/MCP stack supports tool-list change notifications, emit them after the active tool set changes so compatible clients can refresh `tools/list`.

Do not assume every client automatically refreshes.

Document any client-side limitations.

Generic mode should remain available as a reliable fallback.

---

# 14. Config Validation

Before activating a tool config entry, validate:

1. `query_id` exists in the active registry.
2. query is enabled.
3. `tool_name` is valid.
4. tool name does not collide with an existing static tool.
5. tool name does not collide with another generated tool.
6. all exposed parameters exist in registry.
7. all exposed filters exist in registry.
8. all exposed output fields exist in registry.
9. defaults are valid.
10. auxiliary metadata is structurally valid.
11. authorization metadata, if present, does not broaden registry permissions.

Invalid config entries must not become active tools.

Keep the last known-good tool configuration if a refresh fails validation.

---

# 15. Security Model

Config-driven tool generation must not weaken the existing security model.

Every generated call still passes through:

```text
authorization
registry lookup
parameter validation
filter validation
output-field validation
SQL artifact resolution
parameter binding
query timeout
result limits
audit logging
```

The tool config is not trusted as an execution authority.

Never allow config to introduce:

- arbitrary SQL
- arbitrary FIBData (S3) paths
- new DB columns not in registry
- new filter operators not in registry
- new output fields not in registry
- privilege expansion

---

# 16. Authorization

The registry remains authoritative for execution authorization.

The tool config may optionally make visibility narrower.

Example:

```yaml
visible_to_roles:
  - structured_products
```

But tool visibility must never replace server-side execution authorization.

Safe rule:

```text
tool visibility
    can reduce visibility

query authorization
    controls execution
```

---

# 17. Example End-to-End Flow

Registry contains:

```text
cmbs_risk_score
issuer_exposure
portfolio_holdings
trade_history
```

Tool exposure config contains:

```yaml
tools:

  - tool_name: cmbs_risk_score
    query_id: cmbs_risk_score
    enabled: true

  - tool_name: issuer_exposure
    query_id: issuer_exposure
    enabled: true
```

The MCP server exposes:

```text
cmbs_risk_score
issuer_exposure
```

It does not expose:

```text
portfolio_holdings
trade_history
```

even though those queries are registered and may still be available through generic `find_query` / `execute_query` when running in hybrid mode.

Later the config is changed to:

```yaml
tools:

  - tool_name: cmbs_risk_score
    query_id: cmbs_risk_score
    enabled: true

  - tool_name: portfolio_holdings
    query_id: portfolio_holdings
    enabled: true
```

After refresh:

```text
issuer_exposure
    removed

portfolio_holdings
    added
```

No source-code change is required.

---

# 18. Recommended Components

Suggested structure:

```text
query_tool/
    registry.py
    manifest_validator.py
    executor.py

    exposure/
        config_loader.py
        config_models.py
        config_validator.py
        tool_schema_builder.py
        dynamic_tool_factory.py
        tool_reconciler.py
```

Responsibilities:

```text
registry.py
    authoritative query definitions

config_loader.py
    loads tool exposure config

config_validator.py
    validates config against registry

tool_schema_builder.py
    generates MCP schemas

dynamic_tool_factory.py
    registers/updates/removes tools

tool_reconciler.py
    compares desired and active tool sets

executor.py
    central secure execution path
```

---

# 19. Required Tests

Add tests proving:

1. registered query does not automatically become an MCP tool
2. enabled config entry generates a tool
3. disabled config entry does not generate a tool
4. removing config entry removes/disables tool
5. config refresh updates the active tool set without code release
6. exposed parameters are a subset of registry parameters
7. exposed filters are a subset of registry filters
8. exposed output fields are a subset of registry output fields
9. invalid expansion beyond registry is rejected
10. tool name collisions are rejected
11. generated handler binds to the configured query ID
12. LLM cannot override query ID
13. generated tool uses central QueryExecutor
14. updated description/schema causes tool refresh
15. malformed config preserves last known-good tool set
16. query disabled in registry causes tool to become unavailable
17. authorization remains enforced
18. filters remain allow-listed and parameter-bound
19. output fields remain allow-listed
20. SQL text remains hidden
21. generic mode remains unaffected
22. hybrid mode exposes both interfaces
23. tool-list change notification is emitted when supported

---

# 20. Definition of Done

This enhancement is complete when:

- the registry remains the authoritative execution contract
- a separate external config controls which registry queries become MCP tools
- tool metadata can include auxiliary LLM-facing information
- tool configs may narrow but never broaden registry capability
- config changes can add, update, or remove MCP tools without source-code deployment
- generated tools remain query-specific and strongly described
- filters and output fields remain flexible within approved bounds
- all execution flows through the central QueryExecutor
- generic mode remains supported
- config-driven and hybrid modes are configuration-selectable
- existing behavior remains backward compatible

---

# Final Architectural Rule

Treat the system as three layers:

```text
1. QUERY REGISTRY
   What is allowed to execute?

2. TOOL EXPOSURE CONFIG
   What should the LLM see?

3. DYNAMIC TOOL FACTORY
   How is that config presented as MCP tools?
```

The `QueryExecutor` remains the final enforcement point.

This separation allows the SQL/query catalog and the MCP tool catalog to evolve independently while preserving security and avoiding application releases for normal query/tool exposure changes.
