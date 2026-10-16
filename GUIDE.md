# Ledgerly: setup and Supabase guide

Status: **prototype, not production-ready.** Part A is built and was tested by me in an automated headless Chromium browser (add, edit, delete, validation, search, filters, CSV, offline, layout). I have NOT tested it on a real Android phone, iPhone/Safari, or with a real accountant's review. Part B is a step-by-step plan: the SQL and code are written carefully but **I have not run them against your Supabase project**, so test each step.

---------------------------------------------------------------------

## Part A. Run the prototype (no accounts, ₹0)

### A1. What is in the folder

```
ledgerly/
  index.html              <- the whole app (HTML + CSS + JavaScript)
  manifest.webmanifest    <- makes it installable
  sw.js                   <- makes the app shell open offline
  icons/
    icon.svg  icon-192.png  icon-512.png  maskable-512.png
  GUIDE.md                <- this file
```

Keep exactly this structure. The icons are images, so use the zip file rather than copy-pasting them.

### A2. Where to paste the code

If you downloaded `ledgerly.zip`: unzip it, that's it.

If you are copy-pasting by hand on a laptop:
1. Create a folder called `ledgerly`, and inside it a folder called `icons`.
2. Open Notepad (Windows) or TextEdit in plain-text mode (Mac). Paste the contents of `index.html`, then **Save As** `index.html` inside `ledgerly`. Choose "All files" so it does not become `index.html.txt`.
3. Do the same for `manifest.webmanifest` and `sw.js`.
4. Put the four icon files in `ledgerly/icons`.

### A3. Test it

**Quick test (30 seconds):** double-click `index.html`. It opens in your browser with sample data. Offline/install features will not work this way, but everything else will.

**Full test, including install and offline (laptop):**
1. Install Python from python.org if you do not have it (free).
2. Open a terminal in the `ledgerly` folder and run: `python -m http.server 8000`
3. Open Chrome at `http://localhost:8000`
4. Chrome should show an install icon in the address bar. Install it.
5. DevTools (F12) → Application → Service Workers shows `sw.js` running. Tick **Offline** and reload: the app should still open.

**Test on your Android phone:** installing on a phone needs HTTPS, so deploy first (Part B7), then open the link in Chrome → menu → **Install app**.

### A4. Checklist

- Dashboard shows sales, expenses, net result, transaction count in ₹ (Indian grouping, like ₹7,44,050.00).
- Tap or tab to a month in the chart: the line under it shows that month's numbers. "View chart data as a table" shows the same data.
- Add an entry with an empty amount, "abc", "0", and a future date: each should show a clear error.
- Edit and delete an entry. Reload the page: your changes stay.
- Search "Sharma", change dates, use "This financial year (Apr–Mar)".
- Reports → Export CSV, open in Excel. Reports → Print / Save as PDF.
- Reports → Download backup, then Restore from it.

### A5. What this prototype does NOT do (so you don't promise it to anyone)

- No login, no cloud, no sync. Data lives only in that one browser on that one device. Clearing browser data deletes it.
- No multi-user or company separation yet (that is Part B).
- No GST invoices, GST returns, tax calculation, receivables/payables tracking, inventory, or bank import.
- GSTIN is only format-checked. It is not verified with the GST system.
- Not an audited or compliant accounting system. Treat reports as working records.
- Uses emoji icons in the menu, which look slightly different per device.

---------------------------------------------------------------------

## Part B. Supabase (free tier): login, database, company-level access

Facts I checked on Supabase's own pricing pages today (verify again before you rely on them, plans change):
- Free plan: 2 projects, 500 MB database, and projects **pause after 1 week of inactivity**.
- **Automatic backups are not included on Free.** Daily backups start on the paid Pro plan.

That means you must do your own backups (B6) and keep the project active.

### B1. Create the project
1. Go to supabase.com, sign up, create a free organization and project.
2. Pick the closest region (choose a South Asia/Mumbai region if offered).
3. Set a strong database password and store it in a password manager. **You will never put it in the app, in GitHub, or in a chat.**

### B2. Which keys are safe
In Project Settings → API Keys:
- **Publishable key** (`sb_publishable_...`, older name: `anon`) and the **Project URL**: these are designed to be in frontend code. They are safe only because Row Level Security (B3) is on.
- **Secret key** (`sb_secret_...`, older name: `service_role`) and the **database password**: bypass all security. **Never** put them in index.html, config files you upload, GitHub, screenshots, or messages to anyone (including me). If one leaks, rotate it in the dashboard immediately.

