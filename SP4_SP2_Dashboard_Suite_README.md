# SP4 / SP2 Sinter Plant — SCADA Dashboard Suite


---

## At a glance

| # | Dashboard | UID | Plant | Live tags | Purpose |
|---|---|---|---|---|---|
| 1 | **Plant Overview** | `sp4-plant-overview` | SP4 | 45 | Top-level operator screen — health of every major plant area |
| 2 | **Critical Process Monitoring** | `sp4-critical-process` | SP4 | 34 | Sinter strand mimic, ignition hood, permeability radar |
| 3 | **WGF Machine Health** | `sp4-wgf-health` | SP4 | 31 | Single-asset deep dive on the Waste Gas Fan |
| 4 | **Process ESP & DDESP** | `sp4-esp-ddesp` | SP4 | 54 | Gas cleaning path — 4 ESP stages + stack + conveyors + aux motors |
| 5 | **Process ESP (SP2)** | `sp2-process-esp` | SP2 | 47 | Same as #4 but 3 ESP stages, no DDESP |
| 6 | **HT Motors** | `sp4-ht-motors` | SP4 | 59 | 7 HT motors with insulation + bearing health |

**Total: 270 bindings across 6 dashboards, all reading from the same InfluxDB measurement `SP4Iba.94F1A3D6-7319-4C42-8744-CFE06FD3D360` in database `tsdata`.**

---

## Common platform

All 6 dashboards share the same architecture, so this section applies to every one of them.

### Data path

```
PLC / OPC server  →  Litmus Edge (DeviceHub agent)  →  NATS  →  InfluxDB v1  →  Grafana SVG Panel
```

Each sensor/tag streams as one row per sample into InfluxDB. The measurement is the long device UUID `SP4Iba.94F1A3D6-7319-4C42-8744-CFE06FD3D360`. Every row carries the sensor name as an InfluxDB **tag** under the key `"tag"` (e.g. `"tag" = 'FURNACE_TEMPERATURE_1'`), and the reading is stored in the **field** `"value"`.

### Query pattern

Every binding in every dashboard uses the same InfluxQL pattern:

```sql
SELECT last("value") FROM "SP4Iba.94F1A3D6-7319-4C42-8744-CFE06FD3D360" 
WHERE "tag" = '<TAG_NAME>' AND $timeFilter
```

`$timeFilter` is filled in by Grafana from the dashboard's time-range picker.

### How the Marcus Calidus SVG Panel works

A single panel contains three pieces:

1. **SVG document** — the full SCADA mimic, with `id="..."` attributes on every element that needs to be driven by data.
2. **Query list (refId A, B, C, …)** — one InfluxQL query per live binding, each with an `alias` set to the tag name.
3. **Events JS** — JavaScript that runs on every refresh. It reads query results, looks up SVG elements by ID, and updates their text/fill/stroke/position.

The plugin exposes two variables to the Events JS:
- `ctrl.series` — array of series objects, one per query. Each has `.alias`, `.datapoints`, `.refId`, `.stats`.
- `svgnode` — the root `<svg>` element. Use `svgnode.getElementById(...)` to find elements.

### Alias-based series lookup (important)

Each Events JS builds a `seriesByAlias` map once per refresh:

```javascript
var seriesByAlias = {};
for (var i = 0; i < series.length; i++) {
  seriesByAlias[series[i].alias] = series[i];
}
```

Bindings then look up data by **tag name**, not by series position. This matters because Grafana may drop or reorder series that return zero datapoints, which would silently misalign positional indexing. **All 6 dashboards use alias-based lookup** so a temporarily missing tag never breaks the rest of the screen.

### Common color conventions

| Level | Color | Meaning |
|---|---|---|
| Normal | green `#22c55e` | Reading within safe operating range |
| Warning | amber `#f59e0b` | Reading approaching alarm threshold; operator should investigate |
| Critical | red `#ef4444` | Reading past alarm threshold; immediate action required |
| No data | grey `#64748b` | Tag has returned no recent value; sensor may be offline or time range too narrow |

