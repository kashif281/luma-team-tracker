# Luma Visuals — Supabase Implementation Guide

## Overview
This guide walks you through adding a real database backend to your Team Tracker app.
After this, data persists between refreshes, multiple users see live updates, and security
is enforced server-side — not just in the browser.

---

## Step 1: Create Your Supabase Project

1. Go to https://supabase.com and create a free account
2. Click "New Project" — name it `luma-visuals`
3. Choose a strong database password — save it somewhere safe
4. Choose region closest to you (e.g. US East)
5. Wait ~2 minutes for it to spin up
6. Go to **Project Settings → API** and copy:
   - `Project URL` → paste into app as `SUPABASE_URL`
   - `anon public key` → paste into app as `SUPABASE_ANON_KEY`

---

## Step 2: Run This SQL in Supabase SQL Editor

Go to your Supabase dashboard → SQL Editor → paste and run each block.

### Tables

```sql
-- Editors table
CREATE TABLE editors (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  color TEXT DEFAULT '#4a7c59',
  rate_signature NUMERIC DEFAULT 150,
  rate_simple NUMERIC DEFAULT 75,
  rate_training NUMERIC DEFAULT 20,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Clients table (email is the permanent identifier for Zapier matching)
CREATE TABLE clients (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE,
  access_code TEXT UNIQUE NOT NULL,
  notif TEXT DEFAULT 'all',
  frameio_link TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Cards table
CREATE TABLE cards (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  title TEXT NOT NULL,
  client_id UUID REFERENCES clients(id) ON DELETE SET NULL,
  client_name TEXT,
  editor_id UUID REFERENCES editors(id) ON DELETE SET NULL,
  status TEXT DEFAULT 'not_started',
  type TEXT DEFAULT 'signature-timeless',
  shoot_date DATE,
  deadline DATE,
  color_profile TEXT,
  music_type TEXT DEFAULT 'Included',
  is_property BOOLEAN DEFAULT false,
  tip NUMERIC DEFAULT 0,
  notes TEXT,
  frameio_link TEXT,
  payroll BOOLEAN DEFAULT false,
  payroll_date DATE,
  revision_complete BOOLEAN DEFAULT false,
  archived BOOLEAN DEFAULT false,
  timeline JSONB DEFAULT '[]',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Messages table
CREATE TABLE messages (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  card_id UUID REFERENCES cards(id) ON DELETE CASCADE,
  sender_name TEXT NOT NULL,
  sender_type TEXT DEFAULT 'team', -- 'team' or 'client'
  message TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Training hours table
CREATE TABLE training_hours (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  editor_id UUID REFERENCES editors(id) ON DELETE CASCADE,
  period_key TEXT NOT NULL, -- e.g. 'p2026-03-14'
  hours NUMERIC DEFAULT 0,
  UNIQUE(editor_id, period_key)
);

-- Webhooks table
CREATE TABLE webhooks (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  url TEXT NOT NULL,
  event TEXT NOT NULL,
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Row Level Security (RLS) — CRITICAL

```sql
-- Enable RLS on all tables
ALTER TABLE editors ENABLE ROW LEVEL SECURITY;
ALTER TABLE clients ENABLE ROW LEVEL SECURITY;
ALTER TABLE cards ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE training_hours ENABLE ROW LEVEL SECURITY;
ALTER TABLE webhooks ENABLE ROW LEVEL SECURITY;

-- Editors: only admin can read all editors
-- (In practice you'll use Supabase Auth roles — see Step 3)
CREATE POLICY "Admin reads all editors"
  ON editors FOR SELECT
  USING (auth.jwt() ->> 'role' = 'admin');

-- Cards: editors can only see their own cards
CREATE POLICY "Editors see own cards"
  ON cards FOR SELECT
  USING (
    auth.jwt() ->> 'role' = 'admin'
    OR editor_id::TEXT = auth.jwt() ->> 'editor_id'
  );

CREATE POLICY "Editors cannot modify payroll"
  ON cards FOR UPDATE
  USING (auth.jwt() ->> 'role' = 'admin')
  WITH CHECK (
    auth.jwt() ->> 'role' = 'admin'
    OR (
      payroll = OLD.payroll
      AND payroll_date = OLD.payroll_date
      AND tip = OLD.tip
    )
  );

-- Clients: only admin can manage
CREATE POLICY "Admin manages clients"
  ON clients FOR ALL
  USING (auth.jwt() ->> 'role' = 'admin');

-- Client portal: clients read their own data by access code
CREATE POLICY "Client reads own data via code"
  ON cards FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM clients c
      WHERE c.id = cards.client_id
      AND c.access_code = current_setting('app.client_code', true)
    )
  );

-- Messages: clients can insert messages for their own cards
CREATE POLICY "Clients insert messages"
  ON messages FOR INSERT
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM cards ca
      JOIN clients cl ON cl.id = ca.client_id
      WHERE ca.id = messages.card_id
      AND cl.access_code = current_setting('app.client_code', true)
    )
  );