### B3. Create the tables and permissions

Open **SQL Editor → New query**, paste this whole block, click **Run**.

How access works: every business is a `company`. A person can see or change data only if they have a row in `company_members` for that company. Roles: `owner` (everything), `editor` (add/edit/delete records), `viewer` (read only).

```sql
-- ===== Tables =====
create table public.companies (
  id uuid primary key default gen_random_uuid(),
  name text not null check (char_length(name) between 2 and 80),
  created_by uuid not null default auth.uid() references auth.users(id),
  created_at timestamptz not null default now()
);

create table public.company_members (
  company_id uuid not null references public.companies(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  role text not null check (role in ('owner','editor','viewer')),
  created_at timestamptz not null default now(),
  primary key (company_id, user_id)
);

create table public.parties (
  id uuid primary key default gen_random_uuid(),
  company_id uuid not null references public.companies(id) on delete cascade,
  kind text not null check (kind in ('customer','supplier')),
  name text not null check (char_length(name) between 1 and 80),
  phone text check (phone is null or phone ~ '^[6-9][0-9]{9}$'),
  gstin text check (gstin is null or gstin ~ '^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z][1-9A-Z]Z[0-9A-Z]$'),
  city text check (city is null or char_length(city) <= 60),
  note text check (note is null or char_length(note) <= 200),
  created_at timestamptz not null default now(),
  unique (id, company_id)
);

create table public.transactions (
  id uuid primary key default gen_random_uuid(),
  company_id uuid not null references public.companies(id) on delete cascade,
  type text not null check (type in ('sale','expense')),
  txn_date date not null check (txn_date >= date '2000-01-01'),
  amount_paise bigint not null check (amount_paise > 0 and amount_paise <= 999999999999),
  party_id uuid,
  party_name text check (party_name is null or char_length(party_name) <= 80),
  category text not null check (char_length(category) between 1 and 40),
  payment_mode text not null check (char_length(payment_mode) between 1 and 20),
  note text check (note is null or char_length(note) <= 200),
  created_by uuid default auth.uid(),
  created_at timestamptz not null default now(),
  -- a transaction may only point at a party of the SAME company
  foreign key (party_id, company_id) references public.parties (id, company_id)
    on delete set null (party_id)
);

create index on public.transactions (company_id, txn_date desc);
create index on public.parties (company_id, kind);
create index on public.company_members (user_id);

-- ===== Helper: is the logged-in user a member (with one of these roles)? =====
create or replace function public.is_member(cid uuid, roles text[] default array['owner','editor','viewer'])
returns boolean
language sql stable security definer
set search_path = ''
as $$
  select exists (
    select 1 from public.company_members m
    where m.company_id = cid and m.user_id = (select auth.uid()) and m.role = any(roles)
  );
$$;
revoke all on function public.is_member(uuid, text[]) from public, anon;
grant execute on function public.is_member(uuid, text[]) to authenticated;

-- ===== Create a company and make the caller its owner =====
create or replace function public.create_company(company_name text)
returns uuid
language plpgsql security definer
set search_path = ''
as $$
declare cid uuid;
begin
  if auth.uid() is null then raise exception 'not authenticated'; end if;
  insert into public.companies (name) values (company_name) returning id into cid;
  insert into public.company_members (company_id, user_id, role) values (cid, auth.uid(), 'owner');
  return cid;
end;
$$;
revoke all on function public.create_company(text) from public, anon;
grant execute on function public.create_company(text) to authenticated;

-- ===== Row Level Security: ON for every table =====
alter table public.companies enable row level security;
alter table public.company_members enable row level security;
alter table public.parties enable row level security;
alter table public.transactions enable row level security;

create policy companies_select on public.companies for select to authenticated
  using (public.is_member(id));
create policy companies_update on public.companies for update to authenticated
  using (public.is_member(id, array['owner'])) with check (public.is_member(id, array['owner']));

create policy members_select on public.company_members for select to authenticated
  using (user_id = (select auth.uid()) or public.is_member(company_id, array['owner']));
create policy members_insert on public.company_members for insert to authenticated
  with check (public.is_member(company_id, array['owner']));
create policy members_update on public.company_members for update to authenticated
  using (public.is_member(company_id, array['owner'])) with check (public.is_member(company_id, array['owner']));
create policy members_delete on public.company_members for delete to authenticated
  using (public.is_member(company_id, array['owner']));

create policy parties_select on public.parties for select to authenticated
  using (public.is_member(company_id));
create policy parties_insert on public.parties for insert to authenticated
  with check (public.is_member(company_id, array['owner','editor']));
create policy parties_update on public.parties for update to authenticated
  using (public.is_member(company_id, array['owner','editor']))
  with check (public.is_member(company_id, array['owner','editor']));
create policy parties_delete on public.parties for delete to authenticated
  using (public.is_member(company_id, array['owner','editor']));

create policy txns_select on public.transactions for select to authenticated
  using (public.is_member(company_id));
create policy txns_insert on public.transactions for insert to authenticated
  with check (public.is_member(company_id, array['owner','editor']));
create policy txns_update on public.transactions for update to authenticated
  using (public.is_member(company_id, array['owner','editor']))
  with check (public.is_member(company_id, array['owner','editor']));
create policy txns_delete on public.transactions for delete to authenticated
  using (public.is_member(company_id, array['owner','editor']));

-- ===== Table permissions: logged-in users only, nothing for anonymous visitors =====
revoke all on public.companies, public.company_members, public.parties, public.transactions from anon;
grant select, update on public.companies to authenticated;
grant select, insert, update, delete on public.company_members, public.parties, public.transactions to authenticated;
```

