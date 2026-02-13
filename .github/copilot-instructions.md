# Copilot Instructions — Business Process Catalog (BPC)

## Project Purpose

This repository is a **Microsoft Dynamics 365 Business Process Catalog (BPC) reference** used to guide AI agents performing D365 Finance & Supply Chain Management configuration via MCP (Model Context Protocol). The xlsx files in `bpm reference/` are the authoritative source of business process definitions, deliverables, and delivery plans aligned to the **Success by Design** methodology.

## Repository Structure

- `bpm reference/` — Three Excel workbooks (binary xlsx, not directly readable as text):
  - **Business Process Catalog Tree (Why)** (~5,768 rows, 35 cols) — Hierarchical process catalog: Tree → End-to-end → Process area → Process → Scenario → Test cases. 15 end-to-end processes (e.g., `10.xx` Acquire to dispose, `65.xx` Order to cash, `90.xx` Record to report).
  - **Deliverables (What) Tree** (~7,340 rows, 56 cols) — Configuration, documentation, jobs, migrations, integrations, and workflows organized by functional area (Finance, HR, IT, Operations, Sales).
  - **Success by Design Delivery Plan** (~811 rows, 25 cols) — Workshops, tasks, and documentation across phases: Strategize → Initiate → Implement (Design/Develop/Test) → Prepare → Operate.
- `.vscode/mcp.json` — MCP server connection to a D365 F&O environment (`sca-dev-lbl-2`).

## Reading XLSX Files

These are binary files. To read them programmatically:
1. Download from GitHub raw URL to a local temp directory using `urllib.request`
2. Use `openpyxl` (install via `python -m pip install openpyxl`) with `load_workbook(path, data_only=True)`
3. Key columns vary per file — always read the header row first

## Data Model & Key Columns

### Business Process Catalog (Why)
- **Process Sequence ID** (col B): hierarchical numbering `XX.YY.ZZZ.NNN` — first two digits = end-to-end process
- **Work Item Type** (col D): `End to end` → `Process area` → `Process` → `Scenario` → `Test cases` / `System process`
- **Titles 1-7** (cols E-K): hierarchical depth labels
- **Fit gap status**, **Gap solution approach**: track customization decisions
- **Menu path**, **Menu item name**, **Module**: map processes to D365 navigation

### Deliverables (What)
- **Work Item Type** (col C): `Configuration`, `Documentation`, `Job`, `Migration`, `Integration`, `Workflow`
- **Configuration Type**, **Data package**, **Execution sequence**: for configuration deliverables
- **Interface technology/mode/model**, **Source/Target system/entity**: for integrations
- **Configuration keys**, **Feature management keys**: D365 feature flags

### Delivery Plan
- **Work Item Type** (col B): `Workshop`, `Task`, `Documentation`
- **Agenda**, **Key questions**, **Workshop assumptions**: workshop facilitation content
- Phases in Title 2: Strategize, Initiate, Implement (Design/Develop/Test), Prepare, Operate

## Working with the MCP Server

The `.vscode/mcp.json` connects to a D365 F&O instance. When performing configuration:
1. Look up the relevant process in the **Business Process Catalog** by process sequence ID
2. Find required deliverables in the **Deliverables** tree by matching functional area and module
3. Follow the **Delivery Plan** phase sequence (Strategize → Operate)
4. Use MCP `data_*` tools (OData) for CRUD operations; fall back to `form_*` tools only when needed
5. Always include `cross-company=true` in OData queries and filter by `dataAreaId`

## D365 Configuration Conventions

- The example use case targets a **Danish legal entity** (Larsen Consulting, Copenhagen) — apply Danish legislation and localization
- Configuration spans General Ledger, Accounts Receivable, and Accounts Payable modules minimum
- Use the Deliverables tree's `Configuration` work items as a checklist for required setup steps
- Reference `Menu path` and `Menu item name` columns from the catalog to navigate D365 forms
