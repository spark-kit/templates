---
name: nocodb
description: NocoDB v3 REST API reference. Covers data CRUD, meta management, links, filters, sorts, attachments, documents. Includes CLI script.
metadata:
  openclaw:
    requires:
      env:
        - NOCODB_TOKEN
        - NOCODB_URL
      bins:
        - curl
        - jq
    primaryEnv: NOCODB_TOKEN
---

# NocoDB v3 API Reference

Source: https://github.com/nocodb/noco-apis-doc (swagger specs).

NocoDB 2026.04.5+ requires v3 API with PAT tokens (`nc_pat_...`). v1/v2 return 403.

## Setup

```bash
export NOCODB_TOKEN="your-api-token"
export NOCODB_URL="https://nocodb.example.com"   # no trailing slash
```

Auth header: `xc-token: $NOCODB_TOKEN`

Get a token: NocoDB UI → Team & Settings → API Tokens → Add New Token.

## ID Prefixes

`w` = workspace, `p` = base, `m` = table, `c` = column/field, `vw` = view

## Record Format

v3 uses wrapped records: `{id: <pk>, fields: {field1: val1, ...}}` — not flat like older APIs.

Pagination: `page` (default 1) + `pageSize` (default 25).

---

# Data API

Base path: `/api/v3/data/{baseId}/{tableId}`

### List Records

```
GET /api/v3/data/{baseId}/{tableId}/records
```

Query params: `fields` (array or comma-string), `sort` (structured `[{field, direction}]`), `where`, `page` (default 1), `pageSize` (default 25), `viewId`, `nestedPage`

Response:
```json
{"records": [{"id": 1, "fields": {"Name": "Alice"}}], "next": "...", "prev": "..."}
```

### Get Record

```
GET /api/v3/data/{baseId}/{tableId}/records/{recordId}
```

### Create Records

```
POST /api/v3/data/{baseId}/{tableId}/records
Content-Type: application/json

{"fields": {"Name": "Alice"}}              # single
[{"fields": {"Name": "A"}}, {"fields": {"Name": "B"}}]  # bulk
```

### Update Records

```
PATCH /api/v3/data/{baseId}/{tableId}/records
Content-Type: application/json

{"id": 31, "fields": {"Name": "Updated"}}                 # single (wrapped in array by server)
[{"id": 31, "fields": {...}}, {"id": 32, "fields": {...}}] # bulk
```

### Delete Records

```
DELETE /api/v3/data/{baseId}/{tableId}/records
Content-Type: application/json

{"id": 31}                        # single
[{"id": 31}, {"id": 32}]          # bulk
```

Response: `{"records": [{"id": 31, "deleted": true}]}`

### Count

```
GET /api/v3/data/{baseId}/{tableId}/count
```

Query params: `where`, `viewId`

### Links

```
GET    /api/v3/data/{baseId}/{tableId}/links/{linkFieldId}/{recordId}
POST   /api/v3/data/{baseId}/{tableId}/links/{linkFieldId}/{recordId}
DELETE /api/v3/data/{baseId}/{tableId}/links/{linkFieldId}/{recordId}
```

Body for POST/DELETE: `[{"id": 42}]` (max 1000)
Response: `{"success": true}`

### Documents (Business/Enterprise only)

```
GET    /api/v3/docs/{baseId}                    # list documents
POST   /api/v3/docs/{baseId}                    # create document
GET    /api/v3/docs/{baseId}/{docId}            # get document
PATCH  /api/v3/docs/{baseId}/{docId}            # update document (send version for optimistic concurrency)
DELETE /api/v3/docs/{baseId}/{docId}            # delete document
PATCH  /api/v3/docs/{baseId}/{docId}/reorder    # reorder/move document
```

---

# Meta API

Base path: `/api/v3/meta`

### Workspaces

```
GET    /api/v3/meta/workspaces
```

### Bases

```
GET    /api/v3/meta/workspaces/{workspaceId}/bases
POST   /api/v3/meta/workspaces/{workspaceId}/bases
GET    /api/v3/meta/bases/{baseId}
PATCH  /api/v3/meta/bases/{baseId}
DELETE /api/v3/meta/bases/{baseId}
```

### Tables

```
GET    /api/v3/meta/bases/{baseId}/tables
POST   /api/v3/meta/bases/{baseId}/tables
GET    /api/v3/meta/bases/{baseId}/tables/{tableId}
PATCH  /api/v3/meta/bases/{baseId}/tables/{tableId}
DELETE /api/v3/meta/bases/{baseId}/tables/{tableId}
```

### Fields

```
POST   /api/v3/meta/bases/{baseId}/tables/{tableId}/fields
GET    /api/v3/meta/bases/{baseId}/fields/{fieldId}
PATCH  /api/v3/meta/bases/{baseId}/fields/{fieldId}
DELETE /api/v3/meta/bases/{baseId}/fields/{fieldId}
```

