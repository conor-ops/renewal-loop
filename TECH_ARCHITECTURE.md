# Renewal Loop — Technical Architecture

> **Version:** 1.0  
> **Last Updated:** 2026-08-13  
> **Status:** Draft — Pre-Build

---

## 1. Architecture Overview

Renewal Loop's technical stack supports three primary user journeys:

1. **E-commerce** — Browse garments/panels, checkout, manage orders (one-time + subscription)
2. **Inventory & Fulfillment** — Track panel inventory, manage panel compatibility, fulfill subscription shipments
3. **AI Custom Design** — Gemini-powered modular pattern generation for custom panel requests

```
┌─────────────────────────────────────────────────────────┐
│                    USER INTERFACES                        │
│  ┌──────────┐  ┌──────────────┐  ┌─────────────────────┐ │
│  │ Web Store │  │ React Native │  │ Admin Dashboard     │ │
│  │ (Next.js) │  │ Companion App│  │ (Medusa Admin)      │ │
│  └─────┬─────┘  └──────┬───────┘  └──────────┬──────────┘ │
└────────┼───────────────┼─────────────────────┼───────────┘
         │               │                     │
         ▼               ▼                     ▼
┌─────────────────────────────────────────────────────────┐
│                   API GATEWAY LAYER                       │
│  ┌─────────────────┐  ┌────────────────────────────────┐ │
│  │ Medusa.js API   │  │ Panel Inventory API (FastAPI)  │ │
│  │ (Cloud Run)     │  │ (Cloud Run)                    │ │
│  └────────┬────────┘  └───────────┬────────────────────┘ │
└───────────┼───────────────────────┼──────────────────────┘
            │                       │
            ▼                       ▼
┌─────────────────────────────────────────────────────────┐
│                    DATA & SERVICES                        │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌─────────┐ │
│  │PostgreSQL│  │  Stripe   │  │ GCP      │  │ Gemini  │ │
│  │(Cloud SQL│  │  Billing  │  │ Storage  │  │   API   │ │
│  │          │  │  Checkout │  │ (Panel   │  │ (AI Pat-│ │
│  │          │  │           │  │  Assets) │  │  terns) │ │
│  └──────────┘  └───────────┘  └──────────┘  └─────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## 2. E-Commerce — Medusa.js

### 2.1 Platform Choice

**Medusa.js** is selected as the e-commerce platform:

| Criteria | Medusa.js | Shopify | Commerce.js |
|---|---|---|---|
| Open source | ✅ (MIT) | ❌ | ❌ |
| Self-hosted / no transaction fees | ✅ | ❌ (2%+ fees) | ❌ |
| Headless (API-first) | ✅ | ✅ | ✅ |
| Custom product types (panels) | ✅ Full control | Limited | Limited |
| Subscription support | Via plugin | Native | Limited |
| Customization for panel compatibility logic | ✅ Full code access | ❌ | Limited |

**Rationale:** Renewal Loop needs custom product logic (panel ↔ garment compatibility, subscription panel allocation, AI-generated custom SKUs). Medusa.js being open-source and self-hosted gives full control over the data model and no per-transaction fees.

### 2.2 Deployment

- **Hosting:** GCP Cloud Run (containerized)
- **Region:** us-central1 (primary), with multi-region in Year 2
- **Scaling:** Min 1 instance, max 10 instances, autoscale on CPU > 70%
- **Image:** Custom Docker image based on `medusajs/medusa` official image
- **CI/CD:** GitHub Actions → build Docker image → push to Artifact Registry → deploy to Cloud Run

```yaml
# cloud-run-medusa.yaml (simplified)
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: renewal-loop-store
spec:
  template:
    spec:
      containers:
        - image: us-central1-docker.pkg.dev/renewal-loop/medusa:latest
          ports:
            - containerPort: 9000
          env:
            - name: DATABASE_URL
              valueFrom: { secretKeyRef: { name: db-creds, key: url } }
            - name: REDIS_URL
              valueFrom: { secretKeyRef: { name: redis-creds, key: url } }
            - name: STRIPE_API_KEY
              valueFrom: { secretKeyRef: { name: stripe-creds, key: key } }
          resources:
            limits: { cpu: "2", memory: "2Gi" }
            requests: { cpu: "0.5", memory: "512Mi" }
