# Go-Ads 360° — Complete Knowledge & Prompt Guide for Softgen AI

> **Goal:** This single document is the **master brain** for the Go-Ads 360° platform.  
> Softgen AI (and any human developer) should be able to read this and understand every major module, data model, workflow, rule, and AI integration needed to build and extend the app using **Supabase + React + Softgen AI**.

---

## 1. System Overview & Vision

### 1.1 What is Go-Ads 360°?

Go-Ads 360° is a **multi-tenant SaaS platform** for **Out-of-Home (OOH) media management**.  
It digitizes the entire lifecycle:

> **Lead → Client → Media Asset → Plan → Campaign → Operations (Mounting & Proof) → Finance (Quotation/Invoice/Expenses) → Reporting & Analytics → AI Assistant**

### 1.2 Who Uses It?

- **Media Owners** – Own bus shelters, billboards, unipoles, etc.  
- **Agencies** – Do not own media; they buy media from owners and run campaigns for brands.  
- **Platform Admin (Go-Ads)** – Manages onboarding, subscriptions, commissions.  
- **Clients / Brands** – Get a read‑only portal to see campaigns, proofs, and invoices.  
- **Internal Teams** – Sales, Operations, Finance, Management.

### 1.3 What Problems Does It Solve?

- Scattered Excel sheets and PDF invoices  
- No real-time view of vacant media  
- Manual quotation & GST calculations  
- No structured proof of performance (photos, PPTs)  
- Poor visibility for clients on delivery & payments  
- No centralized control for media owners and agencies

---

## 2. Technology Stack

### 2.1 Frontend

- **React 18 + TypeScript**
- **Vite** for fast dev/build
- **Tailwind CSS + shadcn/ui** for UI components
- **React Router** for routing
- **Zustand** for client-side state in complex flows (e.g., Plan Builder)
- **Leaflet** or Mapbox for maps
- **pptxgenjs**, **exceljs**, **pdfkit** for exports

### 2.2 Backend

- **Supabase** (Primary backend)
  - PostgreSQL (multi-tenant DB)
  - Supabase Auth
  - Supabase Storage
  - Edge Functions (Serverless APIs)
  - Row-Level Security (RLS)

- (Legacy reference: Firebase Firestore + Functions → now replaced by Supabase)

### 2.3 AI & Integrations

- **AI Providers**
  - Gemini 2.5 (Google)
  - GPT‑4o (OpenAI)
  - Local LLM via **Ollama** (Llama‑3, Phi‑3, Mistral) – for free/private use

- **Business Integrations (placeholders)**
  - Zoho **CRM** (Leads, Clients)
  - Zoho **Books** (Items/Clients/Invoices)
  - WhatsApp Cloud API (Leads + Campaign proof sharing)
  - Gmail API (Lead parsing from emails)
  - Razorpay (Subscriptions & invoice payments)

### 2.4 Deployment

- Frontend on **Vercel / Netlify**
- Backend on **Supabase**
- Optional local AI server (Ollama) on **Windows / Linux**

---

## 3. Multi-Tenant Data Architecture (Supabase)

Each company (media owner or agency) is a **tenant**.  
Tenant isolation is enforced via a `company_id` field and **RLS** on every table.

### 3.1 Core Tables

#### `companies`

Represents a tenant (media owner, agency, or Go-Ads admin org).

| Column | Type | Description |
|--------|------|-------------|
| id | uuid (PK) | Company/tenant id |
| name | text | Legal / brand name |
| type | enum(`media_owner`,`agency`,`platform_admin`) | Tenant type |
| gstin | text | Company GST number |
| logo_url | text | Branding logo |
| theme_color | text | Primary brand color |
| status | enum(`pending`,`active`,`suspended`) | Onboarding & access state |
| subscription_tier | text | free / pro / enterprise |
| created_at | timestamptz | |
| updated_at | timestamptz | |

#### `company_users`

Links authenticated users to companies and assigns roles.