### Refresh rate

All dashboards refresh at **5 seconds**. The default time range varies by dashboard (15 m, 30 m, 1 h); operators can adjust from the top-right time picker. For radar / windbox-style aggregate views, "Last 1 hour" gives the most reliable rendering.

### Import procedure

For each dashboard:

1. Grafana → **Dashboards → New → Import**
2. Upload the corresponding `*_dashboard.json`
3. When prompted, map the **InfluxDB data source** to your existing InfluxDB v1 (the one pointing to the `tsdata` database)
4. Click Import

The SVG content, JavaScript, and all queries are embedded inside the JSON file — no separate uploads needed.

### File inventory

For each dashboard, three files are produced:

| Suffix | Purpose |
|---|---|
| `<name>_dashboard.json` | The complete importable dashboard. **This is the only file you need to import.** |
| `<name>.svg` | Standalone SVG, useful for editing the visual or previewing outside Grafana |
| `<name>_events.js` | Standalone Events JS, useful for code review or editing thresholds |

---

## 1. Plant Overview

**File:** `SP4_Plant_Overview_dashboard.json`

### Purpose

The first screen an operator sees. Shows the health of every major area of the SP4 plant on a single 1920×1080 surface, with summary KPIs at the top and section-by-section breakdowns below.

### Layout

```
┌────────────────────────────────────────────────────────────────┐
│ HEADER  ·  LITMUS EDGE · SP4 SINTER PLANT · Plant Overview      │
├────────────────────────────────────────────────────────────────┤
│ KPI strip: assets live · warnings · critical · health · streaming│
├────────────────────────────────────────────────────────────────┤
│ FURNACE TEMPERATURES                  (5 cards)                  │
│   Furnace 1 │ Furnace 2 │ Furnace 3 │ Furnace 5 │ Furnace 6      │
│   (each with heat-glow overlay proportional to temp)             │
├────────────────────────────────────────────────────────────────┤
│ WGF FAN & MOTOR                       (8 small cards in a row)   │
├────────────────────────────────────────────────────────────────┤
│ STORAGE BINS                          (11 bin pictograms)        │
│   Each shows fill bar + % value                                  │
├────────────────────────────────────────────────────────────────┤
│ TRANSFORMERS TR 1–15                  (15 small cards)           │
├────────────────────────────────────────────────────────────────┤
│ WINDBOX DIFFERENTIAL PRESSURES        (6 cards)                  │
└────────────────────────────────────────────────────────────────┘
```

### Bindings (45 total)

| Section | Bindings | Tags |
|---|---|---|
| Furnaces | 5 | `FURNACE_TEMPERATURE_{1,2,3,5,6}` |
| WGF Fan & Motor | 8 | DE/NDE vibrations, bearing temps, oil pressures |
| Storage Bins | 11 | `STORAGE_BIN{1..11}_LEVEL` |
| Transformers | 15 | `TR_{1..15}_SECONDARY_CURRENT_mA_` |
| Windboxes | 6 | `WINDBOX_{11,12,13,14,15,17A}_DIFFERENTIAL_PRESSURE` |

### Computed roll-ups (no extra queries)

- **Assets with live data** — count of bindings returning a numeric value
- **Active warnings** — count in amber band
- **Critical alarms** — count in red band
- **Plant health score** — % of live bindings that are normal

### Thresholds

| Signal | Normal | Warn | Critical |
|---|---|---|---|
| Furnace temp | < 1200 °C | 1200–1300 | > 1300 |
| Fan vibration | < 4.5 mm/s | 4.5–7.1 | > 7.1 (ISO 10816) |
| Motor bearing temp | < 80 °C | 80–95 | > 95 |
| Oil pressure | > 1.5 bar | 1.0–1.5 | < 1.0 |
| Bin level | 20–80 % | 10–20 / 80–90 | < 10 / > 90 |
| TR secondary current | < 800 mA | 800–1000 | > 1000 |
| Windbox ΔP | < 25 mbar | 25–35 | > 35 |