```

### 2.3 Storefront

- **Framework:** Next.js (React, SSG/SSR)
- **Hosting:** Vercel (edge network, automatic deploys from GitHub)
- **Integration:** Medusa.js Storefront API (REST + GraphQL)
- **Pages:**
  - `/` — Landing page (hero, product overview, sustainability story)
  - `/products` — Product grid (garments + panels)
  - `/products/[slug]` — Product detail (size selector, color selector, panel compatibility checker)
  - `/subscription` — Refurbish Plan landing + signup
  - `/customize` — AI custom panel design tool (Gemini-powered)
  - `/account` — Customer account (order history, subscription management, panel tracker)
  - `/checkout` — Stripe Checkout integration

### 2.4 Custom Product Model

Medusa's product model is extended for panels:

```
Product (garment or panel)
├── type: "garment" | "panel" | "kit" | "subscription"
├── metadata
│   ├── compatible_garment_sizes: ["S", "M", "L", "XL"]  (for panels)
│   ├── panel_type: "elbow" | "knee" | "collar" | "cuff" | "zipper" | "pocket"
│   ├── snap_count: 6 | 8 | 4 | 3 | 10 | 4
│   ├── color: "black" | "navy" | "olive" | "charcoal" | "sand"
│   ├── is_custom: boolean
│   └── ai_design_id: string (if AI-generated)
├── variants (per size/color combo)
└── inventory_items (tracked in panel inventory API)
```

---

## 3. Database — PostgreSQL on Cloud SQL

### 3.1 Configuration

| Parameter | Value |
|---|---|
| Engine | PostgreSQL 15 |
| Tier | db-custom-2-7680 (2 vCPU, 7.5GB RAM) — Year 1; scale up Year 2 |
| Storage | 50GB SSD (auto-expand) |
| Region | us-central1 |
| High availability | Regional (primary + standby) |
| Backups | Automated daily, 7-day retention |
| Connection pooling | Cloud SQL Auth Proxy sidecar in Cloud Run |

### 3.2 Connection from Cloud Run

Medusa.js and the FastAPI inventory service both connect to Cloud SQL via the **Cloud SQL Auth Proxy**:

```
[Cloud Run] → [Cloud SQL Auth Proxy sidecar] → [Cloud SQL PostgreSQL]
```

- Auth Proxy runs as a sidecar container in the same Cloud Run service
- IAM-based authentication (no static passwords)
- Connection pooling within Medusa (TypeORM) and FastAPI (SQLAlchemy)

### 3.3 Key Database Schemas

#### E-Commerce (managed by Medusa.js)
- `product`, `product_variant`, `product_option`
- `customer`, `customer_group`
- `order`, `order_item`, `fulfillment`
- `payment`, `payment_session`
- `shipping_option`, `shipping_method`

#### Subscription (Medusa Subscription Plugin + custom tables)
- `subscription` — Stripe subscription ID, plan type, renewal date, panel allocation
- `subscription_panel_allocation` — Tracks which panels a subscriber has received each quarter

#### Panel Inventory (custom, managed by FastAPI)
- `panel_sku` — Panel type, size, color, snap_count, compatibility metadata
- `panel_stock` — Current stock level per SKU per warehouse
- `panel_reservation` — Reserved stock for pending orders/subscriptions
- `panel_compatibility` — Mapping of which panel SKUs fit which garment SKUs
- `ai_design` — Custom AI-generated panel designs, status, Gemini request ID

---

## 4. Payments — Stripe

### 4.1 Stripe Checkout (One-Time Purchases)

Used for all one-time transactions: garments, individual panels, Refurbish Kits.

- **Integration:** Stripe Checkout (hosted payment page) → redirect back to store
- **Webhooks:** `checkout.session.completed` → Medusa.js order fulfillment
- **Payment methods:** Card, Apple Pay, Google Pay, Link (Stripe-hosted)
- **Currency:** USD (Year 1), multi-currency Year 2

```
User clicks "Buy" →
  Medusa creates cart →
  Stripe Checkout Session created with cart line items →
  User redirected to Stripe Checkout →
  Payment completes →
  Stripe webhook → Medusa fulfills order →
  Panel inventory API decrements stock
