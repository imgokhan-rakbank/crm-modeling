# Banking CRM 360 — Logical Data Model

> **Version:** 1.2  
> **Date:** May 2026  
> **Source:** CRM Field Requirements v1.1 — Levels & Navigation for Data Modelling  
> **Scope:** RAKBANK CRM 360 — Individual & Corporate customers, all product lines

---

## Table of Contents

1. [Design Principles](#1-design-principles)
2. [Entity Overview](#2-entity-overview)
3. [Domain: Party](#3-domain-party)
4. [Domain: Products — Accounts & Cards](#4-domain-products--accounts--cards)
5. [Domain: Products — Lending](#5-domain-products--lending)
6. [Domain: Products — Investments & Insurance](#6-domain-products--investments--insurance)
7. [Domain: CRM Operations](#7-domain-crm-operations)
8. [Domain: Enquiries & Calculators](#8-domain-enquiries--calculators)
9. [Relationships Summary](#9-relationships-summary)
10. [Key Design Decisions](#10-key-design-decisions)

---

## 1. Design Principles

| Principle | Rationale |
|-----------|-----------|
| **Party Model** | Single `PARTY` supertype with subtypes `PARTY_INDIVIDUAL`, `PARTY_ORGANIZATION`, `PARTY_EMPLOYEE`, and `PARTY_REFERENCE` — the supertype holds only attributes universal to all parties; subtype-specific attributes live in each subtype |
| **Product Catalogue separation** | `PRODUCT` (static definition) is separate from product-instance tables (e.g. `ACCOUNT`, `CREDIT_CARD`) — allows product changes without modifying customer data |
| **Levelled access alignment** | Level-0 (Mini 360) fields are promoted as non-nullable/indexed columns; Level-1+ fields are in child tables or extended attribute tables |
| **Islamic Finance parity** | `ISLAMIC_FINANCE` mirrors retail/SME loan structure but keeps Shariah-specific fields (profit rate, Murabaha/Ijara scheme) |
| **Audit everywhere** | Every mutable entity carries `created_at`, `updated_at`, `created_by`, `updated_by` columns |
| **Soft delete** | Entities use `is_active` / `status` rather than hard deletes for regulatory traceability |
| **Normalisation target: 3NF** | With selective denormalisation of calculated/summary values on `PARTY_PORTFOLIO_SUMMARY` for Mini-360 performance |

---

## 2. Entity Overview

```
PARTY ────────────────────────── PARTY_INDIVIDUAL
      ├─────────────────────────  PARTY_ORGANIZATION
      ├─────────────────────────  PARTY_EMPLOYEE
      └─────────────────────────  PARTY_REFERENCE
      │
      ├── PARTY_IDENTIFICATION
      ├── PARTY_CONTACT
      ├── PARTY_ADDRESS
      ├── PARTY_CONSENT
      ├── PARTY_COMPLIANCE
      ├── PARTY_KYC
      ├── PARTY_CHANNEL_SUBSCRIPTION
      ├── PARTY_ANALYTICS
      ├── PARTY_PORTFOLIO_SUMMARY
      │
      ├── PARTY_RELATIONSHIP ── (self-join: joint holders, guarantors, group members)
      │
      ├── ACCOUNT ─────────────── ACCOUNT_BALANCE
         │   │                       ACCOUNT_INTEREST_CONFIG
         │   │                       ACCOUNT_STATEMENT_CONFIG
         │   │                       ACCOUNT_LIMIT
         │   │                       ACCOUNT_TRANSACTION
         │   │                       ACCOUNT_LIEN
         │   │                       CHEQUE_BOOK ── CHEQUE_LEAF
         │   │                       STANDING_INSTRUCTION
         │   │                       SALARY_TRANSACTION
         │   │                       SWEEP_POOL
         │   │                       TRANSFER_REMITTANCE
         │   └─────────────────────── AUTHORIZED_REPRESENTATIVE
         │
         ├── DEPOSIT
         │
         ├── DEBIT_CARD ──────────── CARD_AUTHORIZATION_TRANSACTION
         │                           CARD_TOKEN
         │                           TOKEN_PROVISIONING
         │
         ├── CREDIT_CARD ─────────── CARD_AUTHORIZATION_TRANSACTION
         │   │                       CARD_TOKEN
         │   │                       TOKEN_PROVISIONING
         │   │                       CREDIT_CARD_BALANCE
         │   │                       CREDIT_CARD_STATEMENT
         │   │                       CREDIT_CARD_INSTALLMENT ── INSTALLMENT_SCHEDULE
         │   │                       CREDIT_CARD_REWARDS
         │   │                       SUPPLEMENTARY_CARD
         │   │                       CREDIT_CARD_INSURANCE
         │   └─────────────────────── CREDIT_CARD_STANDING_INSTRUCTION
         │
         ├── PREPAID_CARD ─────────── CARD_AUTHORIZATION_TRANSACTION
         │                            CARD_TOKEN
         │
         ├── DIGITAL_ACCESS_CARD
         │
         ├── RETAIL_LOAN ─────────── RETAIL_LOAN_AUTO_DETAILS
         │   │                       RETAIL_LOAN_HOME_DETAILS
         │   │                       RETAIL_LOAN_REPAYMENT_SCHEDULE
         │   │                       LOAN_RECEIPT
         │   │                       LOAN_STATEMENT
         │   │                       LOAN_EARLY_CLOSURE
         │   └─────────────────────── LOAN_GUARANTOR
         │
         ├── SME_LOAN ────────────── SME_LOAN_DISBURSEMENT
         │   │                       SME_REPAYMENT_SCHEDULE
         │   │                       CREDIT_LIMIT ─── CREDIT_LIMIT_DOCUMENT
         │   │                       OD_FACILITY        CREDIT_LIMIT_COLLATERAL
         │   └─────────────────────── LOAN_RECOVERY_TRANSACTION
         │
         ├── ISLAMIC_FINANCE
         │
         ├── TRADE_FINANCE
         │
         ├── INVESTMENT_PORTFOLIO ── INVESTMENT_ASSET
         │                           INVESTMENT_INSURANCE
         │
         ├── INSURANCE_POLICY ─────── INSURANCE_BENEFICIARY
         │                            INSURANCE_PAYMENT
         │                            INSURANCE_TRANSACTION
         │
         ├── INTERACTION
         ├── SERVICE_REQUEST
         ├── LEAD
      ├── PARTY_PACKAGE ────────── PACKAGE_UTILIZATION
         │
         └── DOCUMENT
```

---

## 3. Domain: Party

### 3.1 PARTY *(supertype)*

Central party record. One row per CIF. Contains **only attributes universal to every party** — subtype-specific attributes live exclusively in `PARTY_INDIVIDUAL`, `PARTY_ORGANIZATION`, `PARTY_EMPLOYEE`, or `PARTY_REFERENCE`.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `party_id` | UUID | PK | Surrogate key |
| `cif_number` | VARCHAR(20) | UNIQUE NOT NULL | Finacle CIF ID |
| `party_type` | ENUM | NOT NULL | `INDIVIDUAL` \| `ORGANIZATION` \| `EMPLOYEE` \| `REFERENCE` |
| `party_status` | VARCHAR(30) | NOT NULL | `ACTIVE` \| `INACTIVE` \| `DORMANT` \| `DECEASED` \| `CLOSED` |
| `is_blacklisted` | BOOLEAN | NOT NULL DEFAULT FALSE | Finacle blacklisted flag |
| `is_negated` | BOOLEAN | NOT NULL DEFAULT FALSE | Negated flag |
| `party_since_date` | DATE | | CIF creation date |
| `preferred_language` | VARCHAR(10) | | ISO 639-1 code, e.g. `EN`, `AR` |
| `segment_id` | UUID | FK → SEGMENT | |
| `sub_segment_id` | UUID | FK → SEGMENT | |
| `primary_rm_id` | UUID | FK → PARTY | Relationship Manager (a PARTY_EMPLOYEE) |
| `secondary_rm_id` | UUID | FK → PARTY | Secondary RM (a PARTY_EMPLOYEE) |
| `primary_branch_id` | UUID | FK → BRANCH | Home branch / SOL |
| `risk_profile` | VARCHAR(30) | | `LOW` \| `MEDIUM` \| `HIGH` |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |
| `created_by` | UUID | FK → PARTY | |
| `updated_by` | UUID | FK → PARTY | |

---

### 3.2 PARTY_INDIVIDUAL *(subtype)*

One-to-one with `PARTY` where `party_type = INDIVIDUAL`. Contains all individual-specific attributes including those that were incorrectly on the supertype.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `party_id` | UUID | PK, FK → PARTY | |
| `is_vip` | BOOLEAN | NOT NULL DEFAULT FALSE | Privilege 786 flag |
| `is_royal` | BOOLEAN | NOT NULL DEFAULT FALSE | Ruling family flag |
| `is_minor` | BOOLEAN | NOT NULL DEFAULT FALSE | Minor customer flag |
| `is_special_needs` | BOOLEAN | NOT NULL DEFAULT FALSE | People of determination flag |
| `is_pep` | BOOLEAN | NOT NULL DEFAULT FALSE | Politically Exposed Person |
| `party_classification` | VARCHAR(50) | | `RESIDENT` \| `NON_RESIDENT` |
| `party_special_status` | VARCHAR(50) | | Free-text special status note |
| `title` | VARCHAR(10) | | `MR` \| `MRS` \| `MS` \| `DR` etc. |
| `first_name` | VARCHAR(100) | NOT NULL | |
| `middle_name` | VARCHAR(100) | | |
| `last_name` | VARCHAR(100) | NOT NULL | |
| `full_name` | VARCHAR(300) | NOT NULL | Denormalised for search performance |
| `date_of_birth` | DATE | | |
| `gender` | ENUM | | `MALE` \| `FEMALE` \| `OTHER` |
| `nationality` | CHAR(3) | | ISO 3166-1 alpha-3 |
| `country_of_birth` | CHAR(3) | | ISO 3166-1 alpha-3 |
| `marital_status` | VARCHAR(20) | | `SINGLE` \| `MARRIED` \| `DIVORCED` \| `WIDOWED` |
| `employment_type` | VARCHAR(50) | | `SALARIED` \| `SELF_EMPLOYED` \| `RETIRED` \| `STUDENT` etc. |
| `employer_code` | VARCHAR(30) | | |
| `employer_name` | VARCHAR(200) | | |
| `department` | VARCHAR(100) | | |
| `total_years_employment` | NUMERIC(5,2) | | |
| `years_since_in_uae` | NUMERIC(5,2) | | |
| `customer_sourced_by` | VARCHAR(100) | | |
| `source_branch_sol` | VARCHAR(30) | | |
| `reason_inactive` | VARCHAR(500) | | |
| `customer_deceased_date` | DATE | | |
| `notes` | TEXT | | |

---

### 3.3 PARTY_ORGANIZATION *(subtype)*

One-to-one with `PARTY` where `party_type = ORGANIZATION`. Contains all corporate/business-specific attributes.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `party_id` | UUID | PK, FK → PARTY | |
| `company_name` | VARCHAR(300) | NOT NULL | |
| `short_name` | VARCHAR(100) | | |
| `country_of_incorporation` | CHAR(3) | | ISO 3166-1 alpha-3 |
| `principal_place_of_operation` | VARCHAR(100) | | |
| `country_of_origin` | CHAR(3) | | |
| `legal_entity_type` | VARCHAR(50) | | `LLC` \| `SOLE_PROP` \| `PARTNERSHIP` \| `PUBLIC` \| `NGO` etc. |
| `date_of_business_establishment` | DATE | | |
| `trade_license_number` | VARCHAR(50) | | |
| `trade_license_expiry` | DATE | | |
| `gcd_number` | VARCHAR(50) | | Group company identifier |
| `group_id` | UUID | FK → CORPORATE_GROUP | |
| `group_code` | VARCHAR(30) | | |
| `shareholding_pct` | NUMERIC(5,2) | | % holding in group |
| `credit_grading_code` | VARCHAR(20) | | |
| `group_credit_grading` | VARCHAR(20) | | |
| `reason_inactive` | VARCHAR(500) | | |
| `customer_sourced_by` | VARCHAR(100) | | |
| `source_branch_sol` | VARCHAR(30) | | |
| `notes` | TEXT | | |

---

### 3.4 CORPORATE_GROUP

Supports group-level credit structures for SME/Corporate.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `group_id` | UUID | PK | |
| `group_code` | VARCHAR(30) | UNIQUE NOT NULL | |
| `group_name` | VARCHAR(300) | NOT NULL | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |

---

### 3.5 PARTY_IDENTIFICATION

Multiple identification documents per party.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `identification_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `document_type` | VARCHAR(30) | NOT NULL | `PASSPORT` \| `EMIRATES_ID` \| `TRADE_LICENSE` \| `VISA` \| `DRIVING_LICENSE` etc. |
| `document_number` | VARCHAR(50) | NOT NULL | |
| `issue_date` | DATE | | |
| `expiry_date` | DATE | | |
| `issuing_country` | CHAR(3) | | |
| `issuing_authority` | VARCHAR(200) | | |
| `is_preferred` | BOOLEAN | NOT NULL DEFAULT FALSE | Primary identification document |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

*Index: `(customer_id, document_type, is_preferred)` — supports Mini-360 EID/Passport lookup.*

---

### 3.6 PARTY_CONTACT

One row per phone number or email address (`contact_category` discriminates). Address is kept as a dedicated entity (`PARTY_ADDRESS`) due to its richer structure and distinct validation requirements.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `contact_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `contact_category` | ENUM | NOT NULL | `PHONE` \| `EMAIL` |
| `phone_number` | VARCHAR(20) | | Required when `contact_category = PHONE` |
| `phone_type` | VARCHAR(20) | | `MOBILE` \| `HOME` \| `WORK` \| `FAX` |
| `country_code` | VARCHAR(5) | | Dial code e.g. `+971` |
| `area_code` | VARCHAR(5) | | |
| `email_address` | VARCHAR(254) | | Required when `contact_category = EMAIL` |
| `email_type` | VARCHAR(20) | | `PERSONAL` \| `WORK` |
| `is_preferred` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 3.7 PARTY_ADDRESS

One row per postal / physical address.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `address_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `address_type` | VARCHAR(20) | NOT NULL | `HOME` \| `WORK` \| `MAILING` |
| `address_line_1` | VARCHAR(200) | | |
| `address_line_2` | VARCHAR(200) | | |
| `address_line_3` | VARCHAR(200) | | |
| `city` | VARCHAR(100) | | |
| `state_province` | VARCHAR(100) | | |
| `postal_code` | VARCHAR(20) | | |
| `country` | CHAR(3) | | ISO 3166-1 alpha-3 |
| `po_box` | VARCHAR(20) | | |
| `is_preferred` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 3.8 PARTY_CONSENT

Marketing and communication consent preferences. Separated from compliance/regulatory data which belongs in `PARTY_COMPLIANCE`.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `consent_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY UNIQUE NOT NULL | One record per party |
| `marketing_sms` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `marketing_email` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `marketing_phone` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `group_sms` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `group_email` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `group_phone` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `consent_aecb` | BOOLEAN | NOT NULL DEFAULT FALSE | Consent to enquire AECB |
| `consent_source` | VARCHAR(50) | | `BRANCH` \| `DIGITAL` \| `CC` |
| `consent_date` | DATE | | |
| `updated_at` | TIMESTAMP | NOT NULL | |
| `updated_by` | UUID | FK → PARTY | |

---

### 3.9 PARTY_COMPLIANCE

Regulatory and tax compliance data (FATCA / CRS / AML). Kept separate from consent to allow independent governance, access controls, and audit processes.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `compliance_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY UNIQUE NOT NULL | One record per party |
| `us_relation` | BOOLEAN | | FATCA: US person / entity flag |
| `tin_number` | VARCHAR(50) | | Tax Identification Number |
| `fatca_status` | VARCHAR(30) | | e.g. `US_PERSON` \| `NON_US_PERSON` \| `RECALCITRANT` |
| `crs_category` | VARCHAR(30) | | Common Reporting Standard category |
| `crs_criteria` | VARCHAR(200) | | Criteria for classification |
| `criteria_us_entity` | VARCHAR(200) | | Criteria used for US entity classification |
| `updated_at` | TIMESTAMP | NOT NULL | |
| `updated_by` | UUID | FK → PARTY | |

---

### 3.10 PARTY_KYC

Know-Your-Customer record per party.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `kyc_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY UNIQUE NOT NULL | |
| `kyc_held` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `kyc_review_date` | DATE | | |
| `expected_monthly_credit_turnover` | NUMERIC(18,2) | | |
| `expected_monthly_cash_credit_pct` | NUMERIC(5,2) | | % of cash credits |
| `expected_monthly_noncash_credit_pct` | NUMERIC(5,2) | | |
| `kyc_completed_by` | UUID | FK → PARTY | |
| `kyc_completed_date` | DATE | | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 3.11 PARTY_CHANNEL_SUBSCRIPTION

Digital channel enrolment.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `subscription_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY UNIQUE NOT NULL | |
| `internet_banking` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `wap_banking` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `sms_banking` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `phone_banking` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `e_statement` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `e_statement_cards` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `e_statement_accounts` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `e_statement_loans` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `e_statement_deposits` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `email_receipt_atm` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `aani_enrollment_status` | VARCHAR(30) | | `ENROLLED` \| `NOT_ENROLLED` \| `PENDING` |
| `aani_enrollment_date` | DATE | | |
| `online_banking_user_id` | VARCHAR(50) | | |
| `login_enabled` | BOOLEAN | | |
| `transaction_enabled` | BOOLEAN | | |
| `sms_otp_enabled` | BOOLEAN | | |
| `rak_token_status` | VARCHAR(30) | | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 3.12 PARTY_CHANNEL_SUBSCRIPTION_HISTORY

Audit log for online banking / digital channel changes.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `history_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `action` | VARCHAR(50) | NOT NULL | `ENROL` \| `SUSPEND` \| `RESET_PASSWORD` \| `UNLOCK` etc. |
| `reason_code` | VARCHAR(50) | | |
| `action_by` | UUID | FK → PARTY | |
| `action_date` | TIMESTAMP | NOT NULL | |
| `channel` | VARCHAR(30) | | `BRANCH` \| `CC` \| `DIGITAL` |
| `remarks` | VARCHAR(500) | | |

---

### 3.13 PARTY_BLACKLIST_DETAIL

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `blacklist_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `reason_code` | VARCHAR(50) | NOT NULL | |
| `commencement_date` | DATE | NOT NULL | |
| `expiry_date` | DATE | | |
| `status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `EXPIRED` \| `LIFTED` |
| `court_order_date` | DATE | | |
| `court_order_number` | VARCHAR(100) | | |
| `created_by` | UUID | FK → PARTY | |
| `created_at` | TIMESTAMP | NOT NULL | |

---

### 3.14 PARTY_NEGATED_DETAIL

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `negated_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `reason_code` | VARCHAR(50) | NOT NULL | |
| `commencement_date` | DATE | NOT NULL | |
| `status` | VARCHAR(20) | NOT NULL | |
| `unit` | VARCHAR(50) | | Business unit that raised the negation |
| `created_by` | UUID | FK → PARTY | |
| `created_at` | TIMESTAMP | NOT NULL | |

---

### 3.15 PARTY_RELATIONSHIP

Self-referencing table for all inter-party links.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `relationship_id` | UUID | PK | |
| `primary_party_id` | UUID | FK → PARTY NOT NULL | Account holder / main entity |
| `related_party_id` | UUID | FK → PARTY NOT NULL | Related party |
| `relationship_type` | VARCHAR(50) | NOT NULL | `JOINT_HOLDER` \| `GUARANTOR` \| `CO_APPLICANT` \| `AUTHORIZED_REP` \| `DIRECTOR` \| `SHAREHOLDER` \| `GROUP_MEMBER` \| `POWER_OF_ATTORNEY` |
| `bank_relation_type` | VARCHAR(50) | | |
| `relationship_with_entity` | VARCHAR(100) | | e.g. `SPOUSE` \| `PARENT` \| `SIBLING` |
| `entity_type` | VARCHAR(30) | | `INDIVIDUAL` \| `ORGANIZATION` |
| `cif_type` | VARCHAR(30) | | |
| `shareholding_pct` | NUMERIC(5,2) | | Applicable for group structures |
| `start_date` | DATE | | |
| `end_date` | DATE | | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_by` | UUID | FK → PARTY | |

---

### 3.16 AUTHORIZED_REPRESENTATIVE

Per account-level authorizations (separate from customer-level relationships).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `auth_rep_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT | Null if party-level |
| `party_id` | UUID | FK → PARTY | Parent party |
| `rep_party_id` | UUID | FK → PARTY | Representative's CIF (if banked) |
| `rep_name` | VARCHAR(300) | NOT NULL | |
| `rep_designation` | VARCHAR(100) | | |
| `document_type` | VARCHAR(30) | | |
| `document_id` | VARCHAR(50) | | |
| `document_expiry` | DATE | | |
| `can_collect_cheque_book` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `can_collect_debit_card` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `can_do_eft` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `can_do_internal_transfer` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `can_hold_mail` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMP | NOT NULL | |

---

### 3.17 SEGMENT

Lookup / reference table for customer segmentation.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `segment_id` | UUID | PK | |
| `segment_code` | VARCHAR(20) | UNIQUE NOT NULL | |
| `segment_name` | VARCHAR(100) | NOT NULL | |
| `segment_level` | ENUM | NOT NULL | `SEGMENT` \| `SUB_SEGMENT` |
| `parent_segment_id` | UUID | FK → SEGMENT | For sub-segments |
| `industry_segment` | VARCHAR(100) | | |
| `industry_sub_segment` | VARCHAR(100) | | |
| `customer_classification` | VARCHAR(50) | | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |

---

### 3.18 PARTY_EMPLOYEE *(subtype)*

One-to-one with `PARTY` where `party_type = EMPLOYEE`. Bank staff — RMs, branch agents, CC agents. Tracked as parties so their contact details, KYC, and relationships reuse the same party infrastructure.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `party_id` | UUID | PK, FK → PARTY | Shared primary key with PARTY |
| `employee_code` | VARCHAR(20) | UNIQUE NOT NULL | |
| `role` | VARCHAR(30) | NOT NULL | `RM` \| `BRANCH_STAFF` \| `CC_AGENT` \| `SUPERVISOR` \| `ADMIN` |
| `branch_id` | UUID | FK → BRANCH | |
| `business_center_name` | VARCHAR(200) | | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |

---

### 3.19 BRANCH

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `branch_id` | UUID | PK | |
| `branch_code` | VARCHAR(10) | UNIQUE NOT NULL | |
| `branch_name` | VARCHAR(200) | NOT NULL | |
| `sol_id` | VARCHAR(10) | UNIQUE | Finacle SOL identifier |
| `region` | VARCHAR(100) | | |
| `emirate` | VARCHAR(50) | | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |

---

### 3.20 PARTY_PORTFOLIO_SUMMARY

Denormalised Mini-360 portfolio KPIs — refreshed asynchronously.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `portfolio_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY UNIQUE NOT NULL | |
| `portfolio_aggregated_value` | NUMERIC(20,2) | | Total AED equivalent |
| `accounts_count` | INTEGER | | |
| `accounts_total_available_balance` | NUMERIC(20,2) | | |
| `credit_cards_count` | INTEGER | | |
| `credit_card_total_outstanding` | NUMERIC(20,2) | | |
| `credit_card_delinquency` | VARCHAR(10) | | Worst bucket |
| `loans_count` | INTEGER | | Retail loans |
| `loans_total_outstanding` | NUMERIC(20,2) | | |
| `loans_delinquency` | VARCHAR(10) | | |
| `investments_count` | INTEGER | | |
| `investments_total_value` | NUMERIC(20,2) | | |
| `insurance_count` | INTEGER | | |
| `insurance_total_premium` | NUMERIC(20,2) | | Annual |
| `deposits_count` | INTEGER | | |
| `deposits_total_value` | NUMERIC(20,2) | | |
| `lending_count` | INTEGER | | SME/Corporate |
| `lending_total_outstanding` | NUMERIC(20,2) | | |
| `lending_delinquency` | VARCHAR(10) | | |
| `trade_finance_count` | INTEGER | | |
| `trade_finance_total_outstanding` | NUMERIC(20,2) | | |
| `last_refreshed_at` | TIMESTAMP | NOT NULL | |

---

### 3.21 PARTY_ANALYTICS

AI/Analytics-derived data (sentiment, propensity, churn).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `analytics_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY UNIQUE NOT NULL | |
| `sentiment_score` | NUMERIC(5,2) | | −1 to +1 scale |
| `sentiment_label` | VARCHAR(20) | | `POSITIVE` \| `NEUTRAL` \| `NEGATIVE` |
| `churn_risk_score` | NUMERIC(5,2) | | 0–100 |
| `nps_score` | NUMERIC(5,2) | | Net Promoter Score |
| `propensity_to_buy_product` | VARCHAR(100) | | Next best product code |
| `propensity_score` | NUMERIC(5,2) | | |
| `customer_kpi_json` | JSONB | | Flexible KPI bag from analytics |
| `last_refreshed_at` | TIMESTAMP | NOT NULL | |

---

### 3.22 PARTY_PREFERENTIAL_RATE

FX / interest preferential rates offered to high-value parties.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `rate_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `currency` | CHAR(3) | NOT NULL | ISO 4217 |
| `credit_discount_pct` | NUMERIC(6,4) | | |
| `debit_discount_pct` | NUMERIC(6,4) | | |
| `expiry_date` | DATE | | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |

---

### 3.23 PARTY_REFERENCE *(subtype)*

One-to-one with `PARTY` where `party_type = REFERENCE`. Emergency / personal references provided by Individual customers. The reference contact person is registered as a `PARTY` in their own right so their contact details are stored via `PARTY_CONTACT`, and the link to the customer is captured in `PARTY_RELATIONSHIP`.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `party_id` | UUID | PK, FK → PARTY | Shared primary key — the reference contact is a party |
| `referencing_party_id` | UUID | FK → PARTY NOT NULL | The Individual customer who provided this reference |
| `friend_name` | VARCHAR(300) | | Display name (denormalised for convenience) |
| `relationship_to_customer` | VARCHAR(50) | | e.g. `FRIEND` \| `COLLEAGUE` \| `FAMILY` |

---

### 3.24 DOCUMENT

Generic document store — polymorphic (customer, account, loan, limit, etc.).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `document_id` | UUID | PK | |
| `entity_type` | VARCHAR(30) | NOT NULL | `PARTY` \| `ACCOUNT` \| `LOAN` \| `CREDIT_LIMIT` \| `INVESTMENT` |
| `entity_id` | UUID | NOT NULL | FK to the owning entity (polymorphic) |
| `document_code` | VARCHAR(30) | NOT NULL | Finacle document code |
| `document_name` | VARCHAR(200) | | |
| `document_number` | VARCHAR(100) | | |
| `issue_date` | DATE | | |
| `expiry_date` | DATE | | |
| `due_date` | DATE | | For limit documents |
| `document_released_date` | DATE | | |
| `is_preferred` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `storage_reference` | VARCHAR(500) | | Link to DMS / blob storage |
| `created_at` | TIMESTAMP | NOT NULL | |

---

## 4. Domain: Products — Accounts & Cards

### 4.1 PRODUCT *(catalogue)*

Static product definition table.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `product_id` | UUID | PK | |
| `product_code` | VARCHAR(30) | UNIQUE NOT NULL | |
| `product_name` | VARCHAR(200) | NOT NULL | |
| `product_category` | VARCHAR(50) | NOT NULL | `ACCOUNT` \| `DEPOSIT` \| `DEBIT_CARD` \| `CREDIT_CARD` \| `PREPAID_CARD` \| `RETAIL_LOAN` \| `SME_LOAN` \| `ISLAMIC_FINANCE` \| `TRADE_FINANCE` \| `INVESTMENT` \| `INSURANCE` |
| `product_sub_category` | VARCHAR(50) | | e.g. `SAVINGS` \| `CURRENT` \| `AUTO_LOAN` \| `HOME_LOAN` etc. |
| `currency` | CHAR(3) | | Default currency |
| `scheme_code` | VARCHAR(30) | | Finacle scheme code |
| `is_islamic` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |

---

### 4.2 ACCOUNT

Current, savings, and overdraft accounts.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `account_id` | UUID | PK | |
| `account_number` | VARCHAR(20) | UNIQUE NOT NULL | Finacle account ID |
| `iban_number` | VARCHAR(34) | UNIQUE | |
| `party_id` | UUID | FK → PARTY NOT NULL | Primary account holder |
| `product_id` | UUID | FK → PRODUCT NOT NULL | |
| `account_name` | VARCHAR(300) | | |
| `account_status` | VARCHAR(30) | NOT NULL | `ACTIVE` \| `DORMANT` \| `FROZEN` \| `CLOSED` |
| `account_status_date` | DATE | | |
| `currency` | CHAR(3) | NOT NULL | |
| `branch_id` | UUID | FK → BRANCH | |
| `sol_id` | VARCHAR(10) | | |
| `open_date` | DATE | | |
| `close_date` | DATE | | |
| `mode_of_operation` | VARCHAR(50) | | `SINGLE` \| `JOINT_ANY_ONE` \| `JOINT_ALL` |
| `freeze_code_reason_1` | VARCHAR(100) | | |
| `freeze_code_reason_2` | VARCHAR(100) | | |
| `freeze_code_reason_3` | VARCHAR(100) | | |
| `freeze_code_reason_4` | VARCHAR(100) | | |
| `is_staff_account` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 4.3 ACCOUNT_BALANCE

Point-in-time balances (refreshed from core banking).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `balance_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT UNIQUE NOT NULL | |
| `effective_available_balance` | NUMERIC(18,2) | | |
| `clear_balance` | NUMERIC(18,2) | | |
| `hold_amount` | NUMERIC(18,2) | | |
| `float_balance` | NUMERIC(18,2) | | |
| `drawing_power` | NUMERIC(18,2) | | |
| `sanction_limit` | NUMERIC(18,2) | | OD limit |
| `funds_in_clearing` | NUMERIC(18,2) | | |
| `last_updated_at` | TIMESTAMP | NOT NULL | |

---

### 4.4 ACCOUNT_INTEREST_CONFIG

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `interest_config_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT UNIQUE NOT NULL | |
| `pay_interest` | BOOLEAN | | |
| `collect_interest` | BOOLEAN | | |
| `credit_interest_pct_min` | NUMERIC(7,4) | | |
| `credit_interest_pct_max` | NUMERIC(7,4) | | |
| `debit_interest_pct_min` | NUMERIC(7,4) | | |
| `debit_interest_pct_max` | NUMERIC(7,4) | | |
| `credit_interest_amt` | NUMERIC(18,2) | | |
| `last_interest_posted_date_cr` | DATE | | |
| `interest_accrued_cr` | NUMERIC(18,2) | | |
| `debit_interest_amt` | NUMERIC(18,2) | | |
| `interest_accrued_dr` | NUMERIC(18,2) | | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 4.5 ACCOUNT_STATEMENT_CONFIG

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `statement_config_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT UNIQUE NOT NULL | |
| `statement_frequency` | VARCHAR(20) | | `MONTHLY` \| `QUARTERLY` \| `ANNUAL` |
| `dispatch_mode` | VARCHAR(20) | | `EMAIL` \| `POST` \| `BRANCH` |
| `next_print_date` | DATE | | |
| `last_statement_date` | DATE | | |

---

### 4.6 ACCOUNT_TRANSACTION

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `transaction_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT NOT NULL | |
| `transaction_date` | DATE | NOT NULL | |
| `value_date` | DATE | NOT NULL | |
| `posting_date` | DATE | | |
| `transaction_particulars_code` | VARCHAR(20) | | |
| `transaction_particulars` | VARCHAR(500) | | |
| `account_currency` | CHAR(3) | NOT NULL | |
| `transaction_amount` | NUMERIC(18,2) | NOT NULL | |
| `transaction_type` | ENUM | NOT NULL | `DEBIT` \| `CREDIT` |
| `running_balance` | NUMERIC(18,2) | | |
| `channel` | VARCHAR(30) | | `ATM` \| `BRANCH` \| `DIGITAL` \| `POS` \| `CHEQUE` |
| `reference_number` | VARCHAR(50) | | |
| `cheque_number` | VARCHAR(20) | | |
| `created_at` | TIMESTAMP | NOT NULL | |

*Partitioned by `transaction_date` for performance.*

---

### 4.7 ACCOUNT_LIEN

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `lien_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT NOT NULL | |
| `lien_amount` | NUMERIC(18,2) | NOT NULL | |
| `lien_reason` | VARCHAR(200) | | |
| `created_date` | DATE | NOT NULL | |
| `expiry_date` | DATE | | |
| `created_by` | UUID | FK → PARTY | |
| `status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `RELEASED` \| `EXPIRED` |
| `module_type` | VARCHAR(30) | | Source module that created the lien |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 4.8 CHEQUE_BOOK

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `cheque_book_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT NOT NULL | |
| `cheque_type` | VARCHAR(50) | | |
| `number_of_leaves` | INTEGER | | |
| `start_cheque_number` | VARCHAR(20) | | |
| `end_cheque_number` | VARCHAR(20) | | |
| `issue_date` | DATE | | |
| `status` | VARCHAR(20) | | `ACTIVE` \| `EXHAUSTED` \| `CANCELLED` |
| `source` | VARCHAR(30) | | `BRANCH` \| `ATM` \| `DIGITAL` |
| `cheque_name` | VARCHAR(200) | | Name printed on cheques |

---

### 4.9 CHEQUE_LEAF

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `leaf_id` | UUID | PK | |
| `cheque_book_id` | UUID | FK → CHEQUE_BOOK NOT NULL | |
| `account_id` | UUID | FK → ACCOUNT NOT NULL | |
| `cheque_number` | VARCHAR(20) | NOT NULL | |
| `status` | VARCHAR(20) | NOT NULL | `UNUSED` \| `PRESENTED` \| `PAID` \| `RETURNED` \| `STOPPED` \| `CANCELLED` |
| `transaction_date` | DATE | | |
| `cheque_amount` | NUMERIC(18,2) | | |
| `cheque_rejected_date` | DATE | | |
| `return_reason` | VARCHAR(200) | | |
| `user_return_count` | INTEGER | | |
| `system_return_count` | INTEGER | | |

---

### 4.10 STANDING_INSTRUCTION

Covers both account and credit card standing instructions.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `si_id` | UUID | PK | |
| `source_entity_type` | ENUM | NOT NULL | `ACCOUNT` \| `CREDIT_CARD` |
| `source_entity_id` | UUID | NOT NULL | FK polymorphic |
| `target_account_number` | VARCHAR(20) | | |
| `si_type` | VARCHAR(50) | NOT NULL | `INTERNAL_TRANSFER` \| `DIRECT_DEBIT` \| `STANDING_ORDER` \| `UTILITY_PAYMENT` etc. |
| `priority` | INTEGER | | |
| `frequency` | VARCHAR(20) | | `DAILY` \| `WEEKLY` \| `MONTHLY` \| `QUARTERLY` \| `ANNUAL` |
| `next_execution_date` | DATE | | |
| `start_date` | DATE | | |
| `end_date` | DATE | | |
| `amount` | NUMERIC(18,2) | | |
| `currency` | CHAR(3) | | |
| `beneficiary_name` | VARCHAR(300) | | |
| `status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `SUSPENDED` \| `COMPLETED` \| `CANCELLED` |
| `direct_debit_pct` | NUMERIC(5,2) | | For % of bill payments |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 4.11 SALARY_TRANSACTION

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `salary_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT NOT NULL | |
| `salary_date` | DATE | NOT NULL | |
| `salary_type` | VARCHAR(50) | | `WPS` \| `NON_WPS` etc. |
| `salary_amount` | NUMERIC(18,2) | NOT NULL | |
| `salary_month` | CHAR(7) | | `YYYY-MM` |
| `status` | VARCHAR(20) | | |

---

### 4.12 TRANSFER_REMITTANCE

Covers SWIFT, local, instant payments (AANI/FTS), and Quick Remit.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `transfer_id` | UUID | PK | |
| `payment_order_id` | VARCHAR(50) | UNIQUE | |
| `transfer_type` | VARCHAR(30) | NOT NULL | `SWIFT` \| `LOCAL` \| `INSTANT` \| `QUICK_REMIT` \| `INHOUSE` |
| `channel_id` | VARCHAR(30) | | |
| `related_reference` | VARCHAR(50) | | |
| `sender_reference` | VARCHAR(50) | | |
| `debit_account_id` | UUID | FK → ACCOUNT | |
| `credit_account_number` | VARCHAR(34) | | External IBAN / account |
| `debit_amount` | NUMERIC(18,2) | NOT NULL | |
| `credit_amount` | NUMERIC(18,2) | | |
| `debit_currency` | CHAR(3) | NOT NULL | |
| `credit_currency` | CHAR(3) | | |
| `fx_rate` | NUMERIC(12,6) | | |
| `beneficiary_name` | VARCHAR(300) | | |
| `beneficiary_bank_name` | VARCHAR(200) | | |
| `beneficiary_bank_swift` | VARCHAR(11) | | |
| `beneficiary_country` | CHAR(3) | | |
| `purpose_of_payment` | VARCHAR(200) | | |
| `transaction_date` | DATE | NOT NULL | |
| `value_date` | DATE | | |
| `status` | VARCHAR(30) | NOT NULL | `PENDING` \| `PROCESSED` \| `REJECTED` \| `RETURNED` |
| `charge_total_aed` | NUMERIC(18,2) | | |
| `created_at` | TIMESTAMP | NOT NULL | |

---

### 4.13 SWEEP_POOL

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `pool_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT NOT NULL | |
| `pool_description` | VARCHAR(200) | | |
| `currency` | CHAR(3) | | |
| `pool_balance` | NUMERIC(18,2) | | |
| `pool_status` | VARCHAR(20) | | |

---

### 4.14 DEPOSIT

Fixed/term deposits.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `deposit_id` | UUID | PK | |
| `deposit_account_number` | VARCHAR(20) | UNIQUE NOT NULL | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `product_id` | UUID | FK → PRODUCT NOT NULL | |
| `linked_account_id` | UUID | FK → ACCOUNT | Linked current/savings account |
| `currency` | CHAR(3) | NOT NULL | |
| `account_name` | VARCHAR(300) | | |
| `deposit_amount` | NUMERIC(18,2) | NOT NULL | |
| `interest_rate` | NUMERIC(7,4) | | % p.a. |
| `opening_date` | DATE | NOT NULL | |
| `maturity_date` | DATE | | |
| `maturity_amount` | NUMERIC(18,2) | | |
| `renewal_period_months` | INTEGER | | |
| `max_renewals_allowed` | INTEGER | | |
| `renewals_done` | INTEGER | | |
| `auto_renewal` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `auto_closure` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `autorenewal_interest` | VARCHAR(30) | | `CREDIT_TO_ACCOUNT` \| `REINVEST` |
| `status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `MATURED` \| `CLOSED` \| `RENEWED` |
| `preferred_language_code` | VARCHAR(10) | | |
| `account_manager` | UUID | FK → PARTY | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 4.15 CARD *(supertype — shared attributes for all card types)*

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `card_id` | UUID | PK | |
| `card_type` | ENUM | NOT NULL | `DEBIT` \| `CREDIT` \| `PREPAID` \| `DIGITAL_ACCESS` |
| `card_number_masked` | VARCHAR(19) | NOT NULL | e.g. `****-****-****-1234` |
| `card_number_token` | VARCHAR(64) | UNIQUE NOT NULL | Tokenised PAN (never store clear PAN) |
| `party_id` | UUID | FK → PARTY NOT NULL | Primary cardholder |
| `product_id` | UUID | FK → PRODUCT NOT NULL | |
| `card_status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `BLOCKED` \| `EXPIRED` \| `CANCELLED` \| `LOST` \| `STOLEN` |
| `card_expiry_date` | CHAR(5) | | `MM/YY` |
| `issue_date` | DATE | | |
| `last_renew_date` | DATE | | |
| `last_reissue_date` | DATE | | |
| `old_card_number_token` | VARCHAR(64) | | |
| `pin_status` | VARCHAR(20) | | `SET` \| `BLOCKED` \| `NEVER_SET` |
| `pin_last_change_date` | DATE | | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 4.16 DEBIT_CARD *(extends CARD)*

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `card_id` | UUID | PK, FK → CARD | |
| `linked_account_id` | UUID | FK → ACCOUNT NOT NULL | |
| `card_variant` | VARCHAR(50) | | `CLASSIC` \| `GOLD` \| `PLATINUM` |

---

### 4.17 PREPAID_CARD *(extends CARD)*

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `card_id` | UUID | PK, FK → CARD | |
| `linked_account_id` | UUID | FK → ACCOUNT | May be unlinked |
| `prepaid_type` | VARCHAR(30) | | `TRAVEL` \| `GIFT` \| `PAYROLL` |

---

### 4.18 DIGITAL_ACCESS_CARD *(extends CARD)*

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `card_id` | UUID | PK, FK → CARD | |
| `dac_holder` | VARCHAR(30) | | `PRIMARY` \| `SUPPLEMENTARY` |

---

### 4.19 CREDIT_CARD *(extends CARD)*

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `card_id` | UUID | PK, FK → CARD | |
| `card_account_number` | VARCHAR(20) | | Internal credit card account |
| `product_code` | VARCHAR(30) | | |
| `is_vip_normal` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `is_vip_high` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `is_staff_account` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `is_enrolled_falcon` | BOOLEAN | NOT NULL DEFAULT FALSE | Fraud monitoring |
| `falcon_number` | VARCHAR(50) | | |
| `rewards_program` | VARCHAR(50) | | `SKYWARDS` \| `RAKPOINTS` \| `CASHBACK` |
| `customer_profile_status` | VARCHAR(30) | | |
| `authorization_control_profile` | VARCHAR(50) | | |
| `authorization_status` | VARCHAR(20) | | |
| `deactivate_from` | DATE | | Temp deactivation start |
| `deactivate_until` | DATE | | Temp deactivation end |

---

### 4.20 CREDIT_CARD_BALANCE

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `balance_id` | UUID | PK | |
| `card_id` | UUID | FK → CREDIT_CARD UNIQUE NOT NULL | |
| `outstanding_balance` | NUMERIC(18,2) | | |
| `available_funds` | NUMERIC(18,2) | | |
| `credit_limit` | NUMERIC(18,2) | | |
| `cash_limit` | NUMERIC(18,2) | | |
| `unbilled_amount` | NUMERIC(18,2) | | |
| `minimum_due` | NUMERIC(18,2) | | |
| `total_due` | NUMERIC(18,2) | | |
| `due_date` | DATE | | |
| `delinquency_bucket` | VARCHAR(10) | | `CURRENT` \| `1-30` \| `31-60` \| `61-90` \| `90+` |
| `dpd` | INTEGER | | Days Past Due |
| `last_updated_at` | TIMESTAMP | NOT NULL | |

---

### 4.21 CREDIT_CARD_STATEMENT

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `statement_id` | UUID | PK | |
| `card_id` | UUID | FK → CREDIT_CARD NOT NULL | |
| `statement_date` | DATE | NOT NULL | |
| `opening_balance` | NUMERIC(18,2) | | |
| `closing_balance` | NUMERIC(18,2) | | |
| `total_debits` | NUMERIC(18,2) | | |
| `total_credits` | NUMERIC(18,2) | | |
| `minimum_due` | NUMERIC(18,2) | | |
| `total_due` | NUMERIC(18,2) | | |
| `payment_due_date` | DATE | | |
| `quote_for_closure` | NUMERIC(18,2) | | |

---

### 4.22 CARD_AUTHORIZATION_TRANSACTION

Applicable for debit, credit, and prepaid cards.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `auth_transaction_id` | UUID | PK | |
| `card_id` | UUID | FK → CARD NOT NULL | |
| `transaction_datetime` | TIMESTAMP | NOT NULL | |
| `transaction_amount` | NUMERIC(18,2) | NOT NULL | |
| `transaction_currency` | CHAR(3) | NOT NULL | |
| `billing_amount` | NUMERIC(18,2) | | |
| `billing_currency` | CHAR(3) | | |
| `merchant_name` | VARCHAR(300) | | |
| `merchant_category_code` | VARCHAR(10) | | MCC |
| `pos_entry_mode` | VARCHAR(20) | | `CHIP` \| `SWIPE` \| `CONTACTLESS` \| `ONLINE` |
| `authorization_code` | VARCHAR(20) | | |
| `status` | VARCHAR(20) | | `APPROVED` \| `DECLINED` \| `REVERSED` |
| `token_number` | VARCHAR(64) | | If tokenised wallet payment |
| `otp_delivery_ref` | VARCHAR(50) | | 3DS OTP reference |
| `otp_delivery_type` | VARCHAR(20) | | `SMS` \| `EMAIL` |
| `otp_delivery_mobile` | VARCHAR(20) | | |
| `otp_delivery_email` | VARCHAR(254) | | |

---

### 4.23 CARD_TOKEN

Digital wallet tokens (Apple Pay, Google Pay, Samsung Pay).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `token_id` | UUID | PK | |
| `card_id` | UUID | FK → CARD NOT NULL | |
| `token_network` | ENUM | NOT NULL | `MASTERCARD` \| `VISA` |
| `token_type` | VARCHAR(30) | | `DEVICE` \| `COF` \| `HCE` |
| `token_unique_reference` | VARCHAR(64) | UNIQUE NOT NULL | |
| `virtual_card_number_id` | VARCHAR(50) | | Mastercard VCNID |
| `virtual_card_expiry` | CHAR(5) | | |
| `wallet_provider` | VARCHAR(30) | | `APPLE_PAY` \| `GOOGLE_PAY` \| `SAMSUNG_PAY` |
| `wallet_provider_id` | VARCHAR(50) | | |
| `token_requestor` | VARCHAR(50) | | |
| `device_name` | VARCHAR(100) | | |
| `device_type` | VARCHAR(50) | | |
| `serial_number` | VARCHAR(100) | | |
| `token_status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `INACTIVE` \| `SUSPENDED` \| `DELETED` |
| `creation_date` | DATE | | |
| `initial_activation_date` | DATE | | |
| `action_taken_by` | UUID | FK → PARTY | |

---

### 4.24 TOKEN_PROVISIONING

Audit of token provisioning lifecycle steps.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `provisioning_id` | UUID | PK | |
| `card_id` | UUID | FK → CARD NOT NULL | |
| `correlation_id` | VARCHAR(64) | | |
| `eligibility_status` | VARCHAR(30) | | |
| `provisioning_status` | VARCHAR(30) | | |
| `verification_status` | VARCHAR(30) | | |
| `terms_accepted_status` | VARCHAR(30) | | |
| `preparing_status` | VARCHAR(30) | | |
| `digitising_status` | VARCHAR(30) | | |
| `issuer_phone` | VARCHAR(20) | | |
| `issuer_email` | VARCHAR(254) | | |
| `final_provisioning_decision` | VARCHAR(30) | | |
| `issuer_reason_code` | VARCHAR(20) | | |
| `provisioning_datetime` | TIMESTAMP | | |

---

### 4.25 CREDIT_CARD_INSTALLMENT

Buy-now-pay-later / easy payment plans.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `installment_id` | UUID | PK | |
| `card_id` | UUID | FK → CREDIT_CARD NOT NULL | |
| `booked_card_number_token` | VARCHAR(64) | | |
| `description` | VARCHAR(300) | | |
| `merchant_name` | VARCHAR(200) | | |
| `authorization_code` | VARCHAR(20) | | |
| `original_transaction_amount` | NUMERIC(18,2) | | |
| `original_transaction_currency` | CHAR(3) | | |
| `number_of_installments` | INTEGER | | |
| `monthly_installment_amount` | NUMERIC(18,2) | | |
| `interest_rate` | NUMERIC(7,4) | | |
| `installment_type` | VARCHAR(30) | | `QUICK_CASH` \| `CASH_ON_CALL` \| `EASY_PAYMENT` |
| `status` | VARCHAR(20) | | |
| `booking_date` | DATE | | |

---

### 4.26 INSTALLMENT_SCHEDULE

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `schedule_id` | UUID | PK | |
| `installment_id` | UUID | FK → CREDIT_CARD_INSTALLMENT NOT NULL | |
| `sequence_number` | INTEGER | NOT NULL | |
| `initial_amortization_date` | DATE | | |
| `actual_amortization_date` | DATE | | |
| `installment_amount` | NUMERIC(18,2) | | |
| `reason_code` | VARCHAR(20) | | |
| `status` | VARCHAR(20) | | `PENDING` \| `PAID` \| `SKIPPED` |

---

### 4.27 SUPPLEMENTARY_CARD

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `supp_card_id` | UUID | PK | |
| `primary_card_id` | UUID | FK → CREDIT_CARD NOT NULL | |
| `supplementary_card_number_token` | VARCHAR(64) | UNIQUE NOT NULL | |
| `supplementary_party_id` | UUID | FK → PARTY | |
| `supplementary_cif_number` | VARCHAR(20) | | |
| `supplementary_customer_name` | VARCHAR(300) | | |
| `card_expiry_date` | CHAR(5) | | |
| `card_status` | VARCHAR(20) | NOT NULL | |
| `credit_limit` | NUMERIC(18,2) | | |

---

### 4.28 CREDIT_CARD_REWARDS

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `rewards_id` | UUID | PK | |
| `card_id` | UUID | FK → CREDIT_CARD UNIQUE NOT NULL | |
| `bonus_points` | NUMERIC(14,2) | | |
| `spend_points` | NUMERIC(14,2) | | |
| `partner_points` | NUMERIC(14,2) | | |
| `time_based_points` | NUMERIC(14,2) | | |
| `redeemed_points` | NUMERIC(14,2) | | |
| `available_points` | NUMERIC(14,2) | | |
| `expiring_points` | NUMERIC(14,2) | | |
| `expiry_date` | DATE | | |
| `last_updated_at` | TIMESTAMP | NOT NULL | |

---

### 4.29 CREDIT_CARD_INSURANCE

Insurance products attached to a credit card.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `card_insurance_id` | UUID | PK | |
| `card_id` | UUID | FK → CREDIT_CARD NOT NULL | |
| `insurance_name` | VARCHAR(200) | NOT NULL | |
| `activation_date` | DATE | | |
| `insurance_status` | VARCHAR(20) | | `ACTIVE` \| `INACTIVE` \| `CANCELLED` |
| `deactivation_date` | DATE | | |
| `free_cycle_count` | INTEGER | | |

---

## 5. Domain: Products — Lending

### 5.1 RETAIL_LOAN

Personal, auto, and home loans.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `loan_id` | UUID | PK | |
| `agreement_number` | VARCHAR(30) | UNIQUE NOT NULL | |
| `agreement_id` | VARCHAR(30) | UNIQUE | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `product_id` | UUID | FK → PRODUCT NOT NULL | |
| `product_category` | VARCHAR(30) | NOT NULL | `PERSONAL` \| `AUTO` \| `HOME` \| `OTHER` |
| `currency` | CHAR(3) | NOT NULL | |
| `loan_amount` | NUMERIC(18,2) | NOT NULL | |
| `current_outstanding` | NUMERIC(18,2) | | |
| `outstanding_charges` | NUMERIC(18,2) | | |
| `overdue_amount` | NUMERIC(18,2) | | |
| `excess_amount` | NUMERIC(18,2) | | |
| `repayment_account_id` | UUID | FK → ACCOUNT | |
| `repayment_frequency` | VARCHAR(20) | | `MONTHLY` \| `QUARTERLY` |
| `emi_amount` | NUMERIC(18,2) | | |
| `interest_type` | VARCHAR(20) | | `FIXED` \| `FLOATING` |
| `interest_rate` | NUMERIC(7,4) | | |
| `floating_rate_index` | VARCHAR(30) | | e.g. `EIBOR` |
| `fixed_for_months` | INTEGER | | |
| `loan_start_date` | DATE | | |
| `maturity_date` | DATE | | |
| `dpd` | INTEGER | | Days Past Due |
| `delinquency_bucket` | VARCHAR(10) | | |
| `delinquency_string` | VARCHAR(30) | | |
| `loan_status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `CLOSED` \| `WRITTEN_OFF` \| `IN_RECOVERY` |
| `application_receipt_date` | DATE | | |
| `source_branch_id` | UUID | FK → BRANCH | |
| `referrer_code` | VARCHAR(30) | | |
| `referrer_name` | VARCHAR(200) | | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 5.2 RETAIL_LOAN_AUTO_DETAILS

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `auto_detail_id` | UUID | PK | |
| `loan_id` | UUID | FK → RETAIL_LOAN UNIQUE NOT NULL | |
| `vehicle_registration_number` | VARCHAR(20) | | |
| `driving_license` | VARCHAR(30) | | |
| `asset_category` | VARCHAR(50) | | |
| `asset_description` | VARCHAR(200) | | |
| `asset_type` | VARCHAR(50) | | |
| `chassis_number` | VARCHAR(30) | | |
| `engine_number` | VARCHAR(30) | | |
| `model_year` | INTEGER | | |
| `make` | VARCHAR(100) | | |
| `model` | VARCHAR(100) | | |

---

### 5.3 RETAIL_LOAN_HOME_DETAILS

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `home_detail_id` | UUID | PK | |
| `loan_id` | UUID | FK → RETAIL_LOAN UNIQUE NOT NULL | |
| `developer_code` | VARCHAR(30) | | |
| `property_id` | VARCHAR(50) | | |
| `property_type` | VARCHAR(50) | | `APARTMENT` \| `VILLA` \| `TOWNHOUSE` \| `COMMERCIAL` |
| `property_description` | VARCHAR(300) | | |
| `lien_position` | INTEGER | | `1` = first charge |
| `land_department_number` | VARCHAR(50) | | |
| `emirate` | VARCHAR(50) | | |

---

### 5.4 LOAN_GUARANTOR

Supports retail and SME loan guarantors / co-applicants.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `guarantor_id` | UUID | PK | |
| `loan_entity_type` | ENUM | NOT NULL | `RETAIL_LOAN` \| `SME_LOAN` \| `ISLAMIC_FINANCE` |
| `loan_entity_id` | UUID | NOT NULL | Polymorphic FK |
| `applicant_type` | VARCHAR(30) | NOT NULL | `GUARANTOR` \| `CO_APPLICANT` \| `KEYMAN` |
| `party_id` | UUID | FK → PARTY | If existing customer |
| `name` | VARCHAR(300) | | If non-customer |
| `relation` | VARCHAR(50) | | |
| `cif_number` | VARCHAR(20) | | |

---

### 5.5 RETAIL_LOAN_REPAYMENT_SCHEDULE

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `schedule_id` | UUID | PK | |
| `loan_id` | UUID | FK → RETAIL_LOAN NOT NULL | |
| `installment_number` | INTEGER | NOT NULL | |
| `due_date` | DATE | NOT NULL | |
| `opening_principal` | NUMERIC(18,2) | | |
| `installment_amount` | NUMERIC(18,2) | NOT NULL | |
| `principal_component` | NUMERIC(18,2) | | |
| `interest_component` | NUMERIC(18,2) | | |
| `charges_component` | NUMERIC(18,2) | | |
| `balance_after_payment` | NUMERIC(18,2) | | |
| `actual_payment_date` | DATE | | |
| `status` | VARCHAR(20) | | `PENDING` \| `PAID` \| `OVERDUE` \| `WAIVED` |

---

### 5.6 LOAN_EARLY_CLOSURE

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `early_closure_id` | UUID | PK | |
| `loan_entity_type` | ENUM | NOT NULL | `RETAIL_LOAN` \| `SME_LOAN` \| `ISLAMIC_FINANCE` |
| `loan_entity_id` | UUID | NOT NULL | |
| `penalty` | NUMERIC(18,2) | | |
| `penalty_calculated_on` | VARCHAR(50) | | |
| `balance_principal` | NUMERIC(18,2) | | |
| `past_due_installments` | NUMERIC(18,2) | | |
| `other_charges` | NUMERIC(18,2) | | |
| `total_closure_amount` | NUMERIC(18,2) | | |
| `quote_date` | DATE | | |
| `quote_expiry_date` | DATE | | |

---

### 5.7 LOAN_RECEIPT

Payment receipts for retail / Islamic loans.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `receipt_id` | UUID | PK | |
| `loan_entity_type` | ENUM | NOT NULL | `RETAIL_LOAN` \| `ISLAMIC_FINANCE` |
| `loan_entity_id` | UUID | NOT NULL | |
| `advice_id` | VARCHAR(30) | | |
| `transaction_date` | DATE | NOT NULL | |
| `mode_of_transaction` | VARCHAR(30) | | `CASH` \| `CHEQUE` \| `TRANSFER` |
| `cheque_number` | VARCHAR(20) | | |
| `cheque_date` | DATE | | |
| `payment_amount` | NUMERIC(18,2) | NOT NULL | |
| `status` | VARCHAR(20) | | |

---

### 5.8 SME_LOAN

SME and corporate term loans, revolving credit, and OD facilities.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `sme_loan_id` | UUID | PK | |
| `product_number` | VARCHAR(30) | UNIQUE NOT NULL | Finacle account ID |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `product_id` | UUID | FK → PRODUCT NOT NULL | |
| `product_type` | VARCHAR(30) | NOT NULL | `TERM` \| `OD` \| `REVOLVING` \| `DEMAND_LOAN` |
| `currency` | CHAR(3) | NOT NULL | |
| `sanction_amount` | NUMERIC(18,2) | | |
| `disbursed_amount` | NUMERIC(18,2) | | |
| `period_months` | INTEGER | | |
| `period_days` | INTEGER | | |
| `next_payment_date` | DATE | | |
| `next_payment_principal` | NUMERIC(18,2) | | |
| `next_payment_interest` | NUMERIC(18,2) | | |
| `schedule_balance` | NUMERIC(18,2) | | |
| `pay_off_amount` | NUMERIC(18,2) | | |
| `outstanding_selling_price` | NUMERIC(18,2) | | |
| `prepayment_till_date` | NUMERIC(18,2) | | |
| `repricing_plan` | VARCHAR(50) | | |
| `pegging_review_date` | DATE | | |
| `interest_pct_min` | NUMERIC(7,4) | | |
| `interest_pct_max` | NUMERIC(7,4) | | |
| `loan_type` | VARCHAR(30) | | |
| `is_top_up` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `is_auto_reschedule` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `advance_interest_collected` | NUMERIC(18,2) | | |
| `repayment_account_id` | UUID | FK → ACCOUNT | |
| `loan_status` | VARCHAR(20) | NOT NULL | |
| `source_branch_id` | UUID | FK → BRANCH | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 5.9 SME_LOAN_DISBURSEMENT

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `disbursement_id` | UUID | PK | |
| `sme_loan_id` | UUID | FK → SME_LOAN NOT NULL | |
| `disbursement_date` | DATE | NOT NULL | |
| `stage_of_project` | VARCHAR(100) | | For construction loans |
| `currency` | CHAR(3) | | |
| `disbursement_amount` | NUMERIC(18,2) | NOT NULL | |
| `mode_of_payment` | VARCHAR(30) | | `TRANSFER` \| `CHEQUE` \| `BACK_TO_BACK` |
| `credit_account_id` | UUID | FK → ACCOUNT | |

---

### 5.10 CREDIT_LIMIT

Group / customer-level credit facilities (SME/Corporate).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `limit_id` | UUID | PK | |
| `limit_reference` | VARCHAR(50) | UNIQUE NOT NULL | Finacle Limit ID |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `sme_loan_id` | UUID | FK → SME_LOAN | Associated loan |
| `level_number` | VARCHAR(20) | | Hierarchy level in credit structure |
| `serial_number` | INTEGER | | |
| `sanction_amount_limit_ccy` | NUMERIC(18,2) | | |
| `sanction_amount_home_ccy` | NUMERIC(18,2) | | AED equivalent |
| `limit_currency` | CHAR(3) | | |
| `fund_liability` | NUMERIC(18,2) | | |
| `non_fund_liability` | NUMERIC(18,2) | | |
| `total_liability` | NUMERIC(18,2) | | |
| `utilized_limit` | NUMERIC(18,2) | | |
| `available_limit` | NUMERIC(18,2) | | |
| `expiry_date` | DATE | | |
| `review_date` | DATE | | |
| `limit_type` | VARCHAR(30) | | `FUNDED` \| `NON_FUNDED` \| `MIXED` |
| `collateral_code` | VARCHAR(30) | | |
| `status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `EXPIRED` \| `CANCELLED` |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 5.11 CREDIT_LIMIT_COLLATERAL

Collateral linked to a credit limit.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `collateral_link_id` | UUID | PK | |
| `limit_id` | UUID | FK → CREDIT_LIMIT NOT NULL | |
| `collateral_code` | VARCHAR(30) | | |
| `collateral_type` | VARCHAR(50) | | `PROPERTY` \| `CASH` \| `SECURITY` \| `GUARANTEE` \| `VEHICLE` |
| `collateral_description` | VARCHAR(300) | | |
| `collateral_value` | NUMERIC(18,2) | | |
| `collateral_currency` | CHAR(3) | | |
| `account_number` | VARCHAR(20) | | If cash collateral |
| `sol_id` | VARCHAR(10) | | |
| `expiry_date` | DATE | | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |

---

### 5.12 OD_FACILITY

Overdraft facility details (sub-entity of SME_LOAN or ACCOUNT).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `od_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT | |
| `sme_loan_id` | UUID | FK → SME_LOAN | |
| `application_from_date` | DATE | NOT NULL | |
| `expiry_date` | DATE | | |
| `sanction_limit` | NUMERIC(18,2) | | |
| `interest_rate` | NUMERIC(7,4) | | |
| `limit_level_interest` | NUMERIC(7,4) | | |
| `sanction_date` | DATE | | |
| `event` | VARCHAR(50) | | |
| `supercede_record` | BOOLEAN | | |
| `status` | VARCHAR(20) | | |

---

### 5.13 LOAN_RECOVERY_TRANSACTION

Recovery entries for defaulted loans.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `recovery_id` | UUID | PK | |
| `loan_entity_type` | ENUM | NOT NULL | `RETAIL_LOAN` \| `SME_LOAN` \| `ISLAMIC_FINANCE` |
| `loan_entity_id` | UUID | NOT NULL | |
| `transaction_date` | DATE | NOT NULL | |
| `value_date` | DATE | | |
| `transaction_internal_id` | VARCHAR(50) | | |
| `currency` | CHAR(3) | | |
| `transaction_type` | VARCHAR(50) | | |
| `transaction_amount` | NUMERIC(18,2) | NOT NULL | |
| `remarks` | VARCHAR(500) | | |

---

### 5.14 ISLAMIC_FINANCE

Shariah-compliant financing products (Murabaha, Ijara, Diminishing Musharakah, etc.).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `islamic_finance_id` | UUID | PK | |
| `agreement_id` | VARCHAR(30) | UNIQUE NOT NULL | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `product_id` | UUID | FK → PRODUCT NOT NULL | |
| `scheme` | VARCHAR(50) | | `MURABAHA` \| `IJARA` \| `DIM_MUSHARAKAH` \| `WAKALA` |
| `currency` | CHAR(3) | NOT NULL | |
| `maturity_date` | DATE | | |
| `total_finance_amount` | NUMERIC(18,2) | NOT NULL | Selling price |
| `total_outstanding_account_ccy` | NUMERIC(18,2) | | |
| `total_outstanding_aed` | NUMERIC(18,2) | | |
| `overdue_amount` | NUMERIC(18,2) | | |
| `profit_rate` | NUMERIC(7,4) | | |
| `asset_type` | VARCHAR(50) | | |
| `registration_number` | VARCHAR(50) | | |
| `insured` | BOOLEAN | | |
| `finance_status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `CLOSED` \| `DEFAULTED` |
| `repayment_account_id` | UUID | FK → ACCOUNT | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 5.15 TRADE_FINANCE

SME/Corporate trade finance instruments (LCs, LGs, etc.).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `trade_finance_id` | UUID | PK | |
| `product_number` | VARCHAR(30) | UNIQUE NOT NULL | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `product_id` | UUID | FK → PRODUCT NOT NULL | |
| `scheme_code` | VARCHAR(30) | | |
| `scheme_description` | VARCHAR(200) | | |
| `instrument_type` | VARCHAR(50) | | `LC_IMPORT` \| `LC_EXPORT` \| `LG` \| `BILL_DISCOUNTING` \| `FORFAITING` |
| `currency` | CHAR(3) | NOT NULL | |
| `amount` | NUMERIC(18,2) | NOT NULL | |
| `outstanding_amount` | NUMERIC(18,2) | | |
| `issue_date` | DATE | | |
| `expiry_date` | DATE | | |
| `beneficiary_name` | VARCHAR(300) | | |
| `status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `EXPIRED` \| `CLOSED` |
| `created_at` | TIMESTAMP | NOT NULL | |

---

## 6. Domain: Products — Investments & Insurance

### 6.1 INVESTMENT_PORTFOLIO

Customer-level investment wrapper.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `investment_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY UNIQUE NOT NULL | |
| `risk_profile_category` | VARCHAR(30) | | `CONSERVATIVE` \| `MODERATE` \| `AGGRESSIVE` |
| `final_risk_profile_category` | VARCHAR(30) | | |
| `risk_profile_score` | NUMERIC(5,2) | | |
| `final_risk_score` | NUMERIC(5,2) | | |
| `client_classification` | VARCHAR(30) | | `RETAIL` \| `PROFESSIONAL` |
| `total_investment_value_aed` | NUMERIC(20,2) | | |
| `last_refreshed_at` | TIMESTAMP | | |
| `created_at` | TIMESTAMP | NOT NULL | |

---

### 6.2 INVESTMENT_ASSET

One row per holding, per asset class.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `asset_id` | UUID | PK | |
| `investment_id` | UUID | FK → INVESTMENT_PORTFOLIO NOT NULL | |
| `asset_class` | VARCHAR(30) | NOT NULL | `MUTUAL_FUND` \| `EQUITY` \| `BOND` \| `STRUCTURED_PRODUCT` \| `FUTURES` \| `PRECIOUS_METAL` \| `PMS` \| `ALTERNATE` |
| `product_name` | VARCHAR(300) | NOT NULL | |
| `product_code` | VARCHAR(50) | | |
| `product_currency` | CHAR(3) | | |
| `current_investment_value` | NUMERIC(18,2) | | |
| `pct_of_portfolio` | NUMERIC(6,2) | | |
| `quantity` | NUMERIC(18,4) | | Units / shares / contracts |
| `quantity_unit` | VARCHAR(20) | | `UNITS` \| `SHARES` \| `GRAMS` \| `CONTRACTS` |
| `average_cost` | NUMERIC(18,6) | | |
| `unrealized_gain_loss` | NUMERIC(18,2) | | |
| `realized_gain_loss` | NUMERIC(18,2) | | |
| `valuation_date` | DATE | | |
| -- *Mutual Fund specific* | | | |
| `fund_risk_profile` | VARCHAR(30) | | |
| `fund_currency` | CHAR(3) | | |
| `nav` | NUMERIC(14,6) | | Net Asset Value per unit |
| -- *Bond specific* | | | |
| `accrued_interest` | NUMERIC(18,2) | | |
| `coupon_rate` | NUMERIC(7,4) | | |
| `maturity_date` | DATE | | |
| `bond_yield` | NUMERIC(7,4) | | |
| -- *Equity specific* | | | |
| `custody_account_number` | VARCHAR(30) | | |
| `equity_category` | VARCHAR(50) | | `ADX` \| `DFM` \| `INTERNATIONAL` |
| -- *Futures specific* | | | |
| `contract_currency` | CHAR(3) | | |
| `contract_size` | NUMERIC(18,4) | | |
| `number_of_contracts` | INTEGER | | |
| `holding_type` | VARCHAR(20) | | `LONG` \| `SHORT` |
| -- *Precious Metal specific* | | | |
| `product_type` | VARCHAR(50) | | `GOLD` \| `SILVER` \| `PLATINUM` |
| `product_subtype` | VARCHAR(50) | | `BAR` \| `COIN` |
| `product_variant` | VARCHAR(50) | | |
| -- *Alternate Investment specific* | | | |
| `pe_held_away_id` | VARCHAR(50) | | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 6.3 INVESTMENT_INSURANCE

Investment-linked insurance products (unit-linked, endowment).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `inv_insurance_id` | UUID | PK | |
| `asset_id` | UUID | FK → INVESTMENT_ASSET NOT NULL | |
| `policy_number` | VARCHAR(50) | UNIQUE NOT NULL | |
| `product_type` | VARCHAR(50) | | |
| `product_currency` | CHAR(3) | | |
| `basic_sum_assured` | NUMERIC(18,2) | | |
| `application_number` | VARCHAR(50) | | |
| `insurer_code` | VARCHAR(30) | | |
| `channel_id` | VARCHAR(30) | | |
| `premium_payment_frequency` | VARCHAR(20) | | `MONTHLY` \| `QUARTERLY` \| `ANNUAL` |
| `annual_premium_amount` | NUMERIC(18,2) | | |
| `policy_term_years` | INTEGER | | |
| `next_recurring_premium_due_date` | DATE | | |
| `recurring_premium_amount` | NUMERIC(18,2) | | |
| `auto_premium` | BOOLEAN | | |
| `payment_mode` | VARCHAR(30) | | `DIRECT_DEBIT` \| `CC` \| `CASH` |
| `payment_account_id` | UUID | FK → ACCOUNT | |

---

### 6.4 INSURANCE_POLICY

Standalone (non-investment) insurance policies.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `policy_id` | UUID | PK | |
| `policy_number` | VARCHAR(50) | UNIQUE NOT NULL | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `product_id` | UUID | FK → PRODUCT NOT NULL | |
| `product_name` | VARCHAR(200) | NOT NULL | |
| `plan_name` | VARCHAR(200) | | |
| `policy_status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `LAPSED` \| `CANCELLED` \| `MATURED` \| `CLAIMED` |
| `policy_currency` | CHAR(3) | | |
| `policy_type` | VARCHAR(50) | | `LIFE` \| `HEALTH` \| `TRAVEL` \| `HOME` \| `VEHICLE` \| `CREDIT_LIFE` |
| `isp_name` | VARCHAR(200) | | Insurance Service Provider |
| `plan_term_years` | INTEGER | | |
| `monthly_premium` | NUMERIC(14,2) | | |
| `annual_premium` | NUMERIC(14,2) | | |
| `basic_sum_assured` | NUMERIC(18,2) | | |
| `total_sum_insured` | NUMERIC(18,2) | | |
| `start_date` | DATE | | |
| `end_date` | DATE | | |
| `business_cycle` | VARCHAR(30) | | |
| `business_cycle_year` | INTEGER | | |
| `third_party_payment` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `rm_id` | UUID | FK → PARTY | |
| `maker_id` | UUID | FK → PARTY | |
| `accidental_death_benefit` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `ptd_benefit` | BOOLEAN | NOT NULL DEFAULT FALSE | Permanent Total Disability |
| `terminal_illness_benefit` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `critical_illness_benefit` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 6.5 INSURANCE_BENEFICIARY

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `beneficiary_id` | UUID | PK | |
| `policy_id` | UUID | FK → INSURANCE_POLICY NOT NULL | |
| `beneficiary_name` | VARCHAR(300) | NOT NULL | |
| `relationship` | VARCHAR(50) | | |
| `percentage` | NUMERIC(5,2) | | |
| `covered_member_name` | VARCHAR(300) | | For health policies |
| `sum_covered` | NUMERIC(18,2) | | |

---

### 6.6 INSURANCE_PAYMENT

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `payment_id` | UUID | PK | |
| `policy_id` | UUID | FK → INSURANCE_POLICY NOT NULL | |
| `payment_sequence` | INTEGER | | `1` = first premium |
| `payment_mode` | VARCHAR(30) | | `DIRECT_DEBIT` \| `CC` \| `CASH` \| `CHEQUE` |
| `payment_account_id` | UUID | FK → ACCOUNT | |
| `payment_cc_id` | UUID | FK → CREDIT_CARD | |
| `payment_account_currency` | CHAR(3) | | |
| `next_processing_date` | DATE | | |
| `retry_count` | INTEGER | | |
| `amount` | NUMERIC(14,2) | | |
| `status` | VARCHAR(20) | | `PENDING` \| `PROCESSED` \| `FAILED` |

---

### 6.7 INSURANCE_TRANSACTION

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `ins_transaction_id` | UUID | PK | |
| `policy_id` | UUID | FK → INSURANCE_POLICY NOT NULL | |
| `transaction_date` | DATE | NOT NULL | |
| `transaction_type` | VARCHAR(50) | | `PREMIUM` \| `CLAIM` \| `SURRENDER` \| `MATURITY` \| `BONUS` |
| `mode` | VARCHAR(30) | | |
| `account_cc_number` | VARCHAR(30) | | |
| `amount` | NUMERIC(14,2) | | |
| `status` | VARCHAR(20) | | |

---

## 7. Domain: CRM Operations

### 7.1 INTERACTION

Customer interactions — footprints, communications, and online banking requests.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `interaction_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `interaction_type` | VARCHAR(30) | NOT NULL | `FOOTPRINT` \| `COMMUNICATION` \| `OB_REQUEST` |
| `channel` | VARCHAR(30) | NOT NULL | `BRANCH` \| `CC` \| `DIGITAL` \| `ATM` \| `MOBILE` \| `CHAT` |
| `interaction_datetime` | TIMESTAMP | NOT NULL | |
| `priority` | VARCHAR(10) | | `LOW` \| `MEDIUM` \| `HIGH` |
| `category` | VARCHAR(100) | | |
| `description` | TEXT | | |
| `event_name` | VARCHAR(200) | | For communications |
| `content` | TEXT | | Email / SMS / letter body |
| `address_to` | VARCHAR(300) | | Recipient address |
| `reference_id` | VARCHAR(50) | | OB request reference |
| `request_type` | VARCHAR(100) | | |
| `request_nature` | VARCHAR(50) | | |
| `status` | VARCHAR(20) | | `OPEN` \| `CLOSED` \| `PENDING` |
| `created_by` | UUID | FK → PARTY | |
| `created_at` | TIMESTAMP | NOT NULL | |

---

### 7.2 INTERACTION_ADDITIONAL_INFO

Key-value bag for flexible OB request additional fields.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `info_id` | UUID | PK | |
| `interaction_id` | UUID | FK → INTERACTION NOT NULL | |
| `field_name` | VARCHAR(100) | NOT NULL | |
| `field_value` | VARCHAR(1000) | | |

---

### 7.3 INTERACTION_HISTORY

Status changes for online banking requests.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `history_id` | UUID | PK | |
| `interaction_id` | UUID | FK → INTERACTION NOT NULL | |
| `status` | VARCHAR(20) | NOT NULL | |
| `remarks` | VARCHAR(500) | | |
| `action_by` | UUID | FK → PARTY | |
| `processed_datetime` | TIMESTAMP | NOT NULL | |
| `channel_id` | VARCHAR(30) | | |

---

### 7.4 SERVICE_REQUEST

CRM service requests (servicing tasks assigned to staff).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `service_request_id` | UUID | PK | |
| `reference_number` | VARCHAR(30) | UNIQUE NOT NULL | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `request_type` | VARCHAR(100) | NOT NULL | |
| `request_sub_type` | VARCHAR(100) | | |
| `assigned_group` | VARCHAR(100) | | |
| `assigned_to` | UUID | FK → PARTY | |
| `channel` | VARCHAR(30) | | `BRANCH` \| `CC` \| `DIGITAL` \| `EMAIL` |
| `priority` | VARCHAR(10) | | `LOW` \| `MEDIUM` \| `HIGH` \| `URGENT` |
| `status` | VARCHAR(30) | NOT NULL | `OPEN` \| `IN_PROGRESS` \| `PENDING_CUSTOMER` \| `RESOLVED` \| `CLOSED` \| `CANCELLED` |
| `sla_due_datetime` | TIMESTAMP | | |
| `created_datetime` | TIMESTAMP | NOT NULL | |
| `last_updated_datetime` | TIMESTAMP | NOT NULL | |
| `closed_datetime` | TIMESTAMP | | |
| `resolution` | TEXT | | |
| `created_by` | UUID | FK → PARTY | |

---

### 7.5 LEAD

Sales leads for existing customers and prospects.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `lead_id` | UUID | PK | |
| `lead_reference_number` | VARCHAR(30) | UNIQUE NOT NULL | |
| `party_id` | UUID | FK → PARTY | Null if prospect |
| `customer_cif` | VARCHAR(20) | | |
| `prospect_cif` | VARCHAR(20) | | |
| `product_id` | UUID | FK → PRODUCT | |
| `product_code` | VARCHAR(30) | | |
| `lead_status` | VARCHAR(30) | NOT NULL | `NEW` \| `CONTACTED` \| `QUALIFIED` \| `CONVERTED` \| `LOST` \| `DECLINED` |
| `lead_source` | VARCHAR(50) | | `BRANCH` \| `CC` \| `DIGITAL` \| `RM` \| `ANALYTICS` \| `CAMPAIGN` |
| `lead_assigned_to` | UUID | FK → PARTY | |
| `campaign_id` | VARCHAR(50) | | |
| `expected_amount` | NUMERIC(18,2) | | |
| `currency` | CHAR(3) | | |
| `remarks` | TEXT | | |
| `created_datetime` | TIMESTAMP | NOT NULL | |
| `last_updated_datetime` | TIMESTAMP | NOT NULL | |
| `closed_datetime` | TIMESTAMP | | |
| `converted_datetime` | TIMESTAMP | | |
| `created_by` | UUID | FK → PARTY | |

---

### 7.6 PARTY_PACKAGE

RAKValue / loyalty packages.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `package_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `package_name` | VARCHAR(100) | NOT NULL | |
| `enrollment_account_id` | UUID | FK → ACCOUNT | |
| `status` | VARCHAR(20) | NOT NULL | `ACTIVE` \| `EXPIRED` \| `CANCELLED` |
| `last_activity_date` | DATE | | |
| `created_at` | TIMESTAMP | NOT NULL | |

---

### 7.7 PACKAGE_CHANGE_HISTORY

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `change_id` | UUID | PK | |
| `package_id` | UUID | FK → PARTY_PACKAGE NOT NULL | |
| `old_package_name` | VARCHAR(100) | | |
| `new_package_name` | VARCHAR(100) | | |
| `date_of_change` | DATE | NOT NULL | |
| `change_type` | VARCHAR(30) | | `UPGRADE` \| `DOWNGRADE` \| `RENEWAL` \| `CANCELLATION` |

---

### 7.8 PACKAGE_UTILIZATION

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `utilization_id` | UUID | PK | |
| `package_id` | UUID | FK → PARTY_PACKAGE NOT NULL | |
| `utilization_type` | ENUM | NOT NULL | `FINANCIAL` \| `NON_FINANCIAL` |
| `transaction_type` | VARCHAR(50) | | |
| `description` | VARCHAR(200) | | |
| `frequency` | VARCHAR(20) | | `MONTHLY` \| `ANNUAL` |
| `eligible_at_cif_level` | NUMERIC(10,2) | | |
| `availed_at_account_level` | NUMERIC(10,2) | | |
| `remaining` | NUMERIC(10,2) | | |
| `period_start_date` | DATE | | |
| `period_end_date` | DATE | | |

---

### 7.9 MEMO

Internal memos attached to accounts, credit cards, and other entities.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `memo_id` | UUID | PK | |
| `entity_type` | VARCHAR(30) | NOT NULL | `ACCOUNT` \| `CREDIT_CARD` \| `LOAN` |
| `entity_id` | UUID | NOT NULL | Polymorphic FK |
| `memo_date` | TIMESTAMP | NOT NULL | |
| `memo_message` | TEXT | NOT NULL | |
| `memo_user` | UUID | FK → PARTY | |
| `memo_type` | VARCHAR(30) | | |
| `topic` | VARCHAR(100) | | |
| `function` | VARCHAR(50) | | |
| `intent` | VARCHAR(50) | | |

---

### 7.10 ESTATEMENT_DELIVERY

eStatement delivery tracking.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `delivery_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `record_id` | VARCHAR(50) | | |
| `product_entity_type` | VARCHAR(30) | | `ACCOUNT` \| `CREDIT_CARD` \| `LOAN` \| `DEPOSIT` |
| `product_number` | VARCHAR(30) | | |
| `email_address` | VARCHAR(254) | | |
| `send_date` | DATE | | |
| `delivery_date` | DATE | | |
| `delivery_status` | VARCHAR(20) | | `SENT` \| `DELIVERED` \| `BOUNCED` \| `FAILED` |
| `delivery_channel` | VARCHAR(20) | | `EMAIL` \| `SMS` |
| `retry_count` | INTEGER | | |

---

### 7.11 TAX_INVOICE_DELIVERY

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `tax_delivery_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `tax_invoice_number` | VARCHAR(50) | | |
| `tax_credit_note_number` | VARCHAR(50) | | |
| `email_address` | VARCHAR(254) | | |
| `invoice_sent_date` | DATE | | |
| `credit_note_sent_date` | DATE | | |
| `delivery_status` | VARCHAR(20) | | |

---

### 7.12 PHYSICAL_DELIVERY

Tracking of card / cheque book / document physical deliveries.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `delivery_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `booking_date` | DATE | NOT NULL | |
| `booking_number` | VARCHAR(50) | UNIQUE NOT NULL | |
| `delivery_type` | VARCHAR(30) | | `CARD` \| `CHEQUE_BOOK` \| `LETTER` \| `DOCUMENT` |
| `current_status` | VARCHAR(30) | | |
| `receiver_name` | VARCHAR(300) | | |
| `receiver_address` | TEXT | | |
| `awb_number` | VARCHAR(50) | | Airway bill / tracking number |
| `courier_name` | VARCHAR(100) | | |
| `shipment_status` | VARCHAR(30) | | |
| `created_date` | DATE | | |
| `delivered_date` | DATE | | |

---

### 7.13 PAYMENT_ENQUIRY

Loan demand / payment schedule enquiry (shared across loan types).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `payment_enquiry_id` | UUID | PK | |
| `loan_entity_type` | ENUM | NOT NULL | `RETAIL_LOAN` \| `SME_LOAN` \| `ISLAMIC_FINANCE` |
| `loan_entity_id` | UUID | NOT NULL | |
| `demand_type` | VARCHAR(50) | | |
| `flow_id` | VARCHAR(30) | | |
| `demand_effective_date` | DATE | | |
| `installment_amount` | NUMERIC(18,2) | | |
| `collection_amount` | NUMERIC(18,2) | | |
| `outstanding_demand` | NUMERIC(18,2) | | |

---

## 8. Domain: Enquiries & Calculators

### 8.1 DISPUTE_FORM

Disputes raised on cards or accounts.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `dispute_id` | UUID | PK | |
| `dispute_number` | VARCHAR(30) | UNIQUE NOT NULL | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `card_id` | UUID | FK → CARD | |
| `account_id` | UUID | FK → ACCOUNT | |
| `dispute_reason` | VARCHAR(200) | NOT NULL | |
| `report_sent_by` | VARCHAR(30) | | `EMAIL` \| `POST` \| `FAX` |
| `raised_datetime` | TIMESTAMP | NOT NULL | |
| `status` | VARCHAR(20) | NOT NULL | `OPEN` \| `UNDER_INVESTIGATION` \| `RESOLVED` \| `CLOSED` |

---

### 8.2 DIGITAL_ONBOARDING

Digital onboarding application records (prospect to customer).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `onboarding_id` | UUID | PK | |
| `prospect_id` | VARCHAR(30) | UNIQUE NOT NULL | |
| `mobile_number` | VARCHAR(20) | NOT NULL | |
| `product_type` | VARCHAR(50) | | |
| `sub_product_type` | VARCHAR(50) | | |
| `work_item_created_date` | TIMESTAMP | | |
| `application_status` | VARCHAR(30) | NOT NULL | `STARTED` \| `IN_PROGRESS` \| `SUBMITTED` \| `APPROVED` \| `REJECTED` \| `ABANDONED` |
| `cheque_book_required` | BOOLEAN | | |
| `debit_card_required` | BOOLEAN | | |
| `is_stp` | BOOLEAN | | Straight-through processing |
| `converted_party_id` | UUID | FK → PARTY | Once onboarded |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

---

### 8.3 CARD_ONBOARDING_APPLICATION

Physical credit card applications processed by CC / branch.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `application_id` | UUID | PK | |
| `application_number` | VARCHAR(30) | UNIQUE NOT NULL | |
| `party_id` | UUID | FK → PARTY | Null if new applicant |
| `cif_number` | VARCHAR(20) | | |
| `customer_name` | VARCHAR(300) | | |
| `product_id` | UUID | FK → PRODUCT NOT NULL | |
| `application_status` | VARCHAR(30) | NOT NULL | |
| `passport_number` | VARCHAR(20) | | |
| `passport_expiry_date` | DATE | | |
| `date_of_birth` | DATE | | |
| `gender` | VARCHAR(10) | | |
| `nationality` | CHAR(3) | | |
| `created_at` | TIMESTAMP | NOT NULL | |

---

### 8.4 AVERAGE_BALANCE_ENQUIRY

Historical monthly average balance per account.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `avg_balance_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT NOT NULL | |
| `month` | CHAR(7) | NOT NULL | `YYYY-MM` |
| `average_balance` | NUMERIC(18,2) | | |
| `maximum_balance` | NUMERIC(18,2) | | |
| `maximum_balance_days` | INTEGER | | |
| `minimum_balance` | NUMERIC(18,2) | | |
| `minimum_balance_days` | INTEGER | | |

*Composite unique key: `(account_id, month)`*

---

### 8.5 TURNOVER_STATISTICS

Monthly account turnover statistics.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `turnover_id` | UUID | PK | |
| `account_id` | UUID | FK → ACCOUNT NOT NULL | |
| `month` | CHAR(7) | NOT NULL | `YYYY-MM` |
| `debit_turnover` | NUMERIC(18,2) | | |
| `debit_entries` | INTEGER | | |
| `credit_turnover` | NUMERIC(18,2) | | |
| `credit_entries` | INTEGER | | |

---

### 8.6 CALCULATOR_SESSION

Captures inputs and outputs from CRM calculators (for audit and lead follow-up).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `session_id` | UUID | PK | |
| `party_id` | UUID | FK → PARTY NOT NULL | |
| `calculator_type` | VARCHAR(30) | NOT NULL | `CC_QUICK_CASH` \| `CC_CASH_ON_CALL` \| `PERSONAL_LOAN` \| `FX_RATE` |
| `input_json` | JSONB | NOT NULL | Inputs (principle, tenure, rate etc.) |
| `output_json` | JSONB | | Results (EMI, total interest, schedule etc.) |
| `used_by` | UUID | FK → PARTY | |
| `created_at` | TIMESTAMP | NOT NULL | |

---

## 9. Relationships Summary

| Relationship | Cardinality | Notes |
|---|---|---|
| PARTY → PARTY_INDIVIDUAL | 1:0..1 | Subtype — joined on `party_id` |
| PARTY → PARTY_ORGANIZATION | 1:0..1 | Subtype — joined on `party_id` |
| PARTY → PARTY_EMPLOYEE | 1:0..1 | Subtype — joined on `party_id` |
| PARTY → PARTY_REFERENCE | 1:0..1 | Subtype — joined on `party_id` |
| PARTY_ORGANIZATION → CORPORATE_GROUP | M:1 | Group-level structure |
| PARTY → PARTY_IDENTIFICATION | 1:M | Multiple ID documents per CIF |
| PARTY → PARTY_CONTACT | 1:M | Phone numbers and email addresses (contact_category discriminates) |
| PARTY → PARTY_ADDRESS | 1:M | Multiple physical/postal addresses |
| PARTY → PARTY_CONSENT | 1:1 | Single marketing consent record per party |
| PARTY → PARTY_COMPLIANCE | 1:1 | Single regulatory compliance record per party |
| PARTY → PARTY_KYC | 1:1 | |
| PARTY → PARTY_CHANNEL_SUBSCRIPTION | 1:1 | |
| PARTY → PARTY_RELATIONSHIP | M:M (self) | Via junction table |
| PARTY → PARTY_PORTFOLIO_SUMMARY | 1:1 | Materialised for Mini-360 |
| PARTY → PARTY_ANALYTICS | 1:1 | AI/analytics derived |
| PARTY → SEGMENT | M:1 | Segment + sub-segment |
| PARTY → PARTY_EMPLOYEE (RM) | M:1 | Primary and secondary RMs (employee is a PARTY subtype) |
| PARTY → ACCOUNT | 1:M | |
| ACCOUNT → ACCOUNT_BALANCE | 1:1 | |
| ACCOUNT → ACCOUNT_TRANSACTION | 1:M | Partitioned |
| ACCOUNT → ACCOUNT_LIEN | 1:M | |
| ACCOUNT → CHEQUE_BOOK → CHEQUE_LEAF | 1:M:M | |
| ACCOUNT → STANDING_INSTRUCTION | 1:M | |
| ACCOUNT → TRANSFER_REMITTANCE | 1:M | As debit account |
| PARTY → DEPOSIT | 1:M | |
| PARTY → CARD (DEBIT/PREPAID/CREDIT/DAC) | 1:M | Via CARD supertype |
| CARD → CARD_AUTHORIZATION_TRANSACTION | 1:M | |
| CARD → CARD_TOKEN | 1:M | Multiple wallets |
| CREDIT_CARD → CREDIT_CARD_BALANCE | 1:1 | |
| CREDIT_CARD → CREDIT_CARD_STATEMENT | 1:M | Monthly |
| CREDIT_CARD → CREDIT_CARD_INSTALLMENT → INSTALLMENT_SCHEDULE | 1:M:M | |
| CREDIT_CARD → SUPPLEMENTARY_CARD | 1:M | |
| PARTY → RETAIL_LOAN | 1:M | |
| RETAIL_LOAN → RETAIL_LOAN_AUTO_DETAILS | 1:0..1 | |
| RETAIL_LOAN → RETAIL_LOAN_HOME_DETAILS | 1:0..1 | |
| RETAIL_LOAN → RETAIL_LOAN_REPAYMENT_SCHEDULE | 1:M | |
| PARTY → SME_LOAN | 1:M | |
| SME_LOAN → CREDIT_LIMIT | 1:M | |
| CREDIT_LIMIT → CREDIT_LIMIT_COLLATERAL | 1:M | |
| PARTY → ISLAMIC_FINANCE | 1:M | |
| PARTY → TRADE_FINANCE | 1:M | |
| PARTY → INVESTMENT_PORTFOLIO | 1:1 | |
| INVESTMENT_PORTFOLIO → INVESTMENT_ASSET | 1:M | |
| INVESTMENT_ASSET → INVESTMENT_INSURANCE | 1:0..1 | Investment-linked insurance |
| PARTY → INSURANCE_POLICY | 1:M | |
| INSURANCE_POLICY → INSURANCE_BENEFICIARY | 1:M | |
| INSURANCE_POLICY → INSURANCE_PAYMENT | 1:M | |
| INSURANCE_POLICY → INSURANCE_TRANSACTION | 1:M | |
| PARTY → INTERACTION | 1:M | Footprints, comms |
| PARTY → SERVICE_REQUEST | 1:M | |
| PARTY → LEAD | 1:M | |
| PARTY → PARTY_PACKAGE | 1:M | |
| PARTY_PACKAGE → PACKAGE_UTILIZATION | 1:M | |
| PARTY → DOCUMENT | 1:M (polymorphic) | |

---

## 10. Key Design Decisions

### 10.1 Party Hierarchy (Joined Subtypes)

Using a **supertype `PARTY`** table with two subtype tables (`PARTY_INDIVIDUAL`, `PARTY_ORGANIZATION`) joined on `party_id`. The supertype holds **only universally applicable attributes** (status, blacklist flag, language, RM, segment, risk). Individual-specific flags (`is_vip`, `is_royal`, `is_minor`, `is_special_needs`, `is_pep`, `party_classification`) live exclusively in `PARTY_INDIVIDUAL`; corporate-specific fields in `PARTY_ORGANIZATION`. This cleanly supports:
- Joint accounts (both parties are `PARTY` rows)
- Corporate customers with individual directors/shareholders
- Shared segments, contacts, and channel subscriptions

### 10.2 Card Supertype Pattern

A single `CARD` supertype holds the common card state. Subtypes (`DEBIT_CARD`, `CREDIT_CARD`, `PREPAID_CARD`, `DIGITAL_ACCESS_CARD`) contain type-specific attributes. This allows:
- `CARD_AUTHORIZATION_TRANSACTION` and `CARD_TOKEN` to be card-type agnostic
- Uniform card search/status APIs
- PAN tokenisation enforced at the supertype level

### 10.3 Polymorphic Entities

`DOCUMENT`, `MEMO`, `STANDING_INSTRUCTION`, `LOAN_GUARANTOR`, `LOAN_EARLY_CLOSURE`, `LOAN_RECOVERY_TRANSACTION`, and `PAYMENT_ENQUIRY` use `(entity_type, entity_id)` polymorphic FK patterns to avoid table explosion while keeping the schema extensible. Application-layer constraints (check constraints or triggers) enforce referential integrity.

### 10.4 Never Store Clear PAN

The `CARD` entity stores `card_number_masked` (for display) and `card_number_token` (cryptographic token for lookups). Clear PAN is never persisted — mandatory for PCI-DSS compliance.

### 10.5 Islamic Finance as First-Class Entity

`ISLAMIC_FINANCE` is a peer entity to `RETAIL_LOAN` and `SME_LOAN` rather than a flag — honouring the distinct contractual and accounting treatment of Murabaha/Ijara products under AAOIFI standards.

### 10.6 Calculated / Denormalised Fields

`PARTY_PORTFOLIO_SUMMARY` intentionally denormalises portfolio aggregates to power sub-second Mini-360 (Level-0) rendering. These fields are refreshed asynchronously via an event-driven pipeline triggered by product state changes in core banking.

### 10.7 Multi-source Data

The CRM is a **system of record for CRM-owned data** (interactions, leads, analytics, consents) and a **system of reference for core-banking data** (accounts, loans, cards). The latter is cached/replicated — `last_refreshed_at` columns signal staleness to the UI.

### 10.8 Audit & Regulatory Traceability

All mutable entities carry `created_at`, `updated_at`, `created_by`, `updated_by`. Blacklisting, negation, online banking history, and package changes have dedicated history/audit tables to satisfy UAE regulatory examination requirements.

---

*End of Data Model — RAKBANK CRM 360 Logical Model v1.2*
