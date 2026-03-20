# Schema Comparison: Before vs After Refactoring

## Table Count

| Aspect | Before | After | Change |
|--------|--------|-------|--------|
| Total Tables | 15 | 13 | -2 tables |
| Core Concepts | Tenant + Workspace | Tenant only | Merged |
| Membership Tables | 3 (tenant, workspace, channel) | 2 (tenant, channel) | -1 table |
| Role Assignment Tables | 3 (tenant, workspace, channel) | 2 (tenant, channel) | -1 table |

---

## Entity Changes

### ❌ Removed Tables
1. **workspaces** - Merged into tenants
2. **workspace_member_roles** - No longer needed

### ✅ Modified Tables

#### 1. tenants (Merged with workspaces)
**Added fields:**
- `slug` (from workspace.code)
- Data from workspace merged in

**Removed fields:**
- `tenant_code` → `slug`
- `policy_flags_json` → merged to `settings_json`

**Simplified:**
- `branding_json` now includes logo_url
- `settings_json` includes all policies and rules

#### 2. channels
**Removed:**
- `workspace_id` (redundant)
- `code` (not needed, have slug)
- `rules_json` (merged to settings_json)
- `metadata_json` (merged to settings_json)

**Changed:**
- `updated_by` field removed (track via audit log instead)

#### 3. channel_members
**Removed:**
- `tenant_id` (can get via channel)
- `workspace_id` (can get via channel)
- `hidden_at` (use left_at instead)
- `removed_at` (use left_at instead)

**Changed:**
- `membership_status` → `status`

#### 4. messages
**Removed:**
- `tenant_id` (can get via channel)
- `workspace_id` (can get via channel)
- `is_edited` (check `edited_at IS NOT NULL`)
- `is_deleted` (check `deleted_at IS NOT NULL`)
- `updated_at` (messages are immutable except edits)

#### 5. message_reads
**Removed:**
- `tenant_id` (can get via channel)
- `workspace_id` (can get via channel)
- `created_at` (not needed)

#### 6. message_pins
**Removed:**
- `tenant_id` (can get via channel)
- `workspace_id` (can get via channel)

#### 7. message_attachments
**Removed:**
- `attachment_type` (can infer from mime_type)
- `storage_provider` (internal detail)
- `metadata_json` (not needed)

**Added:**
- `storage_path` (internal path)
- `thumbnail_url` (for preview)

#### 8. invitations
**Removed:**
- `workspace_id` (tenant = workspace now)
- `intended_role_scope` → `role_scope`
- `intended_role_code` → `role_code`

**Added:**
- `channel_id` (can invite to specific channel)

#### 9. users
**Changed:**
- `account_status` → `status`

#### 10. user_profiles
**Changed:**
- `presence_status` → `status`
- `internal_phone` → `phone`
- `about_text` → `bio`

#### 11. user_settings
**Changed:**
- `notification_level` removed (in notifications_json)
- `notification_config_json` → `notifications_json`
- `preference_json` → `preferences_json`

#### 12. user_sessions
**Removed:**
- `is_current` (can check via last_seen_at/revoked_at)

**Changed:**
- `expired_at` → `expires_at`

#### 13. password_resets
**Changed:**
- `expired_at` → `expires_at`

---

## Field Count Reduction

| Table | Before | After | Saved |
|-------|--------|-------|-------|
| tenants + workspaces | 28 + 15 = 43 | 15 | -28 fields |
| channels | 18 | 14 | -4 fields |
| channel_members | 15 | 11 | -4 fields |
| messages | 17 | 12 | -5 fields |
| message_reads | 8 | 5 | -3 fields |
| message_pins | 7 | 5 | -2 fields |
| message_attachments | 9 | 8 | -1 field |
| invitations | 14 | 14 | 0 (restructured) |

**Total saved: ~47 redundant fields**

---

## Query Complexity Comparison

### Before: Get user's messages in a channel
```sql
SELECT m.*
FROM messages m
  JOIN channels c ON m.channel_id = c.id
  JOIN workspaces w ON c.workspace_id = w.id
  JOIN tenants t ON w.tenant_id = t.id
WHERE m.channel_id = ?
  AND m.tenant_id = t.id  -- Redundant check
  AND m.workspace_id = w.id  -- Redundant check
```

