# Case Attachments Schema and Mapping

## Database Schema: case_attachments

```sql
CREATE TABLE case_attachments (
    attachment_id VARCHAR(10) PRIMARY KEY,
    case_id VARCHAR(18) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    s3_key VARCHAR(500) NOT NULL,
    sync_status VARCHAR(20) DEFAULT 'synced',
    sf_attachment_id VARCHAR(18),
    sync_error TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT fk_case_attachments_case 
        FOREIGN KEY (case_id) REFERENCES cases(case_id)
        ON DELETE CASCADE
);

CREATE INDEX idx_case_attachments_case_id ON case_attachments(case_id);
CREATE INDEX idx_case_attachments_created_at ON case_attachments(created_at);
CREATE INDEX idx_case_attachments_sf_attachment_id ON case_attachments(sf_attachment_id);
CREATE INDEX idx_case_attachments_file_name ON case_attachments(file_name);
```

## Column Mapping

| Database Column | Salesforce Field | Type | Description |
|----------------|------------------|------|-------------|
| attachment_id | - | VARCHAR(10) | Auto-generated: att_xxxx |
| case_id | Case__c | VARCHAR(18) | Salesforce Case ID (FK) |
| file_name | Name | VARCHAR(255) | Original filename (NOT NULL) |
| s3_key | Link__c (parsed) | VARCHAR(500) | S3 path: uploadFile/... (NOT NULL) |
| sync_status | - | VARCHAR(20) | pending\|syncing\|synced\|sync_failed |
| sf_attachment_id | Id | VARCHAR(18) | SF Cloud_Attachment__c ID |
| sync_error | - | TEXT | Error on sync failure |
| created_at | CreatedDate | TIMESTAMPTZ | When attachment was created |

## SOQL Query to Fetch Attachments

### Basic Query (All Attachments)
```sql
SELECT 
    Id,
    Case__c,
    Name,
    Link__c,
    Attachment_Type__c,
    CreatedDate
FROM Cloud_Attachment__c
ORDER BY CreatedDate DESC
```

## S3 Key Extraction

### Link__c Format
```
https://trimble-gs-prod.s3.amazonaws.com/uploadFile/filename_timestamp.ext
```

### Extracted s3_key
```
uploadFile/filename_timestamp.ext
```

### Extraction Logic
```python
def extract_s3_key(link_url):
    if 'uploadFile/' in link_url:
        key = link_url.split('uploadFile/', 1)[1]
        return f"uploadFile/{key}"
    return None
```

## Sample Data

### Salesforce Cloud_Attachment__c
```json
{
    "Id": "a1rWR00000EUGX7YAP",
    "Case__c": "500WR00000Ive6QYAR",
    "Name": "Produk Poltek Batam.pdf",
    "Link__c": "https://trimble-gs-prod.s3.amazonaws.com/uploadFile/Produk%20Poltek%20Batam_1735006934333.pdf",
    "Attachment_Type__c": "",
    "CreatedDate": "2024-12-24T02:23:23.000+0000"
}
```

### Database Record (Example from Salesforce)
```json
{
    "attachment_id": "att_h20q",
    "case_id": "500WR00000Ive6QYAR",
    "file_name": "Produk Poltek Batam.pdf",
    "s3_key": "uploadFile/Produk%20Poltek%20Batam_1735006934333.pdf",
    "sync_status": "synced",
    "sf_attachment_id": "a1rWR00000EUGX7YAP",
    "sync_error": null,
    "created_at": "2024-12-24T02:23:23+00:00"
}
```

---

## Sample Database Record

Below is an actual sample record from the database (fetched on 2026-01-08):

```json
{
  "attachment_id": "att_9v2z",
  "case_id": "5001234567890ABCDE",
  "file_name": "warranty_document.pdf",
  "s3_key": "salesforce-attachments/2024/12/warranty_document_001.pdf",
  "sync_status": "sync_failed",
  "sf_attachment_id": null,
  "sync_error": "Salesforce API timeout after 30 seconds",
  "created_at": "2025-12-29T17:29:55.990164+00:00"
}
```

**Notes on Sample:**
- The `s3_key` contains the path to the file in S3 storage
- `sync_status` can be: pending, syncing, synced, or sync_failed
- `sf_attachment_id` is populated after successful sync to Salesforce
- `sync_error` contains error details when sync fails

---

## Database Statistics

As of 2026-01-08:
- **Total Attachments:** 18,954
- **Cases with Attachments:** 10,436 (3.5% of total cases)

---

## Notes

- All attachment_id values are auto-generated using pattern: `att_` + 4 random alphanumeric chars
- S3 keys are extracted from Link__c URLs or generated for new uploads
- S3 bucket is typically `trimble-gs-prod` (visible in Link__c URLs from Salesforce)
- File names are preserved exactly as stored in Salesforce
- Both `file_name` and `s3_key` are required fields (NOT NULL)
- Created timestamp is preserved from Salesforce CreatedDate
- Foreign key constraint ensures referential integrity with cases table
- 100% of attachments have valid S3 keys extracted
