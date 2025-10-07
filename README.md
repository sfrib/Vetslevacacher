# 💰 VetExotic Price Comparator
**Služba pro veterináře, která porovnává aktuální B2B ceny a dostupnost u dodavatelů Noviko × Samohýl a umí připravit košík k objednání.**

> Multi-tenant • Realtime srovnání • Smart split košíku • Exporty • Bezpečná práce s přístupy

---

## 🔎 Co to dělá
- Přihlásí se (bezpečně) do B2B portálů **Noviko** a **Samohýl** – ideálně přes **API/CSV/EDI**, fallback přes **Playwright**.
- Porovná **ceny po slevách**, **dostupnost**, **balení/jednotky**, **dopravu** a **prahy pro dopravu zdarma**.
- Navrhne **nejlevnější variantu** nebo **nejrychlejší dodání**.
- Volitelně: připraví **košík** u dodavatele (deep-link / automatizace) a vrátí odkaz k dokončení objednávky.

---

## ✨ Features
- **Vyhledávání dle EAN/SKU/názvu** (fuzzy + mapování).
- **Ceny po slevách** (individuální B2B slevy, promo, množstevní slevy).
- **Dostupnost a rychlost** („skladem dnes“, „do 48h“).
- **Smart Split Košíku** – rozdělí položky mezi dodavatele tak, aby minimalizoval **cenu + dopravu** (zohlední prahy dopravy zdarma).
- **Exporty**: CSV/PDF, historie cen, rozdíl oproti minule.
- **Multi-tenant**: každý veterinář má vlastní přístupy a cenové hladiny.
- **Bezpečnost**: šifrované přístupy, audit logy, RBAC.
- **API-first**: `/api/compare`, `/api/cart/create`, `/api/vendors/sync`.

---

## 🧱 Architektura (high-level)
```
Frontend (Next.js)
  └── UI košík, srovnání, exporty
Backend API (Express/TS)
  ├── /api/compare        → porovnání cen+dostupnosti
  ├── /api/cart/create    → příprava košíků u dodavatelů
  └── /api/vendors/sync   → importy přes API/CSV, cron
Workers (Playwright)
  └── login, vyhledání, přidávání do košíků (secure, rate-limit)
DB (Postgres/Supabase)
  ├── products, prices_cache, vendors, vendor_accounts
  └── carts, cart_items, audits
Redis
  └── fronty (sync, cart flow), cache
Secrets
  └── GitHub Secrets / Doppler / 1Password / Vault
```

---

## 📂 Struktura repozitáře
```
vetexotic-price-comparator/
│
├─ src/
│  ├─ api/
│  │  ├─ compare.controller.ts
│  │  ├─ cart.controller.ts
│  │  └─ vendors.controller.ts
│  ├─ services/
│  │  ├─ compare.service.ts         # logika srovnání + smart split
│  │  ├─ pricing.service.ts         # výpočty cen, slev, dopravy
│  │  ├─ noviko.vendor.ts           # API/CSV/Playwright adapter
│  │  └─ samohyl.vendor.ts          # API/CSV/Playwright adapter
│  ├─ workers/
│  │  ├─ playwright.runner.ts       # izolovaný runner (cart/login)
│  │  └─ tasks.queue.ts             # BullMQ fronty
│  ├─ db/
│  │  ├─ schema.sql                 # inicializační schéma
│  │  └─ queries.ts                 # typované dotazy
│  ├─ utils/
│  │  ├─ parse.ts                   # parsePrice, normalizace jednotek
│  │  ├─ map.ts                     # mapování EAN/SKU, fuzzy matching
│  │  ├─ auth.ts                    # RBAC, tenant context
│  │  └─ secrets.ts                 # načtení a dešifrování přístupů
│  ├─ public/
│  │  └─ index.html                 # jednoduché UI (textarea/table)
│  ├─ server.ts                     # Express bootstrap (routes)
│  └─ config.ts                     # konfigurace (env, thresholds)
│
├─ test/
│  ├─ compare.service.test.ts
│  └─ pricing.service.test.ts
│
├─ .env.example
├─ docker-compose.yml
├─ Dockerfile
├─ package.json
├─ tsconfig.json
├─ README.md
└─ LICENSE
```

---

## ⚙️ Tech stack
- **Next.js (frontend)**, **Express + TypeScript (backend)**, **Playwright (workers)**
- **PostgreSQL/Supabase**, **Redis (BullMQ)**, **Docker**
- **Jest** (testy), **ESLint/Prettier**, **GitHub Actions** (CI/CD)

---

## 🚀 Rychlý start (lokálně)
```bash
git clone https://github.com/your-username/vetexotic-price-comparator.git
cd vetexotic-price-comparator
cp .env.example .env  # vyplň přístupy (lokálně dummy)
npm i
npm run db:init       # spustí SQL schéma (viz scripts)
npm run dev           # backend + frontend (concurrently)
```

**.env.example**
```env
# Server
PORT=3000
NODE_ENV=development

# DB
DATABASE_URL=postgres://user:pass@localhost:5432/vetexotic
REDIS_URL=redis://localhost:6379

# Tenancy (příklad single-tenant lokálně)
TENANT_ID=vetexotic-local

# Playwright
PW_HEADLESS=true
PW_USER_DATA_DIR=.pw-sessions

# (Volitelně) přístupy – ukládat spíš do trezoru, ne do .env
NOVIKO_USER=
NOVIKO_PASS=
SAMOHYL_USER=
SAMOHYL_PASS=
```

