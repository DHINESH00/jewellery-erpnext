# Codebase Analysis Prompt — `jewellery_erpnext`

This document is a **ready-to-use prompt** for instructing an AI agent (or a
human reviewer) to perform a deep, end-to-end analysis of the
`jewellery_erpnext` custom Frappe application that lives in this repository.

It is written so you can paste the **"Prompt"** section (below) directly into
an LLM/agent. The context section above it tells the reviewer where to look
and gives them the repo-specific anchors they need to make the analysis
concrete instead of generic.

---

## 0. Repository context the reviewer must internalize first

Before answering anything, the reviewer must walk the repo and build a mental
map. The relevant entry points in this repo are:

- App package: `jewellery_erpnext/` (Frappe app with two declared modules in
  `jewellery_erpnext/modules.txt`: **Jewellery Erpnext** and
  **Gurukrupa Exports**).
- Frappe wiring: `jewellery_erpnext/hooks.py` — declares `doctype_js`,
  `doctype_list_js`, `doc_events`, `scheduler_events`, `fixtures`,
  `after_migrate`, overrides, etc.
- Migration / patches: `jewellery_erpnext/migrate.py`,
  `jewellery_erpnext/patches.txt`, `jewellery_erpnext/patches/`.
- Server-side business logic:
  - `jewellery_erpnext/jewellery_erpnext/doc_events/` (~25 modules:
    `bom.py`, `sales_order.py`, `quotation.py`, `stock_entry.py`,
    `material_request.py`, `purchase_*.py`, `work_order.py`,
    `job_card.py`, `item.py`, `serial_no.py`, etc.).
  - `jewellery_erpnext/jewellery_erpnext/customization/` (per-DocType
    customization packages: `bom/`, `sales_order/`, `quotation/`,
    `material_request/`, `stock_entry/`, `stock_reconciliation/`,
    `serial_and_batch_bundle/`, `purchase_receipt/`, `sales_invoice/`,
    `stock_ledger_entry/`, `batch/`, `stock/`, `utils/`).
  - Top-level helpers: `jewellery_erpnext/utils.py` (~550 LOC),
    `jewellery_erpnext/erpnext_override.py`, `jewellery_erpnext/query.py`,
    `jewellery_erpnext/interbranch.py`, `jewellery_erpnext/jobs/`.
- DocTypes: `jewellery_erpnext/jewellery_erpnext/doctype/` (≈176 DocTypes,
  including manufacturing-domain DocTypes such as `manufacturing_work_order`,
  `manufacturing_operation`, `parent_manufacturing_order`, `main_slip`,
  `department_ir`, `employee_ir`, `melting_lot`, `custom_refining`,
  `combine_job_card`, plus many child tables ending in `_detail`,
  `_table`, `_multiselect`).
- Second module: `jewellery_erpnext/gurukrupa_exports/` (its own `doctype/`
  and `report/`).
- Reports: `jewellery_erpnext/jewellery_erpnext/report/`
  (`bom_details_against_quotation`, `work_order_status`).
- Client-side: `jewellery_erpnext/public/js/` (form scripts, list scripts,
  CSS overrides referenced from `hooks.py`).
- Fixtures: `jewellery_erpnext/fixtures/` (`custom_field.json`,
  `property_setter.json`, `document_naming_rule.json`,
  `stock_entry_type.json`, `tolerance_weight_type.json`, `doctype.json`).
- Programmatic custom fields / property setters:
  `jewellery_erpnext/jewellery_erpnext/custom_fields/` and
  `jewellery_erpnext/jewellery_erpnext/property_setter/`.
- Tests: only `jewellery_erpnext/jewellery_erpnext/tests/test_bom.py`.
- Tooling: `pyproject.toml`, `.flake8`, `.eslintrc`, `.editorconfig`,
  `scripts/check_max_lines.py`, `requirements.txt`, `setup.py`.

The reviewer should build, at minimum:

1. A DocType inventory (name, module, `is_submittable`, `is_child_table`,
   declared `links`, `field` count, custom vs. standard).
2. A hook map (every entry in `hooks.py` → target callable → file → LOC).
3. A patch map (`patches.txt` → patch module → what schema/data it mutates).
4. A client-script map (`doctype_js`, `doctype_list_js` → JS file →
   triggers/overrides).