### Operator notes

- **Furnace heat-glow effect** — the orange glow overlay on each furnace card scales proportionally with temperature above 1000 °C. A glowing card means hot operation; a non-glowing one means standby/cold.
- **Storage bin pictograms** — the vertical fill bar reflects the % level so an operator can see fill states across all 11 bins at a glance without reading numbers.
- **Use case** — control-room wall display, plant-wide situational awareness, shift handover briefings.

---

## 2. Critical Process Monitoring

**File:** `SP4_Critical_Process_dashboard.json`

### Purpose

The sintering operator's working screen. Shows the **strand permeability** state, **burn-through point (BTP)** location, **ignition hood** balance, and **radar view** of windbox ΔP distribution.

### Layout

```
┌────────────────────────────────────────────────────────────────┐
│ Strand Windbox Map (left, large)     │ Ignition Hood (right)    │
│ - 18 WB cells colored by ΔP          │ - Animated flame icon    │
│ - Live ΔP numbers inside each cell   │ - Hood average           │
│ - BTP marker (red triangle, mobile)  │ - 11 zone tiles E3TI031-41│
│ - 5 bed-temp tiles E4TI021-025       │ - Min/Max/Spread          │
├──────────────────────────────────────┴──────────────────────────┤
│ Windbox ΔP Permeability Radar (bottom-left)                     │
│ - 12-axis radar polygon                                          │
│ - Live insights: Max/Min/Mean/Spread/Permeability Health         │
└────────────────────────────────────────────────────────────────┘
```

### Bindings (34 total)

| Section | Bindings | Tags |
|---|---|---|
| Windbox ΔP | 18 | `WINDBOX_{2-11,13-14,15A,15B,16A,16B,17A,17B}_DIFFERENTIAL_PRESSURE` |
| Strand bed temps | 5 | `Applications_Application_2_E4TI021..025` |
| Ignition hood temps | 11 | `Applications_Application_1_E3TI031..041` |

### Computed roll-ups

- **BTP marker position** — placed dynamically over the windbox closest to the hottest bed temperature. As the burn-through migrates along the strand, the marker moves with it.
- **Strand status text** — "X critical · Y warning · Z normal" count of windboxes outside operating range.
- **Hood average / min / max / spread** — across the 11 zones; flags "uneven combustion" if spread > 150 °C or "over-fired" if avg > 1300 °C.
- **Radar insights** — Max ΔP (which WB), Min ΔP (which WB), Mean, Spread, Permeability Health % (= % of windboxes below 25 mbar).

### Thresholds

| Signal | Normal | Warn | Critical |
|---|---|---|---|
| Windbox ΔP | < 25 mbar | 25–35 | > 35 |
| Strand bed temp (E4TI) | < 1200 °C | 1200–1350 | > 1350 |
| Hood zone temp (E3TI) | < 1200 °C | 1200–1300 | > 1300 |

### Operator notes

- **Reading the radar** — concentric rings = 10, 25, 40 mbar. Polygon close to centre = good permeability everywhere. Polygon spiking outward on certain axes = local choke points. A "starfish" (uneven spike pattern) means the bed isn't uniform.
- **BTP placement** — BTP moves forward when sintering speed increases or when bed is more permeable. Watching it drift towards windbox 12+ is a leading indicator that material is sintering further along the strand than usual.
- **Use case** — process operator tuning strand speed, ignition hood firing, and watching for choke points.

---

## 3. WGF Machine Health

**File:** `SP4_WGF_dashboard.json`

### Purpose

Single-asset deep dive for the **Waste Gas Fan**, the most critical rotating equipment in the plant. Shows electrical drive, motor insulation, bearings, fan vibrations, and all four lubrication oil circuits.

### Layout

