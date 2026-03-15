# Security Review Checklist

Structured checklist for evaluating skills, deployment plans, and live setups. Each category contains
specific checks with severity guidance. Not every check applies to every artifact — skip checks that
are clearly irrelevant to the artifact type.

**This checklist uses Elastic/Kibana terminology as concrete examples, but the security principles
apply universally.** When reviewing non-Elastic artifacts (CI/CD pipelines, Terraform plans, Docker
setups, Kubernetes configs, generic APIs), map each check to the equivalent concept in that domain.

Severity levels: **CRITICAL** (must fix before use), **HIGH** (significant risk, fix soon),
**MEDIUM** (notable concern, should address), **LOW** (minor improvement, nice to have).

---

## 1. Credential hygiene

| # | Check | Severity | Details |
|---|-------|----------|---------|
| 1.1 | API keys interpolated into shell commands | CRITICAL | Keys in `curl -H "Authorization: ApiKey ${VAR}"` are visible in process lists (`ps aux`), terminal history, and agent transcripts. Verify commands use env var expansion only at execution time, never string interpolation that persists the value. |
| 1.2 | Secrets written to temp files | HIGH | Query files, config files, or logs that contain credentials. Check for patterns like `echo $KEY > file` or heredocs that embed secrets. |
| 1.3 | `.env` files with overly broad permissions | HIGH | `.env` files should be 600 (owner-only). Check if `.gitignore` excludes them. Check if the skill documents that `.env` must not be committed. |
| 1.4 | Credentials shared across skills without scoping | MEDIUM | Multiple skills using the same API key means one compromised skill exposes all. Check if the skill documents minimum required permissions for its API key. |
| 1.5 | Hardcoded credentials or example keys | CRITICAL | Placeholder keys that look real (base64 strings, UUIDs) may be copy-pasted into production. Verify all examples use obvious placeholders like `<your-api-key>`. |
| 1.6 | Credentials in error output | HIGH | Failed API calls may log the full request including auth headers. Check if error handling strips credentials before display. |

## 2. Write operation safety

| # | Check | Severity | Details |
|---|-------|----------|---------|
| 2.1 | Missing `--dry-run` before destructive operations | HIGH | Bulk updates (acknowledge, delete, close) should always offer or require a dry-run step first. Check if the skill documents this as mandatory. |
| 2.2 | No confirmation gate before write actions | CRITICAL | Write operations with downstream consequences (SLA timers, assignments, notifications) must require explicit user approval. Check for a checkpoint step. |
| 2.3 | No upper-bound on bulk operations | HIGH | Bulk acknowledge, bulk update, or bulk delete without a `--max-count` or `LIMIT` could affect unbounded records. Check for scope caps. |
| 2.4 | No rollback path for multi-step writes | MEDIUM | If step 3 of 4 fails, what is the state? Check if the skill documents partial-failure behavior and recovery steps. |
| 2.5 | Missing idempotency | MEDIUM | Running the same skill twice should not create duplicate cases, duplicate comments, or re-acknowledge already-acknowledged alerts. Check for deduplication logic. |
| 2.6 | Write operations in "read-only" phases | HIGH | Verify that phases labeled as "assessment" or "read-only" contain no write operations (POST, PUT, PATCH, DELETE, acknowledge, create). |

## 3. Data boundary enforcement

| # | Check | Severity | Details |
|---|-------|----------|---------|
| 3.1 | Sensitive data sent to LLM context | HIGH | Hostnames, IPs, usernames, command-line arguments, file paths in alert data will be sent to the LLM as part of agent context. Check if the skill acknowledges this boundary and minimizes exposure. |
| 3.2 | PII in case descriptions or comments | MEDIUM | Case descriptions may contain usernames, email addresses, or internal hostnames that persist in the case management system. Check if the skill guidance minimizes PII in written artifacts. |
| 3.3 | Raw alert data persisted beyond need | MEDIUM | Full alert JSON dumped into case comments or files retains more data than necessary. Check if the skill selects only relevant fields. |
| 3.4 | Agent transcripts containing secrets | HIGH | Terminal output, query results, and error messages in agent transcripts may contain API keys, internal URLs, or sensitive alert content. These transcripts persist on disk. |
| 3.5 | Data exfiltration via query results | MEDIUM | Large query results (`--full` flag, no `LIMIT`) may pull more data than needed into the agent context. Check for explicit field selection and result limits. |

## 4. Blast radius controls