5. A fixtures map (every fixture file → the standard DocTypes it mutates).

Without those five inventories, the analysis stays generic. The prompt below
demands them as deliverables.

---

## 1. Prompt — paste this into the agent

> You are a senior Frappe / ERPNext engineer performing a forensic review of
> the custom Frappe application `jewellery_erpnext` located in this
> repository. Your goal is **not** to summarize the README or guess; it is to
> read the code, build evidence, and produce an actionable report. Cite every
> finding with a path and line range (`path/to/file.py:120-148`). If a claim
> cannot be backed by a citation, drop it.
>
> Treat the repository as the single source of truth. Standard Frappe /
> ERPNext behaviour may be referenced from official Frappe docs, but every
> statement about *this* app must come from *this* repo.
>
> Deliver the report in the eight sections below, in order, using the
> structure described. Where a section asks for a table, return a real
> Markdown table.

### Section 1 — Comprehensive Code Review (structure)

1. **Repository map.** Produce a tree (depth ≤ 3) of `jewellery_erpnext/`
   annotated with the *role* of each subfolder (e.g. `doc_events/` =
   server-side hook handlers, `customization/<doctype>/` = override
   controllers, `custom_fields/` = programmatic custom field definitions,
   `fixtures/` = JSON fixtures synced via `bench migrate`, `patches/` =
   one-shot migrations registered in `patches.txt`).
2. **Module decomposition.** From `jewellery_erpnext/modules.txt` and
   each DocType's `module` field in its `*.json`, list every declared
   module and the count of DocTypes per module. Flag any DocType whose
   declared `module` does not exist in `modules.txt`.
3. **DocType inventory.** Build a Markdown table with columns:
   `doctype` | `module` | `custom?` | `is_submittable` | `is_child_table` |
   `naming_rule` | `# fields` | `# Link fields` | `# Table fields` |
   `controller_class?` | `controller_LOC`. Source: each
   `jewellery_erpnext/.../doctype/<name>/<name>.json` and the sibling
   `<name>.py`.
4. **Architecture conformance.** For each component, judge how well it fits
   the Frappe app layout convention
   (`<app>/<module>/doctype/<doctype>/{<doctype>.json,.py,.js}`,
   `<app>/<module>/report/...`, `<app>/<module>/page/...`,
   `<app>/public/...`). Highlight deviations: e.g. business logic placed
   under `customization/` instead of in the DocType controller, helper code
   in top-level `jewellery_erpnext/utils.py` vs. module-scoped utils, or
   the parallel `gurukrupa_exports/` tree under the app root.
5. **Restructuring recommendations.** For every deviation, give a concrete
   move/rename plan (source path → destination path, hooks/imports to
   update, fixtures to re-export).

### Section 2 — Understanding Code Functions

For each Python file under `jewellery_erpnext/jewellery_erpnext/doc_events/`,
`jewellery_erpnext/jewellery_erpnext/customization/`,
`jewellery_erpnext/utils.py`, `erpnext_override.py`, `query.py`,
`interbranch.py`, `migrate.py`, and `jewellery_erpnext/jobs/`:

1. **Function-level catalogue.** Markdown table with columns:
   `file` | `qualified_name` | `kind` (free function / method /
   `@frappe.whitelist` / scheduled job) | `inputs` (with types if hinted) |
   `returns` | `side_effects` (DB writes, doc submits, file writes,
   external HTTP, queue enqueues, `frappe.db.commit`) |
   `called_from` (hook in `hooks.py`, JS via `frappe.call`, another
   server file, `patches.txt`, scheduler, **none found**).
2. **Effectiveness assessment.** For each function, state whether it
   accomplishes its stated/implied purpose and flag:
   - missing input validation,
   - silent `try/except` that swallows errors,
   - implicit `frappe.db.commit()` mid-transaction,
   - N+1 queries inside loops,
   - direct SQL where the ORM would suffice and vice versa.
3. **Frappe API usage audit.** For every server file, list which Frappe
   primitives are used and whether they are used correctly:
   `frappe.get_doc`, `frappe.get_cached_doc`, `frappe.get_all`,
   `frappe.get_list`, `frappe.db.get_value`, `frappe.db.get_list`,
   `frappe.db.sql`, `frappe.qb`, `frappe.throw`, `frappe.msgprint`,
   `frappe.enqueue`, `frappe.publish_realtime`, `frappe.cache()`,
   permission helpers, `@frappe.whitelist`. Call out anti-patterns:
   `frappe.get_doc` inside a tight loop, `frappe.db.sql` with f-string
   interpolation (SQL-injection risk), missing `as_dict=True`, missing
   `ignore_permissions` justification, `frappe.throw` without a
   translatable string, etc.