### Views

```
GET    /api/v3/meta/bases/{baseId}/tables/{tableId}/views
```

### Filters

```
GET    /api/v3/meta/bases/{baseId}/views/{viewId}/filters
POST   /api/v3/meta/bases/{baseId}/views/{viewId}/filters
PUT    /api/v3/meta/bases/{baseId}/views/{viewId}/filters    # replace all
PATCH  /api/v3/meta/bases/{baseId}/filters/{filterId}
DELETE /api/v3/meta/bases/{baseId}/filters/{filterId}
```

### Sorts

```
GET    /api/v3/meta/bases/{baseId}/views/{viewId}/sorts
POST   /api/v3/meta/bases/{baseId}/views/{viewId}/sorts
PATCH  /api/v3/meta/bases/{baseId}/sorts/{sortId}
DELETE /api/v3/meta/bases/{baseId}/sorts/{sortId}
```

### Base Users

```
GET    /api/v3/meta/bases/{baseId}/users
POST   /api/v3/meta/bases/{baseId}/users
PATCH  /api/v3/meta/bases/{baseId}/users
DELETE /api/v3/meta/bases/{baseId}/users
```

---

# Where Filter Syntax

### Basic

```
(field,operator,value)
(field,operator)                    # null/blank/checked operators
(field,operator,sub_op)             # date operators
(field,operator,sub_op,value)       # date with value
```

### Operators

| Operator | Description | Example |
|----------|-------------|---------|
| eq | Equal | `(name,eq,John)` |
| neq | Not equal | `(status,neq,archived)` |
| like | Contains (% wildcard) | `(name,like,%john%)` |
| nlike | Does not contain | `(name,nlike,%test%)` |
| in | In list | `(status,in,active,pending)` |
| gt, lt, gte, lte | Numeric comparison | `(price,gt,100)` |
| btw, nbtw | Between / not between | `(price,btw,10,100)` |
| blank, notblank | Null or empty | `(notes,blank)` |
| null, notnull | Is null | `(deleted_at,null)` |
| empty, notempty | Empty string | `(description,empty)` |
| checked, notchecked | Boolean | `(is_active,checked)` |
| allof, anyof, nallof, nanyof | Multi-select | `(tags,anyof,bug,feature)` |

### Date Operators

```
(created_at,eq,today)
(created_at,isWithin,pastWeek)
(created_at,isWithin,pastNumberOfDays,14)
(due_date,lt,today)
(event_date,eq,exactDate,2024-06-15)
(created_at,gte,daysAgo,7)
```

Date sub-ops: `today`, `tomorrow`, `yesterday`, `oneWeekAgo`, `oneWeekFromNow`, `daysAgo`, `daysFromNow`, `exactDate`, `pastWeek`, `pastMonth`, `pastYear`, `nextWeek`, `nextMonth`, `nextYear`, `pastNumberOfDays`, `nextNumberOfDays`

### Combining (tilde prefix!)

```
(name,eq,John)~and(age,gte,18)
(status,eq,active)~or(status,eq,pending)
~not(is_deleted,checked)
```

---

# Field Types

SingleLineText, LongText, Number, Decimal, Currency, Percent, Duration, Email, URL, PhoneNumber, Date, DateTime, Time, SingleSelect, MultiSelect, Checkbox, Rating, Attachment, Links, LinkToAnotherRecord, Lookup, Rollup, Formula, Barcode, QRCode, Geometry, User, JSON, CreatedAt, LastModifiedAt, CreatedBy, LastModifiedBy, Button

---

# CLI Script

The bundled `scripts/nocodb.sh` targets v3 API. Requires `NOCODB_TOKEN`, optionally `NOCODB_URL` (defaults to `https://app.nocodb.com`).

```bash
export NOCODB_TOKEN="..."
export NOCODB_URL="https://your-instance.com"
alias nc="bash /path/to/scripts/nocodb.sh"

nc workspace:list
nc base:list <workspace>
nc table:list <base>
nc field:list <base> <table>
nc record:list <base> <table> [page] [size] [where] [sort] [fields] [viewId]
nc record:create <base> <table> '{"fields":{"Name":"Alice"}}'
nc record:update <base> <table> <id> '{"Name":"Updated"}'
nc record:delete <base> <table> <id>
nc link:list <base> <table> <linkField> <recordId>
nc link:add <base> <table> <linkField> <recordId> '[{"id":42}]'
nc where:help
```

Arguments accept names (resolved automatically) or IDs (faster). Set `NOCODB_VERBOSE=1` to see ID resolution.

⚠️ Name resolution can fail for some tables even when they exist (seen 2026-08-22: `record:list <base> modeles` → `Table 'modeles' not found`). On an unexpected "not found", retry with the raw ID (`m…` table / `c…` field) before digging further — every command accepts IDs wherever it accepts names.
