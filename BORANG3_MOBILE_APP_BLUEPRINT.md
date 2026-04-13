# Borang 3 Mobile App Blueprint

## 1) Full App Structure (Pages + Components)

### App shell
- **Splash + Auth bootstrap**
  - Anonymous/officer login (Firebase Auth)
  - Preload reference data (microbes, sensitivity matrix, CCP rules)
  - Offline cache warm-up
- **Home Dashboard**
  - `New Investigation`
  - `Continue Draft`
  - `Sync Queue`
  - `Reports`

### Investigation flow (mobile-first wizard: Step 1 → Step 5)

#### Step 1 — General Case Information
**Page:** `CaseInfoPage`

**Components:**
- Auto Case ID chip (read-only)
- Date picker (`Tarikh siasatan`)
- Geo capture + map pin (`Lokasi premis`)
- Text fields (`Nama premis`, `Pegawai penyiasat`)
- Smart dropdown (`Jenis premis`)
- Progress bar (20%)

#### Step 2 — Agen Penyebab (Cause Agent)
**Page:** `CauseAgentPage`

**Components:**
- Category selector: Microbe / Chemical / Physical
- Conditional panel:
  - If **Microbe**:
    - Searchable multi-select dropdown of microorganisms
    - Inline cards showing:
      - Nama mikroorganisma
      - Jenis (bacteria/virus/parasite)
      - Associated food
- Quick filter chips: `Bacteria`, `Virus`, `Parasite`
- Progress bar (40%)

#### Step 3 — Bahan Mentah (Raw Ingredient Analysis)
**Page:** `RawIngredientPage`

**Components:**
- Repeating list item for each ingredient:
  - Nama bahan mentah
  - Sumber
  - Cara penyimpanan
  - Suhu penyimpanan
  - Tempoh simpanan
- Auto-analysis badge per ingredient:
  - Sensitivity: Low / Medium / High
  - Risiko pencemaran
- High-risk warning banner if any ingredient=high
- Progress bar (60%)

#### Step 4 — Proses Penyediaan Makanan + Maklumat Pekerja
**Page:** `ProcessAndWorkerPage`

**A) Process components**
- Step repeater:
  - Description
  - Suhu
  - Masa
- System-evaluated fields:
  - `CCP detected?`
  - `Risk indicator`
- Visual highlight for CCP rows

**B) Worker components**
- Worker repeater:
  - Nama pekerja
  - Peranan
  - Status latihan (Yes/No)
  - Status suntikan typhoid (Yes/No)
- Compliance alert card:
  - `⚠️ Ketidakpatuhan pekerja: latihan/suntikan tidak lengkap`
- Progress bar (80%)

#### Step 5 — Evidence + Summary + Submit
**Page:** `EvidenceSummaryPage`

**A) Image evidence**
- Camera capture + gallery upload
- Multi-image picker
- Required tag per image: `premis`, `bahan`, `proses`, `pekerja`

**B) Risk Summary Dashboard**
- Overall risk score (0–100)
- Number of CCP issues
- High-risk ingredients count
- Worker compliance issues count
- Traffic-light color:
  - Green: 0–34
  - Yellow: 35–69
  - Red: 70–100

**C) Actions**
- Save local draft
- Sync now
- Generate PDF
- Share (email/WhatsApp)
- Finalize investigation

### Supporting pages
- **DraftsPage** (offline + cloud statuses)
- **SyncQueuePage** (retry failed sync)
- **ReportsPage** (list PDFs + share/download)
- **SettingsPage** (master data refresh, language, officer profile)

---

## 2) Database Schema (Firestore-first, relational-friendly)

> Target stack: FlutterFlow + Firebase (Auth, Firestore, Storage, Cloud Functions)

### Collections

#### `reference_microorganisms`
- `id` (doc id)
- `name` (string)
- `type` (enum: bacteria|virus|parasite)
- `associated_foods` (array<string>)
- `aliases` (array<string>)
- `is_active` (bool)

#### `reference_ingredient_sensitivity`
- `id`
- `ingredient_name` (string)
- `sensitivity_level` (enum: low|medium|high)
- `contamination_risk` (string)
- `threshold_temp_c_min` (number?)
- `threshold_temp_c_max` (number?)
- `max_storage_hours` (number?)
- `source_tab` (string = "Tab 4")

#### `reference_ccp_rules`
- `id`
- `process_keyword` (string)
- `condition_temp_min` (number?)
- `condition_temp_max` (number?)
- `condition_time_min` (number?)
- `condition_time_max` (number?)
- `ccp_flag` (bool)
- `risk_weight` (number)
- `source_tab` (string = "Tab 5")