4. **Hook coverage matrix.** Cross-reference every `doc_events` entry in
   `hooks.py` with the actual function it points to. Flag dangling
   references (file or symbol missing) and orphan handlers (functions in
   `doc_events/` that *no* hook references).

### Section 3 — Unused and Redundant Code

1. **Dead-code scan.** Identify:
   - Python functions/classes/methods that are defined but never imported
     or called anywhere in the repo (search both Python and JS — a
     function may be reached via `frappe.call` from a `.js` file or via
     `hooks.py`/`patches.txt`).
   - Unused imports (cross-check with `flake8` rules in `.flake8`).
   - Commented-out blocks (e.g. the commented `StockEntry` / `WorkOrder`
     overrides at the top of `hooks.py`, and the recent commit
     "Comment out update functions in on_cancel").
   - Duplicate helpers across `utils.py`, `customization/utils/`, and
     individual `doc_events/*` files.
   - DocTypes declared under `doctype/` that are not referenced by any
     Link field, Table field, controller, hook, fixture, report, or JS.
   - Custom fields in `fixtures/custom_field.json` and
     `custom_fields/*.py` that target standard DocTypes but whose
     fieldname is never read by any controller, hook, JS, or report.
   - Patches in `patches.txt` that are already applied everywhere and
     can be retired.
2. **Output format.** A table with columns:
   `path` | `symbol_or_block` | `evidence_of_non_use` |
   `risk_of_removal` (low/medium/high — e.g. low for a private helper
   with no callers, high for something potentially invoked from a client
   script or workflow).
