# wx/ — Airport Weather

Single-file static page (`index.html`, no build step, no backend). Aviation weather for
any US airport: current METAR plus a 10-day forecast with per-hour flight category.

**This is a flight-planning safety tool.** Errors that make conditions look *better* than
forecast are the dangerous kind. When a judgment call is close, take the conservative side
and write down why.

---

## ForeFlight parity: what ForeFlight is actually doing

Determined 2026-08-16 by fingerprinting ForeFlight screenshots against candidate sources.

**ForeFlight's hourly forecast is the National Blend of Models (NBM).** Evidence:

- Hourly temperature matched NBM **52/52 (100%)** across 4 airport-days, while GFS was off
  by up to 4 °F on the same hours.
- Cloud cover 70 % → ForeFlight prints "Mostly Cloudy", matching an NBM-percentage → text
  mapping exactly (see `skyDesc`).
- Visibility caps at 10 sm in whole miles — the NBM cap.

**ForeFlight's "CEILING" row is not always a ceiling.** In the NBM station bulletin `cig`
and `lcb` are identical whenever a ceiling exists; when `cig = -88` (unlimited) `lcb` still
carries a value. KCMA read `lcb = 250` → ForeFlight displayed "25,000' CEILING". So that
row is **lowest cloud base**, printing "None" only when there is no cloud at all.

We display the same numbers but disambiguate: a real BKN/OVC ceiling renders bold, a cloud
base with no ceiling renders dimmed, italic, **in parentheses**. This matters — two adjacent
hours can legitimately both read `200'` while one is LIFR and the other VFR, and styling
alone is too weak a signal for that.

---

## Field decoding (verified empirically — do not "correct" from memory)

### NBM bulletins (NBS / NBE) — real values, NOT categories

| Field | Unit | Sentinel |
|---|---|---|
| `cig` | hundreds of feet | `-88` = unlimited (no BKN/OVC layer) |
| `lcb` | hundreds of feet | `-88` = no cloud at all |
| `vis` | **tenths** of a statute mile, capped at 100 | — |
| `sky` | percent | — |
| `ifc`/`ifv`/`mvc`/`mvv` | percent probability of IFR/MVFR ceiling/vis | — |