#### `investigations`
- `id` (Case ID, format: `KRM-YYYYMMDD-XXXX`)
- `created_at`, `updated_at`
- `created_by_uid`
- `sync_status` (local_only|queued|synced|failed)
- `general` (map)
  - `tarikh_siasatan`
  - `lokasi_premis` (geopoint + text)
  - `nama_premis`
  - `pegawai_penyiasat`
  - `jenis_premis`
- `cause_agent` (map)
  - `category`
  - `selected_microbe_ids` (array<string>)
- `risk_summary` (map)
  - `overall_score` (number)
  - `overall_level` (green|yellow|red)
  - `ccp_issue_count` (number)
  - `high_risk_ingredient_count` (number)
  - `worker_noncompliance_count` (number)

#### `investigations/{id}/raw_ingredients`
- `id`
- `name`
- `source`
- `storage_method`
- `storage_temp_c`
- `storage_duration_hours`
- `sensitivity_level` (auto)
- `contamination_risk` (auto)
- `high_risk_flag` (auto)

#### `investigations/{id}/process_steps`
- `id`
- `step_no`
- `description`
- `temp_c`
- `time_min`
- `ccp_detected` (auto)
- `risk_indicator` (low|medium|high)
- `rule_match_ids` (array<string>)

#### `investigations/{id}/workers`
- `id`
- `name`
- `role`
- `food_handler_training` (bool)
- `typhoid_vaccinated` (bool)
- `noncompliance_flag` (auto)

#### `investigations/{id}/images`
- `id`
- `storage_path`
- `download_url`
- `tag` (premis|bahan|proses|pekerja)
- `captured_at`
- `captured_by_uid`

#### `sync_logs`
- `id`
- `investigation_id`
- `attempt_no`
- `status`
- `error_message`
- `timestamp`

### Storage buckets/folders
- `/investigations/{caseId}/images/{imageId}.jpg`
- `/investigations/{caseId}/reports/{caseId}.pdf`

---

## 3) Logic Flow (Automation Rules)

### A. Case creation
1. Officer taps `New Investigation`.
2. App generates Case ID locally (`KRM-YYYYMMDD-rand4`).
3. Create local draft immediately for offline resilience.

### B. Cause agent smart logic
1. If category != `Microbe`, skip microbe selector.
2. If category = `Microbe`:
   - Load `reference_microorganisms` from local cache.
   - Enable search + multi-select.
   - Show metadata chips (type + associated foods).

### C. Raw ingredient risk automation
For each ingredient row:
1. Fuzzy match ingredient name to `reference_ingredient_sensitivity`.
2. Apply baseline sensitivity + contamination risk.
3. Adjust risk using entered storage temp and duration:
   - Out-of-range temp => +risk weight
   - Over max duration => +risk weight
4. Persist computed fields.
5. If sensitivity `high`, trigger warning banner.

### D. Process CCP detection
For each process step:
1. Normalize text (lowercase, stemming keyword search).
2. Match against `reference_ccp_rules`.
3. Evaluate temp/time conditions.
4. If match + condition met, set `ccp_detected=true`.
5. Assign `risk_indicator` using cumulative rule weight.
6. Highlight row in UI.

### E. Worker compliance alerts
For each worker:
- Noncompliance if:
  - `food_handler_training = false` OR
  - `typhoid_vaccinated = false`
- Trigger global alert if any worker noncompliant:
  - `⚠️ Ketidakpatuhan pekerja: latihan/suntikan tidak lengkap`

### F. Overall risk score
Suggested formula:
- `score = (highRiskIngredients * 20) + (ccpIssues * 15) + (workerNonCompliance * 10) + (criticalTempViolations * 10)`
- Clamp 0..100.
- Map level:
  - 0–34 Green
  - 35–69 Yellow
  - 70–100 Red

### G. Offline-first + sync
1. All writes go to local store first (Hive/SQLite/Firestore cache).
2. Background listener checks connectivity.
3. On online:
   - Push queued records + images.
   - Then recalculate summary server-side (Cloud Function) for consistency.
4. Store sync result in `sync_logs`.

### H. PDF generation flow
1. On `Generate PDF`, compile report JSON.
2. Render template sections in fixed order.
3. Embed thumbnails/images with tags.
4. Save PDF locally + upload to Storage when online.
5. Expose `Download` and `Share` intents.

---

## 4) UI Wireframe Description (Mobile)

### Navigation model
- **Top:** Title + Case ID chip
- **Body:** Single-step form card
- **Bottom sticky:** Back / Next / Save Draft
- **Progress bar:** 5 steps

### Step wireframes

#### Step 1 (General)
- Card 1: Case metadata (auto ID, date)
- Card 2: Premise details
- Primary CTA: `Seterusnya`

#### Step 2 (Cause Agent)
- Segmented control: category
- If microbe: search field + multi-select list + selected chips

#### Step 3 (Raw Ingredients)
- `+ Tambah Bahan` button (large)
- Each ingredient in collapsible card
- Sensitivity badge right-aligned
- Red warning banner appears at top when high risk exists

