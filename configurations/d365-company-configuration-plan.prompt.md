# D365 Finance & SCM — New Company Configuration Plan

Reusable configuration plan for provisioning a new legal entity in Dynamics 365 Finance and Supply Chain Management with General Ledger, Accounts Receivable, and Accounts Payable modules. Derived from the LARC (Larsen Consulting) implementation.

## Variables

Define these before execution. All steps below reference them by `{{name}}`.

| Variable | Description | Example (LARC) |
|----------|-------------|-----------------|
| `{{ENTITY_ID}}` | 4-character legal entity / company ID | `LARC` |
| `{{COMPANY_NAME}}` | Full company name | `Larsen Consulting` |
| `{{NAME_ALIAS}}` | Short name / search name (max 20 chars) | `Larsen Consulting` |
| `{{COUNTRY_CODE}}` | ISO 3166-1 alpha-3 country code | `DNK` |
| `{{COUNTRY_ISO2}}` | ISO 3166-1 alpha-2 country code | `DK` |
| `{{CITY}}` | Primary address city | `Copenhagen` |
| `{{STREET}}` | Primary address street | `Vesterbrogade 10` |
| `{{ZIP}}` | Postal code | `1620` |
| `{{TIMEZONE}}` | D365 timezone enum value | `GMTPLUS0100BRUSSELS_COPENHAGEN_MADRID` |
| `{{LANGUAGE}}` | Language code | `da` |
| `{{CURRENCY}}` | ISO 4217 accounting currency | `DKK` |
| `{{CURRENCY_NAME}}` | Currency display name | `Danish Krone` |
| `{{FISCAL_CALENDAR}}` | Name of existing fiscal calendar to use | `Fiscal` |
| `{{COA_NAME}}` | Chart of Accounts identifier | `Danish` |
| `{{COA_DESC}}` | Chart of Accounts description | `Danish Statutory Chart of Accounts` |
| `{{ACCT_STRUCTURE_BS}}` | Balance sheet account structure | `Service Industries B/S` |
| `{{ACCT_STRUCTURE_PL}}` | Profit & loss account structure | `Service Industries P&L` |
| `{{VAT_STANDARD_RATE}}` | Standard VAT / sales tax rate (%) | `25` |
| `{{VAT_STANDARD_CODE}}` | Tax code for standard rate | `VAT25` |
| `{{VAT_ZERO_CODE}}` | Tax code for zero rate | `VAT0` |
| `{{VAT_EU_CODE}}` | Tax code for EU reverse charge | `VATEU` |
| `{{VAT_REG_NUMBER}}` | Company VAT registration number | *(set per company)* |

---

## Phase 1: Foundation Setup

### 1.1 Create or Update the Legal Entity

**Tool:** `data_create_entities` or `data_update_entities` on `LegalEntities`  
**Key:** `LegalEntityId` = `{{ENTITY_ID}}`

| Field | Value |
|-------|-------|
| `LegalEntityId` | `{{ENTITY_ID}}` |
| `Name` | `{{COMPANY_NAME}}` |
| `CompanyName` | `{{COMPANY_NAME}}` |
| `NameAlias` | `{{NAME_ALIAS}}` |
| `AddressCountryRegionId` | `{{COUNTRY_CODE}}` |
| `AddressCountryRegionISOCode` | `{{COUNTRY_ISO2}}` |
| `AddressCity` | `{{CITY}}` |
| `AddressStreet` | `{{STREET}}` |
| `AddressZipCode` | `{{ZIP}}` |
| `AddressDescription` | `Headquarters` |
| `AddressLocationRoles` | `Business` |
| `TimeZone` | `{{TIMEZONE}}` |
| `LanguageId` | `{{LANGUAGE}}` |

**Validation:** Query `LegalEntities` filtered by `LegalEntityId eq '{{ENTITY_ID}}'` — confirm `FullPrimaryAddress` includes street, zip, city, country.

### 1.2 Verify Accounting Currency Exists

**Tool:** `data_find_entities` on `Currencies`  
**Filter:** `CurrencyCode eq '{{CURRENCY}}'`

If the currency does not exist, create it:

| Field | Value |
|-------|-------|
| `CurrencyCode` | `{{CURRENCY}}` |
| `Name` | `{{CURRENCY_NAME}}` |
| `Symbol` | *(e.g., `kr` for DKK)* |
| `GeneralRoundingRule` | `0.01` |

### 1.3 Verify Fiscal Calendar Coverage

**Tool:** `data_find_entities` on `FiscalCalendarYears`  
**Filter:** `Calendar eq '{{FISCAL_CALENDAR}}'`

Confirm the calendar has years covering the current year and at least one year forward. If missing years exist, create them via:

**Tool:** `data_create_entities` on `FiscalCalendarYears`

| Field | Value |
|-------|-------|
| `Calendar` | `{{FISCAL_CALENDAR}}` |
| `FiscalYear` | *(e.g., `2027`)* |
| `StartDate` | `2027-01-01T12:00:00Z` |
| `EndDate` | `2027-12-31T12:00:00Z` |
| `Description` | `Fiscal year 2027` |

> **Note:** Fiscal calendar periods within the year must also exist. If created via OData, verify periods using `FiscalCalendars` entity. Alternatively, use the `FiscalCalendars` form UI to auto-generate periods.

---

## Phase 2: Chart of Accounts & Main Accounts

### 2.1 Create Chart of Accounts

**Tool:** `data_create_entities` on `ChartOfAccounts`

| Field | Value |
|-------|-------|
| `ChartOfAccounts` | `{{COA_NAME}}` |
| `Description` | `{{COA_DESC}}` |

### 2.2 Create Main Accounts

**Tool:** `data_create_entities` on `MainAccounts`  
**Required fields:** `MainAccountId`, `Name`, `ChartOfAccounts`, `Type` (enum: `ProfitAndLoss`, `Revenue`, `Expense`, `BalanceSheet`, `Total`)

> Get full metadata with `data_get_entity_metadata` on `MainAccounts` including `includeEnumValues=true` before creating.

Minimum account plan (adapt numbering to local statutory requirements):

#### Assets (BalanceSheet)

| Account | Name | Purpose |
|---------|------|---------|
| 110100 | Bank Account — {{CURRENCY}} | Primary bank |
| 110200 | Petty Cash | Cash on hand |
| 130100 | Accounts Receivable — Domestic | AR trade |
| 130110 | Accounts Receivable — Foreign | AR foreign trade |
| 130200 | Allowance for Doubtful Accounts | AR provision |
| 130700 | Other Receivables | Misc receivables |
| 130800 | VAT Receivable | Input VAT holding |
| 140100 | Inventory | Stock (if applicable) |
| 180100 | Fixed Assets | Tangible assets |
| 180200 | Accum. Depreciation — Fixed Assets | Contra-asset |

#### Liabilities (BalanceSheet)

| Account | Name | Purpose |
|---------|------|---------|
| 200100 | Accounts Payable — Domestic | AP trade |
| 200110 | Accounts Payable — Foreign | AP foreign trade |
| 200200 | Accrued Liabilities | Accruals |
| 211000 | Input VAT (Indgående moms) | Deductible VAT on purchases |
| 212000 | Output VAT (Udgående moms) | VAT collected on sales |
| 213000 | VAT Settlement | Net VAT payable/receivable |
| 220100 | Payroll Liabilities | Salary accruals |

#### Equity (BalanceSheet)

| Account | Name | Purpose |
|---------|------|---------|
| 300100 | Share Capital | Equity capital |
| 310100 | Retained Earnings | P&L accumulation |
| 320100 | Current Year Earnings | Year-end close target |

#### Revenue (Revenue)

| Account | Name | Purpose |
|---------|------|---------|
| 400100 | Sales Revenue | Product sales |
| 410100 | Service Revenue | Consulting / services |
| 420100 | Other Income | Misc income |
| 430100 | Cash Discount Given | AR discounts |
| 801000 | Realized Exchange Gain | FX realized gain |
| 802000 | Unrealized Exchange Gain | FX unrealized gain |

#### Expenses (Expense)

