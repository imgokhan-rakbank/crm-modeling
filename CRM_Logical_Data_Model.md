# CRM Logical Data Model

> Derived from: *CRM Field Requirements v1.1 – Levels & Navigation for Data Modelling (May 2026)*

---

## Overview

The CRM system covers **19 screens / functional areas**. This document defines a normalized logical data model organized into **10 domains**:

| # | Domain | Screens Covered |
|---|--------|----------------|
| 1 | Customer | Individual Customer, Non-Individual Customer |
| 2 | Customer Supporting Structures | Status, Preferences, Channel Subscriptions, Employment, Segments, RM Info, Contact, Documents, KYC/FATCA/CRS |
| 3 | Relationships | Relationships between Customers |
| 4 | Interactions & Enquiries | Footprints, Communications, Online Banking Requests, Customer Enquiries |
| 5 | Account | Current/Savings Accounts and all sub-sections |
| 6 | Deposits | Fixed/Call Deposits |
| 7 | Cards | Debit Card, Credit Card, Prepaid Card, Digital Access Card |
| 8 | Lending | Retail Loan, SME Loan, Islamic Finance, Trade Finance |
| 9 | Investments & Insurance | Investment Portfolio, Insurance Policies |
| 10 | CRM Operations | Service Requests, Leads |

Shared/reusable structures (Transactions, Standing Instructions, Cheque Books, Limits/Collateral, Transfers, Card Tokens) are described in an **11th section – Shared Entities**.

---

## Conventions

- `PK` – Primary Key  
- `FK` – Foreign Key  
- `ENUM` – Constrained value list  
- Fields derived from the Excel requirements are listed without specifying a data type where it is contextually obvious (VARCHAR unless otherwise noted).
- All entities carry implicit audit fields: `created_at`, `updated_at`, `created_by`, `updated_by`.

---

## Domain 1 – Customer

### 1.1 Customer *(abstract base)*

Every customer, whether individual or company, is identified by a **CIF Number**.

| Attribute | Notes |
|-----------|-------|
| `customer_id` **PK** | CIF Number from core system |
| `customer_type` ENUM | `INDIVIDUAL`, `NON_INDIVIDUAL` |
| `customer_status` ENUM | Active, Inactive, Deceased, Dormant, etc. |
| `customer_classification` | |
| `customer_special_status` | |
| `determination_remarks` | |
| `domicile_branch_id` FK → Branch | |
| `relationship_start_date` DATE | |
| `preferred_language` | |
| `customer_since` DATE | |
| `is_vip` BOOLEAN | VIP Indicator |
| `is_staff` BOOLEAN | Staff Flag |
| `staff_id` | Staff / Military ID |
| `is_pep` BOOLEAN | Politically Exposed Person |
| `blacklisted_flag` BOOLEAN | |
| `overall_blacklisted_status` | |
| `negated_flag` BOOLEAN | |
| `overall_negated_status` | |
| `non_resident_indicator` BOOLEAN | |
| `turned_non_resident_on` DATE | |
| `resident_country` | |
| `minor_indicator` BOOLEAN | |
| `guardian_customer_id` FK → Customer | Guardian CIF (when minor) |
| `illiterate_flag` BOOLEAN | (Individual only) |
| `risk_profile` | |
| `risk_profile_expiry_date` DATE | |
| `dormant_flag` BOOLEAN | |
| `pre_dormant_flag` BOOLEAN | |
| `trade_finance_flag` BOOLEAN | |
| `preference_code` | |
| `gcd_number` | |
| `total_pdc_aed` DECIMAL | Total Post-Dated Cheques in AED |
| `customer_sourced_by` | |
| `source_branch_sol_pos` | |
| `e_crn_number` | |
| `notes` TEXT | |
| `primary_rm_id` FK → RelationshipManager | |
| `secondary_rm_id` FK → RelationshipManager | |
| `rak_token_status` | RAK Token status |

---

### 1.2 IndividualCustomer *(extends Customer)*

| Attribute | Notes |
|-----------|-------|
| `title` | Mr, Mrs, Dr, etc. |
| `first_name` | |
| `middle_name` | |
| `last_name` | |
| `full_name` | |
| `short_name` | |
| `mothers_maiden_name` | |
| `date_of_birth` DATE | |
| `gender` ENUM | Male, Female |
| `nationality` | ISO country code |
| `marital_status` | |
| `education` | |
| `years_since_in_uae` INTEGER | |
| `customer_deceased_date` DATE | |
| `reason_for_inactive` | |
| `vehicle_insurance_renewal_date` DATE | |
| `no_of_dependents` INTEGER | |
| `credit_grading_code` | |
| `group_credit_grading` | |

---

### 1.3 NonIndividualCustomer *(extends Customer)*

| Attribute | Notes |
|-----------|-------|
| `short_name` | |
| `country_of_incorporation` | |
| `principal_place_of_operation` | |
| `country_of_origin` | |
| `legal_entity_type` | |
| `primary_callback_contact` | |
| `primary_callback_contact_number` | |
| `secondary_callback_contact` | |
| `secondary_callback_contact_number` | |
| `no_of_permanent_employees` INTEGER | |
| `net_worth` DECIMAL | |
| `primary_parent_company` | |
| `privilege_type` | |
| `business_relationship_segment` | |
| `merchant_acquired` BOOLEAN | |
| `standalone_cheque_direct` BOOLEAN | |
| `standalone_cdm` BOOLEAN | |
| `standalone_cash_pickup` BOOLEAN | |
| `business_name_changed_last_3_years` BOOLEAN | |
| `credit_grading_code` | |
| `group_credit_grading` | |
| `sanction_declaration_status` | |
| `dnfbp_declaration_status` | |
| `reason_for_inactive` | |

---

## Domain 2 – Customer Supporting Structures

### 2.1 CustomerStatusRecord *(Blacklist / Negated)*

One customer may have multiple status records (e.g. multiple blacklist entries).

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `record_type` ENUM | `BLACKLISTED`, `NEGATED` |
| `reason_code` | |
| `commencement_date` DATE | |
| `expiry_date` DATE | (Blacklist only) |
| `status` | |
| `court_order_date` DATE | (Blacklist only) |
| `circular_date` DATE | (Blacklist only) |
| `unit` | |
| `created_by` | |
| `remarks` TEXT | |

---

### 2.2 CustomerPreference

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `marketing_sms` BOOLEAN | |
| `marketing_email` BOOLEAN | |
| `marketing_by_phone` BOOLEAN | |
| `from_group_by_sms` BOOLEAN | |
| `from_group_by_email` BOOLEAN | |
| `from_group_by_phone` BOOLEAN | |
| `survey_consent` BOOLEAN | |
| `mandatory_consent` BOOLEAN | |
| `authentication_flags` | |
| `manual_authentication_flag` | |

---

### 2.3 CustomerConsentRecord *(repeating consent entries)*

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `consent_type` | SMS, Email, Survey, etc. |
| `value` | |
| `date` DATE | |
| `source` | |
| `channel` | |
| `user_id` | |

---

### 2.4 ChannelSubscription

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `internet_banking` BOOLEAN | |
| `wap_banking` BOOLEAN | |
| `sms_banking` BOOLEAN | |
| `phone_banking` BOOLEAN | |
| `e_statement` BOOLEAN | |
| `email_receipt_for_atm` BOOLEAN | |
| `e_statement_cards` BOOLEAN | |
| `e_statement_accounts` BOOLEAN | |
| `e_statement_loans` BOOLEAN | |
| `e_statement_deposits` BOOLEAN | |
| `e_statement_investments` BOOLEAN | |
| `e_statement_transfers` BOOLEAN | |
| `e_statement_trade_finance` BOOLEAN | |

---

### 2.5 EStatementSubscriptionAudit

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `requested_status` | |
| `requested_date` DATE | |
| `requested_user` | |
| `requested_email` | |
| `requested_comments` TEXT | |

