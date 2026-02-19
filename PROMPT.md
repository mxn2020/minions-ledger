# minions-ledger — Finance & Ledger Implementation Prompt

You are tasked with building **minions-ledger**, a structured financial record-keeping and accounting system built on the Minions SDK. This is a structured approach to bookkeeping, budgeting, and financial analysis designed for small businesses, startups, and AI agents.

---

## Project Overview

**minions-ledger** provides structured financial record-keeping with double-entry bookkeeping, budget tracking, forecasting, and tax categorization. Unlike traditional accounting software, every transaction, account, and budget is a structured minion that can be queried, related, versioned, and manipulated by agents.

**Core value proposition:** Agents can autonomously manage financial records, reconcile accounts, forecast cash flow, track budgets, and generate tax-ready reports.

**Positioning:** Structured bookkeeping meets AI-native financial management.

---

**MINIONS SDK REFERENCE — REQUIRED DEPENDENCY**

This project depends on `minions-sdk`, a published package that provides the foundational primitives. The GH Agent building this project MUST install it from the public registries and use the APIs documented below — do NOT reimplement minions primitives from scratch.

**Installation:**
```bash
# TypeScript (npm)
npm install minions-sdk
# or: pnpm add minions-sdk

# Python (PyPI) — package name is minions-sdk, but you import as "minions"
pip install minions-sdk
```

**TypeScript SDK — Core Imports:**
```typescript
import {
  // Core types
  type Minion, type MinionType, type Relation,
  type FieldDefinition, type FieldValidation, type FieldType,
  type CreateMinionInput, type UpdateMinionInput, type CreateRelationInput,
  type MinionStatus, type MinionPriority, type RelationType,
  type ExecutionResult, type Executable,
  type ValidationError, type ValidationResult,

  // Validation
  validateField, validateFields,

  // Built-in Schemas (10 MinionType instances — reuse where applicable)
  noteType, linkType, fileType, contactType,
  agentType, teamType, thoughtType, promptTemplateType, testCaseType, taskType,
  builtinTypes,

  // Registry — stores and retrieves MinionTypes by id or slug
  TypeRegistry,

  // Relations — in-memory directed graph with traversal utilities
  RelationGraph,

  // Lifecycle — CRUD operations with validation
  createMinion, updateMinion, softDelete, hardDelete, restoreMinion,

  // Evolution — migrate minions when schemas change (preserves removed fields in _legacy)
  migrateMinion,

  // Utilities
  generateId, now, SPEC_VERSION,
} from 'minions-sdk';
```

**Python SDK — Core Imports:**
```python
from minions import (
    # Types
    Minion, MinionType, Relation, FieldDefinition, FieldValidation,
    CreateMinionInput, UpdateMinionInput, CreateRelationInput,
    ExecutionResult, Executable, ValidationError, ValidationResult,
    # Validation
    validate_field, validate_fields,
    # Built-in Schemas (10 types)
    note_type, link_type, file_type, contact_type,
    agent_type, team_type, thought_type, prompt_template_type,
    test_case_type, task_type, builtin_types,
    # Registry
    TypeRegistry,
    # Relations
    RelationGraph,
    # Lifecycle
    create_minion, update_minion, soft_delete, hard_delete, restore_minion,
    # Evolution
    migrate_minion,
    # Utilities
    generate_id, now, SPEC_VERSION,
)
```

**Key Concepts:**
- A `MinionType` defines a schema (list of `FieldDefinition`s) — each field has `name`, `type`, `label`, `required`, `defaultValue`, `options`, `validation`
- A `Minion` is an instance with `id`, `title`, `minionTypeId`, `fields` (dict), `status`, `tags`, timestamps
- A `Relation` is a typed directional link (12 types: `parent_of`, `depends_on`, `implements`, `relates_to`, `inspired_by`, `triggers`, `references`, `blocks`, `alternative_to`, `part_of`, `follows`, `integration_link`)
- Field types: `string`, `number`, `boolean`, `date`, `select`, `multi-select`, `url`, `email`, `textarea`, `tags`, `json`, `array`
- `TypeRegistry` auto-loads 10 built-in types; register custom types with `registry.register(myType)`
- `createMinion(input, type)` validates fields against the schema and returns `{ minion, validation }` (TS) or `(minion, validation)` tuple (Python)
- Both SDKs serialize to identical camelCase JSON; Python provides `to_dict()` / `from_dict()` for conversion