```
┌────────────────────────────────────────────────────────────────┐
│ KPI strip: Speed · DC Current · Excitation · Asset Health       │
├────────────────────────────────────────────────────────────────┤
│              MOTOR + FAN SCHEMATIC                              │
│  ┌─────────────┐                ┌────────────────────────┐      │
│  │   MOTOR     │                │       FAN (rotating)   │      │
│  │  U1 V1 W1   │── coupling ────│   6 vibration arrows   │      │
│  │  U2 V2 W2   │ shaft          │   around housing       │      │
│  │  NDE  DE    │                │                        │      │
│  └─────────────┘                └────────────────────────┘      │
├────────────────────────────────────────────────────────────────┤
│ LUBE OIL CIRCUITS (4 cards along the bottom)                    │
│  Fan DE │ Fan NDE │ Motor DE │ Motor NDE                        │
│  (each: flow · pressure · temperature bars + worst-of dot)      │
└────────────────────────────────────────────────────────────────┘
```

### Bindings (31 total)

| Section | Bindings | Tags |
|---|---|---|
| Drive | 3 | `WGF_SPEED`, `WGF_DC_CURRENT`, `WGF_EXCITATION_CURRENT` |
| Motor winding | 6 | `WGF_MOTOR_{U1,V1,W1,U2,V2,W2}_WINDING_TEMPERATURE` |
| Motor bearings | 2 | `WGF_MOTOR_{DE,NDE}_BEARING_TEMPERATURE` |
| Fan vibrations | 6 | `WGF_FAN_{DE,NDE}_{X,Y,AXIAL}_VIBRATION` |
| Motor axial vibration | 2 | `WGF_MOTOR_{DE,NDE}_AXIAL_VIBRATION` |
| Lube oil (4 × 3) | 12 | `WGF_{FAN_DE,FAN_NDE,MOTOR_DE,MOTOR_NDE}_LUBRICATIONOIL_{FLOW,PRESSURE,TEMPERATURE}` |

### Special animation

The fan rotor **rotates in real time at a rate proportional to `WGF_SPEED`**. At 800 rpm the rotor visibly spins at full speed; below ~80 rpm the rotor barely moves; at 0 rpm it stops entirely. Rotation is implemented via `setInterval` (33 ms = 30 fps) inside the Events JS, with the angle preserved across refreshes so it doesn't snap back.

### Thresholds

| Signal | Normal | Warn | Critical |
|---|---|---|---|
| WGF Speed | < 780 rpm | 780–820 | > 820 |
| DC Current | < 1800 A | 1800–2000 | > 2000 |
| Excitation Current | < 140 A | 140–160 | > 160 |
| Winding temperatures | < 130 °C | 130–155 | > 155 |
| Bearing temperatures | < 80 °C | 80–95 | > 95 |
| Vibrations | < 4.5 mm/s | 4.5–7.1 | > 7.1 |
| Lube oil flow | > 30 l/min | 20–30 | < 20 |
| Lube oil pressure | > 1.5 bar | 1.0–1.5 | < 1.0 |
| Lube oil temperature | < 65 °C | 65–75 | > 75 |

### Operator notes

- **Winding pucks** use a heat-map color (cyan → green → amber → red over 40–155 °C) so you can see hotspots at a glance even without reading numbers.
- **Vibration arrows** change color by threshold (green → amber → red). The arrow tip points in the direction of measurement (X = horizontal, Y = vertical, Axial = along shaft).
- **Lube circuit worst-of status** — each lube circuit card's status dot is the worst-of-3 (flow + pressure + temperature). One bad reading turns the whole circuit red.
- **Use case** — predictive maintenance, vibration trend tracking, alarm investigation when something on the Plant Overview lights up under WGF.

---

## 4. Process ESP & DDESP

**File:** `SP4_ESP_dashboard.json`

### Purpose

End-to-end view of the **gas cleaning train**, from the sinter machine where dust-laden flue gas is generated through 4 ESP stages and out the stack. This is the dashboard environmental compliance and ESP optimization operators watch.

### Layout — process flow