```

### 4.2 Stripe Billing (Subscriptions)

Used for the Refurbish Plan ($15/month recurring).

- **Integration:** Stripe Billing + Stripe Customer Portal
- **Plan:** `renewal-loop-refurbish` — $15/month, billed monthly, cancel anytime
- **Trial:** First month free (Stripe trial period: 30 days)
- **Webhooks:**
  - `invoice.payment_succeeded` → Extend subscription, allocate quarterly panels
  - `customer.subscription.updated` → Sync status to Medusa
  - `customer.subscription.deleted` → Cancel in Medusa, end panel allocation
- **Customer Portal:** Subscribers self-manage billing, update card, cancel — no support tickets needed

### 4.3 Stripe Connect (B2B, Year 2)

For B2B custom orders and potential marketplace (third-party panel makers):
- Custom invoicing via Stripe Invoicing
- Net-30 payment terms for B2B accounts
- Separate Stripe Connect account for marketplace sellers (Year 2+)

---

## 5. Inventory — Panel Inventory Tracker (FastAPI + GCP Storage)

### 5.1 Purpose

Panels have complex inventory needs that standard e-commerce platforms handle poorly:
- **Compatibility matrix:** Which panels fit which garments (size cross-compatibility)
- **Subscription allocation:** Reserve panels for quarterly subscriber shipments
- **AI custom panels:** One-off production with unique SKUs
- **Multi-warehouse:** DC + manufacturer buffer stock

### 5.2 Architecture

```
┌──────────────┐
│  FastAPI     │  Python 3.11 + FastAPI + SQLAlchemy
│  Service     │  Deployed on Cloud Run
│  (Cloud Run) │  2 vCPU, 1GB RAM, autoscale 0-5
└──────┬───────┘
       │
       ├──→ PostgreSQL (Cloud SQL) — stock, reservations, compatibility
       ├──→ GCP Cloud Storage — panel design assets, pattern files, images
       └──→ Medusa.js API — sync stock levels, receive order events
```

### 5.3 API Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/panels` | GET | List all panel SKUs with stock levels |
| `/api/v1/panels/{sku}` | GET | Get single panel details + compatibility |
| `/api/v1/panels/compatible` | GET | Query: given garment SKU + size, return compatible panels |
| `/api/v1/panels/stock` | GET | Get current stock for all SKUs |
| `/api/v1/panels/stock/{sku}` | POST | Update stock level (admin/webhook from warehouse) |
| `/api/v1/panels/reserve` | POST | Reserve stock for an order or subscription shipment |
| `/api/v1/panels/release` | POST | Release a reservation (cancelled order) |
| `/api/v1/subscriptions/allocate` | POST | Allocate quarterly panels for a subscriber |
| `/api/v1/ai-designs` | POST | Submit AI custom design request → Gemini API |
| `/api/v1/ai-designs/{id}` | GET | Get AI design status + result |
| `/api/v1/ai-designs/{id}/approve` | POST | Approve design → create custom panel SKU → queue for production |

### 5.4 GCP Cloud Storage

| Bucket | Contents | Access |
|---|---|---|
| `rl-panel-assets` | Panel product images, design files, pattern PDFs | Public read (CDN via Cloud CDN) |
| `rl-ai-designs` | AI-generated panel designs (pre-approval) | Private (authenticated FastAPI only) |
| `rl-production-files` | Approved production-ready pattern files for manufacturer | Private (admin + manufacturer portal) |
| `rl-inventory-reports` | Daily stock reports, subscription allocation logs | Private (admin) |

---

## 6. AI — Gemini API for Modular Pattern Generation

### 6.1 Use Cases

1. **Custom Panel Design Requests:** Customers describe a desired panel design (color, pattern, artwork) via the web store or mobile app. Gemini generates a panel design proposal → user approves → design is converted to a production-ready pattern file → custom SKU created → queued for production.