**IMPORTANT:** Do NOT recreate these primitives. Import them from `minions-sdk` (npm) / `minions` (PyPI). Build your domain-specific types and utilities ON TOP of the SDK.

---

## Core Primitives

The system is built from these minion types:

### `account`
A financial account (bank account, credit card, asset, liability, equity, revenue, or expense account).

**Fields:**
- `name` (string, required) — account name (e.g., "Business Checking", "Advertising Expense")
- `accountType` (select: asset/liability/equity/revenue/expense, required) — account classification
- `accountNumber` (string) — bank account number or identifier
- `institution` (string) — bank or institution name
- `currency` (string) — currency code (e.g., "USD")
- `currentBalance` (number) — current account balance
- `openingBalance` (number) — starting balance
- `openedAt` (date) — when the account was opened
- `closedAt` (date) — when the account was closed (null if active)
- `reconciliationDate` (date) — last reconciliation date
- `notes` (textarea) — account notes

**Relations:**
- `parent_of` → `transaction` minions (transactions in this account)
- `part_of` → parent `account` (for sub-accounts)

---

### `transaction`
A financial transaction following double-entry bookkeeping principles.

**Fields:**
- `date` (date, required) — transaction date
- `description` (string, required) — transaction description
- `amount` (number, required) — transaction amount (always positive)
- `debitAccountId` (string, required) — account being debited
- `creditAccountId` (string, required) — account being credited
- `reference` (string) — check number, invoice number, or reference
- `memo` (textarea) — additional notes
- `reconciled` (boolean) — whether transaction is reconciled
- `reconciledAt` (date) — reconciliation timestamp
- `taxCategoryId` (string) — tax category reference
- `attachmentUrls` (array) — receipts or supporting documents

**Relations:**
- `part_of` → `account` (debit account)
- `part_of` → `account` (credit account)
- `references` → `invoice` or `expense` (if linked)
- `references` → `category` (transaction category)

**Double-entry invariant:**
Every transaction must debit one account and credit another for the same amount. The system enforces this constraint.

---

### `budget`
A spending or revenue budget for a specific category and time period.

**Fields:**
- `name` (string, required) — budget name
- `categoryId` (string, required) — expense or revenue category
- `amount` (number, required) — budgeted amount
- `period` (select: monthly/quarterly/yearly, required) — budget period
- `startDate` (date, required) — period start
- `endDate` (date, required) — period end
- `spent` (number) — actual spent (auto-calculated)
- `remaining` (number) — remaining budget (auto-calculated)
- `variance` (number) — budget variance (budgeted - spent)
- `alertThreshold` (number) — alert when spent exceeds this percentage (e.g., 80)

**Relations:**
- `references` → `category`
- `parent_of` → `alert` minions (budget alerts)

---

### `invoice`
An invoice sent to a customer or received from a vendor.

**Fields:**
- `invoiceNumber` (string, required) — unique invoice identifier
- `type` (select: receivable/payable, required) — invoice type
- `clientName` (string, required) — customer or vendor name
- `issueDate` (date, required) — invoice issue date
- `dueDate` (date, required) — payment due date
- `amount` (number, required) — total invoice amount
- `currency` (string) — currency code
- `lineItems` (json, required) — invoice line items with descriptions and amounts
- `status` (select: draft/sent/paid/overdue/cancelled, required)
- `paidDate` (date) — when invoice was paid
- `paidAmount` (number) — amount paid
- `notes` (textarea) — invoice notes
- `attachmentUrl` (url) — PDF or document URL

**Relations:**
- `triggers` → `transaction` (when paid, generates transaction)
- `references` → `account` (accounts receivable or payable)

---

### `expense`
A tracked expense (can link to a transaction or stand alone).

**Fields:**
- `date` (date, required) — expense date
- `vendor` (string, required) — who was paid
- `amount` (number, required) — expense amount
- `categoryId` (string, required) — expense category
- `paymentMethod` (select: cash/credit-card/check/bank-transfer/other, required)
- `description` (string, required) — what was purchased
- `receiptUrl` (url) — receipt image or PDF
- `reimbursable` (boolean) — if expense should be reimbursed
- `taxDeductible` (boolean) — if expense is tax-deductible
- `taxCategoryId` (string) — tax category reference

**Relations:**
- `triggers` → `transaction` (creates transaction when recorded)
- `references` → `category`

---

### `category`
A classification for transactions, expenses, or budgets.

