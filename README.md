# DA40 Performance Calculator

**Live:** https://gopilot.blog/da40-performance-calculator/

A single-file, AFM-accurate performance calculator for the **Diamond DA40-180** (Lycoming IO-360-M1A, G1000 NXi). Every performance chart in Section 5 of the AFM was digitized point-by-point and verified against the AFM's own worked examples. Enter a METAR (or fetch one live), set your weight and cruise altitude, and get climb / cruise / descent / runway numbers plus a ForeFlight-ready performance profile.

Built for real flight planning at high-density-altitude airports like KPVU (field elev 4,497 ft).

---

## Features

- **Live METAR** — fetch by ICAO through GoPilot's own API (`api.gopilot.blog`, a Cloudflare Worker proxying the NOAA Aviation Weather Data API), with a 5-proxy public fallback and manual raw-METAR paste that auto-parses ICAO, wind, temperature, and altimeter
- **Mass slider** — 850–1200 kg with synced kg/lb inputs, color-coded Utility / Normal / MÄM 40-227 zones, and quick-jump presets
- **Three atmospheric states** — Field, Cruise, and Midpoint PA/DA/OAT computed separately; the field ISA deviation is preserved aloft via the standard lapse rate so you only enter OAT once
- **Climb** — Take-off Climb (Flaps T/O) or Cruise Climb (Flaps UP) modes; ROC from the AFM chart evaluated at the **climb midpoint**, mass-corrected from the chart nomograph; climb IAS interpolated by weight per §4A.3.8
- **Cruise** — TAS from the §5.3.9 chart at **cruise density altitude**, MP and fuel flow from the §5.3.2 engine tables at cruise pressure altitude, ISA power correction applied (±3 % per ±15 °C per the AFM note)
- **Descent** — glidepath-based: `ROD = GS × (6076.12 / 60) × tan(γ)`, default 3.0° (ILS standard), adjustable in 0.1° steps; descent speed selectable as IAS or direct TAS
- **Wind** — course input gives headwind/tailwind and left/right crosswind components; ground speed, time, and distance for every phase, including top-of-descent distance from the field
- **Take-off & landing distance** — ground roll and 50 ft obstacle figures from §5.3.6 / §5.3.10 / §5.3.11, with weight and wind corrections and a Flaps LDG / Flaps UP toggle
- **ForeFlight profile block** — the eight numbers ForeFlight wants (climb/cruise/descent TAS, FF, FPM) in one copy-ready strip

## Data Source

All performance data comes from one document:

> **Diamond DA 40-180 Airplane Flight Manual, Doc. # 6.01.01-E, Revision 10 (18-Sep-2023)**

| AFM section | Content | Digitized grid |
|---|---|---|
| 5.3.1 | Airspeed calibration (IAS→CAS) | Flaps UP / T/O / LDG curves |
| 5.3.2 | Engine performance — MP + fuel flow | 4 power settings × 4 RPM × PA to 17,000 ft |
| 5.3.6 | Take-off distance | 6 PA × 5 OAT |
| 5.3.7 | Take-off climb ROC | 6 PA × 5 OAT @ 1000 kg ref |
| 5.3.8 | Cruise climb ROC | 6 PA × 5 OAT @ 1000 kg ref |
| 5.3.9 | Cruising TAS | 8 DA × 4 power settings |
| 5.3.10 | Landing distance — Flaps LDG | 6 PA × 5 OAT |
| 5.3.11 | Landing distance — Flaps UP | 6 PA × 5 OAT |

Values are looked up with bilinear interpolation between grid points. Out-of-envelope cells (e.g. 75 % power above ~6,000 ft DA) return "out of env." rather than extrapolating.

## Verification

Every digitized chart is anchored to the AFM's own worked example before use:

| AFM worked example | AFM result | Calculator | Error |
|---|---|---|---|
| §5.3.7 — PA 0, 15 °C, 1000 kg | 1160 fpm | 1160 fpm | 0.0 % |
| §5.3.8 — PA 0, 15 °C, 1000 kg | 1050 fpm | 1050 fpm | 0.0 % |
| §5.3.9 — PA 5000, 15 °C, 55 % | 118 KTAS | 117.4 KTAS | −0.5 % |
| §5.3.6 — PA 2000, 15 °C, 1000 kg, HW 10 | 558 / 985 ft | 553 / 979 ft | −0.9 / −0.6 % |
| §5.3.10 — PA 2000, 15 °C, 1000 kg, HW 10 | 624 / 1329 ft | 623 / 1327 ft | −0.2 / −0.2 % |
| §5.3.11 — PA 4000, 8 °C, 1000 kg, HW 8 | 886 / 1903 ft | 887 / 1907 ft | +0.1 / +0.2 % |

Climb fuel flow (not tabulated in the AFM for full-throttle climb) is calibrated to simulator measurements at PA 4,500 ft: **15.1 gph @ 2700 RPM** and **13.6 gph @ 2400 RPM**, with a 3 %-per-1000-ft full-throttle decay.

## Core Formulas

```
PA  = elev + (29.92 − altim) × 1000
ISA = 15 − 1.98 × PA/1000                      [°C]
DA  = PA + 120 × (OAT − ISA)
σ   = (1 − 6.875586e-6 × DA)^4.2561
TAS = CAS / √σ
k   = 1 − 0.03 × (ISAdev / 15)                 [AFM §5.3.2 power correction]
HW  = WS × cos(WindDir − Course)               [+ head / − tail]
XW  = WS × sin(WindDir − Course)               [+ right / − left]
ROD = GS × (6076.12 / 60) × tan(γ)             [1 NM = 6076.12 ft]
```

## Usage

- **Online:** open the live link, paste or fetch a METAR, adjust weight/altitude/power — everything recomputes instantly.
- **Offline:** download `index.html` — it is fully self-contained (no build, no dependencies beyond Google Fonts). METAR paste works with no network at all.
- On mobile, "Add to Home Screen" makes it behave like an app.

## Architecture

```
gopilot.blog/da40-performance-calculator/   ← this repo (GitHub Pages)
api.gopilot.blog                            ← Cloudflare Worker METAR proxy
    └→ aviationweather.gov Data API         (60 s edge cache per ICAO)
```

The METAR proxy exists because aviationweather.gov does not send CORS headers, so browsers cannot call it directly. The Worker adds `Access-Control-Allow-Origin` and caches each ICAO for 60 seconds to respect NOAA's rate-limit guidance.

## Limitations

- Bilinear interpolation between chart grid points — accuracy is anchor-verified at the worked examples, expect roughly ±5 % elsewhere
- Runway distances assume a paved, dry, level runway; apply AFM safety factors for grass, wet, or contaminated surfaces
- Wheel fairings assumed installed (−5 % cruise TAS if removed, per AFM §5.3.9 caution)
- Wind corrections for runway distance are a linearization of the AFM nomograph — for strong winds, read the chart directly

## Disclaimer

**Planning reference only.** This tool is not a substitute for the AFM. Always cross-check performance figures with the AFM charts in the aircraft before flight. AFM data remains the property of Diamond Aircraft Industries; this project digitizes it solely for personal training and flight-planning use.

## Version History

- **v6.0** — own METAR API (`api.gopilot.blog`) as primary source; public proxies demoted to fallback
- **v5.x** — take-off & landing distances, glidepath-based descent, IAS/TAS descent toggle, sim-calibrated climb fuel flow, exact NM→ft conversion
- **v4** — cruise-altitude separation, wind components, midpoint climb/descent averaging, per-phase GS/time/distance
- **v3** — METAR paste parser + CORS-proxy fetch
- **v1–v2** — AFM Rev 10 chart digitization, continuous mass slider

---

Built by **Philip Baek** · UVU Aviation Science · [gopilot.blog](https://gopilot.blog)
