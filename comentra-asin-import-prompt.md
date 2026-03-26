# Claude Code Prompt — Comentra SKU/ASIN Import Tool

Baue ein React-basiertes CSV-Import-Tool für die Comentra SaaS-Plattform.
Ziel: Bestehende Amazon ASINs per SKU mit Produkten in Supabase verknüpfen.
Kein SP-API, kein externes Tool — nur CSV-Upload → Supabase Update.

---

## Stack
- React (bestehende Comentra UI)
- @supabase/supabase-js
- papaparse (CSV-Parsing)
- Supabase Tabelle: product_variants (sku, asin)

---

## Funktionsweise

1. User exportiert aus Amazon Seller Central:
   Lagerbestand → "Bestandsbericht herunterladen" (TSV/CSV enthält SKU + ASIN)

2. User lädt die CSV/TSV-Datei in Comentra hoch

3. Tool parst die Datei, erkennt automatisch:
   - Spalte mit SKU (sucht nach: "seller-sku", "sku", "SKU")
   - Spalte mit ASIN (sucht nach: "asin1", "asin", "ASIN")

4. Tool zeigt Vorschau-Tabelle:
   - Spalten: SKU | ASIN | Status (Gefunden / Nicht gefunden / Bereits verknüpft)
   - Status wird live gegen Supabase geprüft (SELECT WHERE sku = ?)
   - Zeigt Zusammenfassung: X gefunden, Y nicht gefunden, Z bereits verknüpft

5. User klickt "Verknüpfung speichern"
   - UPDATE product_variants SET asin = ? WHERE sku = ? AND tenant_id = ?
   - Nur Zeilen mit Status "Gefunden" werden aktualisiert
   - Nicht gefundene SKUs werden übersprungen und geloggt

6. Ergebnis-Anzeige:
   - X erfolgreich verknüpft
   - Y SKUs nicht in Supabase gefunden (Liste downloadbar als CSV)
   - Z bereits verknüpft (keine Änderung)

---

## Supabase Tabelle

product_variants:
  id (uuid), tenant_id (uuid), product_id (uuid),
  sku (text), name (text), asin (text nullable),
  status (text), is_default (boolean)

Queries IMMER mit tenant_id filtern.

---

## UI Komponenten

### 1. Upload-Bereich
- Drag & Drop oder Datei-Button
- Akzeptiert: .csv, .txt, .tsv
- Zeigt Dateiname + Zeilenanzahl nach Upload

### 2. Spalten-Mapping (falls automatische Erkennung fehlschlägt)
- Dropdown: "Welche Spalte ist die SKU?"
- Dropdown: "Welche Spalte ist die ASIN?"
- Nur anzeigen wenn automatische Erkennung fehlschlägt

### 3. Vorschau-Tabelle
- Max 100 Zeilen anzeigen, Rest zusammengefasst
- Farbkodierung:
  grün  = SKU in Supabase gefunden, ASIN wird eingetragen
  grau  = bereits verknüpft (ASIN identisch)
  gelb  = ASIN wird überschrieben (alte ASIN war anders)
  rot   = SKU nicht in Supabase gefunden

### 4. Aktions-Button
- "X Produkte verknüpfen" (nur aktiv wenn mind. 1 grün)
- Ladeindikator während Supabase-Updates
- Fortschrittsanzeige: "23 / 150 aktualisiert..."

### 5. Ergebnis
- Erfolgs-/Fehlermeldung
- Download-Button für nicht gefundene SKUs als CSV

---

## Dateistruktur (in bestehende Comentra UI einbauen)
src/
  components/
    AsinImport/
      AsinImportPage.jsx     ← Haupt-Komponente
      CsvDropzone.jsx        ← Upload-Bereich
      PreviewTable.jsx       ← Vorschau-Tabelle mit Status
      ImportSummary.jsx      ← Ergebnis nach Import
      csvParser.js           ← papaparse Wrapper + Spalten-Erkennung
      supabaseImport.js      ← Supabase Lookup + Update Logik

---

## Wichtige Hinweise
- tenant_id immer mitgeben bei allen Supabase-Queries
- Batch-Updates in 50er Gruppen (nicht alle auf einmal)
- Keine sensiblen Daten loggen
- Kommentare auf Deutsch
- Kein TypeScript
- Design: Comentra Design System (dunkles Theme, Gold-Akzente #C9A84C)

---

## Amazon CSV Format (Referenz)
Amazon Seller Central exportiert typischerweise TSV mit diesen Spalten:
"item-name", "item-description", "listing-id", "seller-sku",
"price", "quantity", "asin1", "product-id-type" ...

Der Parser soll robust sein und auch abweichende Spaltennamen erkennen.