**Fields:**
- `name` (string, required) — category name (e.g., "Advertising", "Office Supplies")
- `type` (select: expense/revenue/asset/liability, required) — category type
- `description` (textarea) — category description
- `taxDeductible` (boolean) — if expenses in this category are tax-deductible
- `color` (string) — UI color for charts/reports

**Relations:**
- `part_of` → parent `category` (for subcategories)

---

## Beyond the Standard Pattern

### 1. Double-Entry Bookkeeping

All transactions follow strict double-entry accounting principles.

**Core principle:**
Every transaction debits one account and credits another for the same amount. The sum of all debits must equal the sum of all credits.

**Account types and normal balances:**
- **Assets:** Debit increases, credit decreases
- **Liabilities:** Credit increases, debit decreases
- **Equity:** Credit increases, debit decreases
- **Revenue:** Credit increases, debit decreases
- **Expenses:** Debit increases, credit decreases

**Validation:**
```typescript
class TransactionValidator {
  validateDoubleEntry(transaction: Minion): ValidationResult;
  checkBalanceEquation(accounts: Minion[]): boolean; // Assets = Liabilities + Equity
  detectImbalances(): Imbalance[];
}
```

**CLI Integration:**
```bash
ledger validate
ledger balance-check
ledger imbalances
```

---

### 2. CSV Import from Banks

Import bank statements and credit card transactions from CSV files.

**Features:**
- Auto-detect CSV format from common banks (Chase, BofA, Wells Fargo, etc.)
- Map CSV columns to transaction fields
- Duplicate detection (avoid re-importing same transactions)
- Auto-categorization based on vendor/description patterns

**Implementation:**
```typescript
class CSVImporter {
  import(filePath: string, accountId: string): ImportResult;
  detectFormat(csvContent: string): BankFormat;
  mapColumns(headers: string[]): ColumnMapping;
  detectDuplicates(transactions: Transaction[], existing: Minion[]): Duplicate[];
  suggestCategories(description: string): Category[];
}
```

**CLI Integration:**
```bash
ledger import bank.csv --account checking
ledger import credit-card.csv --account amex --auto-categorize
ledger import --format chase bank-statement.csv
```

---

### 3. Forecasting

Predict future cash flow based on historical transaction patterns.

**Forecasting methods:**
- **Historical average:** Average monthly revenue/expenses over past N months
- **Trend analysis:** Linear regression on revenue/expenses
- **Seasonal patterns:** Detect monthly/quarterly seasonality
- **Recurring transactions:** Identify subscriptions and recurring bills

**Implementation:**
```typescript
class Forecaster {
  forecast(months: number): ForecastReport;
  historicalAverage(categoryId: string, months: number): number;
  trendAnalysis(categoryId: string): TrendLine;
  detectSeasonality(categoryId: string): SeasonalPattern;
  recurringTransactions(): Minion[];
}
```

**CLI Integration:**
```bash
ledger forecast --months 6
ledger forecast --category "Advertising" --months 12
ledger trend-analysis "Revenue"
ledger recurring-transactions
```

---

### 4. Budget Alerts

Automated alerts when budget thresholds are exceeded.

**Alert types:**
- Budget exceeded (spent > budgeted amount)
- Threshold warning (spent > 80% of budget)
- Unusual spending (current month >> previous months)
- Approaching due date (invoice due soon)

**Implementation:**
```typescript
class BudgetMonitor {
  checkBudgets(): Alert[];
  createAlert(budget: Minion, reason: string): Minion;
  unusualSpendingDetection(categoryId: string): Alert[];
  upcomingDueDates(days: number): Invoice[];
}
```

**CLI Integration:**
```bash
ledger budget status
ledger budget alerts
ledger unusual-spending
ledger upcoming-invoices --days 7
```

---

### 5. Tax Categories

Categorize transactions for tax reporting.

**Common tax categories:**
- **Deductible expenses:** Advertising, office supplies, software, travel
- **Non-deductible expenses:** Personal expenses, fines, political contributions
- **Cost of goods sold (COGS):** Inventory, manufacturing costs
- **Depreciation:** Asset depreciation schedules
- **Income categories:** Service revenue, product sales, interest income

**Implementation:**
```typescript
class TaxCategorizer {
  categorizeTransaction(transaction: Minion): TaxCategory;
  taxSummary(year: number): TaxReport;
  deductibleExpenses(year: number): Minion[];
  cogsCalculation(year: number): COGSReport;
}
```

