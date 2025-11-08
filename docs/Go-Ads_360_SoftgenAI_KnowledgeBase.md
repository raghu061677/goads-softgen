# Go-Ads 360° — Softgen AI Implementation & Knowledge Base

**Platform Type:** Multi-Tenant SaaS for OOH Media Management  
**Backend:** Supabase (PostgreSQL + Edge Functions + Storage + Auth)  
**Frontend:** React 18 + TypeScript + Tailwind + shadcn/ui  
**Builder:** Softgen AI / AI Studio  
**AI Engine:** Gemini 2.5 / GPT-4o  
**Core Users:**  
• Media Owners (upload assets)  
• Agencies (book assets for clients)  
• Platform Admin (Go-Ads team)  
• Client Portal Users (view proof + invoices)

---

## 1️⃣ Executive Vision

Go-Ads 360° digitizes the entire OOH advertising lifecycle — Lead → Client → Plan → Campaign → Operations → Finance → Reports — and now adds multi-tenant SaaS capability so that other media owners and agencies can onboard, brand, and manage their own inventory and campaigns inside the Go-Ads ecosystem.

### New 2025 Additions
| Module | Goal |
|---------|------|
| Company Onboarding | Register & verify media owners/agencies (KYC, GST, logo) |
| Tenant Isolation | Each company’s data filtered by `company_id` using Supabase RLS |
| Subscription & Billing | Razorpay integration for INR plans (+ GST) |
| Commission Tracking | Portal fee / booking commission on each asset transaction |
| White-Label Branding | Company-specific logo + color theme + client portal branding |

---

## 2️⃣ Multi-Tenant Architecture Overview

### a. Core Concept
All data lives in shared Supabase DB, isolated per tenant via `company_id`.  
RLS ( Row-Level Security ) ensures users only see their company’s records.

### b. Entity Relationships
```
companies (orgs)
 ├─ company_users
 ├─ media_assets
 ├─ clients
 ├─ plans → plan_items
 ├─ campaigns → mounting_assignments
 ├─ invoices
 ├─ expenses
 ├─ subscriptions
 └─ transactions
```

### c. Tenant Types
| Type | Description | Key Capabilities |
|------|--------------|----------------|
| Media Owner | Owns OOH inventory | CRUD media_assets, set rates, share with agencies |
| Agency | Books media for brand clients | Build plans, book assets, upload proofs, pay portal fees |
| Admin (Go-Ads) | Manages platform | Approve companies, manage subscriptions, commissions |

---

## 3️⃣ Supabase Schema (Extended)

### companies
| Field | Type | Notes |
|--------|------|------|
| id | uuid PK | tenant id |
| name | text | company name |
| type | enum (media_owner,agency,admin) | |
| gstin | text | |
| logo_url | text | branding |
| theme_color | text | |
| status | enum (pending,active,suspended) | |
| subscription_tier | text | free / pro / enterprise |
| created_at | timestamptz | |

### company_users
| Field | Type | Notes |
| id | uuid PK | |
| company_id | uuid FK → companies.id | |
| role | enum (admin,sales,ops,finance,viewer) | |
| email | text | |
| name | text | |
| phone | text | |

### subscriptions
| company_id | tier | start_date | end_date | amount | status | razorpay_subscription_id |

### transactions
| id | company_id | booking_id | type (subscription,portal_fee,commission) | amount | gst_amount | paid_status | created_at |

### media_assets
Add → `company_id` (owner tenant) and `is_public` (boolean) to expose asset to marketplace.

### plans / campaigns
Add → `company_id` (agency tenant) and `owner_company_id` (media owner).

---

## 4️⃣ Company Onboarding Flow

**Path:** `/onboarding`  
1️⃣ User registers via Supabase Auth  
2️⃣ Chooses Company Type (Media Owner / Agency)  
3️⃣ Fills form: Company name, GSTIN, logo, contact details  
4️⃣ Creates Admin user record in `company_users`  
5️⃣ Platform Admin approves company (status → active)  
6️⃣ Company dashboard unlocks with modules based on type.

**Branding Effect:**  
Upon login, UI theme and logo auto-apply from `companies.theme_color / logo_url`.

---

## 5️⃣ Subscription & Billing Model (India-Optimized)