If Supabase shows an error, copy the exact message and bring it back; do not skip lines.

**Test that company separation works (do this before any real data):**
1. Create two test accounts (B4), A and B. As A, create a company and add a record.
2. In Authentication → Users, copy B's user id. In SQL Editor run:
```sql
begin;
set local role authenticated;
select set_config('request.jwt.claims', '{"sub":"<B_USER_ID>","role":"authenticated"}', true);
select * from public.transactions;   -- must return 0 rows
rollback;
```
3. Also log in as B in the app: B must see nothing from A's company.

**Adding a teammate (v1 has no invite screen, so the owner does this by SQL):** the teammate signs up first, then run:
```sql
insert into public.company_members (company_id, user_id, role)
select '<COMPANY_ID>', id, 'editor' from auth.users where email = 'teammate@example.com';
```

### B4. Login settings (Authentication)
- Providers: keep Email on. Keep **email confirmation on**.
- URL Configuration: set **Site URL** to your deployed address (B7) and add it under Redirect URLs. Add `http://localhost:8000` only while developing, and remove it later.
- Set a minimum password length of at least 8 (longer is better).
- Supabase's built-in email sender has a very low sending limit, enough for testing but not for real customers. Before real users, set up custom SMTP with a provider that has a free tier (check its limits).

### B5. Connect the app (what changes in the code)
The prototype's data functions (`loadDB`, `saveDB`, add/edit/delete) are replaced with calls to Supabase, and a login screen is added. Do it in this order:

1. Download the Supabase JS library once and save it next to index.html as `vendor/supabase.js`. Use a pinned version (check the current version number on npmjs.com/package/@supabase/supabase-js):
   `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@<VERSION>/dist/umd/supabase.js`
   Hosting it yourself means it also works from the offline cache. Add `vendor/supabase.js` to the `SHELL` list in `sw.js`.