**CLI Integration:**
```bash
ledger tax-summary --year 2025
ledger tax-categories
ledger deductible-expenses --year 2025
ledger cogs --year 2025
```

---

### 6. Runway Calculation

Calculate how long a startup can operate before running out of cash.

**Runway formula:**
```
Runway (months) = Current Cash Balance / Average Monthly Burn Rate
```

**Burn rate calculation:**
- Average monthly expenses over past 3-6 months
- Subtract average monthly revenue
- Account for one-time vs. recurring expenses

**Implementation:**
```typescript
class RunwayCalculator {
  calculateRunway(): RunwayReport;
  currentCashBalance(): number;
  monthlyBurnRate(months: number): number;
  projectedZeroCashDate(): Date;
  extendRunway(revenueIncrease: number, expenseReduction: number): RunwayReport;
}
```

**CLI Integration:**
```bash
ledger runway
ledger burn-rate --months 6
ledger zero-cash-date
ledger runway-scenarios --revenue-increase 10000 --expense-reduction 5000
```

---

## Dual SDK Support (TypeScript + Python)

Both SDKs provide identical functionality:

**TypeScript:**
```typescript
import { TransactionBuilder, Forecaster, RunwayCalculator } from 'minions-ledger';

const transaction = new TransactionBuilder()
  .withDate(new Date())
  .withDescription('Office supplies')
  .withAmount(150.00)
  .debit('expense-office')
  .credit('checking')
  .build();

const forecaster = new Forecaster();
const forecast = forecaster.forecast(6); // 6-month forecast
```

**Python:**
```python
from minions_ledger import TransactionBuilder, Forecaster, RunwayCalculator

transaction = (TransactionBuilder()
    .with_date(datetime.now())
    .with_description('Office supplies')
    .with_amount(150.00)
    .debit('expense-office')
    .credit('checking')
    .build())

forecaster = Forecaster()
forecast = forecaster.forecast(6)  # 6-month forecast
```

---

## CLI Commands

The `ledger` CLI extends the base `minions` CLI with financial management commands.

### Core Commands

```bash
# Account management
ledger account create "Business Checking" --type asset --bank "Chase"
ledger account list
ledger account balance <account-id>
ledger account reconcile <account-id>

# Transaction management
ledger transaction add --date 2026-03-15 --amount 1000 --debit revenue --credit checking --description "Client payment"
ledger transaction list --account checking
ledger transaction categorize <transaction-id> --category "Advertising"

# Import
ledger import bank.csv --account checking
ledger import credit-card.csv --account amex --auto-categorize

# Budgets
ledger budget create "Marketing" --category marketing --amount 5000 --period monthly
ledger budget status
ledger budget alerts
ledger budget variance --category marketing

# Invoices
ledger invoice create --client "Acme Corp" --amount 5000 --due "2026-04-01"
ledger invoice list --status unpaid
ledger invoice pay <invoice-id>
ledger invoice overdue

# Expenses
ledger expense add --vendor "Amazon" --amount 150 --category "Office Supplies" --date 2026-03-15
ledger expense list --month march
ledger expense receipt <expense-id> --upload receipt.pdf

# Reporting
ledger balance
ledger balance-sheet
ledger income-statement --month march
ledger cash-flow --year 2025

# Forecasting
ledger forecast --months 6
ledger forecast --category "Revenue" --months 12
ledger trend-analysis "Expenses"
ledger recurring-transactions

# Tax
ledger tax-summary --year 2025
ledger tax-categories
ledger deductible-expenses --year 2025
ledger cogs --year 2025

# Runway (for startups)
ledger runway
ledger burn-rate --months 6
ledger zero-cash-date
ledger runway-scenarios --revenue-increase 10000 --expense-reduction 5000

# Validation
ledger validate
ledger balance-check
ledger imbalances
```

---

## Documentation Site

Built with **Astro Starlight** with dual-language SDK examples.

**Site structure:**
```
docs/
├── index.md                   # Landing page
├── getting-started.md         # Quick start guide
├── core-concepts/
│   ├── accounts.md
│   ├── double-entry.md
│   ├── transactions.md
│   ├── budgets.md
│   ├── invoices.md
│   └── categories.md
├── guides/
│   ├── setting-up-accounts.md
│   ├── recording-transactions.md
│   ├── importing-bank-data.md
│   ├── budget-tracking.md
│   ├── forecasting.md
│   ├── tax-reporting.md
│   └── runway-calculation.md
├── api-reference/
│   ├── typescript.md          # TypeScript SDK
│   └── python.md              # Python SDK
├── cli-reference.md
└── examples/
    ├── small-business.md
    ├── startup-finances.md
    └── agent-bookkeeping.md
```