```
┌─────────────────────────────────────────────────────────────────┐
│ KPI: Sinter M/C · WGF Speed · Stack PM · System Health          │
├─────────────────────────────────────────────────────────────────┤
│ Sinter → Pass A → Pass B → Pass C → DDESP → Stack               │
│ M/C       (5 TR)  (5 TR)  (5 TR)   (3 TR)  (smoke ↑)            │
│                                                                  │
│   each TR card: kV value, mA value, worst-of status dot          │
├─────────────────────────────────────────────────────────────────┤
│ Downstream Conveyors (CC-1..5, CV-6, CV-7)                       │
├─────────────────────────────────────────────────────────────────┤
│ Auxiliary Motors (F1004, F1005, F1006, F1008, F1015-17, F2008)  │
└─────────────────────────────────────────────────────────────────┘
```

### Bindings (54 total)

| Section | Bindings | Tags |
|---|---|---|
| Pass A | 10 | TR-6..10 (V + I each) |
| Pass B | 10 | TR-1..5 |
| Pass C | 10 | TR-11..15 |
| DDESP | 6 | TR-201, TR-202, TR-203 |
| Sinter M/C | 1 | `SINTER_M_C_CURRENT` |
| Stack PM | 1 | `STACK_PM` |
| WGF speed | 1 | `WGF_SPEED_ACTUAL` |
| Conveyors | 7 | `CC_{1-5}_CURRENT`, `CV_{6,7}_CURRENT` |
| Auxiliary motors | 8 | `F{1004,1005,1006,1008,1015,1016,1017,2008}M01_CURRENT` |

### Animated stack smoke

The smoke ellipse at the stack top has its **opacity scaled with the stack PM reading**. Low PM (< 5 mg/Nm³) = barely-visible wisp at 5% opacity. Approaching the 30 mg/Nm³ limit = thick visible smoke. > 50 mg/Nm³ = opaque dark smoke at 90%. This gives an environmental operator immediate visual feedback on emission level.

### Thresholds

| Signal | Normal | Warn | Critical |
|---|---|---|---|
| TR secondary voltage | < 55 kV | 55–65 | > 65 |
| TR secondary current | < 800 mA | 800–1000 | > 1000 |
| Sinter M/C current | < 380 A | 380–420 | > 420 |
| WGF speed | < 780 rpm | 780–820 | > 820 |
| Stack PM | < 30 mg/Nm³ | 30–50 | > 50 |
| Conveyor / aux motor current | < 80 A | 80–100 | > 100 |

### Operator notes

- **Reading TR cards** — both kV and mA are shown. The status dot reflects worst-of: high secondary current with low kV means heavy corona current draw; high kV with low mA means a flashover may be imminent.
- **Stage by stage filtering** — Pass A (TR-6..10) handles the heaviest dust load; PM should drop progressively through B, C, and DDESP. A spike in Pass C currents usually means upstream stages aren't pulling their weight.
- **Use case** — environmental compliance monitoring, ESP tuning to keep stack PM under permit limit, identifying weak TR fields.

### Dropped from the original reference dashboard

Several panels in the original reference were wired to placeholder tags (TR-12 voltage stood in for "Loss System", "Oil Pump Status", "LCA Driver"); these were **dropped rather than carry the mis-wiring forward**. If real digital status tags for ESP rappers, hopper heaters, oil pumps, etc. become available, they can be added as colored indicator dots inside each ESP housing.

---

## 5. Process ESP (SP2)

**File:** `SP2_ESP_dashboard.json`

### Purpose

Same family as #4, but for the **SP2 plant** which has a different configuration:

| | SP4 (Dashboard #4) | **SP2 (this one)** |
|---|---|---|
| ESP stages | 4 (A, B, C, DDESP) | **3** (A, B, C — no DDESP) |
| TR transformers | 18 | **15** |
| Aux motors | 8 | **7** (no F2008) |
| Total bindings | 54 | **47** |



### Important note on the InfluxDB measurement

The queries currently use the **same measurement** as the SP4 dashboards (`SP4Iba.94F1A3D6-7319-4C42-8744-CFE06FD3D360`). If SP2 streams to a separate DeviceHub agent with its own measurement (e.g. `SP2Iba.xxx`), the queries in this dashboard's JSON need to be updated to point at that measurement. A find-and-replace inside the JSON suffices.

---

## 6. HT Motors

**File:** `SP4_HT_Motors_dashboard.json`

### Purpose

**7 High-Tension motors** monitored in parallel with the same signature (6 stator winding RTDs + 2 bearing RTDs + sometimes a motor current). One screen, one card per motor.

### Layout

```
┌─────────────────────────────────────────────────────────────┐
│ KPI: motors live · warnings · critical · fleet health       │
├─────────────────────────────────────────────────────────────┤
│  ┌───E6022──┐  ┌──A1000──┐  ┌──E6023──┐  ┌──PDF────┐         │
│  │  motor   │  │  motor  │  │  motor  │  │  motor  │         │
│  │ silhouette│ │silhouette│ │silhouette│ │silhouette│         │
│  └──────────┘  └─────────┘  └─────────┘  └─────────┘         │
│  ┌──E6024──┐  ┌──GSC3───┐  ┌──MND────┐                       │
│  │  motor   │  │  motor  │  │  motor  │                       │
│  │silhouette│  │silhouette│ │silhouette│                       │
│  └──────────┘  └─────────┘  └─────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

Each motor card shows a proper motor pictogram: silver/grey **slab housing with horizontal cooling fins**, **junction box** on top, **mounting feet** below, **DE end bell** on the right with an **output shaft**. Six **winding RTD pucks** sit on the housing body (2 rows × 3 columns), the **NDE bearing** indicator sits on the back of the housing, and the **DE bearing** sits inside the end bell.

### Bindings (59 total)

| Motor | Winding (CH1-6) | Bearings (DE + NDE) | Current |
|---|---|---|---|
| E6022 | ✓ 6 RTDs | ✓ 2 RTDs | ❌ (no tag in DB) |
| A1000 | ✓ 6 RTDs | ✓ 2 RTDs | ❌ |
| E6023 | ✓ 6 RTDs | ✓ 2 RTDs | ✓ `E6023M01_CURRENT` |
| PDF | ✓ 6 RTDs | ✓ 2 RTDs | ❌ |
| E6024 | ✓ 6 RTDs | ✓ 2 RTDs | ✓ `E6024M01_CURRENT` |
| GSC3 | ✓ 6 RTDs | ✓ 2 RTDs | ❌ |
| MND | ✓ 6 RTDs | ✓ 2 RTDs | ✓ `MND_ACTUAL_CURRENT` |

Motors without a current tag show a greyed-out "no live tag in DB" placeholder for visual consistency.

### Thresholds

| Signal | Normal | Warn | Critical |
|---|---|---|---|
| Winding RTDs | < 130 °C | 130–155 | > 155 |
| Bearing RTDs | < 80 °C | 80–95 | > 95 |
| Motor current | < 80 A | 80–100 | > 100 |

### Computed roll-ups

- **Fleet health %** = % of all live RTDs that are in the normal band
- **Per-motor status pill** = worst-of all 8 or 9 signals on that motor
- **Per-motor winding stats** = min / max / avg across CH1-6

### Operator notes

- **Heat-map pucks** — each winding RTD's puck is colored on a continuous heat scale (cyan → green → amber → red over 40–155 °C). Asymmetric heating (one phase much hotter than the other two) suggests winding insulation issues.
- **Bearing rings** — the colored stroke around each bearing indicator reflects threshold status. A red ring on either bearing should trigger immediate vibration/lube checks (cross-reference with WGF dashboard if it's that motor).
- **Use case** — predictive maintenance, monthly trending of stator insulation, identifying motors approaching insulation Class B/F limits.

### Missing from this dashboard (intentional)

The original reference dashboard referenced **18 vibration tag IDs** for these motors (TAG IDs 30599–30610) that do not exist in the device's tag list. They've been dropped. If vibration tags are later wired up at the edge (e.g. `E6022_DE_X_VIBRATION`), they can be added as arrows around each motor card following the same pattern as the WGF dashboard.

---

## Troubleshooting

### "Some cards stay grey / 'NO DATA'"

The most common cause is **time range too narrow**. Set the dashboard time range (top right) to "Last 1 hour" or "Last 6 hours" so that even sparsely-updated tags are included.

If only certain cards stay grey, those specific tags probably weren't streaming during the time window. To verify, in InfluxDB run:

```sql
SHOW TAG VALUES FROM "SP4Iba.94F1A3D6-7319-4C42-8744-CFE06FD3D360" 
WITH KEY = "tag" WHERE "tag" =~ /YOUR_TAG_NAME/
```

### "Numbers update but status dots stay grey"

The Events JS didn't run, or `svgnode` reference is broken. Open browser DevTools (F12) → Console — any JavaScript error there will tell you the cause.

### "Time series animation (fan rotor) stopped"

The `setInterval` for the rotor lives in the browser window. If the panel was re-rendered (e.g. dashboard time range changed), the interval may have been cleared. Hit refresh, or temporarily switch the panel to edit mode and back.

### "I changed a threshold but the dashboard still uses the old one"

Edit the Events JS in the panel options ("Events" section in the panel options sidebar), search for the binding id (e.g. `furnace_1` or `tr12_kv`), and change the numbers in the line that sets `level = ...`. Click **Apply** on the panel — no full reimport needed.

### "I want to add a new tag to a dashboard"

1. Add a new query in the panel's Query tab with `alias` set to the tag name
2. Add a new element with a unique `id="..."` to the SVG document
3. Add a new block in the Events JS that calls `valueOfTag('YOUR_TAG_NAME')`, applies threshold logic, and updates the SVG element

The alias-based lookup pattern (see "Common platform" section) means the new tag will work regardless of where the new query falls in the refId list.

### "I'm getting moment.js deprecation warnings in the console"

Harmless. Litmus Edge writes timestamps in a slightly non-standard format and Grafana's moment.js wrapper warns about it. The data still parses correctly and dashboard rendering is unaffected.

---

## Roadmap / known limitations

- **Cross-dashboard navigation** — currently each dashboard is independent. A future improvement is to add Grafana **dashboard links** so an operator can click "WGF" on the Plant Overview and jump directly to the WGF Machine Health dashboard.
- **Click-to-trend** — Marcus Calidus does support click events on SVG elements. A natural next step is to make e.g. clicking a winding puck open a popup with that winding's 24h trend.
- **Annotations** — alarm acknowledgments and operator notes could be wired to Grafana annotations so they appear as vertical lines on any future trend panel.
- **Per-tag last-seen badge** — for sparse tags, a small "last seen X min ago" indicator would help operators distinguish "currently safe" from "currently silent".
- **Mobile layout** — current SVG viewBoxes are 1920×1080 (desktop). On a phone they will scale down but text will become hard to read. A separate mobile-optimized dashboard suite with larger numbers and fewer details per screen would help field engineers.

---

## Maintainer reference

**Tag schema cheatsheet** — when looking up a tag in InfluxDB:

```
database     : tsdata
measurement  : SP4Iba.94F1A3D6-7319-4C42-8744-CFE06FD3D360
tag key      : tag
field key    : value
```

**Build pattern** — each dashboard is generated by a single Python script that produces SVG + Events JS + dashboard JSON in lockstep from one binding spec. To keep the three artefacts in sync, always edit the binding spec at the top of the script, never any of the three outputs by hand.

**Plugin compatibility** — built and tested against:
- Grafana **10.0.1**
- Marcus Calidus SVG Panel **0.4.6**
- InfluxDB **v1.x** with InfluxQL queries