| Tier | Target | Monthly Fee (₹ INR) | Includes |
|-------|---------|------------------|-----------|
| Starter / Free | Small agencies & owners | 0 | Up to 10 assets, basic exports |
| Pro | Growing businesses | 5 000 | Full modules, AI assistant, branding |
| Enterprise | Large media groups | Custom (annual) | Dedicated workspace + white-label |
| Transaction Fee | per booking | 2 % of booking value (+GST) | via Razorpay API |

### Razorpay Integration (Supabase Edge Function)
- Create subscriptions via Razorpay REST API  
- Listen to webhooks → update `subscriptions.status`  
- Invoice generation auto-adds GST (18 %)  

---

## 6️⃣ Access & Permissions Matrix

| Role | Access Scope | Key Modules |
|-------|--------------|-------------|
| Company Admin | Full tenant control | users, assets, plans, billing |
| Sales User | Leads, Clients, Plans | CRUD plans, quotations |
| Operations User | Campaigns, Mounting | upload proofs |
| Finance User | Invoices, Expenses | read/write finance |
| Client Portal User | Read-only campaigns + invoices | proof view + downloads |

---

## 7️⃣ Softgen AI Prompt Library

### A. Generate Company Onboarding Module
```
Create a Supabase-connected “Company Onboarding” module.
Include pages:
 - /onboarding (company type select + KYC form)
 - /admin/companies (list + approve/reject)
Use tables: companies, company_users.
On submit → create company record, link current user, set status='pending'.
Add Supabase RLS to restrict records to user.company_id.
```

### B. Extend Media Assets Module
```
Update media_assets table → add company_id, is_public.
Filter list page by company_id.
If user.type='admin', show all.
If asset.is_public=true, allow agencies to view and book.
```

### C. Generate Marketplace View
```
Path: /marketplace
Show all media_assets where is_public=true.
Include filters (city, type, size).
When agency user clicks “Book”, create booking record linking agency.company_id and owner.company_id.
```

### D. Add Subscription & Payment Module
```
Path: /billing
Display current subscription from subscriptions table.
Allow upgrade/downgrade via Razorpay Checkout.
Edge Function /create-subscription handles payment & updates Supabase.
```

### E. Commission Automation
```
Create Edge Function /onBookingComplete:
Input: booking_id
→ calculate portal_fee = (total_amount * 0.02)
→ insert into transactions table with type='commission'
→ notify finance dashboard.
```

### F. Agency & Owner Dashboards
```
Agency dashboard: KPIs (Active Campaigns, Booked Media, Pending Invoices)
Owner dashboard: KPIs (Available Assets, Revenue, Bookings Received)
Each dashboard filters data by company_id.
```

---

## 8️⃣ RLS Policies Example (Supabase)
```sql
create policy "tenant_isolation" on media_assets
for all using (company_id = auth.jwt()->>'company_id');
```

---

## 9️⃣ Softgen AI Deployment Prompt (Full System)
```
Act as senior SaaS architect.
Build the full Go-Ads 360° multi-tenant platform in Softgen AI.
Use Supabase backend and Razorpay for subscriptions.
Include modules:
 - Company Onboarding & Branding
 - Media Assets CRUD (+ map view)
 - Clients & Leads
 - Plans & Campaigns (with AI rate recommender)
 - Operations Photo Upload
 - Finance (Quotation/Invoice/Expenses)
 - AI Assistant (chat / reports)
 - Marketplace (public asset listing)
 - Subscription & Commission Billing
Ensure RLS tenant isolation by company_id.
Apply shadcn/ui styling and responsive layouts.
```

---

## 🔟 Knowledge Base Articles (for Softgen Docs)
| Article Title | Purpose |
|----------------|----------|
| Architecture & Multi-Tenant Model | Explain Supabase + RLS setup |
| Company Onboarding Guide | Step-by-step for new tenants |
| Subscription Billing Setup | How to integrate Razorpay keys |
| Media Owner vs Agency Roles | Usage differences |
| Marketplace Visibility Rules | When assets become public |
| Commission & Transactions | How portal fees are computed |
| Client Portal Access | Configuring read-only dashboard |
| Data Security & RLS | Compliance overview |

---

## ✅ Outcome

By following this pack, your Softgen AI workspace can now:
- Onboard agencies and media owners with KYC + branding  
- Enforce secure multi-tenant isolation  
- Manage subscriptions and commissions in INR  
- Run the entire OOH workflow under one SaaS platform  
- Scale as a marketplace where owners list and agencies book  