2. **Modular Pattern Generation:** Internal tool for generating new panel shapes and sizes. Given a panel type and target dimensions, Gemini proposes optimized snap placement, fabric cut pattern, and stitching paths.

3. **B2B Custom Design:** Corporate customers upload a logo or brand guidelines → Gemini generates branded panel options → customer selects → production-ready files generated.

### 6.2 Integration Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│ User submits│────▶│  FastAPI     │────▶│  Gemini API     │
│ custom      │     │  Inventory   │     │  (Vertex AI /   │
│ design req  │     │  Service     │     │   Google AI)    │
└─────────────┘     └──────┬───────┘     └────────┬────────┘
                           │                      │
                           ▼                      ▼
                    ┌──────────────┐     ┌─────────────────┐
                    │  PostgreSQL  │     │  GCP Storage    │
                    │  (ai_design  │     │  (design assets │
                    │   table)     │     │   saved here)   │
                    └──────────────┘     └─────────────────┘
```

### 6.3 Gemini API Workflow

```python
# Pseudocode: Custom panel design generation
async def generate_panel_design(request: PanelDesignRequest):
    # 1. Build prompt with panel constraints
    prompt = f"""
    Generate a panel design for a {request.panel_type} panel.
    Constraints:
    - Dimensions: {PANEL_DIMENSIONS[request.panel_type]}
    - Snap points: {PANEL_SNAP_LAYOUT[request.panel_type]} positions
    - Color base: {request.base_color}
    - Customer request: {request.description}
    - Must be printable on 600D recycled polyester canvas
    - Output: design description + SVG pattern file
    """

    # 2. Call Gemini
    response = await gemini_client.generate_content(
        model="gemini-2.0-flash",
        prompt=prompt,
        generation_config={
            "temperature": 0.7,
            "max_output_tokens": 4096,
            "response_mime_type": "application/json"
        }
    )

    # 3. Parse response, save design assets
    design = parse_design_response(response)
    design_id = save_to_db(design)
    save_svg_to_gcs(design.svg, f"ai-designs/{design_id}/pattern.svg")
    save_preview_to_gcs(design.preview, f"ai-designs/{design_id}/preview.png")

    # 4. Return design for user approval
    return { "design_id": design_id, "preview_url": design.preview_url }
