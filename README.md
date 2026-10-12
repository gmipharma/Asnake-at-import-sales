# GMI Pharma Marketplace — Telegram Mini App Prototype
GMI pharma import sales 
## What's included
- `index.html`: responsive, interactive frontend prototype
- English/Amharic language toggle
- Search, category filters, sample product listings and supplier comparison
- Cart and supplier-specific demo orders
- Payment reference submission and demo admin review
- Supplier onboarding form and demo admin approval queue

## Important prototype limits
This is a frontend-only demo. All data lives in browser memory and resets on reload. Supplier names, prices and inventory are illustrative. It does not authenticate Telegram users, upload/store license files securely, contact a bank, verify payment, or create real orders. The admin controls are not secured. Do not use it for real transactions.

## Run locally
Open `index.html` in a modern browser. To test Telegram Mini App behavior, host it over HTTPS and configure the bot's menu button or Web App button to point to that HTTPS URL.

## Recommended production architecture
- Frontend: React + TypeScript (or adapt this prototype), Telegram Mini Apps SDK
- Backend: Node.js + TypeScript using NestJS/Express
- Database: PostgreSQL
- Private document/object storage for license documents and payment evidence
- HTTPS hosting, secrets manager, backups and monitoring

## Suggested PostgreSQL schema
Use UUID primary keys and timestamps in production. Never store secrets or bank credentials in frontend code.

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY,
  telegram_user_id BIGINT UNIQUE NOT NULL,
  display_name TEXT,
  phone TEXT,
  preferred_language TEXT NOT NULL DEFAULT 'en'
    CHECK (preferred_language IN ('en','am')),
  role TEXT NOT NULL DEFAULT 'buyer'
    CHECK (role IN ('buyer','supplier','admin','finance')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE suppliers (
  id UUID PRIMARY KEY,
  legal_name TEXT NOT NULL,
  contact_phone TEXT NOT NULL,
  business_type TEXT NOT NULL,
  license_status TEXT NOT NULL DEFAULT 'pending'
    CHECK (license_status IN ('pending','approved','rejected','suspended','expired')),
  reviewed_by UUID REFERENCES users(id),
  reviewed_at TIMESTAMPTZ,
  review_notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE supplier_documents (
  id UUID PRIMARY KEY,
  supplier_id UUID NOT NULL REFERENCES suppliers(id),
  document_type TEXT NOT NULL,
  private_storage_key TEXT NOT NULL,
  document_number TEXT,
  expires_on DATE,
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','approved','rejected','expired')),
  reviewed_by UUID REFERENCES users(id),
  reviewed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE products (
  id UUID PRIMARY KEY,
  name_en TEXT NOT NULL,
  name_am TEXT,
  category TEXT NOT NULL,
  generic_name TEXT,
  brand_name TEXT,
  strength TEXT,
  dosage_form TEXT,
  manufacturer TEXT,
  registration_reference TEXT,
  requires_prescription_or_restricted_handling BOOLEAN NOT NULL DEFAULT false,
  active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE supplier_listings (
  id UUID PRIMARY KEY,
  supplier_id UUID NOT NULL REFERENCES suppliers(id),
  product_id UUID NOT NULL REFERENCES products(id),
  sku TEXT,
  unit_label TEXT NOT NULL,
  unit_price NUMERIC(14,2) NOT NULL CHECK (unit_price >= 0),
  available_quantity INTEGER NOT NULL DEFAULT 0 CHECK (available_quantity >= 0),
  minimum_order_quantity INTEGER NOT NULL DEFAULT 1 CHECK (minimum_order_quantity > 0),
  batch_number TEXT,
  expiry_date DATE,
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','paused','out_of_stock')),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
  id UUID PRIMARY KEY,
  order_number TEXT UNIQUE NOT NULL,
  buyer_id UUID NOT NULL REFERENCES users(id),
  supplier_id UUID NOT NULL REFERENCES suppliers(id),
  status TEXT NOT NULL DEFAULT 'awaiting_payment'
    CHECK (status IN ('awaiting_supplier_confirmation','awaiting_payment',
      'payment_submitted','paid','processing','dispatched','delivered','cancelled','disputed')),
  currency CHAR(3) NOT NULL DEFAULT 'ETB',
  subtotal NUMERIC(14,2) NOT NULL CHECK (subtotal >= 0),
  delivery_fee NUMERIC(14,2) NOT NULL DEFAULT 0 CHECK (delivery_fee >= 0),
  total NUMERIC(14,2) NOT NULL CHECK (total >= 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
  id UUID PRIMARY KEY,
  order_id UUID NOT NULL REFERENCES orders(id),
  listing_id UUID NOT NULL REFERENCES supplier_listings(id),
  product_name_snapshot TEXT NOT NULL,
  unit_price_snapshot NUMERIC(14,2) NOT NULL,
  quantity INTEGER NOT NULL CHECK (quantity > 0),
  line_total NUMERIC(14,2) NOT NULL
);

CREATE TABLE payment_submissions (
  id UUID PRIMARY KEY,
  order_id UUID NOT NULL REFERENCES orders(id),
  submitted_by UUID NOT NULL REFERENCES users(id),
  transaction_reference TEXT NOT NULL,
  evidence_storage_key TEXT,
  status TEXT NOT NULL DEFAULT 'submitted'
    CHECK (status IN ('submitted','under_review','verified','rejected')),
  reviewed_by UUID REFERENCES users(id),
  reviewed_at TIMESTAMPTZ,
  rejection_reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE supplier_bank_accounts (
  id UUID PRIMARY KEY,
  supplier_id UUID NOT NULL REFERENCES suppliers(id),
  bank_name TEXT NOT NULL,
  account_holder TEXT NOT NULL,
  account_number_encrypted TEXT NOT NULL,
  active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_events (
  id UUID PRIMARY KEY,
  actor_user_id UUID REFERENCES users(id),
  entity_type TEXT NOT NULL,
  entity_id UUID NOT NULL,
  action TEXT NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Telegram bot integration
1. Create or manage `@gmi_pric_bot` using Telegram's official BotFather.
2. Host the Mini App on HTTPS.
3. Configure a menu button or inline keyboard Web App button with the deployed URL.
4. Use Telegram Mini Apps SDK on the frontend for theme/viewport and to close or expand the app as needed.
5. Send Telegram `initData` to your backend over HTTPS.
6. On the backend, validate `initData` using Telegram's documented HMAC validation before creating a session. Reject stale auth dates and never trust `initDataUnsafe` alone.
7. Use bot API calls only from the backend; keep the bot token in server-side environment secrets.
8. Notify buyers and suppliers about order status changes through backend-controlled bot messages.

## Supplier verification workflow
1. Applicant submits business identity, contact and document metadata.
2. Store document uploads in private storage; use short-lived signed URLs and access checks.
3. Admin checks applicable business, pharmaceutical distribution, product registration and other relevant licenses with issuing authorities.
4. Save reviewer, timestamp, expiry dates and decision in an audit trail.
5. Only approved, non-expired suppliers can publish listings. Suspend expired or disputed suppliers until reviewed.
6. Apply different requirements for medicines, diagnostics, devices and general medical consumables as required by Ethiopian law and regulator guidance.

## Bank transfer confirmation workflow
1. Buyer checks out; backend creates one order per supplier and freezes a price/quantity snapshot.
2. Supplier confirms stock and final order amount.
3. Show that supplier's approved bank details and a unique order/payment reference.
4. Buyer submits a transaction reference; optional evidence is stored privately.
5. Mark payment `submitted`, never `paid` just because an image was uploaded.
6. Authorized finance staff verify the credit using bank records or an approved reconciliation channel.
7. Mark verified, record who approved and when, and append an audit event.
8. Prevent duplicate references and separate payment submitter from approver where feasible.

## Deployment checklist
- Register a domain and enable HTTPS.
- Deploy frontend to a static host; deploy backend to a managed application host.
- Create managed PostgreSQL and run migrations.
- Configure environment variables: `BOT_TOKEN`, `DATABASE_URL`, `PUBLIC_APP_URL`, storage credentials, session secrets.
- Set restrictive CORS, rate limits, request validation and role-based authorization.
- Configure BotFather Mini App/menu button URL.
- Test Telegram `initData` validation, order splitting, concurrent stock changes and price snapshots.
- Test access control, file upload restrictions, malware scanning, backup restoration and audit logs.
- Pilot with a few approved suppliers and invited institutional buyers.
- Obtain legal/regulatory review before enabling real pharmaceutical transactions.