---

### 2.6 OnlineBankingSubscription

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `aani_enrollment_status` | (Individual) |
| `enrollment_date` DATE | |
| `user_id` | |
| `login_enabled` BOOLEAN | |
| `transaction_enabled` BOOLEAN | |
| `login_password` | (hashed) |
| `sms_otp` BOOLEAN | |
| `transaction_limit` DECIMAL | (Non-Individual) |
| `access_type` | (Non-Individual) |
| `corporate_type` | (Non-Individual) |

---

### 2.7 OnlineBankingSubscriptionHistory

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `action` | |
| `reason_code` | |
| `user_id` | |
| `action_by` | |
| `action_datetime` TIMESTAMP | |

---

### 2.8 Employment *(Individual Customers only)*

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → IndividualCustomer | |
| `employment_type` | |
| `employer_code` | |
| `employer_name` | |
| `total_years_of_employment` DECIMAL | |
| `department` | |
| `employment_status` | |
| `date_of_joining` DATE | |
| `occupation` | |
| `designation` | |
| `staff_id_military_id` | |
| `salary_transfer_to_rakbank` BOOLEAN | |
| `dealing_countries` | |
| `monthly_income` DECIMAL | |

---

### 2.9 CustomerSegment

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `segment` | |
| `sub_segment` | |
| `industry_segment` | |
| `industry_sub_segment` | |
| `customer_classification` | |

---

### 2.10 RelationshipManager

Shared lookup entity referenced by Customers, Loans, Insurance, and other products.

| Attribute | Notes |
|-----------|-------|
| `rm_id` **PK** | |
| `rm_code` | |
| `rm_name` | |
| `rm_mobile` | |
| `rm_work_phone` | |
| `rm_email` | |
| `business_center_name` | |
| `unit_head_name` | |
| `unit_head_contact` | |

---

### 2.11 CustomerPhone

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `preferred_flag` BOOLEAN | |
| `phone_type` | Home, Work, Mobile, etc. |
| `country_code` | |
| `area_code` | |
| `contact_number` | |
| `extension_number` | |

---

### 2.12 CustomerEmail

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `preferred_flag` BOOLEAN | |
| `email_type` | |
| `email_id` | |

---

### 2.13 CustomerAddress

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `preferred_flag` BOOLEAN | |
| `address_type` | Residence, Work, Mailing, etc. |
| `flat_no_designation` | |
| `building_employer_name` | |
| `street_employee_id` | |
| `residence_type` | |
| `landmark` | |
| `zip_code` | |
| `po_box` | |
| `emirates_city` | |
| `state` | |
| `country` | |
| `years_in_current_address` INTEGER | |
| `return_mail_flag` BOOLEAN | |
| `return_mail_reason` | |
| `hold_mail_indicator` BOOLEAN | |
| `hold_mail_reason` | |
| `business_center_name` | |

---

### 2.14 CustomerDocument

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `preferred_flag` BOOLEAN | |
| `document_code` | |
| `document_name` | Passport, EID, Trade License, etc. |
| `document_id` | Document number |
| `issue_date` DATE | |
| `expiry_date` DATE | |
| `document_verified` BOOLEAN | |
| `id_issued_organisation` | |
| `trade_license_source` | (Non-Individual only) |
| `trade_license_update_status` | (Non-Individual only) |

---

### 2.15 KYCDetails

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `kyc_held` BOOLEAN | |
| `kyc_review_date` DATE | |
| `expected_monthly_credit_turnover` DECIMAL | |
| `expected_monthly_cash_credit_turnover_pct` DECIMAL | |
| `expected_monthly_non_cash_credit_turnover_pct` DECIMAL | |
| `expected_highest_non_cash_credit_transaction` DECIMAL | |
| `expected_highest_cash_credit_transaction` DECIMAL | |
| `consent_aecb` BOOLEAN | Consent to enquire AECB |
| `consent_source` | |
| `consent_date` DATE | |

---

### 2.16 FATCADetails

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `us_relation` BOOLEAN | |
| `tin_number` | |
| `reason_type_of_relation` | |
| `documents_collected` BOOLEAN | |
| `signed_date` DATE | |
| `expiry_date` DATE | |
| `place_of_birth` | (Individual) |
| `country_of_birth` | (Individual) |
| `category` | (Non-Individual) |
| `criteria_for_us_entity` | (Non-Individual) |
| `exempted_us_entity` BOOLEAN | (Non-Individual) |
| `fatca_entity_type` | (Non-Individual) |
| `name_of_security_market` | (Non-Individual) |
| `name_of_traded_corporation` | (Non-Individual) |
| `financial_entity` BOOLEAN | (Non-Individual) |
| `giin_number` | (Non-Individual) |
| `controlling_person_us_relation` BOOLEAN | (Non-Individual) |

---

### 2.17 CRSDetails

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `crs_undocumented_flag` BOOLEAN | |
| `crs_undocumented_flag_reason` | |
| `country_of_tax_residence` | |
| `tin_number` | |
| `no_tin_reason` | |
| `reason_for_selecting_option_b` | |

---

### 2.18 CurrencyPreferentialRate

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `currency` | |
| `credit_discount_pct` DECIMAL | |
| `debit_discount_pct` DECIMAL | |
| `preferential_expiry_date` DATE | |

---

### 2.19 CustomerGroup *(Non-Individual: group membership)*

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → NonIndividualCustomer | |
| `group_id` | |
| `group_code` | |
| `group_name` | |
| `primary_group_indicator` BOOLEAN | |
| `shareholding_pct` DECIMAL | |

---

## Domain 3 – Relationships

### 3.1 CustomerRelationship

Links customers to each other or to external entities.

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `primary_customer_id` FK → Customer | |
| `related_customer_id` FK → Customer | Nullable (external party) |
| `bank_relation_type` | |
| `relationship_with_entity` | |
| `relationship_type` | Director, Guarantor, Signatory, etc. |
| `entity_type` | CIF Type (Individual/Corporate) |
| `type_of_company` | |
| `controlling_person_type` | |
| `name` | (if external party) |
| `title` | |
| `full_name` | |
| `gender` | |
| `passport_trade_license_number` | |
| `dob_date_of_incorporation` DATE | |
| `total_liability_pct` DECIMAL | |
| `total_equity_pct` DECIMAL | |
| `ib_enabled_flag` BOOLEAN | |
| `us_relation` BOOLEAN | |
| `address` | |
| `tin_number` | |
| `controlling_interest_pct` DECIMAL | |
| `country_of_residence` | |
| `place_of_birth` | |
| `country_of_birth` | |
| `crs_undocumented_flag` BOOLEAN | |
| `crs_undocumented_flag_reason` | |
| `country_of_tax_residence` | |
| `no_tin_reason` | |

---

### 3.2 AuthorizedRepresentative

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `representative_name` | |
| `representative_designation` | |
| `document_type` | |
| `document_id` | |
| `document_expiry` DATE | |

---

### 3.3 AuthorizedRepresentativePermission

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `representative_id` FK → AuthorizedRepresentative | |
| `permission_category` ENUM | `DELIVERABLE`, `FINANCIAL_SERVICE` |
| `permission_name` | e.g. Cheque Book, EFT, etc. |
| `allowed` BOOLEAN | |

---

## Domain 4 – Interactions & Enquiries

### 4.1 Footprint

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `date` TIMESTAMP | |
| `channel` | |
| `priority` | |
| `category` | |
| `description` TEXT | |

---

### 4.2 Communication

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `date` TIMESTAMP | |
| `channel` | SMS, Email, Phone, etc. |
| `address_to` | |
| `event_name` | |
| `content` TEXT | |
| `communication_direction` ENUM | `INBOUND`, `OUTBOUND` |
| `message_from` | |
| `communication_type` | |
| `status` | |
| `reason_field` | |
| `network` | |
| `message_content` TEXT | |
| `received_date` TIMESTAMP | |
| `internal_source_system` | |