2. Create `config.js` (public values only):
```js
window.LEDGERLY_CONFIG = {
  SUPABASE_URL: 'https://YOUR-PROJECT-REF.supabase.co',
  SUPABASE_PUBLISHABLE_KEY: 'sb_publishable_xxxxxxxxxxxx'   // NEVER the secret key
};
```
3. In `index.html`, add before the main script: `<script src="vendor/supabase.js"></script>` and `<script src="config.js"></script>`.
4. In the `Content-Security-Policy` meta tag, change `default-src 'self'` to also allow your project, e.g. add: `connect-src 'self' https://YOUR-PROJECT-REF.supabase.co`.
5. Data layer sketch (`sb` is the client created with the two public values):
```js
const sb = window.supabase.createClient(LEDGERLY_CONFIG.SUPABASE_URL, LEDGERLY_CONFIG.SUPABASE_PUBLISHABLE_KEY);

// auth
await sb.auth.signUp({ email, password });
await sb.auth.signInWithPassword({ email, password });
await sb.auth.signOut();
sb.auth.onAuthStateChange((event, session) => { /* show app or login screen */ });

// first login: create a company (the owner row is created by the database function)
const { data: companyId, error } = await sb.rpc('create_company', { company_name: 'My Business' });

// read this company's data
const { data: txns } = await sb.from('transactions').select('*')
  .eq('company_id', companyId).order('txn_date', { ascending: false });

// add / edit / delete (map paise<->amount_paise, date<->txn_date, partyId<->party_id)
await sb.from('transactions').insert({ company_id: companyId, type: 'sale', txn_date: '2026-10-17',
  amount_paise: 250050, category: 'Services', payment_mode: 'UPI' });
await sb.from('transactions').update({ amount_paise: 300000 }).eq('id', id);
await sb.from('transactions').delete().eq('id', id);
```
Every call returns `{ data, error }`; show `error.message` in the existing toast. The database also enforces the same limits as the form, so bad data is rejected even if someone bypasses the page.

Important behaviour change: with Supabase the **data needs internet**. The app shell still opens offline, but records will not load or save offline. The prototype's fully offline mode is lost unless you build a sync queue later (not planned here).

Because this is a bigger edit, a good next step is: finish B1 to B4, send me your Project URL and publishable key (these two are public) and I will produce the full connected version with a login screen. Never send me the secret key or database password.

### B6. Backups (free plan = you do them)
1. **Weekly:** Supabase Dashboard → Table Editor → open each table (`companies`, `company_members`, `parties`, `transactions`) → export as CSV. Save the files in a private folder or private cloud drive. This also counts as activity that helps stop the 1-week pause.
2. **Also weekly:** the app's own Reports → Export CSV (for Excel) of all dates.
3. **Test a restore once** with fake data: create a new empty test project, run the B3 SQL, and import a CSV into a table (Table Editor → Import data) to confirm your backups really work.
4. **Later / advanced:** a full `pg_dump` of the database using the connection string from the dashboard, run from your laptop. Keep the database password out of any shared file.
5. If a project is paused, open the dashboard and click **Resume project**. Supabase's docs say a paused project can be restored for 1 year.
6. When real customers depend on the app, consider the paid Pro plan for daily backups and no pausing.

### B7. Deploy for free (HTTPS is required for install)
**Option 1: GitHub Pages (simple for beginners)**
1. Create a free GitHub account and a **new public repository** named `ledgerly`. (Public is fine because there are no secrets in the files. Anyone can read them.)
2. Add file → Upload files → drag in everything inside the `ledgerly` folder (keep the `icons` folder) → Commit.
3. Settings → Pages → Source: "Deploy from a branch", branch `main`, folder `/ (root)` → Save.
4. After a minute your app is at `https://YOUR-USERNAME.github.io/ledgerly/`. Open it on your phone in Chrome → menu → **Install app**.

**Option 2: Cloudflare Pages** (free static hosting, connect the same GitHub repo; lets you add security headers via a `_headers` file later).

Avoid platforms whose free plan is non-commercial only (for example, check Vercel's Hobby terms) if the app will be used for a business.

**Every time you change the app:** change `CACHE = 'ledgerly-shell-v1'` in `sw.js` to `v2`, `v3`, and so on. Otherwise phones keep showing the old cached version.

After deploying, go back to Supabase → Authentication → URL Configuration and set the real URL.

### B8. Security checklist before real users
- RLS enabled on all four tables (the SQL does this) and the two-account test passed.
- No secret key, service_role key, or database password anywhere in the repository or the page source. Search your files for `sb_secret` and `service_role` before uploading.
- Email confirmation on, password minimum 8+, redirect URLs limited to your real site.
- All text from the database is inserted into the page with the existing `esc()` function (the prototype already does this). Keep doing it for any new code.
- Keep the vendored Supabase library pinned; update it deliberately.
- Do not store card numbers, bank passwords, Aadhaar, or PAN. Phone numbers and GSTINs of customers are personal or business data; India's DPDP Act may apply, so get qualified advice and add a privacy notice before you run this for other businesses.
- This is a bookkeeping aid. It is not a GST filing tool and is not audited. A chartered accountant should review how you present reports to others.
