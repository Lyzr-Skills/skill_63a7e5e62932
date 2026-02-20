---
name: supabase
description: Use Supabase as the system of record for persisting any structured data — leads, demo requests, tasks, logs, user data, or any records the agent needs to store, query, update, or delete.
---

# Supabase System of Record Skill

Use Supabase as the persistent backend whenever the agent needs to store, retrieve, update, or delete structured data. This covers any use case: leads, demo requests, customer records, task tracking, activity logs, inventory, or any other data that should be durably stored.

## Setup

1. Create a Supabase project at https://supabase.com/dashboard
2. Get your project URL and API keys from Settings → API
3. Set environment variables:
   ```bash
   export SUPABASE_URL="https://your-project.supabase.co"
   export SUPABASE_ANON_KEY="your-anon-key"
   export SUPABASE_SERVICE_ROLE_KEY="your-service-role-key"
   ```

## Creating Tables

When a new type of data needs to be stored, create a table via the Supabase SQL editor or REST API. Follow this pattern:

```sql
CREATE TABLE table_name (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  -- your columns here
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Auto-update updated_at on changes
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER table_name_updated_at BEFORE UPDATE ON table_name
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

## Common Table Examples

### Leads / Contacts
```sql
CREATE TABLE leads (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE,
  company TEXT,
  phone TEXT,
  source TEXT,
  status TEXT DEFAULT 'new' CHECK (status IN ('new', 'contacted', 'qualified', 'converted', 'lost')),
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

### Demo Requests
```sql
CREATE TABLE demo_requests (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  lead_id UUID REFERENCES leads(id) ON DELETE CASCADE,
  scheduled_at TIMESTAMPTZ,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'scheduled', 'completed', 'cancelled')),
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

## Usage

All operations use the Supabase REST API (PostgREST). Always use the service role key for agent operations.

### Insert a record

```bash
curl -s -X POST "$SUPABASE_URL/rest/v1/{table}" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"field": "value"}'
```

### List / query records

```bash
# All records, newest first
curl -s "$SUPABASE_URL/rest/v1/{table}?select=*&order=created_at.desc" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY"

# Filter with PostgREST operators
# eq, neq, gt, gte, lt, lte, like, ilike, in, is
curl -s "$SUPABASE_URL/rest/v1/{table}?status=eq.active&select=*" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY"

# Search text fields (case-insensitive)
curl -s "$SUPABASE_URL/rest/v1/{table}?name=ilike.*search_term*&select=*" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY"

# Pagination
curl -s "$SUPABASE_URL/rest/v1/{table}?select=*&order=created_at.desc&limit=20&offset=0" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY"
```

### Update a record

```bash
curl -s -X PATCH "$SUPABASE_URL/rest/v1/{table}?id=eq.{record_id}" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"field": "new_value"}'
```

### Delete a record

```bash
curl -s -X DELETE "$SUPABASE_URL/rest/v1/{table}?id=eq.{record_id}" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY"
```

### Join related tables

```bash
# Fetch demo_requests with their associated lead info
curl -s "$SUPABASE_URL/rest/v1/demo_requests?select=*,leads(name,email,company)" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY"
```

### Upsert (insert or update)

```bash
curl -s -X POST "$SUPABASE_URL/rest/v1/{table}" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: resolution=merge-duplicates,return=representation" \
  -d '{"email": "jane@example.com", "name": "Jane Doe", "status": "contacted"}'
```

### Count records

```bash
curl -s "$SUPABASE_URL/rest/v1/{table}?select=count" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Prefer: count=exact"
```

## When to Use This Skill

- Any time data needs to be **persisted** beyond the current conversation
- Storing leads, contacts, customers, or prospects
- Tracking demo requests, meetings, or scheduled events
- Logging activities, events, or audit trails
- Managing tasks, tickets, or to-do items
- Storing configuration, settings, or user preferences
- Any CRUD operation on structured data
- When the user says "save this", "record that", "keep track of", "store", "log", "remember for later"

## Notes

- Always use the **service role key** for agent operations (bypasses Row Level Security)
- PostgREST filter syntax: `eq.`, `neq.`, `gt.`, `gte.`, `lt.`, `lte.`, `like.`, `ilike.`, `in.(a,b,c)`, `is.null`
- Foreign key joins: `select=*,related_table(col1,col2)`
- Use `Prefer: return=representation` to get the created/updated row back
- Use `Prefer: count=exact` with HEAD or GET to get row counts
- If a table doesn't exist yet for the data being stored, create it first using the SQL editor pattern above
