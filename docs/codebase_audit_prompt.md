# Codebase Audit Prompt — `jewellery_erpnext`

> **Purpose.** This document is a ready-to-paste prompt for driving a deep,
> evidence-based audit of the `jewellery_erpnext` custom Frappe app
> (and its sibling module `gurukrupa_exports`). It is intentionally specific
> to *this* repository — it names the real modules, hooks, override classes,
> doc_events, scheduler jobs, patches, and conventions that an auditor must
> investigate. Use it with any reasoning model or human reviewer that has
> read access to the repo.
>
> The prompt is structured so that the reviewer must produce *findings backed
> by file paths and line numbers*, not generic advice.

---

## 0. Repository Snapshot (give the reviewer the lay of the land)

The auditor must internalize the following before producing any finding:

- **App name:** `jewellery_erpnext` (Frappe v15 / ERPNext v15 custom app).
- **Top-level package:** `jewellery_erpnext/` containing:
  - `hooks.py` — central wiring (doc_events, doctype_js, override classes, scheduler).
  - `modules.txt` — declares two Frappe modules: **`Jewellery Erpnext`** and **`Gurukrupa Exports`**.
  - `jewellery_erpnext/` — primary module (≈176 DocTypes under `doctype/`, plus `doc_events/`, `customization/`, `custom/`, `custom_fields/`, `report/`, `tests/`).
  - `gurukrupa_exports/` — secondary module (≈35 DocTypes, mostly design / sketch / order workflows).
  - `patches/` — migration patches referenced from `patches.txt`.
  - `jobs/stock_reconciliation_template.py` — scheduled job target.
  - `public/js/doctype_js/*.js` — client overrides registered via `doctype_js`.
  - `fixtures/` — Custom Field, Property Setter, DocType, Document Naming Rule, Stock Entry Type, Tolerance Weight Type fixtures.
  - `interbranch.py`, `query.py`, `utils.py`, `migrate.py`, `erpnext_override.py` — top-level helper / override modules.
- **Standard ERPNext DocTypes the app extends or overrides:**
  Quotation, Sales Order, Sales Invoice, Delivery Note, BOM, Work Order, Job Card,
  Item, Item Attribute, Material Request, Stock Entry, Stock Reconciliation,
  Stock Ledger Entry, Serial and Batch Bundle, Batch, Purchase Order,
  Purchase Receipt, Purchase Invoice, Payment Entry, Unreconcile Payment, Warehouse,
  Diamond/Gemstone Weight, Quality Inspection Template, Manufacturer, Supplier, Customer.
- **Override mechanisms in use** (verify each is justified):
  - `override_doctype_class` for Stock Entry, Stock Reconciliation, Stock Ledger Entry,
    Serial and Batch Bundle (see `hooks.py`).
  - `override_whitelisted_methods` for `make_stock_entry` (Job Card, Material Request),
    `make_stock_in_entry` (Stock Entry), and `submit_cancel_or_update_docs` (Bulk Update).
  - Multi-handler `before_validate` chains on Stock Entry and Sales Invoice
    (one handler from `doc_events/`, another from `customization/`).
- **Scheduler:** A single cron entry running **every minute**:
  `jewellery_erpnext.jobs.stock_reconciliation_template.create_stock_reconciliation`.
  This is a hot spot — review carefully.
- **Tests:** Only one file under `jewellery_erpnext/tests/test_bom.py`.
- **Tooling:** `pyproject.toml` (black, isort, line-length 99), `.flake8`,
  `.eslintrc`, `scripts/check_max_lines.py`. No CI workflow committed.

The reviewer must confirm these facts during the audit and call out drift if
any of them have changed.

---

## 1. The Audit Prompt (paste this verbatim into the reviewer)

You are a senior Frappe / ERPNext engineer performing a forensic, evidence-based
audit of the custom Frappe application **`jewellery_erpnext`**. You have read
access to the full repository. Your goal is to produce a single audit report
that a maintainer can act on without further investigation.

### Operating rules

1. **Cite everything.** Every finding must reference concrete file paths and
   line numbers (e.g. `jewellery_erpnext/hooks.py:163-168`). Do not produce
   generic advice that could apply to any Frappe app.
