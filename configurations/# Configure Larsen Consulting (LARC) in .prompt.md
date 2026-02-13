# Configure Larsen Consulting (LARC) in D365 F&O

A new Danish legal entity **LARC** (Larsen Consulting, Copenhagen) will be created with a dedicated Danish Chart of Accounts, DKK as accounting currency, Danish VAT (25% moms), and full GL/AR/AP module configuration. The approach uses OData `data_*` tools as the primary method, falling back to `form_*` tools only where OData doesn't support the operation (e.g., number sequence generation, ledger parameter forms).

## Phase 1: Foundation Setup

1. **Create the Legal Entity** — Use `LegalEntities` OData entity to create LARC with `CompanyName` = "Larsen Consulting", `AddressCountryRegionId` = "DNK", `AddressCity` = "Copenhagen", `AddressCountryRegionISOCode` = "DK", and relevant address fields.

2. **Ensure DKK currency exists** — Already confirmed: `CurrencyCode` = "DKK" (Danish Krone) exists in the system. No action needed.

3. **Verify Fiscal Calendar** — The "Fiscal" calendar exists (from 2014). Verify it covers 2025/2026 periods. If not, create new fiscal year/periods via `FiscalCalendarYears` or the Fiscal calendar form.

## Phase 2: Chart of Accounts & Account Structures

4. **Create "Danish" Chart of Accounts** — Use `ChartOfAccounts` entity to create a new CoA with `ChartOfAccounts` = "Danish", `Description` = "Danish Statutory Chart of Accounts".

5. **Create Main Accounts** — Use `MainAccounts` entity to create a Danish-standard account structure. Minimum accounts:
   - **Assets**: 1100xx (Cash/Bank DKK), 1300xx (Accounts Receivable), 1400xx (Inventory), 1800xx (Fixed Assets)
   - **Liabilities**: 2000xx (Accounts Payable), 2100xx (VAT Payable/Receivable), 2200xx (Accrued Liabilities)
   - **Equity**: 3000xx (Share Capital), 3100xx (Retained Earnings)
   - **Revenue**: 4000xx (Sales Revenue), 4100xx (Service Revenue)
   - **Expense**: 5000xx (COGS), 6000xx (Operating Expenses), 6100xx (Salaries), 7000xx (Depreciation)
   - **Tax-specific**: 2110xx (Input VAT / Indgående moms), 2120xx (Output VAT / Udgående moms), 2130xx (VAT Settlement)
   - **Currency**: Unrealized/realized gain/loss accounts for foreign exchange

6. **Assign Account Structures** — Use the existing "Service Industries B/S" and "Service Industries P&L" structures (MainAccount > BusinessUnit > Department > Project > ServiceLine). These are already active and fit a consulting firm. They will be linked to the LARC ledger.

## Phase 3: Ledger Configuration

7. **Configure Ledger** — Use `Ledgers` entity to set up LARC's ledger:
   - `ChartOfAccounts` = "Danish"
   - `AccountingCurrency` = "DKK"
   - `FiscalCalendar` = "Fiscal"
   - `AccountStructureName1` = "Service Industries B/S"
   - `AccountStructureName2` = "Service Industries P&L"
   - Assign exchange rate gain/loss main accounts

8. **Configure General Ledger Parameters** — Use form tools to open GL Parameters (menu item: `LedgerParameters`) in company LARC and set:
   - Default journal names
   - Sales tax calculation settings
   - Fiscal year close settings
   - Penny difference accounts

## Phase 4: Danish Tax (Moms) Configuration

9. **Create Sales Tax Codes** — Use `TaxCodes` entity in LARC:
   - `VAT25` — Standard Danish VAT 25% (Moms) — `TaxCalculationMethod` = Percentage
   - `VAT0` — Zero-rated (exports, exempt supplies)
   - `VATEU` — EU reverse charge (intracommunity)

10. **Create Tax Code Values/Rates** — Use form tools (`TaxTable` menu) to set the rate of 25% for `VAT25`, effective from the company start date.

11. **Create Sales Tax Ledger Posting Groups** — Map tax codes to GL accounts (Input VAT → 2110xx, Output VAT → 2120xx, Settlement → 2130xx).

12. **Create Sales Tax Groups** — Use `TaxGroups` entity:
    - `DK_DOM` — Domestic (contains VAT25)
    - `DK_EU` — EU trade (contains VATEU)
    - `DK_EXP` — Export (contains VAT0)