| Account | Name | Purpose |
|---------|------|---------|
| 500100 | Cost of Goods Sold | COGS |
| 600100 | Operating Expenses | General OpEx |
| 610100 | Salaries & Wages | Payroll cost |
| 620100 | Rent Expense | Office rent |
| 630100 | Travel Expense | Business travel |
| 640100 | Office Supplies | Consumables |
| 700100 | Depreciation Expense | FA depreciation |
| 710100 | Interest Expense | Finance costs |
| 720100 | Cash Discount Received | AP discounts |
| 803000 | Realized Exchange Loss | FX realized loss |
| 804000 | Unrealized Exchange Loss | FX unrealized loss |
| 900100 | Penny Difference | GL rounding |

#### Totals (Total)

| Account | Name | Purpose |
|---------|------|---------|
| 199999 | TOTAL ASSETS | Summing account |
| 299999 | TOTAL LIABILITIES | Summing account |
| 399999 | TOTAL EQUITY | Summing account |
| 499999 | TOTAL REVENUE | Summing account |
| 999999 | TOTAL EXPENSES | Summing account |

### 2.3 Verify Account Structures

**Tool:** `data_find_entities` on `AccountStructures`  
Confirm `{{ACCT_STRUCTURE_BS}}` and `{{ACCT_STRUCTURE_PL}}` exist and are `Status` = `Active`.

If custom structures are needed, create via form tools (`DimensionConfigureAccountStructure` menu item) — OData creation of account structures is limited.

---

## Phase 3: Ledger Configuration

### 3.1 Configure the Ledger

**Tool:** `data_create_entities` or `data_update_entities` on `Ledgers`  
**Key:** `LegalEntityId` = `{{ENTITY_ID}}`

| Field | Value |
|-------|-------|
| `LegalEntityId` | `{{ENTITY_ID}}` |
| `ChartOfAccounts` | `{{COA_NAME}}` |
| `AccountingCurrency` | `{{CURRENCY}}` |
| `FiscalCalendar` | `{{FISCAL_CALENDAR}}` |
| `AccountStructureName1` | `{{ACCT_STRUCTURE_BS}}` |
| `AccountStructureName2` | `{{ACCT_STRUCTURE_PL}}` |
| `MainAccountIdRealizedGain` | `801000` |
| `MainAccountIdRealizedLoss` | `803000` |
| `MainAccountIdUnrealizedGain` | `802000` |
| `MainAccountIdUnrealizedLoss` | `804000` |

**Validation:** Query `Ledgers` filtered by `LegalEntityId eq '{{ENTITY_ID}}'` — confirm all fields.

### 3.2 Configure General Ledger Parameters

**Tool:** `form_open_menu_item` → `LedgerParameters` (Display) in company `{{ENTITY_ID}}`

Set on the **Ledger** tab:
- Accounting currency displayed = `{{CURRENCY}}`
- Penny difference debit/credit accounts = `900100`

Set on the **Sales tax** tab:
- Sales tax calculation method = Whole amount (or Per line, depending on business requirements)
- Apply sales tax taxation rules = Yes

**Save the form.**

---

## Phase 4: Tax Configuration

### 4.1 Create Sales Tax Settlement Period

**Tool:** `form_open_menu_item` → `TaxPeriod` (Display) in company `{{ENTITY_ID}}`

| Field | Value |
|-------|-------|
| Settlement period ID | `DK_MTH` (or `DK_QTR` for quarterly) |
| Description | `Danish VAT Monthly` |
| Tax authority | *(create or assign)* |
| Period interval | 1 Month |

Create period intervals covering at least the current fiscal year.

### 4.2 Create Sales Tax Ledger Posting Groups

**Tool:** Find entity type `TaxLedgerAccountGroup` or use form `TaxAccountGroup` (Display) in `{{ENTITY_ID}}`

| Posting Group | Sales Tax Payable | Sales Tax Receivable | Settlement |
|---------------|-------------------|----------------------|------------|
| `DK_VAT` | `212000` | `211000` | `213000` |

### 4.3 Create Sales Tax Codes

**Tool:** `data_create_entities` on `TaxCodes` in `{{ENTITY_ID}}`