---

### 4.3 OnlineBankingRequest

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `reference_id` | |
| `request_type` | |
| `nature` | |
| `date` TIMESTAMP | |
| `status` | |
| `backend_reference_id` | |

---

### 4.4 OnlineBankingRequestAdditionalInfo

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `request_id` FK → OnlineBankingRequest | |
| `field_name` | |
| `field_value` | |

---

### 4.5 DisputeRecord

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `dispute_number` | |
| `card_number` | |
| `dispute_reason` | |
| `report_sent_by` | |
| `date_time` TIMESTAMP | |

---

### 4.6 DeliveryEnquiry

Tracks physical delivery of cards, documents, etc.

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `tracking_awb_number` | |
| `consignee` | |
| `shipment_status` | |
| `location` | |
| `created_date` TIMESTAMP | |
| `last_updated_date` TIMESTAMP | |
| `delivery_type` | |
| `product_type` | |
| `delivery_scheduled_date` DATE | |
| `received_by` | |
| `nationality` | |
| `emirates_id_number` | |
| `biometrics_status` | |
| `notes` TEXT | |

---

### 4.7 EStatementDelivery

| Attribute | Notes |
|-----------|-------|
| `id` **PK** (Record ID) | |
| `customer_id` FK → Customer | |
| `product_number` | |
| `email_address` | |
| `send_date` DATE | |
| `delivery_date` DATE | |
| `status` | |
| `document_date` DATE | |
| `e_statement_type` | |
| `is_read` BOOLEAN | |
| `read_date` DATE | |

---

### 4.8 RAKValuePackage

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `package_availed` | |
| `last_activity_date` DATE | |
| `enrollment_account` | |
| `status` | |

---

### 4.9 PackageUtilization

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `package_id` FK → RAKValuePackage | |
| `utilization_type` ENUM | `FINANCIAL`, `NON_FINANCIAL` |
| `transaction_type` | |
| `description` | |
| `frequency` | |
| `eligible_at_cif_level` DECIMAL | |
| `availed_at_account_level` DECIMAL | |
| `balance_at_cif_level` DECIMAL | |
| `utilization_month` DATE | |

---

## Domain 5 – Account

### 5.1 Account

| Attribute | Notes |
|-----------|-------|
| `account_number` **PK** | |
| `customer_id` FK → Customer | |
| `account_name` | |
| `account_status` | |
| `iban_number` | |
| `currency` | ISO currency code |
| `scheme_code` | |
| `scheme_name` | |
| `account_short_name` | |
| `open_date` DATE | |
| `closed_date` DATE | |
| `account_manager_id` FK → RelationshipManager | |
| `freeze_code` | |
| `mode_of_operation` | |
| `joint_account_flag` BOOLEAN | |
| `account_closure_reason_code` | |
| `charge_level_code` | |
| `delivery_channel_id` | |
| `related_to_staff` BOOLEAN | |
| `allow_sweeps` BOOLEAN | |
| `collect_charges` BOOLEAN | |
| `ecs_enabled` BOOLEAN | |
| `secured` BOOLEAN | |
| `scheme_type` | |
| `overdraft_type` | |
| `notes` TEXT | |
| `effective_available_balance` DECIMAL | |
| `clear_balance` DECIMAL | |
| `hold_amount` DECIMAL | |

---

### 5.2 AccountInterest

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `pay_interest` BOOLEAN | |
| `collect_interest` BOOLEAN | |
| `credit_interest_pct_min` DECIMAL | |
| `credit_interest_pct_max` DECIMAL | |
| `debit_interest_pct_min` DECIMAL | |
| `debit_interest_pct_max` DECIMAL | |
| `customer_preferential_interest_cr` DECIMAL | |
| `customer_preferential_interest_dr` DECIMAL | |
| `interest_credit_account` | |
| `interest_debit_account` | |
| `interest_calculation_frequency_cr` | |
| `interest_calculation_frequency_dr` | |
| `next_interest_calculation_date_cr` DATE | |
| `next_interest_calculation_date_dr` DATE | |
| `interest_rate_code` | |
| `interest_accrued_ytd` DECIMAL | |
| `interest_collected_ytd` DECIMAL | |
| `interest_paid_ytd` DECIMAL | |
| `charges_ytd` DECIMAL | |

---

### 5.3 AccountStatement

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `statement_frequency` | |
| `dispatch_mode` | |
| `next_print_date` DATE | |
| `last_statement_date` DATE | |

---

### 5.4 JointAccountLink

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `linked_customer_id` FK → Customer | |
| `relation_type` | |
| `relation_code` | |
| `print_statement` BOOLEAN | |
| `print_advice_for_si` BOOLEAN | |
| `print_deposit_notice` BOOLEAN | |
| `print_loan_notice` BOOLEAN | |
| `initiate_certificate` BOOLEAN | |
| `initiate_rate_change_advice` BOOLEAN | |
| `exclude_from_combined_statement` BOOLEAN | |

---

### 5.5 AccountLien *(see also Shared: Lien)*

*(Links to Lien entity, account_number FK)*

---

### 5.6 SweepPool

| Attribute | Notes |
|-----------|-------|
| `pool_id` **PK** | |
| `pool_description` | |
| `currency` | |
| `pool_balance` DECIMAL | |
| `pool_status` | |

---

### 5.7 SweepPoolMember

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `pool_id` FK → SweepPool | |
| `account_number` FK → Account | |
| `scheme_code` | |
| `order_in_pool` INTEGER | |
| `available_balance` DECIMAL | |
| `status` | |
| `relationship` | |

---

### 5.8 AverageBalanceRecord

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `month` DATE | |
| `average_balance` DECIMAL | |
| `maximum_balance` DECIMAL | |
| `maximum_balance_days` INTEGER | |
| `minimum_balance` DECIMAL | |
| `minimum_balance_days` INTEGER | |

---

### 5.9 TurnoverStatistic

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `month` DATE | |
| `debit_turnover` DECIMAL | |
| `debit_entries` INTEGER | |
| `credit_turnover` DECIMAL | |
| `credit_entries` INTEGER | |

---

## Domain 6 – Deposits

### 6.1 Deposit

| Attribute | Notes |
|-----------|-------|
| `deposit_account_number` **PK** | |
| `customer_id` FK → Customer | |
| `currency` | |
| `account_name` | |
| `scheme_code` | |
| `scheme_name` | |
| `account_status` | |
| `deposit_open_date` DATE | |
| `mode_of_operation` | |
| `value_date` DATE | |
| `maturity_date` DATE | |
| `closure_date` DATE | |
| `deposit_period_mm_dd` | |
| `deposit_amount` DECIMAL | |
| `maturity_value` DECIMAL | |
| `repay_account` | |
| `effective_interest_rate` DECIMAL | |
| `total_interest` DECIMAL | |
| `profit_interest_credit_account` | |
| `deposit_status` | |
| `call_deposit_notice_period_mm_dd` | |
| `notice_date` DATE | |
| `lien_amount` DECIMAL | |
| `auto_closure` BOOLEAN | |
| `auto_renewal` BOOLEAN | |
| `max_renewal_allowed` INTEGER | |
| `renewals_done` INTEGER | |
| `renewal_period_mm_dd` | |
| `autorenewal_interest` | |
| `renewal_currency` | |
| `renewal_exchange_rate` DECIMAL | |
| `renewal_rate` DECIMAL | |
| `renewal_option` | |
| `renewal_amount` DECIMAL | |
| `contracted_interest_rate` DECIMAL | |
| `interest_rate_code` | |
| `account_manager_id` FK → RelationshipManager | |

---

## Domain 7 – Cards

### 7.1 DebitCard