| Column | Type | Description |
|--------|------|-------------|
| id | uuid (PK) | |
| company_id | uuid FK → companies.id | Tenant link |
| user_id | uuid | Supabase auth user id |
| role | enum(`admin`,`sales`,`ops`,`finance`,`viewer`) | Role within company |
| name | text | User name |
| email | text | |
| phone | text | |
| created_at | timestamptz | |

#### `subscriptions`

Manages SaaS subscription state per company.

| Column | Type |
|--------|------|
| id | uuid (PK) |
| company_id | uuid FK → companies |
| tier | text (`free`,`pro`,`enterprise`) |
| start_date | date |
| end_date | date |
| amount | numeric |
| status | enum(`active`,`expired`,`cancelled`) |
| razorpay_subscription_id | text (nullable) |
| created_at | timestamptz |

#### `transactions`

Tracks subscription, portal fee, commission, and other financial transactions.

| Column | Type |
|--------|------|
| id | uuid (PK) |
| company_id | uuid FK → companies |
| booking_id | uuid (nullable) |
| type | enum(`subscription`,`portal_fee`,`commission`,`other`) |
| amount | numeric |
| gst_amount | numeric |
| paid_status | enum(`pending`,`paid`,`failed`) |
| created_at | timestamptz |

### 3.2 Domain Tables (Core Business)

#### `media_assets`

OOH inventory for media owners.

| Column | Type | Notes |
|--------|------|-------|
| id | text PK | Asset ID (e.g., HYD-BSQ-0001) |
| company_id | uuid FK→companies | Owner’s tenant |
| city | text | e.g., Hyderabad |
| area | text | e.g., Begumpet |
| location | text | Description |
| latitude | numeric | |
| longitude | numeric | |
| media_type | text | bus_shelter, hoarding, unipole, etc. |
| direction | text | Facing direction |
| dimension | text | “40x10 ft” |
| total_sqft | numeric | |
| card_rate | numeric | Official rate |
| base_rate | numeric | Internal lowest rate |
| printing_charge | numeric | Default printing |
| mounting_charge | numeric | Default mounting |
| status | enum(`Available`,`Booked`,`Blocked`) | |
| is_public | boolean | Visible in marketplace |
| next_available_from | date (nullable) | |
| municipal_id | text | Authority id |
| municipal_authority | text | GHMC/TGIIC/etc |
| photos | jsonb | List of URLs |
| created_at | timestamptz | |
| updated_at | timestamptz | |

#### `clients`

End customers (agencies’ or owners’ clients).

| Column | Type |
|--------|------|
| id | text PK |
| company_id | uuid |
| name | text |
| gstin | text |
| pan | text |
| billing_address | text |
| shipping_address | text |
| email | text |
| phone | text |
| type | enum(`brand`,`agency`,`direct`) |
| created_at | timestamptz |

#### `client_contacts`

Contact persons per client.

| Column | Type |
|--------|------|
| id | uuid |
| client_id | text FK→clients |
| name | text |
| email | text |
| phone | text |
| designation | text |

#### `leads`

Sales leads from WhatsApp/email/web.

| Column | Type |
|--------|------|
| id | uuid |
| company_id | uuid |
| source | text (`whatsapp`,`email`,`webform`,`manual`) |
| raw_message | text |
| parsed_data | jsonb |
| status | enum(`new`,`qualified`,`converted`,`closed`) |
| linked_client_id | text FK→clients (nullable) |
| created_by | uuid FK→company_users |
| created_at | timestamptz |

#### `plans` and `plan_items`

Media plans (quotations) created by agencies/owners.

`plans`:

| Column | Type |
|--------|------|
| id | text PK (PLAN-YYYYMM-###) |
| company_id | uuid (agency/owner creating plan) |
| client_id | text FK→clients |
| name | text |
| start_date | date |
| end_date | date |
| status | enum(`draft`,`sent`,`approved`,`rejected`,`converted`) |
| gross_amount | numeric |
| discount_amount | numeric |
| taxable_amount | numeric |
| cgst_amount | numeric |
| sgst_amount | numeric |
| total_amount | numeric |
| notes | text |
| created_at | timestamptz |

`plan_items`:

| Column | Type |
|--------|------|
| id | uuid |
| plan_id | text FK→plans |
| asset_id | text FK→media_assets |
| owner_company_id | uuid |
| base_rate | numeric |
| card_rate | numeric |
| negotiated_rate | numeric |
| printing_charge | numeric |
| mounting_charge | numeric |
| quantity | integer (months or display periods) |
| prorata_factor | numeric |
| line_subtotal | numeric |
| line_discount | numeric |
| line_taxable | numeric |
| cgst_amount | numeric |
| sgst_amount | numeric |
| total_line_amount | numeric |

#### `campaigns`

Live or completed campaigns derived from approved plans.

| Column | Type |
|--------|------|
| id | text PK (CMP-YYYYMM-###) |
| company_id | uuid (agency running campaign) |
| client_id | text |
| plan_id | text FK→plans |
| status | enum(`planned`,`running`,`completed`,`cancelled`) |
| start_date | date |
| end_date | date |
| total_amount | numeric |
| proof_zip_url | text (nullable) |
| created_at | timestamptz |

#### `mounting_assignments`

Operational tasks per asset per campaign.

| Column | Type |
|--------|------|
| id | uuid |
| campaign_id | text FK→campaigns |
| asset_id | text |
| mounter_name | text |
| mounter_phone | text |
| status | enum(`assigned`,`installed`,`proof_uploaded`,`verified`) |
| photos | jsonb (photo URLs by type: newspaper, geotag, traffic1, traffic2) |
| scheduled_date | date |
| completed_at | timestamptz |

#### `invoices`, `expenses`, `power_bills`

Finance entities. They link to plans/campaigns and companies.  
(Use your earlier SUPABASE_SCHEMA.md as reference; here we just ensure they have `company_id`, amount breakdown, GST, status, and dates.)

---

## 4. Security & Rules

### 4.1 Row-Level Security (RLS)

**Principle:** Every table that stores tenant data must enforce `company_id` filter.

Example policy for `media_assets`:

```sql
alter table media_assets enable row level security;

create policy "tenant_isolation_media_assets"
on media_assets
for all
using (company_id::text = auth.jwt()->>'company_id');
```

Similar policies apply to `clients`, `leads`, `plans`, `campaigns`, `invoices`, `expenses`, etc.

**Platform Admin Override:** You can allow platform admins to bypass tenant filters by checking a claim like `role = 'platform_admin'` in JWT.

```sql
using (
  company_id::text = auth.jwt()->>'company_id'
  or auth.jwt()->>'global_role' = 'platform_admin'
);
```

### 4.2 Storage Rules (Supabase Storage)

Buckets:

- `media-assets` – photos for assets
- `campaign-proofs` – 4 photos per asset per campaign
- `documents` – PDFs, PPTs, Excel exports, work orders, invoices

General rule: object paths include `company_id` so RLS-like access can be enforced in code.

Example path:  
`media-assets/{company_id}/{asset_id}/{photo_type}.jpg`

Upload & access logic:

- Only authenticated users belonging to that `company_id` can upload or read their objects.  
- Signed URLs for sharing with clients via portal or public links (time-limited).

---

## 5. Module-by-Module Workflows & Rules

Each module section includes:

- Purpose
- Routes (frontend paths)
- Key screens/components
- Workflows
- Rules & calculations
- Softgen AI prompt (for generation)

### 5.1 Module: Authentication & Company Onboarding

**Purpose:**  
Handle user login, company registration (media owner or agency), and role assignment.

**Routes:**
- `/login`
- `/register`
- `/onboarding`
- `/admin/companies` (platform admin)

**Workflow: New Company Onboarding**

1. User hits `/register`, signs up via Supabase Auth (email/password).  
2. Redirect to `/onboarding`:
   - Select **Company Type**: Media Owner / Agency  
   - Enter company details: name, GSTIN, address, logo, brand color, contact person.  
3. Create record in `companies` with `status = 'pending'`.  
4. Create `company_users` record with `role = 'admin'` for that user.  
5. Platform admin reviews in `/admin/companies` and sets `status = 'active'`.  
6. Once active, the user can access `/dashboard` with modules enabled according to `type` and `subscription_tier`.

**Softgen Prompt: Auth & Onboarding**

```text
Create Auth + Onboarding module for Go-Ads 360° using Supabase.
Routes:
 - /login
 - /register
 - /onboarding
 - /admin/companies
Use tables: companies, company_users.
Flow:
 - New user registers with Supabase Auth
 - /onboarding collects company details (type, gstin, logo, theme_color)
 - Insert into companies with status='pending'
 - Create company_users row linking auth.user.id with role='admin'
 - /admin/companies page allows platform_admin to approve/suspend companies
Apply RLS so users only see their own company, except platform_admin which sees all.
```

---

### 5.2 Module: Lead & Client Management

**Purpose:**  
Manage leads from WhatsApp/email/web, convert them to clients, and store full KYC details.

**Routes:**
- `/leads`
- `/leads/new`
- `/leads/[id]`
- `/clients`
- `/clients/new`
- `/clients/[id]`

**Lead Workflow:**
1. Leads arrive via:
   - WhatsApp webhook → Edge Function → insert into `leads`
   - Gmail parser → Edge Function → insert into `leads`
   - Manual entry via `/leads/new`
2. AI parses raw message (area, budget, dates, brand, media type preferences) into `parsed_data` JSON.
3. Sales user opens `/leads` and updates status (`new` → `qualified` → `converted` or `closed`).
4. On “Convert to Client”, create `clients` + `client_contacts` entries and link `leads.linked_client_id`.

**Client Workflow:**
1. `/clients/new` includes tabs: Basic Info, Billing/KYC, Contacts.  
2. Optional GSTIN lookup (placeholder for Zoho/third-party API).  
3. On save:
   - Generate `client_id` with pattern (CLT-YYYY-###).
   - Insert into `clients` and `client_contacts` in transaction.

**Softgen Prompt: Leads & Clients**

```text
Build Leads + Clients module for Go-Ads 360°.

Tables: leads, clients, client_contacts.

Features:
 - /leads: table with filters (status, source, date)
 - /leads/new: form with source, raw_message, parsed_data view
 - Button "Convert to Client" -> creates client + contacts and links lead
 - /clients: data table with search, filters
 - /clients/new: multi-tab form (Basic, Billing/KYC, Contacts)
 - Client ID auto-generation like CLT-2025-001.

Use Supabase queries and ensure company_id is always attached and respected by RLS.
```

---

### 5.3 Module: Media Asset Management

**Purpose:**  
Manage inventory of all OOH assets for each **media owner** company.

**Routes:**
- `/media-assets`
- `/media-assets/new`
- `/media-assets/[id]`
- `/media-assets/map`

**Key Components:**
- **Asset List:** Table with filters (city, area, media_type, status, is_public).
- **Add/Edit Asset Form:** Detailed form with image upload.
- **Asset Detail:** Shows gallery, booking history, power bills, maintenance.
- **Map View:** Leaflet map showing markers with clickable popups.

**Workflow: Adding a New Asset**
1. User navigates to `/media-assets` → “New Asset”.  
2. Select City + Media Type → call Edge Function `generate_asset_id` to get new ID like `HYD-BSQ-0001`.  
3. Fill location, dimension, direction, rates (card_rate, base_rate, printing, mounting), municipal details.  
4. Upload 1–4 photos → stored in Supabase Storage: `media-assets/{company_id}/{asset_id}/{index}.jpg`.  
5. On Save:
   - Insert into `media_assets` with `status = 'Available'` and `is_public` default based on company preference.  
   - Generate search tokens if needed (for quick search).  

**Rules:**
- `status` must change to `Booked` automatically when a campaign is confirmed for that asset within overlapping dates.  
- If multiple campaigns are allowed, store booking periods in a separate `asset_bookings` table.

**Softgen Prompt: Media Assets**

```text
Create Media Asset Management module.

Routes:
 - /media-assets
 - /media-assets/new
 - /media-assets/[id]
 - /media-assets/map

Features:
 - List: table with filters, column toggle, export to CSV.
 - New/Edit: form with auto asset_id generation based on city + media_type.
 - Upload photos to Supabase Storage path media-assets/{company_id}/{asset_id}/.
 - Asset Detail: show gallery, basic info, booking history placeholder.

Use media_assets table with company_id and is_public fields, and enforce RLS.
```

---

### 5.4 Module: Plan Builder (Quotations)

**Purpose:**  
Allow salespeople to create detailed **media plans/quotations** by selecting assets, negotiating rates, and exporting proposals.

**Routes:**
- `/plans`
- `/plans/new`
- `/plans/[id]`
- `/plans/[id]/share/[shareId]` (public view)

**Key Components:**
- Plan list with status, client, dates, total amount.  
- Plan form: basic details (name, client, dates).  
- Asset selection list (filtered from `media_assets` with `status = Available`).  
- Selected assets table with editable rates, discounts, mounting/printing charges.  
- Summary card with totals and GST.  

**Core Calculations:**

For each line item:

```text
effective_rate = negotiated_rate OR card_rate
prorata_factor = (actual_days / billing_cycle_days)   (if prorata enabled)
display_value = effective_rate * prorata_factor
line_subtotal = display_value + printing_charge + mounting_charge
line_discount = (line_subtotal * discount_percent / 100)
line_taxable = line_subtotal - line_discount
cgst_amount = line_taxable * 0.09
sgst_amount = line_taxable * 0.09
total_line_amount = line_taxable + cgst_amount + sgst_amount
```

For the entire plan:

```text
gross_amount = sum(line_subtotal)
discount_amount = sum(line_discount)
taxable_amount = sum(line_taxable)
cgst_amount = sum(cgst_amount)
sgst_amount = sum(sgst_amount)
total_amount = taxable_amount + cgst_amount + sgst_amount
```

**AI Rate Recommender:**

- Edge Function `/rate-recommender`:
  - Input: asset info (city, media_type, size, brand category, period).
  - Query historical `plan_items` for similar assets.  
  - Pass context to Gemini/GPT/local model.  
  - Return recommended range and suggested “target rate.”

**Softgen Prompt: Plan Builder**

```text
Implement a Plan Builder module for Go-Ads 360°.

Routes:
 - /plans
 - /plans/new
 - /plans/[id]
 - /plans/[id]/share/[shareId]

Use tables: plans, plan_items, clients, media_assets.

Features:
 - New Plan: select client, start/end dates, plan name.
 - Asset Selector: show only Available assets for that date range.
 - Selected Assets Table: editable negotiated_rate, prorata, printing, mounting, discount.
 - Plan Summary Card: live totals (gross, discount, taxable, CGST, SGST, total).
 - AI Rate Button: calls Edge Function /rate-recommender to suggest rate based on historical data.
 - Save: inserts/updates plans and plan_items in one transaction.
 - Public Share View: /plans/[id]/share/[shareId] with read-only display for clients.

Follow the calculation formulas defined in the knowledge guide.
```

---

### 5.5 Module: Campaign Management & Automation

**Purpose:**  
Convert approved plans into campaigns, block assets, create mounting tasks, and manage execution.

**Routes:**
- `/campaigns`
- `/campaigns/new` (usually via “Convert Plan”)
- `/campaigns/[id]`
- `/campaigns/[id]/operations`

**Workflow: Plan → Campaign**

1. From `/plans/[id]`, when status = `approved`, user clicks “Convert to Campaign”.  
2. Edge Function `/create-campaign-from-plan`:
   - Create new `campaigns` row.  
   - For each `plan_item`, create `mounting_assignments` row with status = `assigned`.  
   - Update `media_assets.status = 'Booked'` for the asset during the campaign period.  
3. `/campaigns/[id]` shows overview: assets, statuses, proofs, work orders.  

**Rules:**
- Only `approved` plans can be converted.  
- If campaign is cancelled, assets’ `status` or booking windows must be updated.  

**Softgen Prompt: Campaigns & Automation**

```text
Create Campaign Management & Automation from Plans.

When user clicks "Convert to Campaign" on an approved plan:
 - Call Edge Function /create-campaign-from-plan(plan_id).
 - Create campaign record and mounting_assignments for each plan_item.
 - Update media_assets.status to 'Booked' for those dates.

Provide routes:
 - /campaigns
 - /campaigns/[id]
 - /campaigns/[id]/operations

In /campaigns/[id], show campaign summary, assets list, mounting status, export buttons.
```

---

### 5.6 Module: Operations & Proof Upload (Mobile)

**Purpose:**  
Allow field staff (mounters) to receive tasks and upload proof photos (Newspaper, Geo-tag, Traffic 1, Traffic 2).

**Routes:**
- `/operations`
- `/operations/[assignmentId]/upload` (mobile-friendly)

**Workflow:**
1. Operations dashboard `/operations` lists `mounting_assignments` for logged-in ops user.  
2. They open a task → see asset location, map, instructions.  
3. Press “Upload Proof” → `/operations/[id]/upload` opens:  
   - Photo upload slots: Newspaper, Geotag, Traffic 1, Traffic 2.  
   - Capture via mobile camera.  
4. On upload, files go to `campaign-proofs/{company_id}/{campaign_id}/{asset_id}/{type}.jpg`.  
5. `mounting_assignments.photos` is updated with those URLs and `status` = `proof_uploaded`.  
6. Back-office user validates and sets `status` = `verified`.  

**Softgen Prompt: Operations & Proofs**

```text
Build Operations module for mounting assignments and proof uploads.

Tables: mounting_assignments, media_assets, campaigns.

Routes:
 - /operations
 - /operations/[id]/upload

Features:
 - Ops List: Only assignments for the logged-in ops user (role='ops').
 - Upload Screen: 4 required photos (newspaper, geotag, traffic1, traffic2).
 - Save photos to Supabase Storage under campaign-proofs/{company_id}/{campaign_id}/{asset_id}/.
 - After upload, update mounting_assignments.status to 'proof_uploaded' and store URLs in jsonb field.
 - Admin can later change status to 'verified'.

Optimize /operations/[id]/upload for mobile (large buttons, full-width inputs).
```

---

### 5.7 Module: Finance (Quotations, Invoices, Expenses, Zoho Integration)

**Purpose:**  
Generate quotations from plans, create invoices, track payments & expenses, and sync with Zoho Books / CRM.

**Routes:**
- `/finance/quotations`
- `/finance/invoices`
- `/finance/expenses`

**Quotations**

- Each **approved plan** becomes a quotation entity.  
- ID pattern: `EST-YYYY-##`.  
- PDF export with all GST and line items.  
- Optional sync to Zoho Books as “Estimate” via API.

**Invoices**

- For each completed or started campaign, one or more invoices are created.  
- ID pattern: `INV-YYYY-####`.  
- Fields: taxable, CGST, SGST, total, payment status, due date.  
- Optional push to Zoho Books via their Invoice API (placeholder: Edge Function `/zoho-sync/invoice`).

**Expenses**

- Track printing, mounting, electricity, rents, etc.  
- Link to campaign and/or asset.  
- Category-wise reporting.

**Softgen Prompt: Finance & Zoho Placeholders**

```text
Create Finance module for Quotations, Invoices, and Expenses.

Tables: plans, campaigns, invoices, expenses, transactions.

Routes:
 - /finance/quotations
 - /finance/invoices
 - /finance/expenses

Features:
 - Convert approved plan into quotation record and allow PDF export.
 - Create invoices linked to campaigns, with GST breakdown and payment status.
 - Expense tracking: date, category, amount, gst, linked_campaign_id.
 - Add Zoho Books integration placeholders:
    - Edge Function /zoho-sync/estimate(plan_id) to send quotations.
    - Edge Function /zoho-sync/invoice(invoice_id) to send invoices.

Integrate with subscriptions/transactions to record payments via Razorpay.
```

---

### 5.8 Module: Reports & Analytics

**Purpose:**  
Provide insights: vacant media, revenue, occupancy, outstanding payments.

**Routes:**
- `/reports/vacant-media`
- `/reports/revenue`
- `/reports/occupancy`
- `/reports/aging`

**Examples:**
- Vacant Media: list all `media_assets` with `status = Available` in a date range.  
- Revenue: group invoices by client or campaign; chart monthly totals.  
- Occupancy: percentage of time each asset was booked vs available.  
- Aging: outstanding invoices by 0–30/31–60/61–90 days.  

**Softgen Prompt: Reports**

```text
Create Reports section for Go-Ads 360°.

Routes:
 - /reports/vacant-media
 - /reports/revenue
 - /reports/occupancy
 - /reports/aging

Sources:
 - media_assets + campaigns + plan_items for occupancy and vacant media.
 - invoices and expenses for revenue and aging.

Use cards + charts + tables. Allow export to Excel for each report.
```

---

### 5.9 Module: Marketplace & Client Portal

**Marketplace (`/marketplace`)**

- Shows assets from multiple media owners with `is_public = true`.  
- Agencies can search/filter and add to plans.  

**Client Portal (`/portal/dashboard`)**

- Client-specific login (or magic link).  
- Shows campaigns, proofs, invoices, downloadable reports.  

**Softgen Prompt: Marketplace & Client Portal**

```text
Create Marketplace and Client Portal.

Marketplace:
 - Route /marketplace
 - Show public media_assets (is_public=true).
 - Allow agency users to select these assets into their plans.

Client Portal:
 - Route /portal/dashboard (separate layout)
 - Show campaigns, proof galleries, invoices for that client only.
 - Links to download proof PPT, work order PDF, and invoices.

Make sure portal is read-only and does not expose other clients' data.
```

---

## 6. AI Assistant Architectures

### 6.1 Common Flow

1. User types question in `/admin/assistant`.  
2. Frontend calls `/api/ask-ai`.  
3. Edge Function:
   - Identifies intent (`vacant_media`, `pending_invoices`, `client_summary`, etc.).  
   - Builds and executes Supabase SQL query (respecting `company_id`).  
   - Formats result into JSON payload: `{ type: 'table' | 'cards' | 'text', data, summary }`.  
4. Frontend renders cards/tables/text.

### 6.2 Gemini / GPT-4o Cloud Assistant

- Edge Function `ask-ai-cloud`:
  - System prompt describes Go-Ads schema and asks model to return structured JSON with `action` and `filters`.  
  - After parsing, Supabase is queried.  
  - Model is used again to summarize final answer in natural language, but raw data also returned.

### 6.3 Local LLM via Ollama (Option A)

- Run Ollama locally on Raghu’s machine:  
  - Install from official site.  
  - Run model: `ollama run llama3` or `ollama run phi3`.  
- Edge Function `/ask-local-ai` (for local dev only):  
  - Calls `http://localhost:11434/api/generate`.  
  - Uses small prompts to ask model for SQL filters or summary text.

### 6.4 Supabase Edge–Only Assistant (Option C)

- For simple, deterministic queries, skip LLM and use hard-coded keyword matching + SQL.  
- Example: queries like “vacant assets in Hyderabad” can be handled with string search in the Edge Function itself.

**Softgen Prompt: AI Assistant**

```text
Create AI Assistant module for Go-Ads 360°.

Route:
 - /admin/assistant (chat UI)

Endpoints:
 - /api/ask-ai (main)
 - /api/ask-ai-cloud (uses Gemini/GPT for intent + summarization)
 - /api/ask-local-ai (optional, connects to local Ollama server)

Behavior:
 - Parse natural language into {action, filters} either via LLM or simple keyword logic.
 - Run Supabase queries for media_assets, clients, plans, campaigns, invoices, expenses with company_id filter.
 - Return {type, data, summary} where type = 'table' | 'cards' | 'text'.
 - Frontend shows tables for lists, metric cards for KPIs, and plain text for descriptions.

Include fallbacks:
 - if LLM fails, respond with “No data available” or basic keyword-based query.
```

---

## 7. Subscription & Commission Logic (SaaS Model)

### 7.1 Tiers (India-Friendly)

- **Starter (Free):** up to 10 assets, basic modules.  
- **Pro (₹5K/month):** full modules, AI assistant, branding.  
- **Enterprise (Custom):** white-label, dedicated support.

### 7.2 Commission on Bookings

When an agency books owner’s media via Go-Ads:

- Portal fee = (booking_total × 2%) recorded in `transactions` as `type = 'commission'`.  
- Owner receives net amount minus this commission (handled in accounting, not necessarily in app v1).

### 7.3 Softgen Prompt: Subscriptions & Billing

```text
Implement SaaS subscription & commission logic.

Tables: subscriptions, transactions, companies.

Features:
 - /billing page per company showing current tier, usage, and payment history.
 - Integrate Razorpay to create/renew subscriptions (placeholder Edge Function /create-subscription).
 - On plan/campaign booking, calculate commission (2% of total) and insert into transactions as type='commission'.

Make sure all billing data is tied to company_id and visible only to company admins and platform_admin.
```

---

## 8. Developer Prompt Library (Softgen AI)

Below are **ready-to-use prompts** you can paste into Softgen AI to create or modify modules.

### 8.1 Full System Setup

```text
Act as a senior SaaS architect.

Using this knowledge base, scaffold the full Go-Ads 360° multi-tenant app in Softgen AI with Supabase backend.

Include:
 - Auth + Company Onboarding (companies, company_users)
 - Lead & Client Management
 - Media Asset Management
 - Interactive Plan Builder with pricing & GST calculations
 - Campaign Management & Mounting Assignments
 - Operations Proof Upload (mobile-optimized)
 - Finance module (Quotations, Invoices, Expenses) with Zoho placeholders
 - Reports & Analytics (vacant media, revenue, occupancy)
 - AI Assistant (cloud + local options)
 - Marketplace (public assets) and Client Portal
 - Subscription & Commission Billing

Respect company_id for all data access via RLS.
Use Tailwind + shadcn/ui and keep UX clean and dashboard-style.
```

### 8.2 Module-Level Prompt Pattern

For any module, use this template:

```text
You are a full-stack generator working inside Softgen AI.

Task:
Implement the [MODULE NAME] for Go-Ads 360° as described in the knowledge guide.

Key Details:
 - Backend: Supabase (use tables [LIST] with company_id)
 - Frontend routes: [LIST ROUTES]
 - Main screens: [LIST SCREENS]
 - Main actions: [LIST ACTIONS & RULES]
 - Calculations: [EXTRACT FROM SECTION ABOVE]
 - Security: apply RLS by company_id and role-based UI controls.

Output:
 - Frontend components (React + TypeScript + Tailwind + shadcn/ui)
 - API calls using Supabase client
 - Edge Functions if needed (for heavy logic or AI calls)
 - Comments explaining each important function for future devs.
```

You can specialize this template for **Media Assets**, **Plan Builder**, **AI Assistant**, etc., by copying the relevant section above.

---

## 9. Testing & QA Checklist

- ✅ RLS: verify users from one company cannot see another company’s data.  
- ✅ Plan calculations: cross-check totals with manual Excel.  
- ✅ Export files: open PPT/Excel/PDF in real apps and verify formatting.  
- ✅ AI queries: confirm correct answers for “vacant media,” “pending invoices,” “client summary” queries.  
- ✅ Mobile ops: test `/operations/[id]/upload` on actual phones.  
- ✅ Billing: test trial → subscription → cancellation flows with Razorpay sandbox.  
- ✅ Zoho: validate placeholder flows with test account when ready.

---

## 10. Future Enhancements

- Approval flows (multi-level approval for high-value campaigns).  
- Advanced dashboards with charts and drill-downs.  
- Auto-generated proposal emails and WhatsApp replies.  
- On-prem private LLM server for enterprise clients.  
- Marketplace with bidding/auction features for high-demand media.

---

**End of Master Knowledge Guide**  
Use this as the single source of truth for all Go-Ads 360° implementation inside Softgen AI.
