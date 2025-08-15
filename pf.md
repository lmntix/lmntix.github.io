# Chart of Accounts (CoA) Design for Multi-Tenant Microfinance Application

## Overview

This document outlines the design for a multi-tenant microfinance application that supports multiple organizations with separate Chart of Accounts (CoA) mapping for different account types including Savings, Fixed Deposits, Loans, and Recurring Deposits.

## Core Design Principles

1. **Multi-tenancy**: Each organization has isolated data with `organization_id`
2. **Type Safety**: Use PostgreSQL enums for consistent data types
3. **Direct Mapping**: Each product account directly references its GL account
4. **Simplicity**: Avoid complex template tables and unnecessary abstractions
5. **Auditability**: Clear transaction trails with GL integration

## Database Schema (Drizzle ORM)

### Enums Definition

```typescript
// Enums first
export const accountTypeEnum = pgEnum('account_type', ['ASSET', 'LIABILITY', 'INCOME', 'EXPENSE', 'EQUITY']);
export const productTypeEnum = pgEnum('product_type', ['SAVINGS', 'FIXED_DEPOSIT', 'LOAN', 'RECURRING_DEPOSIT']);
export const accountStatusEnum = pgEnum('account_status', ['ACTIVE', 'DORMANT', 'CLOSED']);
export const transactionTypeEnum = pgEnum('transaction_type', ['DEPOSIT', 'WITHDRAWAL', 'INTEREST_CREDIT', 'INTEREST_DEBIT', 'FEE', 'PENALTY']);
```

### Core Tables

#### 1. Organizations
```typescript
const organizations = pgTable('organizations', {
  id: uuid('id').defaultRandom().primaryKey(),
  name: text('name').notNull(),
  code: varchar('code', { length: 10 }).unique().notNull(),
  createdAt: timestamp('created_at').defaultNow(),
});
```

#### 2. Chart of Accounts
```typescript
const chartOfAccounts = pgTable('chart_of_accounts', {
  id: uuid('id').defaultRandom().primaryKey(),
  organizationId: uuid('organization_id').references(() => organizations.id),
  accountCode: varchar('account_code', { length: 20 }).notNull(),
  accountName: text('account_name').notNull(),
  accountType: accountTypeEnum('account_type').notNull(),
  productType: productTypeEnum('product_type'), // NULL for non-product accounts
  isActive: boolean('is_active').default(true),
}, (table) => ({
  uniqueOrgAccount: unique().on(table.organizationId, table.accountCode),
}));
```

### Product Account Tables

#### 3. Savings Accounts
```typescript
const savingsAccounts = pgTable('savings_accounts', {
  id: uuid('id').defaultRandom().primaryKey(),
  organizationId: uuid('organization_id').references(() => organizations.id),
  accountNumber: varchar('account_number', { length: 20 }).unique().notNull(),
  customerId: uuid('customer_id').notNull(),
  glAccountId: uuid('gl_account_id').references(() => chartOfAccounts.id), // Direct mapping to CoA
  balance: decimal('balance', { precision: 18, scale: 2 }).default('0'),
  interestRate: decimal('interest_rate', { precision: 5, scale: 2 }),
  status: accountStatusEnum('status').default('ACTIVE'),
  openedDate: date('opened_date').defaultNow(),
});
```

#### 4. Fixed Deposit Accounts
```typescript
const fixedDepositAccounts = pgTable('fixed_deposit_accounts', {
  id: uuid('id').defaultRandom().primaryKey(),
  organizationId: uuid('organization_id').references(() => organizations.id),
  accountNumber: varchar('account_number', { length: 20 }).unique().notNull(),
  customerId: uuid('customer_id').notNull(),
  glAccountId: uuid('gl_account_id').references(() => chartOfAccounts.id),
  principalAmount: decimal('principal_amount', { precision: 18, scale: 2 }).notNull(),
  interestRate: decimal('interest_rate', { precision: 5, scale: 2 }).notNull(),
  tenure: integer('tenure').notNull(), // in months
  maturityDate: date('maturity_date').notNull(),
  status: accountStatusEnum('status').default('ACTIVE'),
  openedDate: date('opened_date').defaultNow(),
});
```

