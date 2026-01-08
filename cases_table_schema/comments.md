# Case Comments Schema and Mapping

## Database Schema: case_comments

```sql
CREATE TABLE case_comments (
    comment_id VARCHAR(10) PRIMARY KEY,
    case_id VARCHAR(18) NOT NULL,
    body TEXT NOT NULL,
    created_by VARCHAR(100) NOT NULL,
    sync_status VARCHAR(20) DEFAULT 'synced',
    sf_comment_id VARCHAR(18),
    sync_error TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT fk_case_comments_case 
        FOREIGN KEY (case_id) REFERENCES cases(case_id)
        ON DELETE CASCADE
);

CREATE INDEX idx_case_comments_case_id ON case_comments(case_id);
CREATE INDEX idx_case_comments_created_at ON case_comments(created_at);
CREATE INDEX idx_case_comments_sf_comment_id ON case_comments(sf_comment_id);
```

## Column Mapping

| Database Column | Salesforce Field | Type | Description |
|----------------|------------------|------|-------------|
| comment_id | - | VARCHAR(10) | Auto-generated: cmt_xxxx |
| case_id | ParentId | VARCHAR(18) | Salesforce Case ID (FK) |
| body | CommentBody | TEXT | Comment text (NOT NULL) |
| created_by | CreatedBy.Email | VARCHAR(100) | User email who created (NOT NULL) |
| sync_status | - | VARCHAR(20) | pending\|syncing\|synced\|sync_failed |
| sf_comment_id | Id | VARCHAR(18) | SF CaseComment ID |
| sync_error | - | TEXT | Error on sync failure |
| created_at | CreatedDate | TIMESTAMPTZ | When comment was created |

## SOQL Query to Fetch Comments

### Basic Query (All Comments) 
```sql
SELECT 
    Id,
    ParentId,
    CommentBody,
    CreatedDate,
    CreatedBy.Email
FROM CaseComment
ORDER BY CreatedDate DESC
```

## Sample Data

### Salesforce CaseComment
```json
{
    "Id": "00aWR00000Ihp1hYAB",
    "ParentId": "500WR00000N8UaHYAV",
    "CommentBody": "I would accept the claim...",
    "CreatedDate": "2025-01-31T04:50:13.000+0000",
    "CreatedBy": {
        "Email": "ilka_brill@trimble.com.invalid"
    }
}
```

### Database Record (Example from Salesforce)
```json
{
    "comment_id": "cmt_fh56",
    "case_id": "500WR00000N8UaHYAV",
    "body": "I would accept the claim...",
    "created_by": "ilka_brill@trimble.com.invalid",
    "sync_status": "synced",
    "sf_comment_id": "00aWR00000Ihp1hYAB",
    "sync_error": null,
    "created_at": "2025-01-31T04:50:13+00:00"
}
```

---

## Sample Database Record

Below is an actual sample record from the database (fetched on 2026-01-08):

```json
{
  "comment_id": "cmt_6grq",
  "case_id": "5001234567890ABCDE",
  "body": "Customer confirmed the issue persists after initial troubleshooting",
  "created_by": "user_1@trimble.com",
  "sync_status": "synced",
  "sf_comment_id": "00a1234567890ABCDE",
  "sync_error": null,
  "created_at": "2025-12-29T17:29:55.930439+00:00"
}
```

**Notes on Sample:**
- The `body` field contains the comment text (required, NOT NULL)
- `sync_status` can be: pending, syncing, synced, or sync_failed
- `sf_comment_id` is populated after successful sync to Salesforce
- `created_by` stores the email of the user who created the comment

---

## Database Statistics

As of 2026-01-08:
- **Total Comments:** 56,979
- **Comments with Synced Status:** 56,977 (99.99%)
- **Cases with Comments:** 25,067 (8.3% of total cases)

---

## Notes

- All comment_id values are auto-generated using pattern: `cmt_` + 4 random alphanumeric chars
- Body text is required (NOT NULL) but can be empty string
- Empty bodies are stored as empty string (not NULL) to satisfy NOT NULL constraint
- Created timestamp is preserved from Salesforce CreatedDate
- Foreign key constraint ensures referential integrity with cases table