```

### 6.4 Gemini Models Used

| Use Case | Model | Why |
|---|---|---|
| Custom panel design (text → design concept + SVG) | `gemini-2.0-flash` | Fast, multimodal, good for creative + structured output |
| Modular pattern generation (engineering optimization) | `gemini-2.5-pro` | Higher reasoning for snap placement optimization, fabric efficiency |
| B2B brand adaptation (logo → panel design) | `gemini-2.0-flash` | Image understanding + generation |
| Customer support / design iteration chat | `gemini-2.0-flash` | Conversational, fast iteration |

### 6.5 Design Approval → Production Flow

```
1. User submits design request (text description + panel type + color)
2. Gemini generates design proposal (preview image + SVG pattern)
3. User reviews preview in web store or mobile app
4. User approves → design fee charged ($8-$20 depending on complexity)
5. SVG pattern converted to production-ready DXF/PLT file for cutting machine
6. Custom panel SKU created in Medusa + inventory tracker
7. Production file uploaded to GCS `rl-production-files` bucket
8. Manufacturer portal fetches production file → cuts and sews custom panel
9. Custom panel shipped to customer
```

---

## 7. Mobile — React Native Companion App

### 7.1 Purpose

The companion app serves three functions:

1. **Panel Ordering** — Browse and order replacement panels, track shipments
2. **Subscription Management** — View Refurbish Plan status, allocate quarterly panels, manage billing
3. **Panel Tracking** — Inventory of panels the user owns, garment ↔ panel compatibility checker, wear tracking (notifications when it's time to replace a panel based on usage)

### 7.2 Tech Stack

| Component | Technology |
|---|---|
| Framework | React Native 0.74 (Expo SDK 51) |
| Navigation | React Navigation (Stack + Tab) |
| State management | Zustand + React Query (TanStack Query) |
| API client | Axios → Medusa.js Storefront API + FastAPI Inventory API |
| Auth | Medusa.js customer auth (JWT) + biometric (FaceID/Fingerprint) |
| Payments | Stripe React Native SDK (Apple Pay, Google Pay, saved cards) |
| Push notifications | Firebase Cloud Messaging (Android) + APNs (iOS) |
| Analytics | Mixpanel (event tracking) |
| Deployment | EAS Build → App Store + Google Play |

### 7.3 Key Screens

| Screen | Purpose |
|---|---|
| **Home** | Dashboard: active subscription status, panel inventory, reorder suggestions |
| **Shop** | Browse garments + panels, filter by type/color/size, compatibility indicators |
| **Garment Scanner** | Camera-based QR scan on garment label → shows compatible panels + current panel inventory |
| **Custom Design** | Submit AI custom panel design request, review Gemini-generated previews, approve |
| **Subscription** | Refurbish Plan status, quarterly panel allocation, billing (Stripe Customer Portal embedded) |
| **My Panels** | Inventory of owned panels by garment, wear tracking, replacement reminders |
| **Orders** | Order history, shipment tracking |
| **Settings** | Account, address book, payment methods, notifications |

### 7.4 Panel Wear Tracking (Unique Feature)

The app includes a **wear tracking** system:

- User logs wash/wear cycles for each panel (1-tap after laundry)
- App tracks cycles against panel durability rating (100 wash cycles)
- Push notification at 80% wear: "Your elbow panel is nearing end of life — order a replacement?"
- One-tap reorder → suggests same color/size or offers color swap
- Wear data feeds into subscription allocation optimization (subscribers get panels they actually need, not random ones)

### 7.5 Garment QR System

Each garment ships with a **sewn-in QR label**:

- QR encodes garment SKU + size + manufacturing date
- Scanning with the app shows:
  - All compatible panel types and sizes
  - User's current panel inventory for this garment
  - Recommended replacement panels based on wear tracking
  - Direct purchase links
- QR also serves as proof of authenticity for warranty/Refurbish Plan claims

---

## 8. Infrastructure Summary

### 8.1 GCP Services Used

| Service | Purpose | Est. Monthly Cost (Year 1) |
|---|---|---|
| Cloud Run | Medusa.js + FastAPI hosting | $25-50 |
| Cloud SQL (PostgreSQL) | Primary database | $70-100 |
| Cloud Storage | Panel assets, AI designs, production files | $5-15 |
| Cloud CDN | Panel image delivery | $5-10 |
| Cloud Build / Artifact Registry | CI/CD | $5-10 |
| Secret Manager | API keys, DB credentials | $2 |
| VPC Network | Private connectivity | $5 |
| **Total GCP** | | **~$120-200/mo** |

### 8.2 Third-Party Services

| Service | Purpose | Est. Monthly Cost (Year 1) |
|---|---|---|
| Stripe | Payment processing | 2.9% + $0.30/transaction |
| Gemini API (Vertex AI) | AI pattern generation | $20-100 (usage-based) |
| Vercel | Next.js storefront hosting | $0-20 (Hobby → Pro) |
| EAS (Expo Application Services) | React Native builds | $0-29 (Free → Production) |
| Mixpanel | Analytics | $0 (free tier Year 1) |
| SendGrid | Transactional email | $0-20 |
| **Total Third-Party** | | **~$50-200/mo + Stripe %** |

### 8.3 Total Infrastructure Cost

- **Year 1 (low volume):** ~$170-$400/month
- **Year 2 (scaled):** ~$500-$800/month
- **Infra cost as % of revenue:** <1% at Year 1 revenue ($214K)

---

## 9. Security & Compliance

### 9.1 Data Security

- **PII:** Customer data (name, email, address) stored in PostgreSQL (Cloud SQL) with encryption at rest
- **PCI compliance:** No card data stored — all payment data handled by Stripe (PCI DSS Level 1)
- **Secrets:** All API keys, DB credentials in GCP Secret Manager; never in code or env files in repo
- **API auth:** Medusa.js admin API requires JWT auth; FastAPI uses API key + OAuth2
- **HTTPS:** All endpoints TLS-terminated at Cloud Run (Google-managed certs)

### 9.2 GDPR / Privacy

- Cookie consent banner on storefront
- Customer data export endpoint (GDPR Article 15)
- Customer data deletion endpoint (GDPR Article 17 — right to erasure)
- Data retention: Order data retained 7 years (tax compliance); marketing data deleted on unsubscribe

### 9.3 Backup & Disaster Recovery

- **Database:** Cloud SQL automated daily backups + 7-day point-in-time recovery
- **File storage:** GCS bucket versioning enabled + 30-day soft delete
- **RTO:** 4 hours (Cloud Run redeploy + Cloud SQL failover)
- **RPO:** 24 hours (last daily backup)
- **Multi-region:** Year 2 — replicate to us-east1

---

## 10. Development Roadmap (Technical)

| Phase | Months | Technical Deliverables |
|---|---|---|
| **Phase 1** | M1-3 | Landing page (index.html → Next.js); Medusa.js local dev setup; DB schema design; FastAPI inventory API prototype |
| **Phase 2** | M4-6 | Medusa.js deployed to Cloud Run; PostgreSQL on Cloud SQL; Stripe Checkout integration; panel inventory API v1; product catalog seeded |
| **Phase 3** | M7-9 | Next.js storefront deployed; pre-order flow (Stripe Checkout); email signup → SendGrid; admin dashboard |
| **Phase 4** | M10-12 | Stripe Billing subscription integration; React Native companion app MVP (panel ordering + subscription mgmt); Gemini AI custom design v1; panel wear tracking v1 |

---

## 11. Repository Structure

```
renewal-loop/
├── README.md
├── PRODUCT_SPEC.md          # Product specification (this project)
├── BUSINESS_PLAN.md         # Business plan
├── TECH_ARCHITECTURE.md     # This file
├── index.html               # Landing page
├── store/                   # Medusa.js backend (future)
│   ├── src/
│   ├── medusa-config.js
│   ├── Dockerfile
│   └── package.json
├── storefront/              # Next.js storefront (future)
│   ├── src/
│   ├── package.json
│   └── next.config.js
├── inventory-api/           # FastAPI panel inventory tracker (future)
│   ├── main.py
│   ├── models/
│   ├── routers/
│   ├── requirements.txt
│   └── Dockerfile
├── mobile/                  # React Native companion app (future)
│   ├── src/
│   ├── App.tsx
│   ├── app.json
│   └── package.json
├── ai/                      # Gemini integration (future)
│   ├── prompts/
│   ├── design_generator.py
│   └── pattern_optimizer.py
└── infra/                   # GCP infrastructure (future)
    ├── cloud-run-medusa.yaml
    ├── cloud-run-fastapi.yaml
    ├── cloud-sql.yaml
    └── cloudbuild.yaml
