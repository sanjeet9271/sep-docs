# Cases Table - Complete Documentation

This document provides comprehensive documentation for the `cases` table, including schema definition, API mappings, and SOQL queries.

---

## Table of Contents

1. [Schema Definition](#schema-definition)
2. [API to Database Mapping](#api-to-database-mapping)
3. [SOQL Queries](#soql-queries)

---

# Schema Definition

## Table: `cases`

Stores cases synced from Salesforce. This is the source of truth for submitted cases.

---

## PostgreSQL Table Definition

```sql
CREATE TABLE cases (
    case_id VARCHAR(18) PRIMARY KEY,              -- Salesforce Case ID (18 characters)
    case_number VARCHAR(20) NOT NULL,             -- Case # (e.g., CS-00012345)
    case_type VARCHAR(50),                        -- WarrantyClaim, RMARepair, etc.
    account_id VARCHAR(50),                       -- Account ID for filtering
    status VARCHAR(50),                           -- Open, In Progress, Closed
    progress INTEGER,                             -- Progress (0-100) [DEPRECATED - FE calculates]
    serial_number VARCHAR(100),                   -- Serial # for display + filtering
    part_number VARCHAR(100),                     -- Part # for display + filtering
    product_description VARCHAR(255),             -- Product description for display
    subject VARCHAR(255),                         -- Subject/title for display
    submitted_at TIMESTAMPTZ,                     -- Original submission date
    case_data JSONB NOT NULL,                     -- Full case details from SF
    synced_at TIMESTAMPTZ NOT NULL DEFAULT NOW()  -- Last sync from Salesforce
);
```

---

## Column Descriptions

| Column | Data Type | Nullable | Description | Source |
|--------|-----------|----------|-------------|--------|
| `case_id` | VARCHAR(18) | NOT NULL (PK) | Salesforce Case ID (18-char format) | Salesforce `Id` |
| `case_number` | VARCHAR(20) | NOT NULL | Human-readable case number | Salesforce `CaseNumber` |
| `case_type` | VARCHAR(50) | NULL | Case type for filtering | Extracted from `case_data` |
| `account_id` | VARCHAR(50) | NULL | Account ID for filtering/authorization | Extracted from `case_data` |
| `status` | VARCHAR(50) | NULL | Current case status | Extracted from `case_data` (prefer External Status) |
| `progress` | INTEGER | NULL | **DEPRECATED** - Frontend calculates from status | N/A |
| `serial_number` | VARCHAR(100) | NULL | Serial number for display/filtering | Extracted from `case_data` |
| `part_number` | VARCHAR(100) | NULL | Part number for display/filtering | Extracted from `case_data` |
| `product_description` | VARCHAR(255) | NULL | Product description for display | Derived from part/serial data |
| `subject` | VARCHAR(255) | NULL | Case subject/title | Extracted from `case_data` |
| `submitted_at` | TIMESTAMPTZ | NULL | Original submission timestamp | Extracted from `case_data` or `CreatedDate` |
| `case_data` | JSONB | NOT NULL | Complete case payload from Salesforce | Full API response |
| `synced_at` | TIMESTAMPTZ | NOT NULL | Last synchronization timestamp | System timestamp |

---

## `case_data` JSONB Structure

The `case_data` column stores the complete case information as a JSONB object.

### Root Structure

```json
{
  "case": { ... },              // Core case fields
  "serviceParts": [ ... ],      // Array of service parts
  "laborCharges": [ ... ],      // Array of labor charges
  "otherCharges": [ ... ]       // Array of other costs
}
```

**Note**: Comments and attachments are stored in separate database tables (`case_comments` and `case_attachments`) and NOT in `case_data` JSONB.

---

### Section 1: `case` Object

Core case information from Salesforce. Field names are simplified for easier database querying.

```json
{
  "case": {
    // Salesforce IDs
    "Id": "500WR00000i20KLYAY",
    "CaseNumber": "06224721",
    
    // Basic Information
    "Subject": "Intermittent communication on TM200",
    "CaseType": "Dealer Warranty Claim",
    "Origin": "MTP",
    "Status": "Closed",
    "ExternalCaseStatus": "Closed",
    "Priority": "Medium",
    "CurrencyIsoCode": "USD",
    "Description": "Intermittent communication on TM200 after AG815 installed.",
    
    // Product & Serial Information
    "SerialNumber": {
      "Name": "5934550042",
      "UniqueId": "5934550042####95060-00"
    },
    "Part": {
      "ExternalNumber": "95060-00",
      "Description": "TM200 Display Module"
    },
    "RepairedProductPartNumber": {
      "ExternalNumber": "string|null"
    },
    
    // Account & Contact Information
    "Account": {
      "ExternalID": "1686224845368638"
    },
    "Contact": {
      "Name": "James Lancaster",
      "ExternalId": "james_lancaster@vantage-au.com"
    },
    "Technician": {
      "Name": "James Lancaster",
      "ExternalId": "james_lancaster@vantage-au.com"
    },
    
    // Warranty Claim Details
    "SalesOrderNumber": "string|null",
    "Symptom": "Communication",
    "Failure": "Intermittent",
    "RepairDescription": "Replaced Digital Board in TM200.",
    "AtlasCaseNumber": "string|null",
    "FailureCategory": "Part Failure",
    "PartnerCRMCase": "302225B",
    "MTPCaseReference": "string|null",
    
    // Dates
    "DateTimeReceived": "2025-08-01T11:00:00.000+0000",
    "DateTimeRepaired": "2025-08-15T11:00:00.000+0000",
    "DateTimePreviousRepair": "ISO-8601|null",
    
    // Warranty Information
    "PolicyType": "Factory Warranty",
    "PolicyNumber": "string|null",
    "PlanType": "string|null",
    "TypeOfContract": "string|null",
    
    // Failure Verification
    "FailureVerified": false,
    "StepsTakenToVerifyTheFailure": "string|null",
    "ApplicableFailureCategories": "Discretionary Reimbursement,Part Failure",
    
    // Exception Flow (Optional/Future)
    "WarrantyException": "Service provided within Warranty",
    "ServiceBulletin": {
      "ExternalId": "string|null"
    },
    "OtherServiceBulletins": "string|null",
    "PreviousAtlasNumber": "string|null",
    "PartAssistance": "string|null",
    "TechnicalAnalysis": "string|null",
    "ReturnToLocationText": "string|null"
  }
}
```

---

### Section 2: `serviceParts` Array

Service parts used in the repair/claim. Field names simplified.

```json
{
  "serviceParts": [
    {
      "PartNumber": {
        "ExternalNumber": "118002-00S"
      },
      "Price": 4500.0,
      "Quantity": 2.0,
      "TotalPartCost": 9000.0,
      "AdditionalReimbursement": 10.0,
      "AdditionalReimbursementAmount": 900.0,
      "CurrencyIsoCode": "USD",
      "ReturnPart": false,
      "IsWarrantable": true,
      "MissingPrice": false,
      "IsPartRecentlyPurchased": false,
      "ExternalId": "string",
      "IsPartPickedFromBOM": false,
      "APIPrice": 4500.0,
      "APIMSG": "string|null"
    }
  ]
}
```

---

### Section 3: `laborCharges` Array

Labor charges associated with the case. Field names simplified.

```json
{
  "laborCharges": [
    {
      "RequestedHours": 2.0,
      "Hours": 3.0,
      "Rate": 75.0,
      "TotalLaborCost": 225.0,
      "CurrencyIsoCode": "USD",
      "IsWarrantable": true
    }
  ]
}
```

---

### Section 4: `otherCharges` Array

Other costs (Freight, Duty, Other). Field names simplified.

```json
{
  "otherCharges": [
    {
      "CostCategoryType": "Freight",
      "UnitUsage": 1.0,
      "AmountPerUnit": 40.0,
      "Amount": 40.0,
      "CurrencyIsoCode": "USD",
      "IsWarrantable": true
    }
  ]
}
```

---

## Indexes

```sql
-- Primary key (implicit index)
-- case_id (PRIMARY KEY)

-- Proposed indexes for filtering and performance
CREATE INDEX idx_cases_account_id ON cases (account_id);
CREATE INDEX idx_cases_type_status ON cases (case_type, status);
CREATE INDEX idx_cases_status ON cases (status);
CREATE INDEX idx_cases_serial_number ON cases (serial_number);
CREATE UNIQUE INDEX ux_cases_case_number ON cases (case_number);

-- Optional: JSONB indexes for specific queries
CREATE INDEX idx_cases_case_data_status ON cases ((case_data->'case'->>'Status'));
CREATE INDEX idx_cases_case_data_external_status ON cases ((case_data->'case'->>'ExternalCaseStatus'));
```

---

## Sample Database Record

Below is an actual sample record from the database (fetched on 2026-01-08):

```json
{
  "case_id": "500WR00000TbFu7YAF",
  "case_number": "06063218",
  "case_type": null,
  "account_id": "000000000000000000",
  "status": "New",
  "progress": null,
  "serial_number": null,
  "part_number": null,
  "product_description": "Not Product Related",
  "subject": "Issues connecting the tablet to the MX7",
  "submitted_at": "2024-01-08T20:45:04+00:00",
  "synced_at": "2025-12-25T18:41:05.959730+00:00",
  "case_data": {
    "case": {
      "Id": "500WR00000TbFu7YAF",
      "CaseNumber": "06063218",
      "Subject": "Issues connecting the tablet to the MX7",
      "CaseType": "",
      "Origin": "E-Mail",
      "Status": "Closed",
      "ExternalCaseStatus": "New",
      "Priority": "Medium",
      "CurrencyIsoCode": "USD",
      "Description": "The small green light on the power box is flashing. Customer isn't sure if this is normal. Also, he tried backing up a mission and got the \"connection lost\" error message. The wifi stick didn't work anymore in the usb port but was working in another usb port. Note: issue was reported on 2024-02-04.",
      "SerialNumber": {
        "Name": "",
        "UniqueId": ""
      },
      "Part": {
        "ExternalNumber": "Not Product Related",
        "Description": "Not Product Related"
      },
      "Account": {
        "ExternalID": "000000000000000000"
      },
      "Contact": {
        "Name": "Scott Mullin",
        "ExternalId": "smullin@publicworks1.com"
      },
      "Technician": {
        "Name": "",
        "ExternalId": ""
      }
    },
    "serviceParts": [],
    "laborCharges": [],
    "otherCharges": []
  }
}
```

**Notes on Sample:**
- This is a support case (not a warranty claim), hence many fields are empty or null
- Status column shows "New" (from ExternalCaseStatus) while case_data.case.Status shows "Closed"
- The `case_data` JSONB contains the full case details with simplified field names
- Service parts, labor charges, and other charges are empty arrays for this particular case

---

## Database Statistics

As of 2026-01-08:
- **Total Cases:** 300,182
- **Cases with Comments:** 25,067 (8.3%)
- **Cases with Attachments:** 10,436 (3.5%)

---

## Notes

1. **Progress Field**: The `progress` column is deprecated. Frontend should calculate progress based on `status`.

2. **Status Handling**: 
   - Store both `Status` and `ExternalCaseStatus` in `case_data`
   - The `status` column should prefer `ExternalCaseStatus` (if not null), otherwise use `Status`

3. **Product Description**: 
   - Not directly available in Salesforce response
   - Can be derived from `SerialNumber.Name` or `Part.ExternalNumber`

4. **Simplified Field Names**:
   - Internal `case_data` uses simplified names (e.g., `CaseType` instead of `twodscp__Case_Type__c`)
   - This makes JSONB querying cleaner: `case_data->'case'->>'CaseType'`
   - Mapping to Salesforce API fields is maintained in the mapping section

5. **Data Consistency**:
   - Extracted columns must match JSONB values
   - Update both when syncing from Salesforce

---

# API to Database Mapping

This section maps database columns and JSONB fields to Salesforce API response paths.

## API Request Reference

- **Request 1 (caseDetailsRequest)**: Fetches core case information
- **Request 2 (caseRelatedDataRequest)**: Fetches service parts, labor, other costs

---

## Table Columns Mapping

| DB Column | API Response Path | Sample Value | Notes |
|-----------|-------------------|--------------|-------|
| `case_id` | `Id` | "500WR00000i20KLYAY" | SF Case ID (18 chars) |
| `case_number` | `CaseNumber` | "06224721" | Human-readable case number |
| `case_type` | `twodscp__Case_Type__c` | "Dealer Warranty Claim" | Extract to column |
| `account_id` | `Account.twodscp__External_ID__c` | "1686224845368638" | External account ID |
| `status` | `twodscp__External_Case_Status__c` OR `Status` | "Closed" | Prefer External Status |
| `progress` | **DEPRECATED** | NULL | Frontend calculates from status |
| `serial_number` | `twodscp__Serial_Number__r.twodscp__Unique_Id__c` | "5934550042####95060-00" | Format: serial####part |
| `part_number` | `twodscp__Part__r.twodscp__External_Number__c` | "95060-00" | From part external number |
| `product_description` | `twodscp__Part__r.Description` | "TM200 Display Module" | From part description |
| `subject` | `Subject` | "Intermittent communication..." | Case subject |
| `submitted_at` | `CreatedDate` | "2025-08-01T11:00:00Z" | Original submission date |
| `case_data` | **See JSONB Mapping below** | {...} | Complete payload |
| `synced_at` | System timestamp (NOW()) | "2025-12-23T12:00:00Z" | Last sync timestamp |

---

## JSONB `case_data` Field Mapping

### Section 1: Core Case Fields (`case_data.case`)

Maps to **Request 1** response

| case_data Field | Salesforce API Field | Sample Value |
|-----------------|----------------------|--------------|
| `case.Id` | `Id` | "500WR00000i20KLYAY" |
| `case.CaseNumber` | `CaseNumber` | "06224721" |
| `case.Subject` | `Subject` | "Intermittent communication..." |
| `case.CaseType` | `twodscp__Case_Type__c` | "Dealer Warranty Claim" |
| `case.Origin` | `Origin` | "MTP" |
| `case.Status` | `Status` | "Closed" |
| `case.ExternalCaseStatus` | `twodscp__External_Case_Status__c` | "Closed" |
| `case.Priority` | `Priority` | "Medium" |
| `case.CurrencyIsoCode` | `CurrencyIsoCode` | "USD" |
| `case.Description` | `Description` | "Intermittent communication..." |
| `case.Symptom` | `twodscp__Symptom__c` | "Communication" |
| `case.Failure` | `twodscp__Failure__c` | "Intermittent" |
| `case.RepairDescription` | `twodscp__Repair_Description__c` | "Replaced Digital Board..." |
| `case.AtlasCaseNumber` | `twodscp__Atlas_Case_Number__c` | null |
| `case.FailureCategory` | `twodscp__Failure_Category__c` | "Part Failure" |
| `case.PartnerCRMCase` | `twodscp__Partner_CRM_Case__c` | "302225B" |
| `case.MTPCaseReference` | `twodscp__MTP_Case_Reference__c` | null |
| `case.SalesOrderNumber` | `twodscp__Sales_Order_Number__c` | null |
| `case.SerialNumber.Name` | `twodscp__Serial_Number__r.Name` | "5934550042" |
| `case.SerialNumber.UniqueId` | `twodscp__Serial_Number__r.twodscp__Unique_Id__c` | "5934550042####95060-00" |
| `case.Part.ExternalNumber` | `twodscp__Part__r.twodscp__External_Number__c` | "95060-00" |
| `case.Part.Description` | `twodscp__Part__r.Description` | "TM200 Display Module" |
| `case.RepairedProductPartNumber.ExternalNumber` | `twodscp__Repaired_Product_Part_Number__r.twodscp__External_Number__c` | null |
| `case.Account.ExternalID` | `Account.twodscp__External_ID__c` | "1686224845368638" |
| `case.Contact.Name` | `Contact.Name` | "James Lancaster" |
| `case.Contact.ExternalId` | `Contact.twodscp__External_Id__c` | "james_lancaster@vantage-au.com" |
| `case.Technician.Name` | `twodscp__Technician__r.Name` | "James Lancaster" |
| `case.Technician.ExternalId` | `twodscp__Technician__r.twodscp__External_Id__c` | "james_lancaster@vantage-au.com" |
| `case.DateTimeReceived` | `twodscp__Date_Time_Received__c` | "2025-08-01T11:00:00.000+0000" |
| `case.DateTimeRepaired` | `twodscp__Date_Time_Repaired__c` | "2025-08-15T11:00:00.000+0000" |
| `case.DateTimePreviousRepair` | `twodscp__Date_Time_Previous_Repair__c` | null |
| `case.PolicyType` | `twodscp__PolicyType__c` | "Factory Warranty" |
| `case.PolicyNumber` | `twodscp__Policy_Number__c` | null |
| `case.PlanType` | `twodscp__Plan_Type__c` | null |
| `case.TypeOfContract` | `twodscp__Type_of_Contract__c` | null |
| `case.FailureVerified` | `twodscp__Failure_Verified__c` | false |
| `case.StepsTakenToVerifyTheFailure` | `twodscp__Steps_Taken_To_Verify_The_Failure__c` | null |
| `case.ApplicableFailureCategories` | `twodscp__Applicable_Failure_Categories__c` | "Discretionary Reimbursement,Part Failure" |
| `case.WarrantyException` | `twodscp__Warranty_Exception__c` | "Service provided within Warranty" |
| `case.ServiceBulletin.ExternalId` | `twodscp__Service_Bulletin__r.twodscp__External_Id__c` | null |
| `case.OtherServiceBulletins` | `twodscp__Other_Service_Bulletins__c` | null |
| `case.PreviousAtlasNumber` | `twodscp__Previous_Atlas_Number__c` | null |
| `case.PartAssistance` | `twodscp__Part_Assistance__c` | null |
| `case.TechnicalAnalysis` | `twodscp__Technical_Analysis__c` | null |
| `case.ReturnToLocationText` | `twodscp__Return_To_Location_Text__c` | null |

---

### Section 2: Service Parts (`case_data.serviceParts[]`)

Maps to **Request 2** response (attachments)

| case_data Field | Salesforce API Field | Sample Value |
|-----------------|----------------------|--------------|
| `serviceParts[n].PartNumber.ExternalNumber` | `twodscp__Part_Number__r.twodscp__External_Number__c` | "118002-00S" |
| `serviceParts[n].Price` | `twodscp__Price__c` | 4500.0 |
| `serviceParts[n].Quantity` | `twodscp__Quantity__c` | 2.0 |
| `serviceParts[n].TotalPartCost` | `twodscp__Total_Part_Cost__c` | 9000.0 |
| `serviceParts[n].AdditionalReimbursement` | `twodscp__Additional_Reimbursement__c` | 10.0 |
| `serviceParts[n].AdditionalReimbursementAmount` | `twodscp__Additional_Reimbursement_Amount__c` | 900.0 |
| `serviceParts[n].CurrencyIsoCode` | `CurrencyIsoCode` | "USD" |
| `serviceParts[n].ReturnPart` | `twodscp__Return_Part__c` | false |
| `serviceParts[n].IsWarrantable` | `twodscp__Is_Warrantable__c` | true |
| `serviceParts[n].MissingPrice` | `twodscp__Missing_Price__c` | false |
| `serviceParts[n].IsPartRecentlyPurchased` | `twodscp__Is_Part_Recently_Purchased__c` | false |
| `serviceParts[n].ExternalId` | `twodscp__External_Id__c` | "EXT123" |
| `serviceParts[n].IsPartPickedFromBOM` | `twodscp__Is_Part_Picked_From_BOM__c` | false |
| `serviceParts[n].APIPrice` | `twodscp__API_Price__c` | 4500.0 |
| `serviceParts[n].APIMSG` | `twodscp__API_MSG__c` | null |

---

### Section 3: Labor Charges (`case_data.laborCharges[]`)

Maps to **Request 2** response (comments)

| case_data Field | Salesforce API Field | Sample Value |
|-----------------|----------------------|--------------|
| `laborCharges[n].RequestedHours` | `twodscp__Requested_Hours__c` | 2.0 |
| `laborCharges[n].Hours` | `twodscp__Hours__c` | 3.0 |
| `laborCharges[n].Rate` | `twodscp__Rate__c` | 75.0 |
| `laborCharges[n].TotalLaborCost` | `twodscp__Total_Labor_Cost__c` | 225.0 |
| `laborCharges[n].CurrencyIsoCode` | `CurrencyIsoCode` | "USD" |
| `laborCharges[n].IsWarrantable` | `twodscp__Is_Warrantable__c` | true |

---

### Section 4: Other Charges (`case_data.otherCharges[]`)

Maps to **Request 2** response (feed items)

| case_data Field | Salesforce API Field | Sample Value |
|-----------------|----------------------|--------------|
| `otherCharges[n].CostCategoryType` | `twodscp__Cost_Category_Type__c` | "Freight" |
| `otherCharges[n].UnitUsage` | `twodscp__Unit_Usage__c` | 1.0 |
| `otherCharges[n].AmountPerUnit` | `twodscp__Amount_Per_Unit__c` | 40.0 |
| `otherCharges[n].Amount` | `twodscp__Amount__c` | 40.0 |
| `otherCharges[n].CurrencyIsoCode` | `CurrencyIsoCode` | "USD" |
| `otherCharges[n].IsWarrantable` | `twodscp__Is_Warrantable__c` | true |

---

# SOQL Queries

This section provides the SOQL queries needed to fetch complete case data from Salesforce.

## Query Strategy

To populate the `cases` table completely, we need **TWO composite API requests**:

1. **Request 1 (caseDetailsRequest)**: Fetches core case information
2. **Request 2 (caseRelatedDataRequest)**: Fetches service parts, labor, other costs

---

## Request 1: Case Details Query

### Composite Request Body

```json
{
  "allOrNone": false,
  "compositeRequest": [
    {
      "method": "GET",
      "url": "/services/data/v60.0/query?q=<SOQL_QUERY>",
      "referenceId": "caseDetailsQuery"
    }
  ]
}
```

### SOQL Query (Formatted)

```sql
SELECT 
  Id,
  CaseNumber,
  
  -- Basic Information
  Subject,
  twodscp__Case_Type__c,
  Origin,
  Status,
  twodscp__External_Case_Status__c,
  Priority,
  CurrencyIsoCode,
  Description,
  
  -- Product & Serial Information (Relationships)
  twodscp__Serial_Number__r.Name,
  twodscp__Serial_Number__r.twodscp__Unique_Id__c,
  twodscp__Part__r.twodscp__External_Number__c,
  twodscp__Part__r.Description,
  twodscp__Repaired_Product_Part_Number__r.twodscp__External_Number__c,
  
  -- Account & Contact Information (Relationships)
  Account.twodscp__External_ID__c,
  Contact.Name,
  Contact.twodscp__External_Id__c,
  twodscp__Technician__r.Name,
  twodscp__Technician__r.twodscp__External_Id__c,
  
  -- Warranty Claim Details
  twodscp__Sales_Order_Number__c,
  twodscp__Symptom__c,
  twodscp__Failure__c,
  twodscp__Repair_Description__c,
  twodscp__Atlas_Case_Number__c,
  twodscp__Failure_Category__c,
  twodscp__Partner_CRM_Case__c,
  twodscp__MTP_Case_Reference__c,
  
  -- Dates
  twodscp__Date_Time_Received__c,
  twodscp__Date_Time_Repaired__c,
  twodscp__Date_Time_Previous_Repair__c,
  CreatedDate,
  LastModifiedDate,
  
  -- Warranty Information
  twodscp__PolicyType__c,
  twodscp__Policy_Number__c,
  twodscp__Plan_Type__c,
  twodscp__Type_of_Contract__c,
  
  -- Failure Verification
  twodscp__Failure_Verified__c,
  twodscp__Steps_Taken_To_Verify_The_Failure__c,
  twodscp__Applicable_Failure_Categories__c,
  
  -- Exception Flow (Optional)
  twodscp__Warranty_Exception__c,
  twodscp__Service_Bulletin__r.twodscp__External_Id__c,
  twodscp__Other_Service_Bulletins__c,
  twodscp__Previous_Atlas_Number__c,
  twodscp__Part_Assistance__c,
  twodscp__Technical_Analysis__c,
  twodscp__Return_To_Location_Text__c

FROM Case
```
---

### Query 2.1: Service Parts

#### Formatted SOQL

```sql
SELECT 
  Id,
  twodscp__Case__c,
  twodscp__Case__r.CaseNumber,
  twodscp__Case__r.Id,
  
  twodscp__Part_Number__r.twodscp__External_Number__c,
  twodscp__Price__c,
  twodscp__Quantity__c,
  twodscp__Total_Part_Cost__c,
  twodscp__Additional_Reimbursement__c,
  twodscp__Additional_Reimbursement_Amount__c,
  CurrencyIsoCode,
  
  twodscp__Return_Part__c,
  twodscp__Is_Warrantable__c,
  twodscp__Missing_Price__c,
  twodscp__Is_Part_Recently_Purchased__c,
  twodscp__External_Id__c,
  twodscp__Is_Part_Picked_From_BOM__c,
  twodscp__API_Price__c,
  twodscp__API_MSG__c

FROM twodscp__Service_Part__c
```
---

### Query 2.2: Labor Charges

#### Formatted SOQL

```sql
SELECT 
  Id,
  twodscp__Case__c,
  twodscp__Case__r.CaseNumber,
  twodscp__Case__r.Id,
  
  twodscp__Requested_Hours__c,
  twodscp__Hours__c,
  twodscp__Rate__c,
  twodscp__Total_Labor_Cost__c,
  CurrencyIsoCode,
  twodscp__Is_Warrantable__c

FROM twodscp__Labor__c
```

---

### Query 2.3: Other Costs

#### Formatted SOQL

```sql
SELECT 
  Id,
  twodscp__Case__c,
  twodscp__Case__r.CaseNumber,
  twodscp__Case__r.Id,
  
  twodscp__Cost_Category_Type__c,
  twodscp__Unit_Usage__c,
  twodscp__Amount_Per_Unit__c,
  twodscp__Amount__c,
  CurrencyIsoCode,
  twodscp__Is_Warrantable__c

FROM twodscp__Other_Cost__c
```
---

## Query Performance Tips

1. **Use CaseNumber for lookups** - It's more readable and indexed
2. **Limit bulk queries** - Use `LIMIT 200` for batch operations
3. **Filter by date** - Use `LastModifiedDate` for incremental syncs
4. **Query by account** - Filter by `Account.twodscp__External_ID__c` for tenant isolation
5. **Composite API** - Use composite requests to reduce roundtrips (max 25 subrequests per call)

---

## Notes

- All relationship queries use `__r` notation (e.g., `twodscp__Serial_Number__r.Name`)
- Always include `Id` and `CaseNumber` for reliable joins
- Child queries reference `twodscp__Case__r.CaseNumber`
- Simplified field names in `case_data` make JSONB queries cleaner
- Comments and attachments are queried separately and stored in their own tables