| Attribute | Notes |
|-----------|-------|
| `card_number` **PK** | |
| `account_number` FK → Account | |
| `customer_id` FK → Customer | |
| `product_name` | |
| `card_status` | |
| `card_expiry_date` DATE | |
| `customer_name` | |
| `embossed_name` | |
| `card_activation_date` DATE | |
| `dispatch_channel` | |
| `card_dispatch_date` DATE | |
| `total_otb` DECIMAL | |
| `cash_otb` DECIMAL | |
| `cash_transactions_left` INTEGER | |
| `total_transactions_left` INTEGER | |
| `card_type` | |
| `card_sub_type` | |
| `created_date` DATE | |
| `dispatched_date` DATE | |
| `delivery_date` DATE | |

---

### 7.2 CreditCard

| Attribute | Notes |
|-----------|-------|
| `card_number` **PK** | |
| `customer_id` FK → Customer | |
| `product_code` | |
| `product_name` | |
| `card_status` | |
| `activation_status` | |
| `card_expiry_date` DATE | |
| `credit_limit` DECIMAL | |
| `primary_card_limit` DECIMAL | |
| `card_currency` | |
| `card_type` | |
| `primary_card_flag` BOOLEAN | |
| `primary_card_number` FK → CreditCard | (for supplementary) |
| `customer_name` | |
| `embossed_name` | |
| `embossed_company_name` | |
| `statement_cycle` | |
| `card_activation_date` DATE | |
| `due_date` DATE | |
| `card_account_status` | |
| `card_delivery_date` DATE | |
| `card_dispatch_date` DATE | |
| `authorization_status` | |
| `financial_status` | |
| `embossing_status` | |
| `card_account_number` | |
| `staff_account` BOOLEAN | |
| `last_renew_date` DATE | |
| `last_reissue` DATE | |
| `old_card_number` | |
| `fee_profile` | |
| `creation_date` DATE | |
| `card_closure_date` DATE | |
| `card_closure_reason` | |
| `retention_flag` BOOLEAN | |
| `retention_reason` | |
| `card_dispatch_channel` | |
| `last_status_change_date` DATE | |
| `virtual_card_flag` BOOLEAN | |
| `card_crn` | |
| `card_account_type` | |
| `marketing_code` | |
| `promotion_code` | |
| `annual_fee_waiver_eligibility` BOOLEAN | |
| `annual_spent_amount` DECIMAL | |
| `annual_spent_remaining` DECIMAL | |
| `enrollment_date` DATE | |
| `reward_profile` | |
| `rvc_package_code` | |
| `rvc_package_status` | |
| `charity_organization_id` | |
| `charity_flat_amount` DECIMAL | |
| `charity_percentage_amount` DECIMAL | |
| `notes` TEXT | |
| `vip_normal` BOOLEAN | |
| `vip_high` BOOLEAN | |
| `direct_debit_flag` BOOLEAN | |
| `direct_debit_account` | |
| `direct_debit_pct` DECIMAL | |

---

### 7.3 CreditCardBalance

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `card_number` FK → CreditCard | |
| `outstanding_balance` DECIMAL | |
| `available_funds` DECIMAL | |
| `credit_limit` DECIMAL | |
| `cash_limit` DECIMAL | |
| `cash_otb` DECIMAL | |
| `total_otb` DECIMAL | |
| `last_payment_date` DATE | |
| `last_payment_amount` DECIMAL | |
| `unbilled_amount` DECIMAL | |
| `last_statement_date` DATE | |
| `minimum_due` DECIMAL | |
| `overdue_amount` DECIMAL | |
| `last_statement_balance` DECIMAL | |
| `next_statement_date` DATE | |
| `reward_balance` DECIMAL | |
| `temporary_credit` DECIMAL | |

---

### 7.4 CardStatement

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `card_number` FK → CreditCard | |
| `statement_date` DATE | |
| `opening_balance` DECIMAL | |
| `closing_balance` DECIMAL | |
| `debits` DECIMAL | |
| `credits` DECIMAL | |
| `minimum_due` DECIMAL | |
| `overdue_amount` DECIMAL | |
| `pay_in_full_amount` DECIMAL | |
| `billing_date` DATE | |
| `due_date` DATE | |
| `rewards_opening_balance` DECIMAL | |
| `rewards_closing_balance` DECIMAL | |
| `earned_points` INTEGER | |
| `redeemed_points` INTEGER | |

---

### 7.5 CardInstallment

| Attribute | Notes |
|-----------|-------|
| `installment_number` **PK** | |
| `card_number` FK → CreditCard | |
| `installment_booked_card_number` | |
| `description` | |
| `merchant_name` | |
| `authorization_code` | |
| `installment_type` | |
| `original_installment_amount` DECIMAL | |
| `tenure` INTEGER | |
| `start_date` DATE | |
| `end_date` DATE | |
| `next_billing_date` DATE | |
| `outstanding_amount` DECIMAL | |
| `emi` DECIMAL | |
| `emi_pending` INTEGER | |
| `status` | |
| `total_installment_amount` DECIMAL | |
| `total_interest_amount` DECIMAL | |
| `outstanding_principal_amount` DECIMAL | |
| `outstanding_interest_amount` DECIMAL | |

---

### 7.6 CardInsurance

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `card_number` FK → CreditCard | |
| `insurance_name` | |
| `activation_date` DATE | |
| `insurance_status` | |
| `deactivation_date` DATE | |
| `free_cycle_count` INTEGER | |
| `re_activation_date` DATE | |
| `un_enrollment_user` | |
| `auto_un_enrollment_date` DATE | |
| `free_period_end_date` DATE | |
| `un_enrolled_reason` | |

---

### 7.7 CardReward

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `card_number` FK → CreditCard | |
| `date` DATE | |
| `points` INTEGER | |
| `message` | |
| `status` | |
| `transaction_type` | |

---

### 7.8 PrepaidCard *(same structure as DebitCard, different product)*

| Attribute | Notes |
|-----------|-------|
| `card_number` **PK** | |
| `account_number` FK → Account | |
| `customer_id` FK → Customer | |
| *(same attributes as DebitCard 7.1)* | |

---

### 7.9 DigitalAccessCard

| Attribute | Notes |
|-----------|-------|
| `card_number` **PK** | |
| `product_name` | |
| `card_status` | |
| `card_expiry_date` DATE | |
| `dac_holder_name` | |
| `dac_holder_customer_id` FK → Customer | |
| `card_activation_date` DATE | |
| `dispatch_channel` | |
| `created_date` DATE | |
| `dispatched_date` DATE | |
| `delivery_date` DATE | |

---

### 7.10 CardToken

Covers both Mastercard and VISA tokenization (digital wallets).

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `card_number` FK → (DebitCard / CreditCard / PrepaidCard) | |
| `token_network` ENUM | `MASTERCARD`, `VISA` |
| `token_number` | |
| `token_type` | |
| `token_expiry_date` DATE | |
| `status` | |
| `wallet_provider` | Apple Pay, Google Pay, etc. |
| `token_requestor_id` | |
| `token_requestor_name` | |
| `device_type` | |
| `device_name` | |
| `device_serial_number` | |
| `device_id` | |
| `token_unique_reference` | |
| `creation_date` DATE | |
| `initial_activation_date` DATE | |
| `expiration_date` DATE | |

---

### 7.11 CardAuthorizationTransaction

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `card_number` FK → (any Card) | |
| `transaction_datetime` TIMESTAMP | |
| `transaction_amount` DECIMAL | |
| `transaction_currency` | |
| `billing_amount` DECIMAL | |
| `billing_currency` | |
| `transaction_type` | |
| `pos_entry` | |
| `merchant_name` | |
| `merchant_id` | |
| `merchant_city` | |
| `country` | |
| `terminal_id` | |
| `acquiring_id` | |
| `acquiring_country` | |
| `mcc_code` | |
| `transaction_mode` | |
| `response_code` | |
| `response_code_description` | |
| `auth_id` | |
| `reason_code` | |
| `status` | |
| `failure_reason` | |
| `card_expiry_date` DATE | |
| `token_number` FK → CardToken | |
| `token_expiry` DATE | |
| `requestor_id` | |
| `device_type` | |
| `matched_date` DATE | |