| Field | `{{VAT_STANDARD_CODE}}` | `{{VAT_ZERO_CODE}}` | `{{VAT_EU_CODE}}` |
|-------|------------------------|--------------------|--------------------|
| `TaxCode` | `VAT25` | `VAT0` | `VATEU` |
| `TaxName` | `Danish VAT 25%` | `Zero-rated VAT` | `EU Reverse Charge` |
| `TaxCurrencyCodeId` | `{{CURRENCY}}` | `{{CURRENCY}}` | `{{CURRENCY}}` |
| `TaxPostingGroupId` | `DK_VAT` | `DK_VAT` | `DK_VAT` |
| `TaxPeriodId` | `DK_MTH` | `DK_MTH` | `DK_MTH` |
| `TaxBase` | `PercentOfNetAmount` | `PercentOfNetAmount` | `PercentOfNetAmount` |
| `TaxCountryRegionType` | `Domestic` | `Domestic` | `EU` |

### 4.4 Set Tax Code Rates

**Tool:** `form_open_menu_item` → `TaxTable` (Display) in `{{ENTITY_ID}}`  
Navigate to each tax code → **Values** button:

| Tax Code | From Date | Rate (%) |
|----------|-----------|----------|
| `{{VAT_STANDARD_CODE}}` | `2025-01-01` | `{{VAT_STANDARD_RATE}}` |
| `{{VAT_ZERO_CODE}}` | `2025-01-01` | `0` |
| `{{VAT_EU_CODE}}` | `2025-01-01` | `{{VAT_STANDARD_RATE}}` |

### 4.5 Create Sales Tax Groups

**Tool:** `data_create_entities` on `TaxGroups` in `{{ENTITY_ID}}`

| TaxGroupCode | Description |
|--------------|-------------|
| `DK_DOM` | Domestic trade |
| `DK_EU` | EU intracommunity trade |
| `DK_EXP` | Export (outside EU) |

Then add tax codes to groups via `TaxGroupDatas` entity or `TaxGroupTable` form:

| Tax Group | Tax Code |
|-----------|----------|
| `DK_DOM` | `{{VAT_STANDARD_CODE}}` |
| `DK_EU` | `{{VAT_EU_CODE}}` |
| `DK_EXP` | `{{VAT_ZERO_CODE}}` |

### 4.6 Create Sales Tax Item Groups

**Tool:** `data_create_entities` on `TaxItemGroups` in `{{ENTITY_ID}}`

| TaxItemGroupCode | TaxCodeId | Description |
|------------------|-----------|-------------|
| `FULL` | `{{VAT_STANDARD_CODE}}` | Full VAT |
| `ZERO` | `{{VAT_ZERO_CODE}}` | Zero-rated |

---

## Phase 5: Accounts Receivable Configuration

### 5.1 Create Customer Groups

**Tool:** `data_create_entities` on `CustomerGroups` in `{{ENTITY_ID}}`

| CustomerGroupId | Description | PaymentTermId | TaxGroupId |
|-----------------|-------------|---------------|------------|
| `DOM` | Domestic customers | `Net30` | `DK_DOM` |
| `EU` | EU customers | `Net30` | `DK_EU` |
| `EXP` | Export customers | `Net30` | `DK_EXP` |

### 5.2 Create Customer Posting Profiles

**Tool:** `data_create_entities` on `CustomerPostingProfiles`

| PostingProfile | Description |
|----------------|-------------|
| `GEN` | General customer posting |

**Tool:** `data_create_entities` on `CustomerPostingProfileLines`

| PostingProfile | AccountCode | SummaryAccount | CashDiscountAccount |
|----------------|-------------|----------------|---------------------|
| `GEN` | `All` | `130100` | `430100` |

### 5.3 Create Payment Terms

**Tool:** `data_create_entities` on `PaymentTerms` (shared, not company-specific)

> Check if they already exist first. Payment terms are cross-company.

| PaymentTermId | Description | PaymentMethod | NumberOfDays |
|---------------|-------------|---------------|--------------|
| `Net14` | Net 14 days | `Net` | `14` |
| `Net30` | Net 30 days | `Net` | `30` |
| `COD` | Cash on delivery | `COD` | `0` |

### 5.4 Create Customer Payment Methods

**Tool:** `data_create_entities` on `CustomerPaymentMethods` in `{{ENTITY_ID}}`

