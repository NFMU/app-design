# Database Schema Refactoring - Simplified Design

## 🎯 Mục tiêu
Đơn giản hóa database schema bằng cách loại bỏ các concept duplicate, field redundant và cải thiện tính nhất quán.

## 📋 Các thay đổi chính

### 1. **Merge Tenant và Workspace**

**Trước đây:**
- `tenants` table: Chứa tenant-level info
- `workspaces` table: Chứa workspace-level info
- Mối quan hệ: 1 tenant → N workspaces

**Vấn đề:**
- Trong use case, mỗi tenant luôn có "default workspace"
- Duplicate fields: `uuid`, `status`, `metadata_json`, `created_at`, `updated_at`, `deleted_at`
- Tạo complexity không cần thiết

**Sau khi refactor:**
- Chỉ giữ `tenants` table
- Merge tất cả thông tin quan trọng vào 1 entity
- Đổi `tenant_code` → `slug` (dễ hiểu hơn cho URL)
- Merge `rules_json`, `policy_flags_json` → `settings_json`
- Merge `logo_url` → `branding_json`

```sql
tenants {
  id, uuid, plan_id, owner_user_id
  name, slug, domain, status
  timezone, locale
  branding_json  -- Contains logo, colors, theme
  settings_json  -- Contains policies, rules, retention
}
```

---

### 2. **Loại bỏ workspace_members**

**Trước đây:**
- `tenant_members`: Members của tenant
- `workspace_members`: Members của workspace (có cả `tenant_id` và `workspace_id`)

**Vấn đề:**
- Duplicate concept vì tenant = workspace
- Confusion về việc quản lý membership

**Sau khi refactor:**
- Chỉ giữ `tenant_members`
- Loại bỏ `workspace_member_roles`, chỉ giữ `tenant_member_roles` và `channel_member_roles`

---

### 3. **Loại bỏ redundant tenant_id và workspace_id**

**Trước đây:**
Nhiều bảng có cả `tenant_id` VÀ `workspace_id`:
- `channels`: tenant_id + workspace_id
- `channel_members`: tenant_id + workspace_id + channel_id
- `messages`: tenant_id + workspace_id + channel_id
- `message_reads`: tenant_id + workspace_id + channel_id
- `message_pins`: tenant_id + workspace_id + channel_id

**Vấn đề:**
- Data redundancy
- Nếu có channel_id thì có thể JOIN để lấy tenant_id
- Khó maintain consistency

**Sau khi refactor:**
- `channels`: Chỉ giữ `tenant_id`
- `channel_members`: Chỉ giữ `channel_id` (có thể lấy tenant_id qua channels)
- `messages`: Chỉ giữ `channel_id`
- `message_reads`: Chỉ giữ `channel_id`
- `message_pins`: Chỉ giữ `channel_id`

---

### 4. **Đơn giản hóa các field**

#### 4.1. Messages table
**Trước:**
```sql
is_edited: boolean
edited_at: datetime
is_deleted: boolean
deleted_at: datetime
```

**Sau:**
```sql
edited_at: datetime    -- NULL = not edited
deleted_at: datetime   -- NULL = not deleted
```
- Loại bỏ boolean flags redundant
- Chỉ cần check `IS NOT NULL`

#### 4.2. User sessions
**Trước:**
```sql
is_current: boolean
expired_at: datetime
```

**Sau:**
```sql
expires_at: datetime   -- Consistent naming
```
- Loại bỏ `is_current` (có thể check qua `last_seen_at` hoặc `revoked_at`)

#### 4.3. User settings
**Trước:**
```sql
notification_level: varchar
notification_config_json: json
preference_json: json
```

**Sau:**
```sql
notifications_json: json   -- Merge all notification settings
preferences_json: json     -- Consistent naming
```

#### 4.4. Channels
**Trước:**
```sql
code: varchar
metadata_json: json
rules_json: json
```

**Sau:**
```sql
settings_json: json  -- Merge rules và config
```
- Loại bỏ `code` (không cần thiết, đã có `slug`)

---

### 5. **Cải thiện naming convention**

| Old | New | Lý do |
|-----|-----|-------|
| `tenant_code` | `slug` | Rõ ràng hơn cho URL |
| `account_status` | `status` | Ngắn gọn hơn |
| `presence_status` | `status` | Consistent |
| `membership_status` | `status` | Consistent |
| `expired_at` | `expires_at` | Consistent với các datetime khác |
| `internal_phone` | `phone` | Đơn giản hơn |
| `about_text` | `bio` | Ngắn gọn hơn |

---

### 6. **Cải thiện message attachments**

**Thêm:**
- `storage_path`: Đường dẫn storage internal
- `thumbnail_url`: URL của thumbnail cho images/videos

**Loại bỏ:**
- `attachment_type`: Có thể infer từ `mime_type`
- `storage_provider`: Không cần expose ra DB level

---

### 7. **Cải thiện invitations**

**Trước:**
```sql
workspace_id: bigint
intended_role_scope: varchar
intended_role_code: varchar
```

**Sau:**
```sql
channel_id: bigint (nullable)  -- Can invite to tenant or specific channel
role_scope: varchar            -- Shorter name
role_code: varchar             -- Shorter name
```

---

## 📊 So sánh trước và sau

### Trước refactoring:
- **15 tables** với nhiều redundancy
- **Duplicate concepts**: tenant vs workspace
- **Redundant fields**: tenant_id ở nhiều nơi
- **Inconsistent naming**: expired_at vs expires_at, account_status vs status

### Sau refactoring:
- **13 tables** (loại bỏ workspaces, workspace_member_roles)
- **Single source of truth**: tenants
- **Minimal redundancy**: Chỉ giữ FK cần thiết
- **Consistent naming**: Tất cả datetime fields đều dùng `*_at`

---

## 🎨 Cải thiện về mặt thiết kế

### Data Integrity
- ✅ Ít redundancy → ít risk về data inconsistency
- ✅ Rõ ràng hơn về ownership và hierarchy
- ✅ Dễ maintain constraints

### Performance
- ✅ Ít JOIN khi query
- ✅ Ít index cần maintain
- ✅ Smaller storage footprint

### Developer Experience
- ✅ Dễ hiểu hơn
- ✅ Ít confusion về concept
- ✅ Clear naming convention
- ✅ Better documentation với notes

---

## 🔄 Migration Strategy

Nếu đã có data cũ, migration steps:
1. Migrate `workspaces` data vào `tenants` (merge fields)
2. Update `workspace_members` → `tenant_members`
3. Drop redundant `tenant_id` columns
4. Update foreign keys
5. Migrate JSON fields (merge branding, settings)

---

## 📝 Notes

- Tất cả các bảng đều có `created_at` để audit
- Soft delete sử dụng `deleted_at` (NULL = active)
- Status fields luôn là varchar để flexible
- JSON fields được document rõ ràng trong notes
- Timestamps luôn dùng datetime type