2. **Prefer evidence over opinion.** When you claim something is "unused",
   "slow", or "dead", show the search that proves it (e.g. "no callers found
   for `X` across `*.py`, `*.js`, `*.json`, `hooks.py`, `patches.txt`,
   `fixtures/*.json`, and DocType `*.json` script fields"). Acknowledge the
   limits of static analysis (Frappe resolves a lot of names dynamically via
   strings in DocType JSON, Server Scripts, Client Scripts, Workflows, Custom
   Fields, Property Setters, and Print Formats).
3. **Respect Frappe v15 idioms.** Compare findings against current Frappe v15 /
   ERPNext v15 conventions (qb / `frappe.qb`, `frappe.db.get_value` with
   `for_update`, `frappe.cache()`, `frappe.enqueue`, hook signatures, the
   Serial and Batch Bundle redesign in v15, the new permission engine, etc.).
4. **Distinguish modules.** Treat **`Jewellery Erpnext`** and
   **`Gurukrupa Exports`** as separate modules and report on them separately
   where relevant. Note any cross-module coupling.
5. **Quantify where possible.** Counts of DocTypes, doc_events handlers,
   functions per file, untested public APIs, queries without indexes, etc.
6. **Be specific about risk.** For each finding tag it as
   `Critical | High | Medium | Low | Nit` and explain the user-visible impact
   (data corruption, wrong stock balance, slow page load, migration failure,
   etc.). Do **not** estimate calendar time; describe technical scope instead
   (which subsystems change, blast radius, migration concerns).

### Deliverable format

Produce a single Markdown report with the following sections **in this order**.
Each section has required sub-checks listed below. If a sub-check is not
applicable, say so explicitly and explain why.

---

#### Section A — Repository map and architectural overview

- A.1 Reconstruct the directory tree of `jewellery_erpnext/` to depth 3 and
  describe the responsibility of each top-level package
  (`doctype/`, `doc_events/`, `customization/`, `custom/`, `custom_fields/`,
  `report/`, `tests/`, `patches/`, `public/`, `fixtures/`, `templates/`,
  `jobs/`, `gurukrupa_exports/`).
- A.2 Diagram (ASCII or Mermaid) the wiring declared in `hooks.py`:
  `doc_events`, `doctype_js`, `doctype_list_js`, `override_doctype_class`,
  `override_whitelisted_methods`, `scheduler_events`, `after_migrate`,
  `app_include_js/css`, `fixtures`. Highlight which standard ERPNext DocTypes
  are touched and through how many distinct handlers.
- A.3 List both Frappe modules declared in `modules.txt` and map every
  DocType under `doctype/` to its module via the DocType JSON `"module"`
  field. Flag DocTypes whose folder location does not match their declared
  module.
- A.4 Identify the two coexisting customization styles
  (`jewellery_erpnext/doc_events/*.py` vs `jewellery_erpnext/customization/<doctype>/*`)
  and explain when each is used. Recommend a single convention.
- A.5 Catalog every duplicated / near-duplicated DocType name
  (e.g. `recoverd_diamond` vs `recovered_diamond`, `recoverd_gemstone` vs
  `recovered_gemstone`, `service_type` vs `service_type_2`,
  `final_sketch_approval___hold`, `designer_assignment___cad`,
  `final_sketch_approval_cmo___rejected`). For each pair, determine which is
  live, which is legacy, and recommend a deprecation path.

#### Section B — `hooks.py` audit

- B.1 For each entry in `doc_events`, verify the target python path resolves
  to an actual function and that the function signature matches what Frappe
  passes (`(doc, method)` for document hooks). Flag missing targets.
- B.2 For multi-handler chains
  (Stock Entry `before_validate`, Sales Invoice `before_validate`,
  Stock Entry `on_submit`), document execution order and whether any handler
  mutates state that the next handler depends on. This is a common source of
  hard-to-debug regressions.
- B.3 For `override_doctype_class` (`CustomStockEntry`, `CustomStockReconciliation`,
  `CustomStockLedgerEntry`, `CustomSerialandBatchBundle`):
  - confirm each subclass actually overrides at least one method,
  - list every overridden / monkey-patched method,
  - assess `super()` discipline (calls to parent `validate`, `on_submit`, etc.),
  - check for v15-specific concerns (Serial and Batch Bundle redesign,
    repost ledger entries, GL entry timing, batch-wise valuation).
- B.4 For `override_whitelisted_methods`, verify the replacement function
  preserves the original signature, `@frappe.whitelist` decorator, permission
  semantics (no broadening of access), and JSON-serializable return types.
  Special attention to `bulk_update.custom_submit_cancel_or_update_docs`
  — confirm it still respects DocPerm and that it cannot be abused to
  bypass workflow states.
- B.5 Audit `scheduler_events`. The cron `* * * * *`
  (`stock_reconciliation_template.create_stock_reconciliation`) runs every
  minute — this is unusual. Determine:
  - average runtime,
  - whether it is idempotent,
  - whether it acquires a lock (`frappe.cache().lock` or equivalent) to
    prevent overlapping runs,
  - whether it should instead be enqueued or moved to `cron(*/5)` / `hourly`.
- B.6 Identify all commented-out blocks in `hooks.py` (there are several,
  including the `app_include_js`/`StockEntry` patches and the
  `GSTTransactionData` override). For each: was it intentionally disabled,
  is it dead, and should it be deleted from version control?

#### Section C — DocType inventory and relationship map

- C.1 Produce a table of every DocType in both modules with columns:
  `name | module | is_child_table | is_submittable | is_tree | autoname |
  has_controller_py | has_client_js | has_workflow | created_by_fixture?`.
  Source the data from each DocType `*.json`.
- C.2 Build the **link graph**: for every `Link` and `Table` field across all
  DocTypes, record `(source_doctype, fieldname, target_doctype)`. Highlight:
  - links pointing to standard ERPNext DocTypes (integration surface),
  - links pointing to deprecated / duplicate DocTypes (Section A.5),
  - orphan DocTypes (no incoming links, no controller logic, no fixtures, no
    JS — strong dead-code candidates),
  - cycles between custom DocTypes.
- C.3 Identify *implicit* relationships not modelled as Link fields:
  Naming Series prefixes, status strings, `frappe.db.get_value` lookups by
  name, and any "linking" performed only inside event handlers. Recommend
  promoting frequently-used implicit links to actual Link fields.
- C.4 For DocTypes with `autoname` set to a server-side function or hook
  (e.g. `Batch` has both `autoname` and `validate` overridden in
  `customization/batch/batch.py`), confirm the autoname function is
  deterministic, collision-safe, and respects existing names on rename.
- C.5 List every Custom Field and Property Setter shipped via fixtures
  (`fixtures/custom_field.json`, `fixtures/property_setter.json`). For each:
  - target standard DocType,
  - whether it is referenced from python or JS in this app,
  - whether it duplicates a field that already exists in v15 ERPNext.

#### Section D — Function-level review of business logic

For each file in `jewellery_erpnext/doc_events/`,
`jewellery_erpnext/customization/**/*.py`, and the top-level
`utils.py`, `query.py`, `interbranch.py`, `erpnext_override.py`,
`migrate.py`, plus `jobs/stock_reconciliation_template.py`:

- D.1 List every function/method with: signature, one-line purpose,
  inputs, outputs, side effects, and the hook(s) that invoke it (if any).
- D.2 Mark each function as `hook-bound | whitelisted API | helper |
  unreachable`. "Unreachable" means no caller in python, JS, hooks, fixtures,
  patches, or DocType JSON. Show the search you ran.
- D.3 Flag functions that:
  - swallow exceptions silently (`except: pass`, bare `except`),
  - use `frappe.db.sql` with f-string / `%s` interpolation that risks SQL
    injection (compare with `frappe.qb` / parameterized queries),
  - perform N+1 queries inside loops over child tables (very common
    anti-pattern in Frappe customizations),
  - call `doc.save()` / `frappe.db.set_value` from inside `validate`
    (causes recursion or stale cache),
  - call `frappe.db.commit()` from inside a document hook (almost always
    wrong; corrupts the surrounding transaction),
  - re-implement utilities already provided by `frappe.utils` or
    `erpnext.stock.utils`.
- D.4 For every `@frappe.whitelist`-decorated function, evaluate:
  - permission check (`frappe.has_permission`, role guard, owner check),
  - input validation (types, allowed values, length),
  - whether it should be `allow_guest=True` (almost certainly not),
  - whether returning ORM docs leaks sensitive fields.
- D.5 Specifically deep-dive these high-risk files (size + business
  criticality):
  - `doc_events/stock_entry.py`,
  - `doc_events/work_order.py`,
  - `doc_events/material_request.py`,
  - `doc_events/sales_order.py`,
  - `doc_events/job_card.py`,
  - `doc_events/bom.py` and `doc_events/bom_utils.py`,
  - `customization/stock_entry/stock_entry.py` (`CustomStockEntry`),
  - `customization/serial_and_batch_bundle/serial_and_batch_bundle.py`,
  - `customization/stock_ledger_entry/stock_ledger_entry.py`,
  - `interbranch.py` (cross-branch stock movement),
  - `utils.py` (general-purpose helpers — usually a magnet for dead code).

#### Section E — Dead / redundant code

- E.1 Produce a ranked list of dead-code candidates with citations.
  Categories to look for:
  - Functions with zero static callers (Section D.2).
  - Imports that are unused in their file (run pyflakes / ruff and dump
    the report).
  - Commented-out code blocks (especially in `hooks.py`,
    `doc_events/*.py`, and `customization/*`).
  - DocTypes without controller logic, without fixtures, without inbound
    Link fields, and with zero rows referenced anywhere in the code
    (Section C.2 orphans).
  - Patches in `patches.txt` whose target column/field/DocType no longer
    exists, or that are guarded by `if frappe.db.exists(...)` and now always
    short-circuit.
  - Public JS files in `public/js/` that are not registered via
    `app_include_js` or `doctype_js` and are not imported from another JS
    file.
  - Reports / Print Formats / Web Forms shipped but never referenced.
- E.2 For each candidate, give the **context of original inclusion**
  (mine git history with `git log --diff-filter=A --follow -- <path>` and
  `git log -p -- <path> | head`) and the **risk of removal**
  (search fixtures and database for runtime references; remember Frappe will
  call functions named in DocType JSON `"controller"` fields, Server Scripts,
  Client Scripts, Workflow Actions, Notifications, and Print Formats).
- E.3 Recommend a process to keep dead code out: pre-commit
  `ruff --select F401,F841`, a CI job that runs `vulture` against
  `jewellery_erpnext/`, and a quarterly DocType usage report driven by
  `tabSingles` / `tabDocType` row counts in production.

#### Section F — Custom relationships and "real use" analysis

- F.1 For each non-standard relationship pattern found in this app
  (multi-select child tables such as `category_multiselect`,
  `operation_multiselect`, `purchase_type_multiselect`,
  `sales_type_multiselect`, `territory_multi_select`,
  `parcel_place_multiselect`, `batch_multiselect`; and "bridge" DocTypes
  such as `manufacturing_plan_sales_order`, `mwo_table`,
  `bom_diamond_detail`, `bom_finding_detail`, `bom_gemstone_detail`,
  `bom_metal_detail`):
  - describe the relationship (M:N, 1:N, polymorphic, denormalized cache),
  - identify the parent DocType(s) and consumers,
  - show one user-facing workflow that depends on it,
  - measure usage if a database is available (`SELECT COUNT(*) FROM
    \`tab<DocType>\``); otherwise mark as "usage unknown — recommend
    production query".
- F.2 Evaluate the **manufacturing pipeline** specifically — it is the
  heart of this app:
  `Manufacturing Plan → Manufacturing Work Order → Parent Manufacturing Order
  → Manufacturing Operation → Department IR → Employee IR → Main Slip →
  Operation Card → Job Card → Stock Entry → Serial and Batch Bundle →
  Stock Ledger Entry`.
  For each hop, document:
  - which DocType triggers the next,
  - which doc_event handler performs the hand-off,
  - what data is copied vs. linked,
  - failure modes when a step is cancelled / amended.
- F.3 Evaluate the **refining / loss tracking** subsystem
  (`refining`, `refining_details`, `refining_department_detail`,
  `refining_operation_details`, `custom_refining*`,
  `metal_loss*`, `operation_metal_loss`, `employee_metal_loss`,
  `workstation_metal_loss`, `manually_book_loss_details`,
  `loss_details`, `loss_type`, `metal_loss_purity`,
  `variant_loss_table`, `variant_loss_warehouse`).
  Identify duplicate concepts and recommend consolidation.
- F.4 Evaluate the **stock reconciliation extensions**
  (`stock_reconciliation_template`, `stock_reconciliation_template_item`,
  `child_stock_reconcilation`, `child_stock_reconcilation_item` —
  note the misspelling, plus `customization/stock_reconciliation/`).
  Confirm the cron job in B.5 only acts on records that need processing
  and that it cannot create reconciliations in a loop.
- F.5 For every "custom relationship" identified above, render a verdict:
  `Used and justified | Used but should be standard ERPNext | Built but
  unused | Built and harmful` with the evidence.

#### Section G — Performance

- G.1 Static query review. For every `frappe.db.sql`, `frappe.db.get_all`,
  `frappe.db.get_list`, `frappe.qb` call:
  - extract the query,
  - identify the columns in `WHERE` / `JOIN` and check if those columns are
    indexed (`search_index` / `unique` flag in DocType JSON, or standard
    Frappe indexes on `name`, `parent`, `parenttype`, `creation`,
    `modified`),
  - flag `SELECT *` / `fields=["*"]` usage,
  - flag missing `limit` on user-facing endpoints,
  - flag queries inside loops (N+1).
- G.2 Hot-path review. List every function called from a `validate` /
  `before_validate` / `on_submit` hook of high-volume DocTypes
  (Stock Entry, Sales Invoice, Job Card, Stock Ledger Entry, Serial and
  Batch Bundle). For each, classify cost
  (`O(1) | O(items) | O(items * batches) | O(N items * M ledger rows)`).
- G.3 Caching opportunities. Identify lookups that are stable per request
  (e.g. company defaults, item attributes, manufacturing settings) and
  recommend `frappe.client_cache` / `frappe.cache()` / request-scope
  memoization.
- G.4 Background-job opportunities. Identify long-running synchronous work
  (bulk reconciliation, IR processing, large BOM rebuilds) that should move
  to `frappe.enqueue` with idempotency keys.
- G.5 Re-evaluate the every-minute cron (B.5). Quantify the cost: how many
  rows does `create_stock_reconciliation` scan per tick?

#### Section H — Coding standards, style, documentation

- H.1 Run `ruff check`, `ruff format --check`, `black --check`, `isort
  --check-only` (line length 99, per `pyproject.toml`) and report counts of
  violations per file. Run `eslint` against `public/js/**/*.js` using the
  committed `.eslintrc`.
- H.2 Use `scripts/check_max_lines.py` to find oversized files and
  recommend splits.
- H.3 Naming. Flag DocType / fieldname inconsistencies
  (`recoverd_*` vs `recovered_*`, `child_stock_reconcilation*` misspelling,
  trailing-underscore variants, `service_type_2`, double-underscore folder
  names from copy/paste).
- H.4 Docstrings & comments. For every public function in
  `doc_events/`, `customization/`, `utils.py`, `query.py`, `interbranch.py`:
  presence of docstring, presence of type hints, presence of inline
  explanation for non-obvious branches. Produce coverage % per file.
- H.5 Error handling. Recommend a single project-wide pattern
  (`frappe.throw` for user-facing errors, `frappe.log_error` for diagnostics,
  custom exception classes inheriting from `frappe.ValidationError` for
  programmatic catching).

#### Section I — Frappe integration & security

- I.1 Permissions. Cross-check every `@frappe.whitelist` function and every
  `override_whitelisted_methods` entry against expected DocPerm /
  Role Permission Manager rules. Identify any function that bypasses
  `frappe.has_permission`.
- I.2 SQL safety. Re-check Section D.3 findings; classify each as
  parameterized / interpolated / unknown.
- I.3 File / attachment handling. If the app accepts uploads or generates
  files (Print Formats, CAD images, sketches in `gurukrupa_exports`),
  confirm `is_private`, MIME validation, and that filenames are not
  user-controlled paths.
- I.4 Workflow. Verify the workflows installed by
  `patches/add_ir_workflows.py` and the IR-status-update patches still
  match the current DocType states; flag drift.
- I.5 Fixtures hygiene. Custom Fields and Property Setters shipped via
  fixtures must round-trip cleanly (`bench export-fixtures` produces no diff
  on a fresh install). Identify fixtures with environment-specific values
  (UUIDs, branch names, account names).
- I.6 Third-party / sibling-app dependencies. The commented-out
  `gst_india` override in `hooks.py` and the `erpnext_override.py` file
  hint at past coupling. Document current runtime dependencies on `frappe`,
  `erpnext`, and any other apps; pin them in `pyproject.toml`
  (`tool.bench.frappe-dependencies` already pins `frappe>=15,<16`; do the
  same for `erpnext` and any others).

#### Section J — Tests & QA

- J.1 Coverage baseline. There is currently a single test file
  (`jewellery_erpnext/tests/test_bom.py`). Report what it actually covers
  and its pass/fail status under
  `bench --site <site> run-tests --app jewellery_erpnext`.
- J.2 Risk-ranked test backlog. List the ten most critical untested code
  paths (likely candidates: stock entry validation, serial-and-batch bundle
  override, scheduler reconciliation job, interbranch transfer, BOM event
  chain, material-request → stock-entry conversion, work-order completion,
  refining loss accounting, sales-invoice before_validate chain,
  whitelisted overrides). For each, sketch a minimal unit / integration
  test using `FrappeTestCase`.
- J.3 Fixture-based test data. Recommend a `tests/fixtures/` layout that
  seeds the minimum set of Items, Warehouses, Customers, BOMs needed for
  the manufacturing flow.
- J.4 CI proposal. There is no `.github/workflows/` directory. Propose a
  minimal GitHub Actions matrix that:
  - sets up a Frappe v15 bench,
  - installs this app,
  - runs `ruff`, `black --check`, `isort --check-only`, `eslint`, and
    `bench run-tests --app jewellery_erpnext`,
  - publishes coverage.
- J.5 Manual QA checklist. Provide a one-page checklist a release engineer
  can run before each deploy, focused on the riskiest user flows
  (creating a Sales Order → Manufacturing Plan → Work Order → Job Card →
  Stock Entry → Stock Reconciliation cycle, plus interbranch transfer and
  refining).

#### Section K — Prioritized action plan

Conclude with a single table of every recommendation gathered above, sorted by
risk × scope, with columns:

| # | Severity | Area | Recommendation | Files affected | Migration / data risk | Verification step |

Do not estimate calendar time. Use technical scope language
(e.g. "isolated helper change", "touches Stock Ledger posting — requires
repost test on staging", "DocType rename — requires patch + fixture export").

---

## 2. How to drive the audit

Run the prompt above against the repository in a fresh review session. The
reviewer should be allowed to:

- read any file,
- run `git log` / `git blame`,
- run static analysis (`ruff`, `vulture`, `pyflakes`, `eslint`,
  `scripts/check_max_lines.py`),
- run the existing test suite,
- *not* mutate the database or push branches.

If a live site is available, additionally allow read-only SQL against
`tab*` tables to answer "is this DocType actually used?" questions in
Sections C, E, and F.

## 3. Definition of done

The audit is complete when:

1. Every section A–K is filled with concrete, cited findings.
2. Every recommendation in Section K maps back to at least one citation in
   Sections A–J.
3. The dead-code list in Section E is reproducible from the cited searches.
4. The performance findings in Section G include either measured numbers
   (when a profiling environment is available) or a clearly stated
   assumption set.
5. The test backlog in Section J.2 is small enough to be executed
   incrementally without a single mega-PR.