| Name | Description | PaymentType | BridgingPostingEnabled | BridgingAccount |
|------|-------------|-------------|------------------------|-----------------|
| `BANK` | Bank transfer | `Ledger` | `No` | — |
| `CASH` | Cash payment | `Ledger` | `No` | — |

### 5.5 Configure AR Parameters

**Tool:** `form_open_menu_item` → `CustParameters` (Display) in company `{{ENTITY_ID}}`

- **General** tab: Default payment terms = `Net30`
- **Posting** tab: Default posting profile = `GEN`
- **Number sequences** tab: Assign sequences for Customer account, Free text invoice, Payment journal, etc.

**Save the form.**

---

## Phase 6: Accounts Payable Configuration

### 6.1 Create Vendor Groups

**Tool:** `data_create_entities` on `VendorGroups` in `{{ENTITY_ID}}`

| VendorGroupId | Description | PaymentTermId | TaxGroupId |
|---------------|-------------|---------------|------------|
| `DOM` | Domestic vendors | `Net30` | `DK_DOM` |
| `EU` | EU vendors | `Net30` | `DK_EU` |
| `EXP` | Foreign vendors | `Net30` | `DK_EXP` |

### 6.2 Create Vendor Posting Profiles

**Tool:** `data_create_entities` on `PostingProfileHeaders` (Vendor posting profile header)

| PostingProfileId | Description |
|------------------|-------------|
| `GEN` | General vendor posting |

**Tool:** `data_create_entities` on `PostingProfileLines` (Vendor posting profile lines)

| PostingProfileId | AccountCode | SummaryAccount | CashDiscountAccount |
|------------------|-------------|----------------|---------------------|
| `GEN` | `All` | `200100` | `720100` |

### 6.3 Create Vendor Payment Methods

**Tool:** `data_create_entities` on `VendorPaymentMethods` in `{{ENTITY_ID}}`

| Name | Description | PaymentType |
|------|-------------|-------------|
| `BANK` | Bank transfer | `Ledger` |
| `EFT` | Electronic funds transfer | `Electronic` |

### 6.4 Configure AP Parameters

**Tool:** `form_open_menu_item` → `VendParameters` (Display) in company `{{ENTITY_ID}}`

- **General** tab: Default payment terms = `Net30`
- **Invoice** tab: Default posting profile = `GEN`
- **Invoice validation**: Three-way matching = Optional (set based on business needs)
- **Number sequences** tab: Assign sequences for Vendor account, Invoice register, Payment journal, etc.

**Save the form.**

---

## Phase 7: Number Sequences & Journal Names

### 7.1 Generate Number Sequences

**Tool:** `form_open_menu_item` → `NumberSequenceTableListPage` (Display) in `{{ENTITY_ID}}`  
Or use the Number Sequence Wizard: `form_open_menu_item` → `NumberSequenceWizard` (Action)

Generate number sequences for all areas — minimum coverage:

| Area | Reference | Format Example | Continuous |
|------|-----------|----------------|------------|
| General ledger | Journal batch number | `LARC-GL-######` | No |
| General ledger | Voucher | `LARC-V-######` | Yes |
| Accounts receivable | Customer account | `LARC-C-####` | No |
| Accounts receivable | Free text invoice | `LARC-FTI-######` | No |
| Accounts payable | Vendor account | `LARC-V-####` | No |
| Accounts payable | Invoice number | `LARC-INV-######` | No |

> Ensure sequences are scoped to `Legal entity` = `{{ENTITY_ID}}`. After generation, assign them on the Number sequences tabs of `LedgerParameters`, `CustParameters`, and `VendParameters`.

### 7.2 Create Journal Names

**Tool:** `data_create_entities` on `JournalNames` in `{{ENTITY_ID}}`

> Get metadata first: `data_get_entity_metadata` on `JournalNames` (include keys + field constraints).

| JournalNameId | Description | JournalType | VoucherSeries |
|---------------|-------------|-------------|---------------|
| `GenJrn` | General journal | `Daily` | *(assign voucher number sequence)* |
| `CustPay` | Customer payment journal | `CustPayment` | *(assign voucher number sequence)* |
| `VendPay` | Vendor payment journal | `VendPayment` | *(assign voucher number sequence)* |
| `VendInv` | Vendor invoice journal | `VendInvoiceRegister` | *(assign voucher number sequence)* |

