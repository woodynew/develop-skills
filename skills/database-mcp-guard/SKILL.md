---
name: database-mcp-guard
description: Prefer project-matched, database-capable MCP tools before performing database work, including Google MCP Toolbox for Databases, generic SQL MCPs, cloud database MCPs, and local custom project MCPs. Use when the user asks Codex to inspect, query, modify, migrate, repair, seed, clean, import/export, or otherwise operate on a database; when SQL, tables, schemas, records, migrations, MySQL, PostgreSQL, SQLite, Redis, MongoDB, D1, Supabase, Prisma, Drizzle, or ORM-backed data operations are involved; or when deciding whether a database action is safe. Requires explicit user confirmation for dangerous operations such as dropping databases or tables, destructive deletes, important data removal, or complex schema changes.
---

# Database MCP Guard

Use this skill as a safety and routing layer for database work. Prefer a matching database-capable MCP for the current project, verify that the MCP points at the project database, then perform the user's requested operation through that MCP when safe.

## Workflow

1. Classify the request.
   - Treat queries, schema inspection, migrations, data fixes, imports/exports, seeds, cleanup, and ORM-backed data changes as database operations.
   - Identify the intended project, environment, database engine, database name, tables, and requested action from the user's request and local project context.

2. Search for a matching database-capable MCP.
   - Inspect available MCP/tool namespaces and deferred tool metadata for tools that can list schemas, inspect tables, run SQL, execute prepared operations, query records, or manage migrations.
   - Recognize database capability by behavior, not only by server name. A usable database MCP may be named after a project, environment, cloud provider, framework, or local service instead of `database`.
   - Consider common database-capable MCP families such as Google MCP Toolbox for Databases, generic MySQL/PostgreSQL/SQLite SQL MCPs, Cloudflare D1 or other cloud database MCPs, Supabase-like MCPs, ORM/project-specific MCPs, and local custom MCPs created for the current repository.
   - Prefer tools whose names, namespaces, descriptions, available operations, connection metadata, schemas, configured database names, or target resources match the current project.
   - If multiple MCPs could match, use read-only evidence to disambiguate before choosing.

3. Verify that the MCP is for the current project database.
   - Compare the MCP target with local evidence such as `AGENTS.md`, README files, `.env.example`, config files, ORM config, migration folders, docker compose files, package scripts, or application database names.
   - When needed, run only safe read-only MCP checks such as current database, version, table list, or schema summary.
   - Do not assume an MCP is correct just because it is available.

4. Execute through the matched MCP when safe.
   - Use read-only operations first to understand current state.
   - Prefer scoped SQL with explicit table and key predicates.
   - For data changes, preview affected rows or counts before mutation when practical.
   - For table-creation migration SQL, follow the migration guard below before execution.
   - Report which MCP/database was used and what verification supports that match.

5. If no suitable MCP matches, hand off to the normal AI path.
   - Continue using project files, code, migrations, and user-provided context.
   - State that no matching current-project database MCP was found.
   - Do not pretend to have live database access.
   - Only use direct database clients or credentials when the user explicitly asked for that route or the project clearly provides a local/dev database path.

## MCP Matching

Match MCPs by capability and project fit, not by a fixed list of product names.

1. Discover candidates.
   - Use the available tool discovery surfaces to search for database-capable tools before falling back to ordinary code reasoning.
   - Include tools exposed by Google MCP Toolbox for Databases even when the server exposes domain-specific tool names instead of generic SQL names.
   - Include local custom MCPs when their namespace, description, or operations suggest access to the repository's database.
   - Treat operations such as `execute_sql`, `query`, `list_tables`, `describe_table`, `get_schema`, `get_query_plan`, `run_migration`, `insert`, `update`, or `delete` as database-capability signals.

2. Score project fit.
   - Highest confidence: the MCP target database name, schema names, table names, service name, config path, or environment label matches the current repository.
   - Medium confidence: the MCP engine and visible schema align with project config, migrations, ORM models, or known table prefixes.
   - Low confidence: the MCP is database-capable but has no evidence linking it to the current project.
   - Do not use a low-confidence MCP for writes. Use only read-only probes or fall back to AI handling.