-- Webhooks: admin only
CREATE POLICY "Admin manages webhooks"
  ON webhooks FOR ALL
  USING (auth.jwt() ->> 'role' = 'admin');
```

### Trigger: auto-update `updated_at`

```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER cards_updated_at
  BEFORE UPDATE ON cards
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## Step 3: Supabase Auth Setup

Instead of storing passwords in your JavaScript, use Supabase Auth.

### Create users in Supabase Auth dashboard

1. Go to **Authentication → Users → Invite User**
2. Create one user per editor using their `@lumavisuals.co` email
3. Create one admin user (e.g. `admin@lumavisuals.co`)

### Add custom claims (role + editor_id)

In **SQL Editor** run:

```sql
-- Example: set Guillermo as editor
UPDATE auth.users
SET raw_user_meta_data = jsonb_build_object(
  'role', 'editor',
  'editor_id', 'e1',  -- match the id in your editors table
  'name', 'Guillermo'
)
WHERE email = 'guillermo@lumavisuals.co';

-- Set admin
UPDATE auth.users
SET raw_user_meta_data = jsonb_build_object('role', 'admin')
WHERE email = 'admin@lumavisuals.co';
```

---

## Step 4: Add Supabase to Your App

At the top of your `<script>` tag in `index.html`, add:

```javascript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const SUPABASE_URL = 'https://YOUR_PROJECT.supabase.co';
const SUPABASE_ANON_KEY = 'your-anon-key-here';
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

### Replace login with Supabase Auth

```javascript
// Replace checkPass() with:
async function checkPass() {
  const email = document.getElementById('li-email').value;
  const pass = document.getElementById('li-pass').value;
  const { data, error } = await supabase.auth.signInWithPassword({ email, password: pass });
  if (error) { document.getElementById('passErr').style.display = 'block'; return; }
  const role = data.user.user_metadata.role;
  const editorId = data.user.user_metadata.editor_id;
  enterApp(role, editorId);
}

// Logout:
async function doLogout() {
  await supabase.auth.signOut();
  // ... rest of logout
}
```

### Real-time board updates

```javascript
// Subscribe to card changes — all users see live updates
supabase
  .channel('cards')
  .on('postgres_changes', { event: '*', schema: 'public', table: 'cards' }, () => {
    loadCards(); // reload cards from DB
    renderBoard();
  })
  .subscribe();
```

### Loading cards from DB

```javascript
async function loadCards() {
  const { data, error } = await supabase.from('cards').select('*').eq('archived', false);
  if (!error) S.cards = data;
}
```

---

## Step 5: Zapier Integration (email-matching)

### Set up the Zap

1. **Trigger**: Webhook (catch hook) — Zapier gives you a URL → paste in your website's order form
2. **Action 1**: Supabase → Find Row in `clients` table, match by `email` field
3. **Action 2 (if found)**: Supabase → Insert Row in `cards` table, set `client_id` from step 2
4. **Action 3 (if not found)**: Supabase → Insert Row in `clients` table, generate access code, then insert card

### Webhook payload your website should send

```json
{
  "client_email": "buyer@example.com",
  "client_name": "John Smith",
  "project_type": "signature-timeless",
  "shoot_date": "2026-04-15",
  "notes": "Airbnb property, 3 bedrooms"
}
```

---

## Step 6: Deploy (GitHub Pages → Vercel)

For Supabase to work properly, switch from GitHub Pages to Vercel (free):

1. Push your `index.html` to GitHub
2. Go to https://vercel.com → Import your GitHub repo
3. Add environment variables:
   - `SUPABASE_URL`
   - `SUPABASE_ANON_KEY`
4. Deploy — you get a URL like `luma-visuals.vercel.app`
5. In Supabase → **Authentication → URL Configuration** → add your Vercel URL

---

## Security Checklist

- [ ] RLS enabled on all tables (done above)
- [ ] Admin password NOT stored in frontend JavaScript
- [ ] Supabase `anon key` is safe to expose — it's public by design, RLS controls access
- [ ] Service role key is NEVER put in frontend code — only use in server functions
- [ ] Client portal uses access code via database lookup, not client-side array
- [ ] Editor passwords managed via Supabase Auth, not hardcoded strings
- [ ] All webhooks fire over HTTPS only

---

## Timeline Estimate

| Task | Time |
|------|------|
| Create Supabase project + tables | 30 min |
| Set up Auth users | 20 min |
| Wire Supabase into the app JS | 2–4 hours (developer) |
| Set up Zapier Zap | 1 hour |
| Testing + QA | 1–2 hours |
| **Total** | **~1 day with a developer** |

---

## Next Steps After Supabase

1. **Email notifications** — use Supabase Edge Functions + Resend (free tier) to send real emails when card status changes
2. **Client magic links** — Supabase can send one-time login links to clients (no code needed) — secure and client-friendly
3. **Audit log** — add a `card_history` table that auto-records every change with who made it and when