#### 5. Loan Accounts
```typescript
const loanAccounts = pgTable('loan_accounts', {
  id: uuid('id').defaultRandom().primaryKey(),
  organizationId: uuid('organization_id').references(() => organizations.id),
  accountNumber: varchar('account_number', { length: 20 }).unique().notNull(),
  customerId: uuid('customer_id').notNull(),
  glAccountId: uuid('gl_account_id').references(() => chartOfAccounts.id),
  loanAmount: decimal('loan_amount', { precision: 18, scale: 2 }).notNull(),
  outstandingBalance: decimal('outstanding_balance', { precision: 18, scale: 2 }),
  interestRate: decimal('interest_rate', { precision: 5, scale: 2 }).notNull(),
  tenure: integer('tenure').notNull(), // in months
  status: accountStatusEnum('status').default('ACTIVE'),
  disbursementDate: date('disbursement_date'),
});
```

#### 6. Recurring Deposit Accounts
```typescript
const recurringDepositAccounts = pgTable('recurring_deposit_accounts', {
  id: uuid('id').defaultRandom().primaryKey(),
  organizationId: uuid('organization_id').references(() => organizations.id),
  accountNumber: varchar('account_number', { length: 20 }).unique().notNull(),
  customerId: uuid('customer_id').notNull(),
  glAccountId: uuid('gl_account_id').references(() => chartOfAccounts.id),
  monthlyDeposit: decimal('monthly_deposit', { precision: 18, scale: 2 }).notNull(),
  tenure: integer('tenure').notNull(),
  interestRate: decimal('interest_rate', { precision: 5, scale: 2 }).notNull(),
  totalDeposited: decimal('total_deposited', { precision: 18, scale: 2 }).default('0'),
  status: accountStatusEnum('status').default('ACTIVE'),
  openedDate: date('opened_date').defaultNow(),
});
```

### Transaction Management

#### 7. Transactions (General Ledger)
```typescript
const transactions = pgTable('transactions', {
  id: uuid('id').defaultRandom().primaryKey(),
  organizationId: uuid('organization_id').references(() => organizations.id),
  transactionType: transactionTypeEnum('transaction_type').notNull(),
  debitAccountId: uuid('debit_account_id').references(() => chartOfAccounts.id),
  creditAccountId: uuid('credit_account_id').references(() => chartOfAccounts.id),
  amount: decimal('amount', { precision: 18, scale: 2 }).notNull(),
  description: text('description'),
  transactionDate: timestamp('transaction_date').defaultNow(),
  productAccountType: productTypeEnum('product_account_type'), // Which product table
  productAccountId: uuid('product_account_id'), // ID in that product table
});
```

## Sample Data Structure

### Organizations Table
| id | name | code |
|---|---|---|
| org-001 | ABC Microfinance | ABC |
| org-002 | XYZ Credit Union | XYZ |

### Chart of Accounts Table
| id | organization_id | account_code | account_name | account_type | product_type |
|---|---|---|---|---|---|
| coa-001 | org-001 | 1100 | Cash and Bank | ASSET | NULL |
| coa-002 | org-001 | 1200 | Loan Portfolio | ASSET | LOAN |
| coa-003 | org-001 | 2100 | Savings Deposits | LIABILITY | SAVINGS |
| coa-004 | org-001 | 2200 | Fixed Deposits | LIABILITY | FIXED_DEPOSIT |
| coa-005 | org-001 | 2300 | Recurring Deposits | LIABILITY | RECURRING_DEPOSIT |
| coa-006 | org-001 | 4100 | Interest Income | INCOME | NULL |
| coa-007 | org-001 | 5100 | Interest Expense | EXPENSE | NULL |