---

## Phase 8: Localization & Compliance

### 8.1 Set VAT Registration Number

**Tool:** `data_update_entities` on `LegalEntities`

| Field | Value |
|-------|-------|
| `VATNum` | `{{VAT_REG_NUMBER}}` |

### 8.2 EU Sales List Configuration

**Tool:** `form_open_menu_item` → `EUSalesList` (Display) in `{{ENTITY_ID}}`

Ensure each sales tax group used for EU trade (`DK_EU`) has `EUSalesListType` configured on its tax item group lines.

### 8.3 Intrastat Configuration (EU countries)

**Tool:** `form_open_menu_item` → `ForeignTradeParameters` or `IntrastatParameters` (Display) in `{{ENTITY_ID}}`

- Set default commodity codes, transaction codes, and transport methods if the company trades goods within the EU.

### 8.4 Electronic Payment Formats (Optional)

For Denmark-specific payment formats (NETS, Betalingsservice, SEPA):
- Import Electronic Reporting (ER) configurations from LCS/Dataverse
- Assign ER formats to vendor/customer payment methods

---

## Verification Checklist

Run these checks after completing all phases:

| # | Check | How |
|---|-------|-----|
| 1 | Legal entity details | Query `LegalEntities` for `{{ENTITY_ID}}` — verify name, address, country, timezone |
| 2 | Ledger configuration | Query `Ledgers` for `{{ENTITY_ID}}` — verify CoA, currency, fiscal calendar, structures |
| 3 | Main accounts visible | Open `MainAccount` form in `{{ENTITY_ID}}` — confirm full account list |
| 4 | Tax codes with rates | Open `TaxTable` form — verify `{{VAT_STANDARD_CODE}}` = `{{VAT_STANDARD_RATE}}`%, `{{VAT_ZERO_CODE}}` = 0% |
| 5 | Tax groups complete | Query `TaxGroups` — verify `DK_DOM`, `DK_EU`, `DK_EXP` exist with correct tax codes |
| 6 | Customer posting | Create a test free text invoice → post → verify GL entries hit `130100` (AR) and `212000` (Output VAT) |
| 7 | Vendor posting | Create a test vendor invoice → post → verify GL entries hit `200100` (AP) and `211000` (Input VAT) |
| 8 | Number sequences | Open `NumberSequenceTableListPage` → confirm sequences generate correctly |
| 9 | Journal names | Open `LedgerJournalTable` → create a new journal → verify journal names appear |
| 10 | VAT registration | Query `LegalEntities` → confirm `VATNum` is populated |

---

## Implementation Status

Track progress per phase:

| Phase | Description | Status |
|-------|-------------|--------|
| **1** | Foundation Setup (Legal Entity, Currency, Fiscal Calendar) | **Completed** |
| **2** | Chart of Accounts & Main Accounts | Not started |
| **3** | Ledger Configuration | Not started |
| **4** | Tax Configuration (Danish VAT / Moms) | Not started |
| **5** | Accounts Receivable | Not started |
| **6** | Accounts Payable | Not started |
| **7** | Number Sequences & Journal Names | Not started |
| **8** | Localization & Compliance | Not started |

---

## Appendix: Tool Reference

| Tool | When to Use |
|------|-------------|
| `data_find_entity_type` | Discover OData entity set names |
| `data_get_entity_metadata` | Get field names, types, keys, enums before CRUD |
| `data_find_entities` | Read / query records (always use `cross-company=true`) |
| `data_create_entities` | Create new records via OData |
| `data_update_entities` | Update existing records via OData |
| `form_open_menu_item` | Open a D365 form (use for parameters, wizards, complex setup) |
| `form_set_control_values` | Set field values on an open form |
| `form_click_control` | Click buttons (New, Save, etc.) |
| `form_save_form` | Save the current form |
| `form_close_form` | Close the current form |

> **Rule:** Prefer `data_*` tools over `form_*` tools. Use forms only when OData doesn't support the operation (e.g., number sequence wizard, parameter forms, tax code value entry).