---

### 7.12 OTPDeliveryDetail

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `auth_transaction_id` FK → CardAuthorizationTransaction | |
| `otp_delivery_ref_no` | |
| `card_usage_date` TIMESTAMP | |
| `otp_delivery_type` | |
| `otp_delivery_mobile_number` | |
| `otp_delivery_email_id` | |
| `message_received_timestamp` TIMESTAMP | |
| `message_response_timestamp` TIMESTAMP | |

---

## Domain 8 – Lending

### 8.1 RetailLoan

| Attribute | Notes |
|-----------|-------|
| `agreement_id` **PK** | |
| `agreement_number` | |
| `customer_id` FK → Customer | |
| `product_category` | |
| `product` | Personal, Auto, Home, etc. |
| `currency` | |
| `scheme_code` | |
| `scheme_name` | |
| `status` | |
| `disbursal_status` | |
| `emi_start_date` DATE | |
| `maturity_date` DATE | |
| `loan_amount` DECIMAL | |
| `current_outstanding` DECIMAL | |
| `outstanding_charges` DECIMAL | |
| `excess_amount` DECIMAL | |
| `overdue_amount` DECIMAL | |
| `emi` DECIMAL | |
| `tenure` INTEGER | |
| `balance_tenure_months` INTEGER | |
| `balance_repayments` INTEGER | |
| `interest_rate` DECIMAL | |
| `installment_mode` | |
| `frequency` | |
| `npa_stage_id` | |
| `topped_up_amount` DECIMAL | |
| `due_day` INTEGER | |
| `repayment_mode` | |
| `dda_reference_number` | |
| `repayment_account_number` FK → Account | |
| `application_date` DATE | |
| `application_type` | |
| `disbursal_date` DATE | |
| `disbursed_amount` DECIMAL | |
| `source_branch` | |
| `referrer_code` | |
| `channel_code` | |
| `dpd` INTEGER | Days Past Due |
| `bucket` | Delinquency bucket |
| `dpd_string` | 12-month DPD string |

---

### 8.2 RetailLoanAutoDetails *(Auto Loan product-specific)*

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `agreement_id` FK → RetailLoan | |
| `vehicle_registration_number` | |
| `driving_license` | |
| `asset_category` | |
| `asset_description` | |
| `asset_type` | |
| `type_of_vehicle` | |
| `manufacturer` | |
| `make` | |
| `model` | |
| `year_model` INTEGER | |
| `dealer_name` | |
| `vehicle_cost` DECIMAL | |
| `insurance_cost` DECIMAL | |
| `down_payment` DECIMAL | |
| `asset_age_at_maturity` | |
| `down_payment_pct` DECIMAL | |
| `appraiser_value` DECIMAL | |
| `gross_ltv` DECIMAL | |
| `net_ltv` DECIMAL | |
| `asset_color` | |

---

### 8.3 RetailLoanHomeDetails *(Home Loan product-specific)*

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `agreement_id` FK → RetailLoan | |
| `developer_code` | |
| `property_id` | |
| `property_type` | |
| `property_description` | |
| `lien_position` | |
| `lien_value` DECIMAL | |
| `total_collateral_value` DECIMAL | |
| `property_transaction_type` | |
| `property_address` | |

---

### 8.4 LoanGuarantorCoApplicant

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `loan_type` ENUM | `RETAIL`, `SME`, `ISLAMIC` |
| `loan_id` | FK to relevant loan table |
| `applicant_type` | Guarantor, Co-Applicant |
| `customer_id` FK → Customer | Nullable (external) |
| `name` | |
| `relation` | |

---

### 8.5 RetailLoanRepaymentSchedule

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `agreement_id` FK → RetailLoan | |
| `installment_number` INTEGER | |
| `due_date` DATE | |
| `opening_principal` DECIMAL | |
| `installment_amount` DECIMAL | |
| `principal_component` DECIMAL | |
| `interest_component` DECIMAL | |
| `days` INTEGER | |
| `closing_principal` DECIMAL | |
| `rate` DECIMAL | |
| `pi_premium_amount` DECIMAL | |
| `li_premium_amount` DECIMAL | |
| `total_repayable` DECIMAL | |
| `deferral` BOOLEAN | |

---

### 8.6 SMELoan

| Attribute | Notes |
|-----------|-------|
| `product_number` **PK** | |
| `customer_id` FK → Customer | |
| `currency` | |
| `product_type` | |
| `amount` DECIMAL | |
| `period_months_days` | |
| `status` | |
| `scheme_code` | |
| `open_date` DATE | |
| `repayment_method` | |
| `repayment_rate` DECIMAL | |
| `operative_account_number` FK → Account | |
| `short_name` | |
| `maturity_date` DATE | |
| `mode_of_operation` | |
| `net_interest_rate` DECIMAL | |
| `penal_interest_code` | |
| `schedule_balance` DECIMAL | |
| `payoff_amount` DECIMAL | |
| `prepayment_till_date` DECIMAL | |
| `outstanding_selling_price` DECIMAL | |
| `delinquency_string` | |
| `top_up_indicator` BOOLEAN | |
| `insurance_finance` BOOLEAN | |
| `deviation` | |
| `cbd_sme_rm_name` | |
| `dsa_id` | |
| `abf_rm_name` | |
| `abf_reference_number` | |

---

### 8.7 IslamicFinance

| Attribute | Notes |
|-----------|-------|
| `agreement_id` **PK** | |
| `customer_id` FK → Customer | |
| `product` | Murabaha, Ijara, etc. |
| `currency` | |
| `maturity_date` DATE | |
| `scheme` | |
| `status` | |
| `tenor` | |
| `next_installment_date` DATE | |
| `total_loan_amount` DECIMAL | |
| `total_outstanding_account_ccy` DECIMAL | |
| `total_outstanding_aed` DECIMAL | |
| `overdue_amount` DECIMAL | |
| `profit_rate` DECIMAL | |
| `total_disbursed_amount` DECIMAL | |
| `bucket` | |
| `funding_account_no` FK → Account | |
| `repayment_mode` | |
| `last_payment_received_date` DATE | |
| `next_installment_amount` DECIMAL | |
| `overdue_principal` DECIMAL | |
| `overdue_profit` DECIMAL | |
| `overdue_charges` DECIMAL | |
| `last_deferral_date` DATE | |
| `balance_installments` INTEGER | |
| `keyman_guarantor_signatory` | |
| `insured` | |
| `asset_type` | |
| `registration_number` | |

---

### 8.8 TradeFinance

| Attribute | Notes |
|-----------|-------|
| `product_number` **PK** | |
| `customer_id` FK → Customer | |
| `scheme_code` | |
| `scheme_description` | |
| `currency` | |
| `amount` DECIMAL | |

---

## Domain 9 – Investments & Insurance

### 9.1 InvestmentPortfolio

| Attribute | Notes |
|-----------|-------|
| `investment_id` **PK** | |
| `customer_id` FK → Customer | |
| `risk_profile_category` | |
| `final_risk_profile_category` | |
| `final_risk_score` DECIMAL | |
| `risk_profile_score` DECIMAL | |
| `risk_profile_expiry_date` DATE | |
| `mode_of_operation` | |
| `preferred_currency` | |
| `status` | |
| `total_value` DECIMAL | |

---

### 9.2 MutualFundHolding

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `investment_id` FK → InvestmentPortfolio | |
| `customer_risk_profile` | |
| `fund_risk_profile` | |
| `client_classification` | |
| `fund_name` | |
| `fund_currency` | |
| `payment_type` | |
| `dividend` | |
| `folio_number` | |
| `holding_date` DATE | |
| `number_of_units` DECIMAL | |
| `average_cost` DECIMAL | |
| `current_nav` DECIMAL | |
| `book_value` DECIMAL | |
| `market_value` DECIMAL | |
| `market_value_preferred_ccy` DECIMAL | |
| `unrealized_gain_loss` DECIMAL | |
| `unrealized_gain_loss_pct` DECIMAL | |
| `nav_date` DATE | |

