---
name: database-migrations
description: |
  Database migration patterns for Supabase, Prisma, and Drizzle ORM.
  Best practices for schema changes, rollbacks, and production deployments.
  Use when managing database schema evolution in production applications.
---

# Database Migration Patterns

Best practices for database migrations with modern ORMs.

## Supabase Migrations

### Setup

```bash
# Install CLI
npm install -g supabase

# Initialize
supabase init

# Link to project
supabase link --project-ref your-project-ref
```

### Creating Migrations

```bash
# Create new migration
supabase migration new create_users_table
```

```sql
-- supabase/migrations/20240115_create_users_table.sql

-- Up Migration
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Create policy
CREATE POLICY "Users can view own profile"
ON users FOR SELECT
USING (auth.uid() = id);

CREATE POLICY "Users can update own profile"
ON users FOR UPDATE
USING (auth.uid() = id);

-- Create updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();
```

### Running Migrations

```bash
# Apply locally
supabase db reset  # Reset and apply all migrations

# Push to remote
supabase db push

# Pull remote changes
supabase db pull
```

### Adding Columns Safely

```sql
-- supabase/migrations/20240116_add_user_role.sql

-- Add column with default (non-blocking)
ALTER TABLE users 
ADD COLUMN role TEXT DEFAULT 'user' NOT NULL;

-- Add constraint separately
ALTER TABLE users
ADD CONSTRAINT users_role_check 
CHECK (role IN ('user', 'admin', 'moderator'));
```

---

## Prisma Migrations

### Setup

```bash
npm install prisma @prisma/client
npx prisma init
```

### Schema Definition

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  role      Role     @default(USER)
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id        String   @id @default(uuid())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([authorId])
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### Creating Migrations

```bash
# Create migration
npx prisma migrate dev --name add_posts_table

# Apply in production
npx prisma migrate deploy

# Reset database (dev only)
npx prisma migrate reset
```

### Safe Production Migrations

```prisma
// Adding a required column safely

// Step 1: Add as optional
model User {
  newField String?
}

// Step 2: Backfill data
// npx prisma migrate dev --name add_new_field_optional

// Step 3: Make required
model User {
  newField String
}

// npx prisma migrate dev --name make_new_field_required
```

### Custom Migration SQL

```bash
# Create empty migration
npx prisma migrate dev --create-only --name custom_migration
```

```sql
-- prisma/migrations/20240115_custom_migration/migration.sql

-- Add your custom SQL
CREATE INDEX CONCURRENTLY idx_posts_title 
ON "Post" USING gin (to_tsvector('english', title));
```

---

## Drizzle Migrations

### Setup

```bash
npm install drizzle-orm drizzle-kit
```

### Schema Definition

```typescript
// src/db/schema.ts
import { 
  pgTable, 
  uuid, 
  text, 
  timestamp, 
  boolean,
  pgEnum 
} from 'drizzle-orm/pg-core'

export const roleEnum = pgEnum('role', ['user', 'admin', 'moderator'])

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').unique().notNull(),
  name: text('name'),
  role: roleEnum('role').default('user').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
})

export const posts = pgTable('posts', {
  id: uuid('id').primaryKey().defaultRandom(),
  title: text('title').notNull(),
  content: text('content'),
  published: boolean('published').default(false).notNull(),
  authorId: uuid('author_id').references(() => users.id).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
})
```

### Configuration

```typescript
// drizzle.config.ts
import type { Config } from 'drizzle-kit'

export default {
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
} satisfies Config
```

### Running Migrations

```bash
# Generate migration
npx drizzle-kit generate

# Apply migration
npx drizzle-kit migrate

# Push schema directly (dev only)
npx drizzle-kit push

# View studio
npx drizzle-kit studio
```

---

## Production Best Practices

### 1. Non-Blocking Column Addition

```sql
-- ❌ Blocking: Adding NOT NULL without default
ALTER TABLE users ADD COLUMN status TEXT NOT NULL;

-- ✅ Non-blocking: Add with default
ALTER TABLE users ADD COLUMN status TEXT DEFAULT 'active' NOT NULL;

-- ✅ Three-step for existing tables:
-- Step 1: Add nullable column
ALTER TABLE users ADD COLUMN status TEXT;

-- Step 2: Backfill data
UPDATE users SET status = 'active' WHERE status IS NULL;

-- Step 3: Add constraint
ALTER TABLE users ALTER COLUMN status SET NOT NULL;
```

### 2. Safe Index Creation

```sql
-- ❌ Blocking
CREATE INDEX idx_posts_author ON posts (author_id);

-- ✅ Non-blocking (PostgreSQL)
CREATE INDEX CONCURRENTLY idx_posts_author ON posts (author_id);
```

### 3. Renaming Columns

```sql
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN full_name TEXT;

-- Step 2: Copy data
UPDATE users SET full_name = name;

-- Step 3: Update application to use both
-- Step 4: Remove old column
ALTER TABLE users DROP COLUMN name;
```

### 4. Dropping Columns

```sql
-- Step 1: Stop writing to column in application
-- Step 2: Deploy application changes
-- Step 3: Wait for all old versions to drain
-- Step 4: Drop column
ALTER TABLE users DROP COLUMN deprecated_field;
```

### 5. Enum Modifications

```sql
-- ✅ Adding values (safe)
ALTER TYPE role ADD VALUE 'guest';

-- ❌ Removing values (unsafe, requires recreate)
-- Must create new type and migrate
```

---

## Migration Checklist

### Before Migration

- [ ] Backup database
- [ ] Test migration on staging
- [ ] Review migration SQL
- [ ] Check for blocking operations
- [ ] Estimate downtime if any
- [ ] Prepare rollback plan

### During Migration

- [ ] Monitor database locks
- [ ] Watch for long-running queries
- [ ] Check application errors
- [ ] Verify data integrity

### After Migration

- [ ] Verify schema changes
- [ ] Test critical flows
- [ ] Monitor performance
- [ ] Update documentation

---

## Rollback Patterns

### Supabase

```bash
# Create rollback migration
supabase migration new rollback_feature_x
```

```sql
-- Manual rollback SQL
DROP TABLE IF EXISTS new_table;
ALTER TABLE users DROP COLUMN IF EXISTS new_column;
```

### Prisma

```bash
# Rollback last migration
npx prisma migrate resolve --rolled-back <migration-name>
```

### Drizzle

```bash
# Drizzle uses push model, rollback via new migration
npx drizzle-kit generate  # Generate rollback migration
```

## Best Practices Summary

1. **Always backup** before production migrations
2. **Test thoroughly** on staging first
3. **Use CONCURRENTLY** for index creation
4. **Add columns as nullable** then make required
5. **Never drop columns** without deprecation period
6. **Monitor locks** during migration
7. **Have rollback plan** ready
8. **Document changes** in migration files