### After: Get user's messages in a channel
```sql
SELECT m.*
FROM messages m
WHERE m.channel_id = ?
```

**Improvement:**
- From 4 JOINs → 0 JOINs
- No redundant checks needed
- Much faster and cleaner

---

### Before: Get user's tenant info
```sql
SELECT t.*, w.*
FROM tenant_members tm
  JOIN tenants t ON tm.tenant_id = t.id
  JOIN workspace_members wm ON wm.user_id = tm.user_id AND wm.tenant_id = t.id
  JOIN workspaces w ON wm.workspace_id = w.id
WHERE tm.user_id = ?
  AND w.is_default = true
```

### After: Get user's tenant info
```sql
SELECT t.*
FROM tenant_members tm
  JOIN tenants t ON tm.tenant_id = t.id
WHERE tm.user_id = ?
```

**Improvement:**
- From 4 tables → 2 tables
- No is_default flag needed
- Direct relationship

---

## Index Reduction

### Removed Indexes (due to removed fields):
1. `workspace_members(tenant_id, workspace_id, user_id)` - Table removed
2. `channels(workspace_id)` - Field removed
3. `channels(tenant_id, workspace_id)` - workspace_id removed
4. `channel_members(tenant_id, workspace_id, channel_id)` - Fields removed
5. `messages(tenant_id, workspace_id, channel_id)` - Fields removed
6. `message_reads(tenant_id, workspace_id, channel_id)` - Fields removed
7. `message_pins(tenant_id, workspace_id, channel_id)` - Fields removed

**Estimated reduction: ~7-10 indexes**

---

## Storage Impact

### Field Type Savings per Row:

| Field Type | Size | Count | Total Saved |
|------------|------|-------|-------------|
| bigint (tenant_id redundant) | 8 bytes | ~7 tables × millions of rows | Significant |
| bigint (workspace_id) | 8 bytes | ~7 tables × millions of rows | Significant |
| boolean flags | 1 byte | ~3 fields × millions | Moderate |
| varchar status fields | ~10-30 bytes | Various | Moderate |

**For a system with 1M messages:**
- Removed `tenant_id` + `workspace_id` = 16 bytes × 1M = **16 MB**
- Removed `is_edited` + `is_deleted` = 2 bytes × 1M = **2 MB**
- Similar savings across other tables

**Total estimated savings for 1M rows across all tables: ~100-200 MB**

---

## Maintenance Impact

### Before:
- ❌ Need to maintain consistency between tenant_id in multiple tables
- ❌ Need to ensure workspace_id matches tenant_id
- ❌ Need to sync tenant_members and workspace_members
- ❌ Complex cascading deletes
- ❌ More indexes to maintain

### After:
- ✅ Single source of truth for tenant
- ✅ No workspace_id to sync
- ✅ Single membership model
- ✅ Simpler cascading rules
- ✅ Fewer indexes

---

## Developer Experience

### Before:
```javascript
// Complex query to get channel with tenant info
const channel = await db.channels.findOne({
  where: { id: channelId },
  include: [
    {
      model: db.workspaces,
      include: [{ model: db.tenants }]
    }
  ]
});
const tenantId = channel.workspace.tenant_id;
```

### After:
```javascript
// Simple query to get channel with tenant info
const channel = await db.channels.findOne({
  where: { id: channelId },
  include: [{ model: db.tenants }]
});
const tenantId = channel.tenant_id;
```

**Improvement:** One less level of nesting, clearer code

---

## Summary

### Quantitative Improvements:
- **-2 tables** (13.3% reduction)
- **-47 redundant fields** (~20% reduction)
- **-7 to 10 indexes**
- **~100-200 MB storage savings** (estimated for 1M rows)
- **50-70% query simplification** in most cases

### Qualitative Improvements:
- ✅ Clearer data model
- ✅ Less confusion about tenant vs workspace
- ✅ Easier to understand for new developers
- ✅ Lower risk of data inconsistency
- ✅ Simpler API design
- ✅ Better performance

### Migration Effort:
- Medium complexity
- Need to merge workspace data into tenants
- Need to update all FKs
- Need to update application code
- Estimated: 1-2 weeks for full migration