---

### 9.3 StructuredProductHolding

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `investment_id` FK → InvestmentPortfolio | |
| `product_name` | |
| `product_currency` | |
| `order_id` | |
| `current_investment_amount` DECIMAL | |
| `settlement_price` DECIMAL | |
| `settlement_value_product_ccy` DECIMAL | |
| `market_price` DECIMAL | |
| `market_price_date` DATE | |
| `market_value_product_ccy` DECIMAL | |
| `accrued_interest_product_ccy` DECIMAL | |
| `unrealized_gain_loss_product_ccy` DECIMAL | |
| `unrealized_gain_loss_preferred_ccy` DECIMAL | |

---

### 9.4 EquityHolding

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `investment_id` FK → InvestmentPortfolio | |
| `custody_account_number` | |
| `equity_category` | |
| `security_name` | |
| `security_currency` | |
| `number_of_shares` DECIMAL | |
| `average_cost` DECIMAL | |
| `market_price` DECIMAL | |
| `price_date` DATE | |
| `book_value_security_ccy` DECIMAL | |
| `market_value_security_ccy` DECIMAL | |
| `book_value_preferred_ccy` DECIMAL | |
| `market_value_preferred_ccy` DECIMAL | |
| `unrealized_gain_loss_security_ccy` DECIMAL | |
| `unrealized_gain_loss_pct` DECIMAL | |

---

### 9.5 BondHolding

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `investment_id` FK → InvestmentPortfolio | |
| `bond_name` | |
| `bond_currency` | |
| `total_quantity_units` DECIMAL | |
| `total_quantity_amount` DECIMAL | |
| `accrued_interest` DECIMAL | |
| `average_cost` DECIMAL | |
| `average_cost_pct` DECIMAL | |
| `market_price` DECIMAL | |
| `market_price_pct` DECIMAL | |
| `price_date` DATE | |
| `book_value` DECIMAL | |
| `market_value` DECIMAL | |
| `notional_increase` DECIMAL | |
| `unrealized_gain_loss` DECIMAL | |
| `coupon` DECIMAL | |

---

### 9.6 FuturesHolding

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `investment_id` FK → InvestmentPortfolio | |
| `futures_name` | |
| `holding_type` | |
| `contract_currency` | |
| `contract_size` DECIMAL | |
| `number_of_contracts` INTEGER | |
| `net_mark_to_market_contract_ccy` DECIMAL | |
| `initial_margin_value_contract_ccy` DECIMAL | |
| `initial_margin_value_preferred_ccy` DECIMAL | |

---

### 9.7 PreciousMetalHolding

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `investment_id` FK → InvestmentPortfolio | |
| `product_name` | |
| `product_type` | |
| `product_subtype` | |
| `product_unit` | |
| `product_variant` | |
| `quantity` DECIMAL | |
| `average_cost` DECIMAL | |
| `book_value_product_ccy` DECIMAL | |
| `closing_price` DECIMAL | |
| `price_date` DATE | |
| `market_value_product_ccy` DECIMAL | |
| `product_currency` | |
| `unrealized_gain_loss_product_ccy` DECIMAL | |
| `unrealized_gain_loss_pct` DECIMAL | |

---

### 9.8 PMSHolding *(Portfolio Management Service)*

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `investment_id` FK → InvestmentPortfolio | |
| `product_name` | |
| `product_currency` | |
| `pms_id` | |
| `investment_amount` DECIMAL | |
| `realized_gain_loss` DECIMAL | |
| `return_pct` DECIMAL | |
| `valuation_date` DATE | |
| `market_value` DECIMAL | |
| `market_value_preferred_ccy` DECIMAL | |
| `unrealized_gain_loss_product_ccy` DECIMAL | |
| `unrealized_gain_loss_pct` DECIMAL | |

---

### 9.9 AlternateInvestmentHolding

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `investment_id` FK → InvestmentPortfolio | |
| `product_type` | PE, Held-away Assets, etc. |
| `product_name` | |
| `product_currency` | |
| `pe_held_away_id` | |
| `valuation_date` DATE | |
| `number_of_units` DECIMAL | |
| `average_cost` DECIMAL | |
| `book_value_product_ccy` DECIMAL | |
| `current_market_price` DECIMAL | |
| `price_date` DATE | |
| `market_value_product_ccy` DECIMAL | |
| `unrealized_gain_loss_product_ccy` DECIMAL | |
| `unrealized_gain_loss_pct` DECIMAL | |

---

### 9.10 InvestmentInsurancePolicy

| Attribute | Notes |
|-----------|-------|
| `id` **PK** (Insurance Investment ID) | |
| `investment_id` FK → InvestmentPortfolio | |
| `product_type` | |
| `product_code` | |
| `product_name` | |
| `policy_number` | |
| `policy_status` | |
| `policy_commencement_date` DATE | |
| `policy_issuance_date` DATE | |
| `maturity_date` DATE | |
| `basic_sum_assured` DECIMAL | |
| `valuation_date` DATE | |
| `market_value_product_ccy` DECIMAL | |
| `surrender_value_product_ccy` DECIMAL | |
| `insurer_code` | |
| `channel_id` | |
| `application_number` | |
| `premium_payment_frequency` | |
| `annual_premium_amount` DECIMAL | |
| `policy_term_years` INTEGER | |
| `next_recurring_premium_due_date` DATE | |
| `recurring_premium_amount` DECIMAL | |

---

### 9.11 InsurancePolicy *(standalone, screen 10)*

| Attribute | Notes |
|-----------|-------|
| `policy_number` **PK** | |
| `customer_id` FK → Customer | |
| `product_name` | |
| `plan_name` | |
| `policy_status` | |
| `policy_currency` | |
| `type` | |
| `isp_name` | |
| `plan_term_years` INTEGER | |
| `monthly_premium` DECIMAL | |
| `premium_amount` DECIMAL | |
| `payment_frequency` | |
| `effective_policy_date` DATE | |
| `recovery_type` | |
| `business_cycle` | |
| `business_cycle_year` INTEGER | |
| `third_party_payment` BOOLEAN | |
| `rm_id` FK → RelationshipManager | |
| `maker_id` | |
| `application_number` | |
| `staff_case` BOOLEAN | |
| `document_expiry_date` DATE | |
| `sourced_by` | |
| `notes` TEXT | |
| `accidental_death_benefit` BOOLEAN | |
| `ptd` BOOLEAN | |
| `terminal_illness_benefit` BOOLEAN | |
| `critical_illness_benefit` BOOLEAN | |
| `waiver_of_contribution_benefit` BOOLEAN | |

---

### 9.12 InsuranceBeneficiary

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `policy_number` FK → InsurancePolicy | |
| `beneficiary_name` | |
| `relationship` | |
| `covered_member_name` | |
| `sum_covered` DECIMAL | |
| `reason_for_cover` | |

---

### 9.13 InsurancePremiumPayment

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `policy_number` FK → InsurancePolicy | |
| `first_payment_mode` | |
| `first_payment_account_number` FK → Account | |
| `first_payment_credit_card_number` FK → CreditCard | |
| `si_required` BOOLEAN | |
| `si_payment_mode` | |
| `total_premiums_paid` DECIMAL | |
| `total_number_of_premiums_paid` INTEGER | |
| `total_premium_amount_missed` DECIMAL | |
| `total_number_of_premiums_missed` INTEGER | |
| `si_amount` DECIMAL | |
| `si_date` DATE | |
| `si_from_date` DATE | |
| `si_to_date` DATE | |
| `last_premium_collection_date` DATE | |