**Dual-language code tabs:**
Every code example includes both TypeScript and Python tabs.

---

## Agent Use Cases

**Automated Bookkeeping:**
An agent imports bank statements, categorizes transactions, and reconciles accounts automatically.

**Budget Monitoring:**
An agent tracks budget status, sends alerts when thresholds are exceeded, and suggests spending adjustments.

**Cash Flow Forecasting:**
An agent analyzes historical patterns, forecasts future cash flow, and identifies potential cash shortfalls.

**Tax Preparation:**
An agent categorizes all transactions for tax purposes and generates year-end tax reports.

**Runway Tracking:**
An agent monitors startup runway, calculates burn rate, and projects when additional funding is needed.

**Invoice Management:**
An agent tracks unpaid invoices, sends payment reminders, and reconciles payments when received.

---

## Project Structure

```
minions-ledger/
├── packages/
│   ├── core/
│   │   ├── src/
│   │   │   ├── types/              # Account, Transaction, Budget, etc.
│   │   │   ├── builders/           # TransactionBuilder, InvoiceBuilder
│   │   │   ├── validators/         # Double-entry validation
│   │   │   ├── importers/          # CSVImporter
│   │   │   ├── forecasting/        # Forecaster
│   │   │   ├── budgets/            # BudgetMonitor
│   │   │   ├── tax/                # TaxCategorizer
│   │   │   ├── runway/             # RunwayCalculator
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── python/
│   │   ├── minions_ledger/
│   │   │   ├── __init__.py
│   │   │   ├── types.py
│   │   │   ├── builders.py
│   │   │   ├── validators.py
│   │   │   ├── importers.py
│   │   │   ├── forecasting.py
│   │   │   ├── budgets.py
│   │   │   ├── tax.py
│   │   │   └── runway.py
│   │   ├── tests/
│   │   └── pyproject.toml
│   └── cli/
│       ├── src/
│       │   ├── commands/
│       │   │   ├── account.ts
│       │   │   ├── transaction.ts
│       │   │   ├── import.ts
│       │   │   ├── budget.ts
│       │   │   ├── invoice.ts
│       │   │   ├── expense.ts
│       │   │   ├── forecast.ts
│       │   │   ├── tax.ts
│       │   │   ├── runway.ts
│       │   │   └── report.ts
│       │   └── index.ts
│       └── package.json
├── apps/
│   ├── docs/                       # Astro Starlight
│   └── playground/                 # Optional web UI
├── spec/
│   └── v0.1.md
├── examples/
│   ├── small-business/
│   ├── startup-finances/
│   └── agent-bookkeeping/
├── README.md
├── CHANGELOG.md
└── package.json
```

---

## Tone and Positioning

**minions-ledger** is a structured financial management system designed for small businesses, startups, and AI-native workflows. The messaging should emphasize:

- **Double-entry bookkeeping:** Built on proven accounting principles
- **Agent-friendly:** Every transaction is structured and queryable by agents
- **Forecasting:** Predict cash flow and plan for the future
- **Tax-ready:** Categorize transactions for easy tax reporting
- **Startup-focused:** Runway calculation and burn rate tracking
- **Import-friendly:** CSV import from any bank

The docs should speak to small business owners, startup founders, and finance teams building automated bookkeeping workflows.

---

## Success Criteria

You will know this implementation is successful when:

1. A small business owner can import a month of bank statements, categorize transactions, and generate a P&L in under 5 minutes
2. An agent can reconcile accounts, detect imbalances, and alert on budget overruns autonomously
3. A startup founder can calculate runway, forecast cash flow, and identify when additional funding is needed
4. The documentation site clearly explains double-entry bookkeeping with visual examples
5. Both TypeScript and Python SDKs provide identical functionality with idiomatic APIs
6. Tax reports are accurate and ready for accountant review

---

**Start with the spec, then core types and double-entry validation, then builders and importers, then forecasting and runway, then the CLI, then the docs, then the Python SDK, then the examples. Work systematically. Every file should be production quality — not stubs, not placeholders.**