3. Verify with safe probes.
   - Prefer read-only probes such as current database, current schema, database version, `SELECT 1`, table listing, table count, or schema summary.
   - Compare probe results against local project evidence from `.env.example`, config files, migration names, ORM models, Docker Compose services, README files, and package scripts.
   - Never use write probes to test whether an MCP works.

4. Resolve conflicts.
   - If one MCP clearly matches the project and another is generic, choose the project-matched MCP.
   - If multiple MCPs all match the current project but target different database engines, models, stores, or products, ask the user which database to operate on before any write, migration, or destructive action.
   - Treat MySQL vs PostgreSQL, relational SQL vs Redis, MongoDB vs SQL, local SQLite vs cloud SQL, primary app database vs analytics/search/cache database, and ORM-generated database vs manually configured database as materially different targets.
   - If two MCPs target different environments, prefer the one explicitly requested by the user; otherwise prefer local/dev over staging/production for exploratory work.
   - If the target remains ambiguous and the requested action is a write or migration, ask one concise clarification question before proceeding.

## Table-Creation Migration SQL

When the user asks to run or generate migration SQL that creates a table, inspect the current database state before applying it.

1. Identify the target table and intended structure.
   - Parse the `CREATE TABLE` statement, ORM migration, or generated SQL enough to know the table name, columns, indexes, primary keys, unique constraints, foreign keys, defaults, nullability, and comments when relevant.
   - If the migration contains an explicit `DROP TABLE`, `DROP TABLE IF EXISTS`, or equivalent drop-before-create statement supplied or requested by the user, treat that as the user's chosen replacement path after confirming the target database is correct.

2. Check whether the table already exists.
   - Use the matched database MCP to inspect the table list and existing table definition.
   - If the table does not exist, apply the create-table migration normally.

3. If the table exists with the same structure, skip the create-table operation.
   - Treat semantically equivalent definitions as the same even when formatting, constraint names, or harmless engine-specific defaults differ.
   - Report that the table already exists and no migration was needed.

4. If the table exists with a different structure, check whether it contains data.
   - If the user explicitly included or requested `DROP`, execute the drop-and-create path directly after the current-project database check.
   - If the table contains data and the user did not explicitly request `DROP`, ask whether to perform an incremental schema modification or drop and recreate the table.
   - Include the row count, schema differences, and consequences of each option in the confirmation prompt.
   - Do not drop or rewrite a populated table without the user's explicit choice.

## Dangerous Operations

Always ask for explicit confirmation before executing dangerous operations, even when an MCP matches.

Dangerous operations include:

- Dropping or deleting a database, schema, table, collection, namespace, or index.
- Truncating tables or collections.
- Deleting important data, broad deletes, deletes without a precise key predicate, or deletes affecting many rows.
- Bulk updates that overwrite important fields or affect many rows.
- Complex schema changes such as destructive migrations, column type changes, column drops, table rewrites, partition changes, constraint rewrites, or irreversible data transforms.
- Production or production-like writes, unless the user has already provided explicit approval for the exact operation.

Exception: when the user-provided migration SQL or instruction explicitly includes `DROP TABLE` / drop-before-create for the target table, treat that as confirmation for that specific drop after verifying the MCP points at the intended current-project database. Do not expand that confirmation to other tables, databases, or environments.

Before asking for confirmation, summarize:

- The exact operation to run.
- The target MCP, environment, database, schema/table, and predicate.
- The expected impact and affected-row estimate when available.
- Backup, rollback, or recovery options when known.

Proceed only after the user clearly confirms the specific dangerous operation. If confirmation is vague, narrow the operation and ask again.

## Safety Defaults

- Prefer read-only discovery before writes.
- Prefer transactions, dry-runs, backups, or reversible steps when supported.
- Avoid cross-environment ambiguity; never write when unsure whether the target is local, staging, or production.
- Refuse or pause when the user asks for destructive production actions without adequate target clarity.
- Keep SQL and tool calls minimal, auditable, and scoped to the user's request.