---

## Domain 10 – CRM Operations

### 10.1 ServiceRequest

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | |
| `reference_number` | |
| `request_type` | |
| `request_sub_type` | |
| `assigned_group` | |
| `assigned_to` | |
| `product_number` | |
| `status` | |
| `requested_by` | |
| `requested_on` TIMESTAMP | |
| `channel` | |
| `request_remarks` TEXT | |
| `additional_remarks` TEXT | |
| `source` | |
| `notes` TEXT | |
| `backend_reference_id` | |

---

### 10.2 Lead

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `customer_id` FK → Customer | Existing customer (nullable for prospects) |
| `prospect_cif` | |
| `lead_reference_no` | |
| `product_code` | |
| `lead_assigned_to` | |
| `lead_status` | |
| `creation_date` TIMESTAMP | |
| `creator` | |
| `creator_unit` | |
| `assigned_to_unit` | |
| `source_description` | |
| `campaign_name` | |
| `company_name` | |
| `product_description` | |
| `follow_up_date` DATE | |
| `customer_type` | |
| `customer_first_name` | |
| `customer_middle_name` | |
| `customer_last_name` | |
| `mobile_number` | |
| `email_id` | |
| `preferred_language` | |
| `preferred_contact_date` DATE | |
| `preferred_contact_time` | |
| `notes` TEXT | |

---

## Domain 11 – Shared Entities

### 11.1 AccountTransaction

Used by Accounts (current/savings), Deposits, and Loans.

| Attribute | Notes |
|-----------|-------|
| `transaction_id` **PK** | |
| `source_type` ENUM | `ACCOUNT`, `LOAN_SME`, `LOAN_RETAIL` |
| `source_id` | FK to Account or Loan |
| `transaction_date` DATE | |
| `value_date` DATE | |
| `transaction_particulars_code` | |
| `transaction_particulars` | |
| `account_currency` | |
| `indicator` ENUM | `CREDIT`, `DEBIT` |
| `amount_in_account_currency` DECIMAL | |
| `transaction_currency` | |
| `transaction_amount` DECIMAL | |
| `running_balance` DECIMAL | |
| `instrument_number` | |
| `instrument_type` | |
| `instrument_date` DATE | |
| `remarks_1` | |
| `remarks_2` | |
| `status` | |
| `transaction_remarks` | |
| `transaction_type` | |
| `rate_code` | |
| `rate` DECIMAL | |
| `reference_number` | |
| `entry_user_id` | |
| `entered` TIMESTAMP | |
| `posted` TIMESTAMP | |
| `verified` TIMESTAMP | |

---

### 11.2 StandingInstruction

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `standing_instruction_type` | |
| `priority` | |
| `frequency` | |
| `next_execution_date` DATE | |
| `start_date` DATE | |
| `end_date` DATE | |
| `status` | |
| `debit_account_number` | |
| `credit_account_number` | |
| `amount_type` | |
| `currency` | |
| `si_amount` DECIMAL | |
| `remarks` TEXT | |
| `suspend_start_date` DATE | |
| `suspended_till` DATE | |
| `remittance_type` | |
| `remit_currency` | |
| `payment_system_id` | |
| `beneficiary_name` | |
| `beneficiary_address_line_1` | |
| `beneficiary_address_line_2` | |
| `beneficiary_country` | |
| `executed_count` INTEGER | |
| `failed_count` INTEGER | |
| `execution_time` | |
| `notes` TEXT | |

---

### 11.3 ChequeBook

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `cheque_start_number` | |
| `finish_number` | |
| `cheque_issue_date` DATE | |
| `no_of_leaves` INTEGER | |
| `issued_not_acknowledged` INTEGER | |
| `passed` INTEGER | |
| `unused` INTEGER | |
| `stopped` INTEGER | |
| `cautioned` INTEGER | |
| `destroyed` INTEGER | |
| `returned_paid` INTEGER | |
| `rejected` INTEGER | |
| `transferred` INTEGER | |

---

### 11.4 ChequeLeaf

| Attribute | Notes |
|-----------|-------|
| `cheque_number` **PK** | |
| `cheque_book_id` FK → ChequeBook | |
| `status` | |

---

### 11.5 Lien

| Attribute | Notes |
|-----------|-------|
| `lien_id` **PK** | |
| `account_number` FK → Account | |
| `created_date` TIMESTAMP | |
| `lien_amount` DECIMAL | |
| `expiry_date` DATE | |
| `lien_reason` | |
| `requested_by` | |
| `request_department` | |
| `contact_number` | |
| `created_by` | |
| `last_modified_by` | |
| `verified_by` | |
| `lien_placement_date` DATE | |
| `modification_date` DATE | |
| `notes` TEXT | |

---

### 11.6 CreditFacility *(Limit)*

Shared by SME Loans, Retail Loans, and standalone credit limits.

| Attribute | Notes |
|-----------|-------|
| `limit_id` **PK** | |
| `customer_id` FK → Customer | |
| `parent_limit_id` FK → CreditFacility | For hierarchical limit nodes |
| `description` | |
| `currency` | |
| `limit_type` | |
| `limit_type_id` | |
| `fund_liability` DECIMAL | |
| `non_fund_liability` DECIMAL | |
| `total_liability` DECIMAL | |
| `utilized_limit` DECIMAL | |
| `number_of_accounts` INTEGER | |
| `available_sanction_limit` DECIMAL | |
| `available_drawing_power` DECIMAL | |
| `sanction_amount_limit_ccy` DECIMAL | |
| `sanction_amount_home_ccy` DECIMAL | |
| `approval_limit` DECIMAL | |
| `drawing_power_indicator` BOOLEAN | |
| `drawing_power_margin_retained` DECIMAL | |
| `original_limit_expiry_date` DATE | |
| `limit_approval_date` DATE | |
| `limit_expiry_date` DATE | |
| `contract_sign_date` DATE | |
| `limit_effective_date` DATE | |
| `limit_expiry_extended_upto` DATE | |
| `number_of_extensions` INTEGER | |
| `limit_review_date` DATE | |
| `type_of_loan` | |
| `min_collateral_value_based_on` | |
| `approval_authority` | |
| `approval_level` | |
| `limit_status` | |
| `interest_table_code` | |
| `limit_approval_reference` | |
| `pattern_of_funding` | |
| `notes` TEXT | |
| `master_limit_node` | |
| `fees_debit_account` FK → Account | |
| `la_sanction_limit` DECIMAL | |
| `tangible_security` BOOLEAN | |
| `tangible_security_amount` DECIMAL | |
| `intangible_security_amount` DECIMAL | |
| `arrangement_fees` DECIMAL | |

---

### 11.7 CollateralLink

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `limit_id` FK → CreditFacility | |
| `account_number` | |
| `sol_id` | |
| `expiry_date` DATE | |
| `collateral_code` | |
| `collateral_type` | |
| `apportioned_value` DECIMAL | |
| `apportioned_pct` DECIMAL | |
| `primary_secondary` ENUM | |
| `collateral_value` DECIMAL | |
| `currency` | |
| `liability` DECIMAL | |
| `liability_currency` | |
| `ceiling_limit` DECIMAL | |
| `loan_to_value_pct` DECIMAL | |
| `guarantor_type` | |
| `guarantor_id` | |
| `guarantor_name` | |
| `lodged_date` DATE | |
| `withdrawn_date` DATE | |
| `guarantee_type` | |
| `guarantee_pct` DECIMAL | |
| `guarantee_amount` DECIMAL | |
| `received_date` DATE | |
| `notes` TEXT | |

---

### 11.8 LimitDocument

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `limit_id` FK → CreditFacility | |
| `document_code` | |
| `document_description` | |
| `document_date` DATE | |
| `due_date` DATE | |
| `document_released_date` DATE | |
| `document_expiry_date` DATE | |
| `notes` TEXT | |
| `status` | |
| `reminder_date` DATE | |
| `waiver_date` DATE | |
| `document_lodgment_date` DATE | |
| `value_in_aed` DECIMAL | |