13. **Create Sales Tax Item Groups** — Use `TaxItemGroups` entity:
    - `FULL` — Full tax (linked to VAT25)
    - `ZERO` — Zero-rated (linked to VAT0)

14. **Create Tax Settlement Period** — Via form tools, set up a monthly or quarterly VAT settlement period for Danish reporting.

## Phase 5: Accounts Receivable Configuration

15. **Create Customer Groups** — Use `CustomerGroups` entity in LARC:
    - `DOM` — Domestic customers
    - `EU` — EU customers
    - `EXP` — Export customers

16. **Create Customer Posting Profiles** — Use `CustomerPostingProfiles` + `CustomerPostingProfileLines` entities:
    - Default profile mapping to AR summary account (1300xx)
    - Revenue account, discount accounts

17. **Create Customer Payment Methods** — Use `CustomerPaymentMethods` entity:
    - `BANK` — Bank transfer (standard Danish payment)
    - `CHECK` — Check payment
    - Assign bridging accounts where needed

18. **Create Payment Terms** — Use `PaymentTerms` entity (if not already existing):
    - `Net14` — Net 14 days (common Danish terms)
    - `Net30` — Net 30 days
    - `COD` — Cash on delivery

19. **Configure AR Parameters** — Via form tools (`CustParameters` menu) in LARC:
    - Default posting profile
    - Default payment method and terms
    - Number sequence assignments

## Phase 6: Accounts Payable Configuration

20. **Create Vendor Groups** — Use `VendorGroups` entity in LARC:
    - `DOM` — Domestic vendors
    - `EU` — EU vendors
    - `EXP` — Export/foreign vendors

21. **Create Vendor Posting Profiles** — Use `PostingProfileHeaders` + `PostingProfileLines` entities:
    - Default profile mapping to AP summary account (2000xx)
    - Discount and prepayment accounts

22. **Create Vendor Payment Methods** — Use `VendorPaymentMethods` entity:
    - `BANK` — Bank transfer
    - `EFT` — Electronic funds transfer (common in Denmark)

23. **Configure AP Parameters** — Via form tools (`VendParameters` menu) in LARC:
    - Default posting profile
    - Default payment method and terms
    - Invoice matching policies
    - Number sequence assignments

## Phase 7: Number Sequences & Journal Names

24. **Generate Number Sequences** — Use form tools (Number sequences wizard or `NumberSequenceTableListPage`) to generate sequences for LARC covering:
    - GL: Voucher numbers, journal batch numbers
    - AR: Customer account numbers, free text invoice numbers, payment journal numbers
    - AP: Vendor account numbers, invoice register numbers, payment journal numbers

25. **Create Journal Names** — Use `JournalNames` entity for LARC:
    - `GenJrn` — General journal
    - `CustPay` — Customer payment journal
    - `VendPay` — Vendor payment journal
    - `VendInv` — Vendor invoice journal

## Phase 8: Danish Localization

26. **EU Trade / Intrastat** — Configure Intrastat parameters if EU trade reporting is required for Denmark (via `ForeignTradeParameters` or form tools).

27. **Danish compliance** — Ensure:
    - VAT registration number field is populated on the legal entity
    - EU Sales List reporting is enabled on relevant tax groups
    - Danish payment formats (NETS/Betalingsservice) are configured if electronic payments are needed

## Verification

- Switch to company LARC and verify the legal entity details (name, address, country = DNK)
- Open the Ledger form — confirm CoA = "Danish", Currency = DKK, Fiscal Calendar = "Fiscal", account structures assigned
- Open Trial Balance — confirm main accounts are visible and correctly categorized
- Verify Tax Codes: open Tax codes form and confirm VAT25 = 25%, VAT0 = 0%
- Create a test free-text invoice to verify AR posting profile → GL integration
- Create a test vendor invoice to verify AP posting profile → GL integration
- Validate number sequences are generating correctly for each module

## Decisions

- **LARC** chosen as the 4-character entity ID (user-provided)
- **New "Danish" CoA** over reusing "Shared" — allows Danish-specific account numbering and statutory structure
- **Service Industries B/S + P&L** account structures — fits consulting company (includes Project and ServiceLine dimensions)
- **Standard D365 best practices** used rather than parsing BPC xlsx files — sufficient for GL/AR/AP scope
- **Danish VAT at 25% flat** — Denmark uses a single standard rate (no reduced rates unlike most EU countries)