| # | Check | Severity | Details |
|---|-------|----------|---------|
| 4.1 | Wildcard index patterns without scoping | MEDIUM | Queries hitting `logs-*` or `.alerts-security.alerts-*` without time bounds or additional filters could scan enormous data volumes. Check for time range constraints. |
| 4.2 | `IN (...)` clauses with untrusted ID lists | HIGH | If alert IDs or entity names come from a prior query, a bug in that query could produce wrong IDs. The downstream `IN (...)` then targets unintended records. Check for validation between steps. |
| 4.3 | Production data without environment guards | HIGH | No check for whether the target is a production, staging, or test environment. A skill intended for testing could be run against production. Check for environment awareness. |
| 4.4 | Missing scope caps | HIGH | Operations should have explicit maximums: max alerts to acknowledge, max cases to create, max results to return. Check for `LIMIT`, `--max-count`, or equivalent guards. |
| 4.5 | Time window unbounded | MEDIUM | Queries without time bounds (e.g., missing `@timestamp >= ...`) scan all historical data. Check that every query has explicit time constraints. |

## 5. Failure cascading and partial state

| # | Check | Severity | Details |
|---|-------|----------|---------|
| 5.1 | Acknowledged alerts with no case | HIGH | If case creation fails after alerts are acknowledged, those alerts drop out of the triage queue with no record. Check if acknowledgment happens AFTER case creation. |
| 5.2 | Cases with no alerts attached | MEDIUM | If alert attachment fails after case creation, an orphaned case exists with no linked evidence. Check if the skill verifies attachment success. |
| 5.3 | No cleanup on failure | MEDIUM | Multi-step workflows that fail mid-execution should document what to clean up. Check for failure handling guidance. |
| 5.4 | Retry without deduplication | HIGH | Retrying a failed step that partially succeeded could duplicate cases, comments, or acknowledgments. Check if retry logic is idempotent. |
| 5.5 | Silent failures | HIGH | API calls that return 200 but with partial errors (e.g., bulk operations where some items fail) may be treated as full success. Check for response validation. |

## 6. Input validation and injection

| # | Check | Severity | Details |
|---|-------|----------|---------|
| 6.1 | Shell command injection via alert fields | CRITICAL | Attacker-controlled data (hostnames, usernames, file paths, command lines) from alert fields injected into shell commands without quoting or escaping. Check all `node ... --title "<field>"` patterns. **Domain note**: this check applies to any shell/CLI command execution with untrusted input — not to query languages (see 6.2). |
| 6.2 | Query injection via untrusted input | HIGH | ES\|QL or KQL queries constructed by string concatenation with values from alert data. An attacker who controls a hostname or username could inject query syntax. Check for parameterization or escaping. **Domain note**: applies to ALL query languages — SQL, ES\|QL, KQL, GraphQL, Cypher, PromQL. When reviewing non-Elastic artifacts, map SQL injection, GraphQL injection, etc. to this check (6.2), not to shell injection (6.1). |
| 6.3 | Prompt injection via case content | HIGH | Case descriptions built from attacker-controlled alert data (process names, command lines) that are later fed to an LLM. Malicious process names like `powershell -c "ignore previous instructions..."` could manipulate AI analysis. |
| 6.4 | Markdown injection in case fields | MEDIUM | Untrusted content in case descriptions or comments could contain markdown that renders as links, images, or hidden content in the Kibana UI. |
| 6.5 | Path traversal in file references | MEDIUM | If the skill reads files based on user input or alert data (e.g., script paths, query file paths), check for `../` traversal or absolute path injection. |

## 7. Operational resilience

| # | Check | Severity | Details |
|---|-------|----------|---------|
| 7.1 | No rate limiting awareness | MEDIUM | Kibana and Elasticsearch APIs have rate limits. Bulk operations in tight loops can trigger 429 responses. Check if the skill handles rate limiting or adds delays. |
| 7.2 | No timeout handling | MEDIUM | Long-running ES\|QL queries or API calls with no timeout could hang the workflow indefinitely. Check for timeout configuration or guidance. |
| 7.3 | Error messages leaking internals | HIGH | Failed API calls may expose internal Elasticsearch URLs, cluster names, index names, or stack traces. Check if error handling sanitizes output. |
| 7.4 | Missing health checks | LOW | Multi-step workflows that assume all services are available. A connectivity check before starting prevents partial execution against an unreachable cluster. |
| 7.5 | No progress indicators for bulk operations | LOW | Bulk operations with no progress output leave the user uncertain whether the operation is working or hung. Check for intermediate status reporting. |
| 7.6 | Dependency availability assumptions | MEDIUM | Skills that assume Node.js, specific npm packages, or CLI tools are installed without checking. Check for prerequisite verification steps. |