---

### 11.9 OverdraftFacility

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `application_from_date` DATE | |
| `expiry_date` DATE | |
| `sanction_limit` DECIMAL | |
| `normal_interest_rate` DECIMAL | |
| `penal_interest_rate` DECIMAL | |
| `margin_interest_rate` DECIMAL | |
| `status` | |
| `sanction_date` DATE | |
| `review_date` DATE | |
| `sanction_level` | |
| `sanction_authority` | |
| `sanction_reference_no` | |
| `collateral_description` | |
| `remarks` TEXT | |

---

### 11.10 Transfer *(Remittance)*

| Attribute | Notes |
|-----------|-------|
| `payment_order_id` **PK** | |
| `account_number` FK → Account | |
| `channel_id` | |
| `related_reference` | |
| `sender_reference` | |
| `debit_account_number` | |
| `status` | |
| `inward_outward_indicator` ENUM | |
| `requested_execution_date` DATE | |
| `payment_product` | |
| `exchange_rate_code` | |
| `fx_rate` DECIMAL | |
| `debit_execution_date` DATE | |
| `remittance_currency` | |
| `remittance_amount` DECIMAL | |
| `ordered_currency` | |
| `ordered_amount` DECIMAL | |
| `debit_value_date` DATE | |
| `payment_currency` | |
| `payment_amount` DECIMAL | |
| `credit_execution_date` DATE | |
| `settlement_mode` | |
| `credit_value_date` DATE | |
| `beneficiary_name` | |
| `beneficiary_account_number` | |
| `beneficiary_bank_code` | |
| `beneficiary_bic` | |
| `nature_of_payment` | |
| `charge_option` | |
| `net_charges` DECIMAL | |
| `purpose_code` | |
| `transaction_reference` | |
| `rejection_code` | |
| `rejection_description` | |
| `reason_for_deletion` | |
| `reason_for_reject` | |

---

### 11.11 InstantPayment

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `channel` | |
| `message_id` | |
| `transaction_id` | |
| `value_date` DATE | |
| `receiver_bank` | |
| `sender` | |
| `sender_mobile` | |
| `amount` DECIMAL | |
| `settlement_amount` DECIMAL | |
| `settlement_status` | |
| `current_status` | |
| `settlement_date` DATE | |
| `creditor_name` | |
| `creditor_bank` | |
| `creditor_iban` | |
| `sender_iban` | |
| `charge_type` | |
| `purpose_code` | |
| `turn_around_time` | |
| `host_reference_number` | |
| `reversal_reference_number` | |

---

### 11.12 SalaryTransaction

| Attribute | Notes |
|-----------|-------|
| `id` **PK** | |
| `account_number` FK → Account | |
| `date` DATE | |
| `salary_type` | |
| `salary_amount` DECIMAL | |
| `salary_for_month` DATE | |
| `status` | |

---

## Entity Relationship Summary

```
Customer (1) ──────────────────────────────────────────────────────────────────
   │                                                                          │
   ├─< CustomerStatusRecord (blacklist/negated)                    ─────── Account (N)
   ├─< CustomerPreference (1:1)                                              │
   ├─< CustomerConsentRecord (N)                                             ├─< AccountInterest
   ├─< ChannelSubscription (1:1)                                             ├─< AccountStatement
   ├─< OnlineBankingSubscription (1:1)                                       ├─< JointAccountLink ──> Customer
   ├─< Employment (1:1, Individual only)                                     ├─< AccountTransaction
   ├─< CustomerSegment (N)                                                   ├─< StandingInstruction
   ├─< CustomerPhone (N)                                                     ├─< ChequeBook ─< ChequeLeaf
   ├─< CustomerEmail (N)                                                     ├─< Lien
   ├─< CustomerAddress (N)                                                   ├─< OverdraftFacility
   ├─< CustomerDocument (N)                                                  ├─< Transfer
   ├─< KYCDetails (1:1)                                                      ├─< InstantPayment
   ├─< FATCADetails (1:1)                                                    ├─< SweepPool (M:N)
   ├─< CRSDetails (1:1)                                                      └─< SalaryTransaction
   ├─< CurrencyPreferentialRate (N)
   ├─< CustomerGroup (N, Non-Individual)                          ─────── Deposit (N)
   ├─< CustomerRelationship (N)
   ├─< AuthorizedRepresentative (N)                               ─────── DebitCard (N) ─< CardToken
   ├─< Footprint (N)                                                                     ─< CardAuthTxn
   ├─< Communication (N)
   ├─< OnlineBankingRequest (N)                                   ─────── CreditCard (N) ─< CardToken
   ├─< RAKValuePackage (N)                                                               ─< CardBalance
   ├── RM → RelationshipManager                                                          ─< CardStatement
   │                                                                                     ─< CardInstallment
   ├─< RetailLoan (N) ─< LoanGuarantorCoApplicant                                       ─< CardInsurance
   │                   ─< RepaymentSchedule                                              ─< CardReward
   │                   ─< RetailLoanAutoDetails
   │                   ─< RetailLoanHomeDetails                   ─────── PrepaidCard (N)
   │
   ├─< SMELoan (N)                                                ─────── DigitalAccessCard (N)
   │
   ├─< IslamicFinance (N)                                         ─────── InvestmentPortfolio (1:1)
   │                                                                        ─< MutualFundHolding
   ├─< TradeFinance (N)                                                     ─< StructuredProductHolding
   │                                                                        ─< EquityHolding
   ├─< CreditFacility/Limit (N) ─< CollateralLink                          ─< BondHolding
   │                             ─< LimitDocument                          ─< FuturesHolding
   │                                                                        ─< PreciousMetalHolding
   ├─< InsurancePolicy (N)                                                  ─< PMSHolding
   │    ─< InsuranceBeneficiary                                             ─< AlternateInvestmentHolding
   │    ─< InsurancePremiumPayment                                          ─< InvestmentInsurancePolicy
   │
   ├─< ServiceRequest (N)
   └─< Lead (N)
```

---

## Notes on Design Decisions

1. **Customer inheritance**: `IndividualCustomer` and `NonIndividualCustomer` both extend `Customer`. In a relational database this is implemented as a base `Customer` table with `customer_type` discriminator + two sub-tables. In document stores or object models, polymorphism is used directly.

2. **Shared CRM sections** (Status, Preferences, KYC, Contact, Documents) are identical in structure for both customer types and are modeled once with a `customer_id` FK.

3. **Card unification**: `DebitCard`, `CreditCard`, `PrepaidCard`, and `DigitalAccessCard` are kept as separate entities because they have meaningfully different attribute sets (credit limits, installments, rewards) but share a common `CardAuthorizationTransaction` and `CardToken` structure.

4. **Loan product hierarchy**: `RetailLoan` has two product-specific sub-tables (`Auto`, `Home`). `SMELoan` and `IslamicFinance` are separate entities despite structural overlap because they have distinct financial concepts (profit rate vs. interest rate, Murabaha vs. conventional, etc.).

5. **CreditFacility/Limit** is shared across `SMELoan` and `RetailLoan` screens – the same limit node hierarchy, collateral links and document tracking structure applies to both.

6. **Transactions** are unified in `AccountTransaction` across current/savings accounts, deposits (via `Deposit.deposit_account_number`), and SME loan accounts; each has a `source_type` discriminator.

7. **Enquiry-only data** (calculators, delivery tracking, FX rates) represents read-only views derived from core system data and are modeled as view/query structures rather than persisted entities in the CRM.

8. **Audit trail entities** (OnlineBankingSubscriptionHistory, CardActivityHistory, PackageHistory, InsurancePolicyHistory, BalanceHistory, LienHistory) are kept as append-only log tables with `customer_id` / product FK and `action_datetime`.
