# Claude Code Prompt — Comentra Marketplace Connector (Eigenbau)

Baue einen vollständigen, eigenen Marktplatz-Connector für die Comentra SaaS-Plattform.
Kein Billbee, kein n8n, keine externe Middleware. Alles komplett selbst gebaut.
Laufumgebung: Hostinger VPS, Node.js + PM2, Supabase als Datenbank.

Ziel: Bestellungen von Amazon.de, eBay.de und Otto.de automatisch in Supabase
einspielen. Multi-Tenant — jeder Tenant hat eigene Marktplatz-Credentials,
die in der Supabase `channels` Tabelle gespeichert sind (credentials: jsonb).

---

## Stack
- Node.js (ES Modules)
- @supabase/supabase-js
- amazon-sp-api (Amazon SP-API)
- ebay-api (npm package für eBay REST API)
- axios (für Otto Partner API)
- node-cron (Polling alle 5 Minuten)
- dotenv
- PM2 ecosystem.config.cjs

---

## Supabase Tabellen (exakte Struktur verwenden)

### channels
  id (uuid), tenant_id (uuid), type (text: "amazon"|"ebay"|"otto"),
  name (text), region (text), credentials (jsonb), settings (jsonb),
  status (text: "active"|"inactive"), last_sync_at (timestamptz)

  credentials Struktur pro Typ:
  - Amazon:  { refresh_token, client_id, client_secret, marketplace_id }
  - eBay:    { access_token, refresh_token, client_id, client_secret }
  - Otto:    { access_token, token_url, api_url }

### orders
  id (uuid), tenant_id (uuid), channel_id (uuid), customer_id (uuid),
  channel_order_id (text), status (text), payment_status (text),
  currency (text), subtotal (numeric), shipping_cost (numeric),
  tax_amount (numeric), discount_amount (numeric), total (numeric),
  shipping_address (jsonb), billing_address (jsonb), notes (text),
  tags (array), metadata (jsonb), ordered_at (timestamptz),
  paid_at (timestamptz), shipped_at (timestamptz),
  created_at (timestamptz), updated_at (timestamptz)

### order_items
  id (uuid), tenant_id (uuid), order_id (uuid), variant_id (uuid),
  listing_id (uuid), name (text), sku (text), qty (integer),
  unit_price (numeric), tax_rate (numeric), total (numeric),
  status (text), purchase_price (numeric), discount_amount (numeric),
  created_at (timestamptz)

---

## Architektur

comentra-connector/
  index.js                  ← Cron-Orchestrierung, lädt alle aktiven Channels
  connectors/
    amazon.js               ← Amazon SP-API Wrapper
    ebay.js                 ← eBay REST API Wrapper
    otto.js                 ← Otto Partner API Wrapper
  services/
    order-sync.js           ← Upsert-Logik für orders + order_items
    customer-sync.js        ← Kunden in customers Tabelle anlegen/updaten
    channel-service.js      ← Channels aus Supabase laden, last_sync_at updaten
  mappers/
    amazon-mapper.js        ← Amazon Response → Comentra Schema
    ebay-mapper.js          ← eBay Response → Comentra Schema
    otto-mapper.js          ← Otto Response → Comentra Schema
  utils/
    retry.js                ← Exponential Backoff für Rate Limits
    logger.js               ← Timestamp-Logging mit Tenant-Kontext
  .env.example
  ecosystem.config.cjs
  package.json

---

## Kern-Logik

### 1. Cron (alle 5 Minuten)
- Lade alle channels WHERE status = 'active' aus Supabase
- Gruppiere nach type (amazon / ebay / otto)
- Für jeden Channel: rufe den passenden Connector auf
- Nach erfolgreichem Sync: update last_sync_at in channels Tabelle

### 2. Amazon Connector (amazon-sp-api)
- getOrders() mit LastUpdatedAfter = channel.last_sync_at
- OrderStatuses: Unshipped, PartiallyShipped, Shipped, Canceled
- getOrderItems() pro Order
- Credentials kommen aus channel.credentials (refresh_token, client_id, etc.)
- Rate Limit: max 12 Requests / Stunde für FBA Reports → retry.js nutzen

### 3. eBay Connector (ebay-api)
- GET /sell/fulfillment/v1/order mit filter=lastModifiedDate
- Access Token aus channel.credentials.access_token
- Falls Token abgelaufen: automatisch refreshen via refresh_token
  und credentials in Supabase updaten
- Rate Limit: 429 → exponential backoff

### 4. Otto Connector (axios)
- Otto Partner API: GET /v4/orders?fulfillmentStatus=PROCESSABLE
- Bearer Token aus channel.credentials.access_token
- Falls abgelaufen: POST zu token_url → neuen Token holen und speichern
- Pagination: nextLink aus Response weiterverfolgen bis keine mehr

### 5. Status-Mapping (alle Kanäle → Comentra)
Amazon:
  Unshipped          → "confirmed"
  PartiallyShipped   → "partially_shipped"
  Shipped            → "shipped"
  Canceled           → "cancelled"
  PaymentComplete    → payment_status: "paid"
  sonst              → payment_status: "pending"

eBay:
  IN_CHECKOUT        → "pending"
  PAYMENT_DISPUTE    → "on_hold"
  FULFILLED          → "shipped"
  CANCELLED          → "cancelled"

Otto:
  PROCESSABLE        → "confirmed"
  SENT               → "shipped"
  RETURNED           → "returned"
  CANCELLED_BY_*     → "cancelled"

### 6. Order Upsert (order-sync.js)
- Prüfe ob channel_order_id + channel_id bereits existiert
- Wenn neu: INSERT
- Wenn vorhanden und Status geändert: UPDATE (nur status, updated_at)
- Alle Queries IMMER mit tenant_id (RLS-sicher)
- order_items: upsert auf order_id + sku

### 7. Customer Sync (customer-sync.js)
- Prüfe ob Kunde (email) bereits in customers Tabelle für diesen Tenant
- Wenn neu: anlegen, customer_id an Order hängen
- Wenn vorhanden: customer_id referenzieren

### 8. Fehlerbehandlung
- Try/catch pro Channel (ein fehlerhafter Channel stoppt nicht andere)
- Try/catch pro Order (eine fehlerhafte Order stoppt nicht den Rest)
- Bei 429 Rate Limit: exponential backoff (1s → 2s → 4s → 8s, max 5 Versuche)
- Alle Fehler mit Timestamp, tenant_id und channel_id loggen

---

## .env (nur Supabase — Marktplatz-Credentials kommen aus DB)
SUPABASE_URL=
SUPABASE_SERVICE_KEY=

---

## Wichtige Hinweise
- tenant_id ist bei JEDER Supabase-Operation Pflicht
- Credentials nie in Logs ausgeben
- Token-Refresh für eBay + Otto direkt in DB zurückschreiben
- Kommentare auf Deutsch
- Kein TypeScript, kein Express
- PM2 Anleitung am Ende für Hostinger VPS Deployment

---

## Testmodus
Baue einen --dry-run Flag: liest Orders von den APIs, loggt sie,
schreibt aber nichts in Supabase. Zum Testen ohne Datenbankänderungen.