#### Step 4 (Process + Worker)
- Tab-like subheader: `Proses | Pekerja`
- Proses list with CCP highlighted in amber/red border
- Worker cards with compliance icon (✅/⚠️)
- Sticky compliance alert bar at bottom when triggered

#### Step 5 (Evidence + Summary)
- Large buttons: `Ambil Gambar`, `Muat Naik`
- Tag selector per image (required)
- Risk dashboard tiles (score, CCP, ingredients, workers)
- Final CTA row:
  - `Simpan`
  - `Jana PDF`
  - `Hantar`

### UX decisions for field officers
- One-hand use, large touch targets (min 44px)
- Default values + smart picks to reduce typing
- Validation in plain Bahasa Melayu
- Autosave on every step transition

---

## 5) Sample Data

### A. Microorganisms (Tab 2)
```json
[
  {
    "id": "micro_001",
    "name": "Salmonella enterica",
    "type": "bacteria",
    "associated_foods": ["ayam", "telur", "susu"]
  },
  {
    "id": "micro_002",
    "name": "Norovirus",
    "type": "virus",
    "associated_foods": ["makanan sedia dimakan", "air tercemar"]
  },
  {
    "id": "micro_003",
    "name": "Giardia lamblia",
    "type": "parasite",
    "associated_foods": ["air", "sayur mentah"]
  }
]
```

### B. Ingredient sensitivity (Tab 4)
```json
[
  {
    "ingredient_name": "ayam mentah",
    "sensitivity_level": "high",
    "contamination_risk": "Pertumbuhan bakteria cepat jika >5°C",
    "threshold_temp_c_max": 5,
    "max_storage_hours": 24
  },
  {
    "ingredient_name": "nasi masak",
    "sensitivity_level": "medium",
    "contamination_risk": "Risiko Bacillus cereus jika suhu bilik terlalu lama",
    "threshold_temp_c_max": 60,
    "max_storage_hours": 4
  }
]
```

### C. CCP rules (Tab 5)
```json
[
  {
    "process_keyword": "memasak ayam",
    "condition_temp_min": 75,
    "condition_time_min": 2,
    "ccp_flag": true,
    "risk_weight": 25
  },
  {
    "process_keyword": "penyejukan",
    "condition_temp_max": 5,
    "ccp_flag": true,
    "risk_weight": 20
  }
]
```

### D. Investigation sample
```json
{
  "id": "KRM-20260413-4F9A",
  "general": {
    "tarikh_siasatan": "2026-04-13",
    "lokasi_premis": "Sekolah Menengah Seri Murni, Shah Alam",
    "nama_premis": "Kantin Seri Murni",
    "pegawai_penyiasat": "Pn. Aina",
    "jenis_premis": "Kantin Sekolah"
  },
  "cause_agent": {
    "category": "Microbe",
    "selected_microbe_ids": ["micro_001"]
  },
  "risk_summary": {
    "overall_score": 78,
    "overall_level": "red",
    "ccp_issue_count": 2,
    "high_risk_ingredient_count": 1,
    "worker_noncompliance_count": 1
  }
}
```

---

## 6) PDF Report Template Layout

### Document metadata
- Header:
  - Logo/Jata + `Borang 3 Laporan Penyiasatan Keracunan Makanan`
  - Case ID, Date, Officer
- Footer:
  - Page number
  - Generated timestamp

### Section order
1. **Ringkasan Kes**
   - Case info table
2. **Agen Penyebab**
   - Category + selected microorganisms
3. **Analisis Bahan Mentah**
   - Table: ingredient, source, storage, sensitivity, contamination risk
   - Highlight high-risk rows
4. **Proses Penyediaan & CCP**
   - Step table with temp/time
   - CCP flag + risk indicator
5. **Maklumat Pekerja & Pematuhan**
   - Worker table
   - Compliance issue summary + alert text
6. **Dashboard Risiko**
   - Overall score + color bar
   - KPI counts (CCP, ingredient, worker)
7. **Lampiran Gambar Bukti**
   - Image grid by tags (premis/bahan/proses/pekerja)
   - Captions and timestamps
8. **Perakuan Pegawai**
   - Signature block
   - Name and designation

### Export actions
- `Download PDF` (local file)
- `Share` (WhatsApp/email via native share sheet)

---

## Recommended Build Stack (Low-code friendly)

- **UI / App builder:** FlutterFlow
- **Backend:** Firebase (Firestore, Storage, Auth, Functions)
- **Offline:** Flutter local persistence + queued sync
- **PDF engine:** Cloud Function (server render) or device-side PDF package
- **AI assistance (optional):** Google AI Studio for anomaly explanation text only

This architecture keeps the workflow fast for field officers, supports offline inspections, and automates HACCP-style risk interpretation without manual calculations.
