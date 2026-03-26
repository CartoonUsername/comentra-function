# Claude Code Prompt — Comentra Amazon Product Upload Wizard

Baue einen dynamischen Amazon Produkt-Upload Wizard für die Comentra SaaS-Plattform.
Eigenständige Seite, nur für Amazon. Kein statisches Formular — alle Felder werden
live von der Amazon Product Type Definitions API abgerufen und dynamisch gerendert.

Comentra nutzt eigene SP-API App-Credentials (eine App für alle Tenants).
Tenants autorisieren einmalig via OAuth — der Refresh Token wird in channels.credentials gespeichert.

---

## Stack
- React (bestehende Comentra UI)
- @supabase/supabase-js
- axios (SP-API Calls vom VPS Backend)
- Node.js Express Microservice auf Hostinger VPS (SP-API darf nicht direkt vom Browser aufgerufen werden)
- Comentra Design System (dunkles Theme, Gold-Akzente #C9A84C)

---

## Architektur

Browser (React) → Comentra Backend API (Express/VPS) → Amazon SP-API
                                    ↓
                              Supabase (Schema Cache + Produkt-Daten)

Der React-Frontend ruft NIE direkt die Amazon SP-API auf.
Alle SP-API Calls laufen über einen eigenen Express-Endpunkt auf dem VPS.

---

## Backend API Endpunkte (Express, VPS)

### GET /api/amazon/product-types?query=bilderrahmen&tenant_id=xxx
- Ruft Amazon Product Type Definitions API auf: searchDefinitions()
- Gibt Liste passender Produkttypen zurück (z.B. PICTURE_FRAME, PHOTO_FRAME)
- Schema wird 24h in Supabase gecacht (Tabelle: schema_cache)

### GET /api/amazon/schema/:productType?tenant_id=xxx
- Ruft vollständiges JSON Schema für den Produkttyp ab
- requirements=LISTING, locale=de_DE, marketplaceId=A1PA6795UKMFR9
- Parsed das Schema: extrahiert Pflichtfelder, optionale Felder, Enums, Validierungen
- Gibt strukturiertes Schema-Objekt zurück (nicht rohe JSON Schema)
- 24h Cache in Supabase

### POST /api/amazon/upload
- Nimmt ausgefüllte Produktdaten + productType + tenant_id
- Baut korrekten SP-API Listings Items putListingsItem Payload
- Sendet an Amazon: PUT /listings/2021-08-01/items/{sellerId}/{sku}
- Gibt Status zurück: success / validation_errors / pending

### GET /api/amazon/upload-status/:sku?tenant_id=xxx
- Pollt Amazon auf Verarbeitungsstatus des Uploads
- Gibt issues zurück falls Amazon Fehler meldet

---

## Supabase Tabellen

### schema_cache (neu anlegen)
  id (uuid), product_type (text), marketplace_id (text),
  schema (jsonb), cached_at (timestamptz), expires_at (timestamptz)

### amazon_uploads (neu anlegen)
  id (uuid), tenant_id (uuid), product_id (uuid), variant_id (uuid),
  product_type (text), sku (text), status (text), payload (jsonb),
  issues (jsonb), submitted_at (timestamptz), updated_at (timestamptz)

### channels (bestehend)
  credentials: {
    client_id, client_secret, refresh_token, marketplace_id,
    seller_id  ← neu hinzufügen
  }

---

## React Wizard (5 Schritte)

### Schritt 1 — Produkttyp wählen
- Suchfeld: User tippt "Bilderrahmen", "Sandkasten" etc.
- Live-Suche gegen Backend → zeigt passende Amazon Produkttypen
- User wählt einen aus (z.B. PICTURE_FRAME)
- Zeigt kurze Beschreibung was dieser Typ bedeutet

### Schritt 2 — Produkt verknüpfen
- User wählt aus bestehenden Supabase-Produkten (products Tabelle)
- Oder: "Neues Produkt anlegen" → öffnet Mini-Formular
- Zeigt welche Varianten das Produkt hat

### Schritt 3 — Dynamisches Formular (KERN-FEATURE)
Schema wird vom Backend geladen, dann:

PFLICHTFELDER zuerst (rot markiert):
  - Jedes Feld wird basierend auf dem Schema-Typ gerendert:
    string    → Text-Input
    number    → Number-Input mit Unit-Dropdown (kg/g, cm/mm/m etc.)
    enum      → Dropdown mit allen erlaubten Werten
    boolean   → Toggle
    array     → Multi-Input (z.B. 5 Bullet Points)
    object    → Grouped Fields (z.B. dimensions: L + B + H + unit)

VARIANTEN-SEKTION (falls Produkt Varianten hat):
  - Zeigt Variation Theme Dropdown (COLOR, SIZE, SIZE_NAME, COLOR_NAME etc.)
  - Pro Variante: eigene SKU, EAN, Preis, Bestand + variantenspezifische Felder
  - "Variante hinzufügen" Button
  - Bei 400 Varianten: Bulk-Import via CSV möglich

OPTIONALE FELDER (ausgeklappt/eingeklappt):
  - Gruppiert nach propertyGroups (Offer, Images, Compliance, SEO etc.)
  - Können einzeln aufgeklappt werden

ECHTZEIT-VALIDIERUNG:
  - Pflichtfelder rot wenn leer
  - Enum-Felder: nur erlaubte Werte
  - Numerische Felder: Min/Max aus Schema
  - Fortschrittsanzeige: "23 / 40 Pflichtfelder ausgefüllt"

### Schritt 4 — Vorschau & Compliance Check
- Zeigt Zusammenfassung aller eingegebenen Daten
- Compliance-Warnungen (GPSR, Gefahrgut, CE falls relevant)
- Schätzt Listing-Qualität (Titel-Länge, Bullet-Points-Qualität etc.)
- "Zurück" Button zum Bearbeiten

### Schritt 5 — Upload & Status
- "Jetzt bei Amazon hochladen" Button
- Zeigt Upload-Status in Echtzeit
- Bei Erfolg: ASIN wird in product_variants gespeichert
- Bei Fehler: Zeigt exakte Amazon-Fehlermeldungen mit Hinweis welches Feld fehlt
- Ermöglicht direkte Korrektur ohne von vorne anzufangen

---

## Schema Parser (Backend, wichtig)

Das Amazon JSON Schema ist sehr komplex (verschachtelte allOf, anyOf, if/then).
Der Parser muss:
- Alle required Felder flach extrahieren
- Enum-Werte pro Feld sammeln
- Units pro numerischem Feld extrahieren
- Conditional requirements auflösen (wenn Feld X gesetzt, wird Feld Y Pflicht)
- propertyGroups für UI-Gruppierung nutzen
- Felder die für de_DE / A1PA6795UKMFR9 nicht relevant sind, ausfiltern

---

## Varianten-Logik (Parent/Child)

Parent-Produkt:
  - parentage_level: "parent"
  - variation_theme: z.B. "SIZE" oder "COLOR_NAME" oder "SIZE_NAME"
  - Keine SKU-spezifischen Felder (kein Preis, kein Bestand)

Kind-Produkte (eine Submission pro Variante):
  - parentage_level: "child"
  - parent_sku: SKU des Parent
  - relationship_type: "variation"
  - variation_theme: gleich wie Parent
  - child_parent_sku_relationship: [{ child_relationship_type: "variation", parent_sku: "..." }]
  - Eigene SKU, EAN, Preis, Bestand

Bei 400 Varianten:
  - Bulk-CSV Upload: SKU, EAN, Preis, Bestand, Varianten-Wert (z.B. Größe)
  - Backend verarbeitet alle 400 als separate putListingsItem Calls
  - Rate Limit beachten: 5 req/s, max 10 burst → Queue mit Delay

---

## Dateistruktur

Backend (VPS):
comentra-amazon-api/
  server.js                    ← Express Server
  routes/
    productTypes.js            ← GET /api/amazon/product-types
    schema.js                  ← GET /api/amazon/schema/:type
    upload.js                  ← POST /api/amazon/upload
    status.js                  ← GET /api/amazon/upload-status/:sku
  services/
    spApiClient.js             ← SP-API Wrapper (LWA OAuth + Requests)
    schemaParser.js            ← JSON Schema → UI-freundliches Format
    uploadBuilder.js           ← UI-Daten → SP-API Payload
    variantQueue.js            ← Rate-limited Queue für Massen-Uploads
  middleware/
    tenantAuth.js              ← tenant_id validieren gegen Supabase
  .env.example
  ecosystem.config.cjs

Frontend (React):
src/components/AmazonWizard/
  AmazonWizardPage.jsx         ← Haupt-Seite, Schritt-Steuerung
  steps/
    ProductTypeStep.jsx        ← Schritt 1
    ProductSelectStep.jsx      ← Schritt 2
    DynamicFormStep.jsx        ← Schritt 3 (Kernkomponente)
    PreviewStep.jsx            ← Schritt 4
    UploadStep.jsx             ← Schritt 5
  components/
    DynamicField.jsx           ← Rendert ein Feld basierend auf Schema-Typ
    VariantEditor.jsx          ← Varianten hinzufügen/bearbeiten
    VariantBulkImport.jsx      ← CSV-Import für viele Varianten
    SchemaProgress.jsx         ← "23/40 Pflichtfelder" Anzeige
  hooks/
    useProductTypeSearch.js    ← Debounced Suche
    useSchema.js               ← Schema laden + cachen
    useUpload.js               ← Upload + Polling

---

## .env (Backend)
SUPABASE_URL=
SUPABASE_SERVICE_KEY=
AMAZON_CLIENT_ID=              ← Comentra eigene App
AMAZON_CLIENT_SECRET=          ← Comentra eigene App
AMAZON_MARKETPLACE_ID=A1PA6795UKMFR9
PORT=3001

---

## Wichtige Hinweise
- tenant_id bei jeder Supabase-Operation Pflicht
- Credentials nie in Logs
- Schema-Cache 24h in Supabase (spart SP-API Calls)
- Bei Rate Limit (429): exponential backoff
- Alle Kommentare auf Deutsch
- Kein TypeScript
- PM2 Anleitung für VPS Deployment am Ende