### Savings Accounts Table
| id | organization_id | account_number | customer_id | gl_account_id | balance | interest_rate | status |
|---|---|---|---|---|---|---|---|
| sav-001 | org-001 | SAV0001 | cust-001 | coa-003 | 5000.00 | 4.5 | ACTIVE |
| sav-002 | org-001 | SAV0002 | cust-002 | coa-003 | 12000.00 | 4.5 | ACTIVE |

### Fixed Deposit Accounts Table
| id | organization_id | account_number | customer_id | gl_account_id | principal_amount | interest_rate | tenure | maturity_date | status |
|---|---|---|---|---|---|---|---|---|---|
| fd-001 | org-001 | FD0001 | cust-001 | coa-004 | 50000.00 | 7.5 | 12 | 2025-08-15 | ACTIVE |

### Loan Accounts Table
| id | organization_id | account_number | customer_id | gl_account_id | loan_amount | outstanding_balance | interest_rate | tenure | status |
|---|---|---|---|---|---|---|---|---|---|
| loan-001 | org-001 | LN0001 | cust-002 | coa-002 | 100000.00 | 85000.00 | 12.5 | 36 | ACTIVE |

### Recurring Deposit Accounts Table
| id | organization_id | account_number | customer_id | gl_account_id | monthly_deposit | tenure | interest_rate | total_deposited | status |
|---|---|---|---|---|---|---|---|---|---|
| rd-001 | org-001 | RD0001 | cust-003 | coa-005 | 2000.00 | 24 | 6.5 | 8000.00 | ACTIVE |

### Transactions Table (Sample GL Entries)
| id | organization_id | transaction_type | debit_account_id | credit_account_id | amount | description | product_account_type | product_account_id |
|---|---|---|---|---|---|---|---|---|
| txn-001 | org-001 | DEPOSIT | coa-001 | coa-003 | 1000.00 | Savings Deposit | SAVINGS | sav-001 |
| txn-002 | org-001 | INTEREST_CREDIT | coa-007 | coa-003 | 18.75 | Savings Interest | SAVINGS | sav-001 |
| txn-003 | org-001 | DEPOSIT | coa-001 | coa-004 | 50000.00 | FD Opening | FIXED_DEPOSIT | fd-001 |
| txn-004 | org-001 | WITHDRAWAL | coa-002 | coa-001 | 100000.00 | Loan Disbursement | LOAN | loan-001 |

## Transaction Flow Example

When a customer deposits ₹1000 into savings account SAV0001:

```sql
-- 1. Update Savings Account Balance
UPDATE savings_accounts 
SET balance = balance + 1000 
WHERE id = 'sav-001';

-- 2. Create Double Entry in GL
INSERT INTO transactions (
  organization_id, transaction_type,
  debit_account_id, credit_account_id,
  amount, description,
  product_account_type, product_account_id
) VALUES (
  'org-001', 'DEPOSIT',
  'coa-001', -- Cash and Bank (DEBIT)
  'coa-003', -- Savings Deposits (CREDIT)
  1000.00, 'Savings Deposit - SAV0001',
  'SAVINGS', 'sav-001'
);
```

## Key Benefits

1. **Type Safety**: Enums prevent invalid data entry
2. **Direct Mapping**: Each product account directly references its GL account
3. **Multi-tenant**: Each organization maintains separate books
4. **Simplicity**: No complex template tables or abstractions
5. **Auditability**: Clear transaction trails with product linkage
6. **Flexibility**: Organizations can define custom CoA structures
7. **Scalability**: Easy to add new product types

## Implementation Guidelines

1. **Use Row-Level Security (RLS)** in PostgreSQL for data isolation
2. **Create proper indexes** on foreign keys and query columns
3. **Implement transaction atomicity** - update both product balance and GL together
4. **Add validation constraints** to ensure accounting equation balance
5. **Consider table partitioning** by organization_id for large datasets

## Next Steps

- [ ] Add customer management tables
- [ ] Implement interest calculation logic
- [ ] Add reporting views and procedures
- [ ] Define business rules and validations
- [ ] Create API endpoints for account operations

---

*This document will be enhanced and updated as we add more features and refine the design.*