```

---

## 12. Key Technical Decisions & Rationale

| Decision | Choice | Why |
|---|---|---|
| E-commerce platform | Medusa.js (not Shopify) | Open-source, no transaction fees, full control over custom panel product model, self-hosted on GCP |
| Cloud provider | GCP (not AWS) | Simpler pricing, Cloud Run is ideal for containerized Medusa + FastAPI, Vertex AI for Gemini access |
| Database | PostgreSQL on Cloud SQL (not Supabase/Firebase) | Managed, HA, familiar SQL, Medusa.js native support |
| Payments | Stripe (not PayPal/Square) | Best subscription billing (Stripe Billing), hosted checkout reduces PCI scope, Customer Portal reduces support load |
| Inventory | Custom FastAPI (not Medusa built-in) | Panel compatibility matrix + subscription allocation logic is too custom for standard e-commerce inventory |
| AI | Gemini API (not OpenAI) | Multimodal (text → image + structured SVG), Vertex AI integration on GCP, competitive pricing |
| Mobile | React Native (not native/Flutter) | Single codebase iOS + Android, Expo for OTA updates, team can use TypeScript across stack |
| Storefront | Next.js on Vercel (not Medusa storefront) | Better DX, SSG for SEO, edge network, free hosting tier |

---

*See also: `PRODUCT_SPEC.md` for product details, `BUSINESS_PLAN.md` for revenue model and roadmap, `index.html` for landing page.*