3. **Context per item.** For each entry, write 1-3 sentences on:
   - the *likely* original intent (infer from surrounding code, commit
     history via `git log -p -- <path>`, and naming),
   - what removing it would impact,
   - safer alternatives if you cannot fully prove disuse (e.g. "mark
     `@deprecated` and log on call for one release").
4. **Process recommendations.** Propose guardrails to prevent dead code
   accumulating: pre-commit `ruff`/`flake8` on unused imports, periodic
   `vulture` runs, a CI job that diffs `hooks.py` references against
   actual symbols, code-owner review before adding new top-level
   helpers in `utils.py`.

### Section 4 — Customization and Real Use Analysis

1. **Custom relationship inventory.** From every DocType JSON under
   `jewellery_erpnext/jewellery_erpnext/doctype/` and
   `jewellery_erpnext/gurukrupa_exports/doctype/`, extract:
   - Link fields → table with `(source_doctype, fieldname, target_doctype,
     reqd, in_list_view)`.
   - Table fields (child tables) → `(parent, child_doctype)` and the
     reverse mapping.
   - `Dynamic Link` fields → `(source_doctype, options_field, link_field)`.
   - Implicit many-to-many patterns (a child table whose only purpose is
     to hold a Link to another DocType — e.g. the `*_multiselect` and
     `*_table` DocTypes in this repo).
2. **Custom-field relationships.** From `fixtures/custom_field.json` and
   the programmatic definitions under `custom_fields/`, list every Link
   or Table field added to a *standard* ERPNext DocType (Sales Order,
   Quotation, BOM, Item, Stock Entry, Work Order, Job Card, Material
   Request, Purchase *, Delivery Note, Sales Invoice, Stock
   Reconciliation, Payment Entry, Journal Entry, Manufacturer, Supplier,
   Customer, Operation, Quality Inspection Template). For each, record:
   `target_doctype` | `fieldname` | `fieldtype` | `options` |
   `read_in_code` (which controllers/hooks/JS read it) |
   `written_in_code` (which controllers/hooks/JS write it).
3. **Real-usage assessment.** For each custom relationship and each
   custom DocType, decide:
   - **Actively used** — referenced by at least one controller, hook,
     report, JS file, or workflow.
   - **Only persisted, never read** — written by a hook but never read
     anywhere → candidate for removal.
   - **Only read, never written** — read by a report or JS but never
     populated → likely broken.
   - **Orphaned** — neither read nor written.
   Provide evidence (citations) for each classification.
4. **Justification & impact.** For each *actively used* custom
   relationship, answer:
   - What user workflow does it serve? (Trace from a triggering event
     in `doc_events/` or a button in a `*.js` file to the final
     persisted state.)
   - Does it duplicate functionality already provided by ERPNext
     (e.g. ERPNext's own subcontracting, manufacturing, batch/serial
     tracking)? If yes, justify the duplication or recommend
     consolidation.
   - Performance/usability/maintainability impact: validation cost on
     every save, extra joins in list views, increased
     `before_validate`/`validate` chain length, fixtures bloat.

### Section 5 — Code Performance Optimization

1. **Static performance scan.** For every server file, flag:
   - `frappe.get_doc(...)` or `frappe.get_all(...)` inside `for`/`while`
     loops (N+1).
   - Calls to `frappe.db.sql` that build query strings via `%s`
     concatenation or f-strings (correctness *and* perf risk).
   - Loops that call `.save()` / `.submit()` / `.insert()` per row
     instead of bulk `frappe.db.set_value`/`db_update_all`.
   - Use of `frappe.db.commit()` inside request handlers (breaks the
     single-transaction guarantee and can mask perf bugs).
   - Recursive doc traversals on submittable DocTypes (`Manufacturing
     Work Order`, `Parent Manufacturing Order`, `Department IR`,
     `Employee IR`, `Main Slip`, `Custom Refining`, `Combine Job Card`)
     without batching.
   - Heavy work performed inside `validate`/`before_validate` instead of
     `on_submit`/`on_update_after_submit` or a background job.
   - Missing `frappe.enqueue` for known long-running operations
     (multi-document IR transitions, melting lot postings).
2. **Caching opportunities.** Identify lookups that are repeated per
   request and could use `frappe.cache()`, `frappe.get_cached_doc`,
   `frappe.get_cached_value`, or `@redis_cache` — especially settings
   reads from `Jewellery Settings`, `Manufacturing Setting`,
   `Certification Settings`, `Alloy Settings`.
3. **Index/query review.** From the queries discovered above, propose
   indexes (`add_index` in a patch) on hot columns, and rewrite the
   worst offenders using `frappe.qb` with concrete before/after diffs.
4. **Profiling plan.** Describe how to verify the findings empirically:
   - `bench --site <site> execute jewellery_erpnext.<path>.<fn>`
     wrapped in `cProfile`,
   - Frappe's "Recorder" (`/app/recorder`) for request-level SQL counts,
     enabled around the heaviest workflows (Manufacturing Work Order
     creation, Department IR transition, Stock Entry submission for
     manufacturing operations),
   - `EXPLAIN` on the top 10 slow queries discovered in Recorder.

### Section 6 — Code Consistency and Standards

1. **Linter conformance.** Run (or simulate) the project's own configs:
   `.flake8` (Python), `.eslintrc` (JS), `pyproject.toml` (any `ruff`/
   `black` config), `.editorconfig`, and `scripts/check_max_lines.py`.
   Report counts of violations by rule and the top files by violation
   density.
2. **Naming conventions.** Verify against Frappe conventions:
   - DocType names in Title Case, `name` field of fixtures matches the
     folder name.
   - Python module/file names in `snake_case` and matching the DocType
     they belong to.
   - JS file names matching the DocType (`hooks.py` declares
     `public/js/doctype_js/<doctype>.js` — confirm each path exists).
   - Custom field `fieldname` prefixed (commonly `custom_*`) when added
     to standard DocTypes.
   - Avoid PascalCase Python files, mixed `camelCase` Python identifiers,
     and stray Hindi/Gujarati transliterations in symbol names.
3. **Documentation quality.** For each public function (especially
   anything in `doc_events/`, `customization/`, `utils.py`,
   `erpnext_override.py`, `query.py`, `interbranch.py`, every
   `@frappe.whitelist`, and every scheduled job):
   - Does it have a docstring?
   - Does the docstring describe args, returns, side effects, and the
     hook/event that invokes it?
   - Are README/in-repo docs accurate? (Note that `README.md` is
     currently a stub of ~5 lines.)
   Produce a table `file:function | has_docstring | docstring_quality
   (none/poor/ok/good) | recommended_summary`.
4. **Style fixes.** Provide a short, prioritized patch plan: 10 highest-
   impact lint/style fixes that improve reviewability without changing
   behaviour.

### Section 7 — Integration with Frappe Features

1. **Permissions & roles.** From each DocType JSON's `permissions`
   array and from any code calling `frappe.has_permission`,
   `ignore_permissions=True`, `frappe.set_user`, `frappe.session.user`,
   produce:
   - A role × DocType matrix of CRUD permissions.
   - A list of every `ignore_permissions=True` site, with justification
     (or "missing justification") and the user-impact if abused.
2. **Authentication & API surface.** Enumerate every `@frappe.whitelist`
   in the repo. For each: arguments, whether `allow_guest=True` is set
   (must not be unless explicitly intended), whether it validates input,
   whether it calls `frappe.has_permission` on the documents it mutates.
3. **Validation mechanisms.** Catalogue uses of `frappe.throw`,
   `frappe.msgprint`, `validate_*` helpers, child-table cleanup, and
   `link_doctype` validations. Confirm error messages are translatable
   (`_(...)`) and user-actionable.
4. **Fixtures & overrides safety.** From `hooks.py`:
   - `doctype_js` / `doctype_list_js` overrides — confirm each JS file
     exists under `public/js/...` and does not silently override core
     handlers without calling `frm.trigger` or `super`.
   - `fixtures` / `after_migrate` (`jewellery_erpnext.migrate.after_migrate`)
     — confirm the migration is idempotent and does not blindly
     overwrite user data.
   - `override_doctype_class`, `override_whitelisted_methods`,
     `monkey_patch_*` style code in `erpnext_override.py` — assess
     forward-compatibility with ERPNext upgrades.
5. **Dependencies.** From `requirements.txt`, `setup.py`, and
   `pyproject.toml`, list runtime deps and pin status. Cross-check
   against imports actually used in the code (no unused deps; no
   undeclared deps).

### Section 8 — Code Testing and Quality Assurance

1. **Current coverage.** The repo currently ships only
   `jewellery_erpnext/jewellery_erpnext/tests/test_bom.py`. Read it,
   describe what it actually tests, and compute a rough coverage ratio
   (tested public functions ÷ total public functions catalogued in
   Section 2).
2. **Test gaps.** List the highest-value missing tests, prioritized by
   blast radius:
   - Manufacturing Work Order lifecycle (create → operations →
     department/employee IR → submit → cancel).
   - Stock Entry / Stock Reconciliation customizations (especially the
     manufacturing-driven flows).
   - BOM validations and the BOM detail child tables.
   - Sales Order / Quotation custom hooks.
   - Job Card combine and operation card transitions.
   - Refining and melting lot calculations.
   - Inter-branch stock entry (`interbranch.py`).
   For each, propose a minimal test scaffold using Frappe's
   `FrappeTestCase`, including required fixtures/factories.
3. **CI & QA recommendations.**
   - Add a GitHub Actions workflow that boots a Frappe site (using the
     official `frappe/frappe_docker` images), installs this app, runs
     `bench --site <site> run-tests --app jewellery_erpnext`, and
     enforces lint (`ruff`, `flake8`, `eslint`) and
     `scripts/check_max_lines.py`.
   - Add `pre-commit` config running `ruff`, `eslint`, JSON-sort for
     fixtures, and a hook that re-exports fixtures so they don't drift.
   - Adopt a TDD policy for new DocTypes: every new submittable DocType
     must ship with at least one happy-path and one cancel-path test.
   - Track coverage with `coverage.py` and publish to PR comments.

---

### Output rules the reviewer must follow

- **Cite or it didn't happen.** Every claim references
  `path/to/file.ext:start-end`. Use `git log` for historical claims.
- **Tables over prose** for inventories.
- **No invented APIs.** If unsure whether a Frappe helper exists in the
  version this app targets, say so and check `requirements.txt` /
  ERPNext compatibility.
- **No calendar estimates.** Express effort as scope (files touched,
  schema changes, fixture re-exports, migration patches required) and
  risk (data-loss potential, upgrade fragility), not days/weeks.
- **Actionable closer.** End the report with a single "Top 15 actions,
  ranked by (impact ÷ risk)" list, each item linking back to the
  section that justifies it.