Verified against KMRY fog (`vis 2` = 0.2 sm, `cig 2` = 200'), KSFO, and KCMA cirrus
(`lcb 250` = 25,000').

### MOS / LAMP (MAV, MEX, LAV) — categorical

```
CIG: 1=<200  2=200-400  3=500-900  4=1000-1900  5=2000-3000  6=3100-6500  7=6600-12000  8=>12000/unlimited
VIS: 1=<1/2  2=1/2-<1   3=1-<2     4=2-<3       5=3-5        6=6         7=>6
```

We take the **lower bound** of each bin. The bins are drawn on flight-category boundaries,
so the lower edge always yields the correct *and* conservative category.

> An earlier version of this file had both tables wrong — `CIG 5` mapped to 4,500 ft (it is
> 2,000–3,000) and `VIS 6` to 3 sm (it is 6). That made the page report **VFR for MVFR
> ceilings**. If you touch these tables, re-derive them from the NWS MOS documentation.

---

## Source ladder

Best available source wins per hour (`SRC_ORDER` in `index.html`), each hour badged with
where it came from:

| Rank | Source | Range | Cadence | Notes |
|---|---|---|---|---|
| 1 | **TAF** | 0–30 h | irregular | Authoritative, human-issued. Always wins. AVWX → aviationweather.gov fallback. TEMPO groups skipped. |
| 2 | **LAMP** (`LAV`) | 0–38 h | hourly | Terminal-calibrated aviation MOS |
| 3 | **NBM short** (`NBS`) | 6–72 h | 3-hourly | Real ceiling/vis values — what ForeFlight uses |
| 4 | **GFS MOS** (`GFS`/MAV) | 0–72 h | 3-hourly | Fallback where no NBM bulletin exists |
| 5 | **NBM extended** (`NBE`) | to +264 h | 12-hourly | **No ceiling or visibility fields at all** |
| 6 | **GFS extended** (`MEX`) | to +192 h | 12-hourly | `cig`/`vis` are also null — only `cld` categories |
| 7 | `~est` | — | hourly | Derived from Open-Meteo cloud cover + LCL |

Bulletins come from IEM (`mesonet.agron.iastate.edu/api/1/mos.json`). Hourly backbone
(temp / dewpoint / wind / gusts / cloud) is Open-Meteo with `models=ncep_nbm_conus,best_match`.

---

## Measured accuracy vs ForeFlight

52 hours, 4 airport-days (KCMA + KMRY, Mon/Tue/Wed), contemporaneous data, 2026-08-16:

| Field | Agreement |
|---|---|
| Temperature | **52/52 (100 %)** |
| Flight category | 36/52 (69 %) after the NBE fix |
| Visibility | 21/52 |
| Ceiling | 4/52 |

**Error direction matters more than the rate:** 21 of 24 category misses were us calling
*worse* conditions than ForeFlight. The 3 optimistic misses were all KCMA 05–07, where we
showed IFR 700' against ForeFlight's LIFR 200' — both solidly no-go for VFR, but that is
the failure mode to watch.

### Why ceiling agreement is low, and why it is not fixable here

ForeFlight reads the **hourly** NBM grid; we can only reach the **3-hourly** bulletin, so a
sharp two-hour fog event gets smeared across seven hours and loses its depth. All three
alternatives were checked and ruled out:

- NOMADS serves the NBM grid as GRIB2 with **no CORS** — unusable from a browser.
- Open-Meteo's NBM exposes **no ceiling field**: `cloud_base` is 100 % null.
- `MEX` carries no `cig`/`vis` either.

Closing this gap requires a **server-side job** to pull NBM GRIB and expose the station
point. That is the single highest-value improvement available to this page.

### Practical confidence, by horizon

- **0–30 h — trust it.** TAF/LAMP, hourly and authoritative.
- **30–72 h — direction right, timing and depth wrong.** Do not use for a marginal
  instrument-approach-minimums call.
- **Beyond 72 h — advisory only.** Badged `NBMX`; ~67 % of a 10-day view. No real ceiling
  or visibility data exists at this range from any browser-reachable source.

---

## Invariants — do not regress these

1. **`fltCat` bounds are inclusive at the top.** MVFR is a ceiling of 1,000 **to 3,000** ft
   and vis **3 to 5** sm. Strict `<` reports a 3,000' ceiling as VFR. Boundary test in
   "Testing" below covers all 13 edge cases.
2. **`roundCeil` floors, never rounds to nearest** — 999' → 950', still IFR. Rounding to
   nearest would let a value drift optimistically across a category boundary.
3. **Category is computed from the value actually displayed**, so the badge and the number
   beneath it can never disagree.
4. **Never interpolate a ceiling across an unlimited ↔ finite transition.** Switch at the
   midpoint (`lerpCeil`). A marine layer arrives as a step, not a slow descent.
5. **Cloud base is never a category driver** — only a true BKN/OVC ceiling is.
6. **NBE ceiling threshold is sky ≥ 80 %, and this number is load-bearing.** A plain BKN
   threshold (62.5 %) invented an MVFR ceiling for 8 straight hours at KCMA. Tightening to
   OVC (87.5 %) is *worse than either*: Monterey's real marine-layer IFR sits at 80–87 %
   with an LCL near 700 ft, and 87.5 % would suppress genuine IFR.

---

## API gotchas (each of these caused a real bug)

- **Open-Meteo suffixes keys with the model name only while that model covers the point.**
  Outside the CONUS NBM domain (Alaska, Hawaii, Puerto Rico, Guam) it silently drops the
  model and returns **bare, unsuffixed keys**. Without the fallback in `omPick`, every
  lookup returns null and those airports render an empty all-VFR forecast — far worse than
  an error. Always go through `omH`/`omB`/`omD`.
- **Open-Meteo timestamps are naive local wall-clock** (`"2026-08-17T05:00"`, no offset).
  `new Date()` resolves them in the *browser's* zone, shifting every model lookup by hours
  whenever the user is not in the airport's timezone. Always use `omMs(fc, str)`.
- **Open-Meteo NBM `precipitation_probability` is not populated** — 168/168 hours ≈ 0
  (max 2 %). PoP comes from `best_match` via `omB`. ForeFlight shows values we cannot match.
- **NBM `weather_code` never exceeds 2**, so it cannot express "cloudy". Icons derive from
  cloud-cover percentage and only defer to the WMO code at ≥ 45 (fog/precip).
- **IEM `mos.json` model whitelist**: `AVN|GFS|ETA|NAM|NBS|NBE|ECM|LAV|MEX`. `NBH` (hourly
  NBM) is **not** available.
- **NWS `api.weather.gov` gridpoints are not a ceiling source** — `ceilingHeight` and
  `visibility` came back empty for LOX, and PoP is only 6-hourly.

---

## Testing

No test framework. Validate by running the page's real script in Node against live APIs —
extract the `<script>` block, `vm.runInContext` it with stubs for `document` /
`localStorage` / `location` / `history` / `requestAnimationFrame`, then call
`loadAirport(code)` and `renderDayHourly(dayIdx)` and read `daily-hourly`'s `innerHTML`.

Before shipping a change to the forecast layer:

1. **Check `error-box` first.** `_fc` is assigned before rendering, so a render crash still
   leaves stale state that will silently produce a plausible-looking table. A sweep that
   ignores `error-box` will report false passes — PHNL once "passed" while displaying
   all-null data as uniform ☀️ VFR.
2. Sweep ≥ 8 airports × 10 days and assert **zero** blank visibility/temperature cells and
   zero invalid categories (last run: 1,920 cells, all clean).
3. Cover the awkward cases: `KMRY` (fog), `PHNL`/`PANC` (outside CONUS NBM), `KASE`
   (mountain), a bad code such as `ZZZZ` (graceful error).
4. Re-run the `fltCat` boundary cases: ceiling 3000/3001/1000/999/500/499 and vis
   5/5.1/3/2.9/1/0.9.

**Comparing against ForeFlight screenshots:** model runs turn over every few hours, so a
screenshot more than a few hours old is not a valid reference. An early comparison here was
wasted on a 24-hour-old screenshot and produced a marine layer "disagreement" that did not
exist. Confirm the screenshot timestamp is contemporaneous with the data being fetched.