**package.json (skripty – výřez)**
```json
{
  "scripts": {
    "dev": "concurrently \"ts-node-dev src/server.ts\" \"serve ./src/public -l 5173\"",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/server.js",
    "test": "jest",
    "db:init": "psql \"$DATABASE_URL\" -f src/db/schema.sql",
    "lint": "eslint ."
  }
}
```

---

## 🔌 API Spec (MVP)

### `POST /api/compare`
Porovná položky mezi dodavateli.

**Body**
```json
{
  "items": [
    { "key": "8591234567890", "qty": 2 },
    { "key": "NOV-ABC-001", "qty": 1 }
  ],
  "prefer": "price|speed", 
  "includeUnavailable": false
}
```

**Response (příklad)**
```json
{
  "summary": {
    "strategy": "smart-split",
    "total_price_vat": 1560.4,
    "shipping": { "noviko": 0, "samohyl": 99 }
  },
  "lines": [
    {
      "key": "8591234567890",
      "name": "Stříkačky 2 ml",
      "best_supplier": "noviko",
      "best_price_vat": 24.2,
      "best_availability": "skladem",
      "offers": [
        { "supplier": "noviko",  "price_vat": 24.2, "availability": "skladem" },
        { "supplier": "samohyl", "price_vat": 25.9, "availability": "do 48h" }
      ],
      "unit": "ks",
      "pack_size": "100 ks"
    }
  ],
  "recommendation": {
    "basket": {
      "noviko": { "subtotal_vat": 1200.0, "items": [{ "key": "859...", "qty": 2 }] },
      "samohyl": { "subtotal_vat": 360.4,  "items": [{ "key": "NOV...", "qty": 1 }] }
    }
  }
}
```

### `POST /api/cart/create`
Vytvoří/naplní košík u dodavatelů a vrátí odkazy.

**Body**
```json
{
  "basket": {
    "noviko": [{ "key": "8591234567890", "qty": 2 }],
    "samohyl": [{ "key": "NOV-ABC-001", "qty": 1 }]
  }
}
```

**Response (příklad)**
```json
{
  "carts": {
    "noviko":  { "status": "ok", "checkout_url": "https://b2b.noviko.cz/cart/..." },
    "samohyl": { "status": "ok", "checkout_url": "https://b2b.samohyl.cz/cart/..." }
  }
}
```

### `POST /api/vendors/sync`
Načtení čerstvých dat (API/CSV). Používá se i cronem.

**Body**
```json
{ "vendor": "noviko", "mode": "csv|api", "force": false }
```

---

## 🗃️ SQL schéma (základ)
```sql
create table tenants (id text primary key, name text not null);
create table vendors (
  id text primary key,
  name text not null,
  shipping_free_from numeric default 0,
  shipping_fee numeric default 0
);
create table vendor_accounts (
  id uuid primary key default gen_random_uuid(),
  tenant_id text references tenants(id) on delete cascade,
  vendor_id text references vendors(id),
  enc_username bytea,
  enc_password bytea,
  user_data_dir text,
  unique (tenant_id, vendor_id)
);
create table products (
  id uuid primary key default gen_random_uuid(),
  ean text,
  name text,
  unit text,
  pack_size text,
  sku_noviko text,
  sku_samohyl text
);
create table prices_cache (
  id uuid primary key default gen_random_uuid(),
  product_id uuid references products(id) on delete cascade,
  vendor_id text references vendors(id),
  price_no_vat numeric,
  vat_rate numeric,
  price_after_discounts numeric,
  availability text,
  fetched_at timestamptz default now(),
  valid_until timestamptz
);
create table carts (
  id uuid primary key default gen_random_uuid(),
  tenant_id text references tenants(id),
  vendor_id text references vendors(id),
  status text default 'draft',
  created_at timestamptz default now(),
  total_vat numeric
);
create table cart_items (
  id uuid primary key default gen_random_uuid(),
  cart_id uuid references carts(id) on delete cascade,
  product_id uuid references products(id),
  qty numeric,
  unit_price_vat numeric
);
create table audits (
  id uuid primary key default gen_random_uuid(),
  tenant_id text references tenants(id),
  actor text,
  action text,
  payload jsonb,
  created_at timestamptz default now()
);
```

---

## 🧮 Jak počítáme
- `cena_s_DPH = price_no_vat * (1 + vat_rate/100)`
- `cena_po_slevě = apply_discounts(cena_s_DPH, klient_sleva, promo, množstevní)`
- **Smart Split**: greedy algoritmus – minimalizuje cenu + dopravu, sleduje prahy dopravy zdarma.

---

## 🔐 Bezpečnost
- Nikdy necommitovat `.env` ani přístupy. Produkčně používat **Secrets**.
- Playwright session v `PW_USER_DATA_DIR` mimo git.
- Rate-limit, random delay, robotická etiketa; ideálně **svolení dodavatelů** k automatizaci.

---

## 🛣️ Roadmap
- **v0.1.0**: MVP (compare + jednoduché UI)
- **v0.2.0**: exporty CSV/PDF, historie cen, diff
- **v0.3.0**: cache + background refresh
- **v0.4.0**: smart split (greedy + heuristiky dopravy)
- **v0.5.0**: košík (deep-link / Playwright microflow)
- **v0.6.0**: multi-tenant, RBAC, audit
- **v1.0.0**: monitoring, SLA, dohody s dodavateli

---

## 🤝 Příspěvky
PR i issues vítány. Dodržujte prosím lint/testy:  
`npm run lint && npm test`

---

## 🧾 Licence
MIT © VetExotic Group s.r.o.
