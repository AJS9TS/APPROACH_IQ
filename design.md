# Design Document: APPROACH IQ

## Overview

APPROACH IQ is a standalone single-file HTML web application for golfers to log practice shots, visualise shot dispersion on a golf green canvas, compute per-club KPIs with benchmark comparisons, export charts as images, track handicap index over time, and receive ranked coaching advice. All data persists in browser localStorage with no backend dependencies. The app is deployed as a Progressive Web App (PWA) via GitHub Pages.

## Architecture

The application is a single `index.html` file containing all HTML, CSS, and JavaScript inline. No build step, bundler, or framework is required.

```
┌─────────────────────────────────────────────────────────────┐
│                      index.html                              │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  <style> — All CSS (responsive, dark/light themes)     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  <body> — Semantic HTML structure                      │ │
│  │  ├── Hero / Branding                                   │ │
│  │  ├── Tab Navigation (desktop) / Bottom Nav (mobile)    │ │
│  │  ├── Page Layout (sidebar ads + main content)          │ │
│  │  │   ├── Log View (Manual / Click-to-Place / Quick)    │ │
│  │  │   ├── Charts View (Green Canvas + filters)          │ │
│  │  │   ├── KPIs View (Per-club metric cards)             │ │
│  │  │   └── Coach's Corner (Practice plans + rankings)    │ │
│  │  ├── Handicap Tracker                                  │ │
│  │  ├── Reviews Carousel                                  │ │
│  │  └── Footer                                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  <script> — All application logic (vanilla JS)         │ │
│  │  ├── Constants & Configuration                         │ │
│  │  ├── Data Layer (localStorage read/write)              │ │
│  │  ├── Validation                                        │ │
│  │  ├── KPI & Dispersion Calculations                     │ │
│  │  ├── Tab Navigation                                    │ │
│  │  ├── Entry Modes (Manual, Click-to-Place, Quick Log)   │ │
│  │  ├── Canvas Rendering (Green, Entry Green)             │ │
│  │  ├── Chart.js Integration (Trends, Handicap)           │ │
│  │  ├── Coaching Algorithm                                │ │
│  │  ├── Handicap Tracker                                  │ │
│  │  ├── UI Utilities (Toast, Theme, Carousel, Weather)    │ │
│  │  └── Initialisation                                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  service-worker.js  — Caching for offline PWA support       │
│  manifest.json      — PWA manifest                          │
│  icons/             — SVG icon + icon generation page        │
└─────────────────────────────────────────────────────────────┘
```

### External Dependencies (CDN)

| Dependency | Purpose |
|------------|---------|
| Chart.js 4.4.7 | Trend line charts (KPI cards) and Handicap Index chart |
| Google Fonts (Inter, Space Grotesk) | Typography |

### Data Flow

1. **Entry**: User submits shot (manual/click/quick) → validated → persisted to localStorage → stats updated
2. **Display**: Tab switched → reads shots from localStorage → renders view (canvas or Chart.js)
3. **Compute**: Shots filtered by club + distance bucket → KPIs/dispersion calculated in pure functions
4. **Export**: Green canvas → `canvas.toDataURL('image/png')` → download via anchor element
5. **Offline**: Service worker caches HTML + CDN assets on first load → serves from cache when offline

---

## Data Models

### Shot

```javascript
{
  id: string,               // UUID (crypto.randomUUID or fallback)
  club: string,             // One of CLUBS array
  verticalDistance: number,  // yards, +long / -short
  horizontalDistance: number,// yards, +right / -left
  date: string              // 'YYYY-MM-DD'
}
```

### Handicap Entry

```javascript
{
  hcp: number,   // Handicap index value (0-54)
  date: string   // 'YYYY-MM-DD'
}
```

### Constants

```javascript
const CLUBS = ['Wedge', '9 Iron', '8 Iron', '7 Iron', '6 Iron', '5 Iron', '3 Wood', 'Driver'];

const DISTANCE_BUCKETS = {
  'Wedge': 10, '9 Iron': 13, '8 Iron': 15, '7 Iron': 25,
  '6 Iron': 30, '5 Iron': 30, '3 Wood': 50, 'Driver': 55
};

const BENCHMARKS = {
  'Wedge': { pro: 2.5, good: 4, avg: 6 },
  '9 Iron': { pro: 3.5, good: 5.5, avg: 8 },
  // ... per club
};
```

### localStorage Schema

| Key | Value | Purpose |
|-----|-------|---------|
| `golf-practice-tracker-shots` | `Shot[]` (JSON) | All shot data |
| `approachiq-handicap-history` | `HcpEntry[]` (JSON) | Handicap index entries |
| `approachiq-theme` | `'light'` or `'dark'` | Theme preference |

---

## Entry Modes

### Manual Entry
User types Vertical_Distance and Horizontal_Distance as signed numbers. Standard form submission with validation.

### Click-to-Place
User taps/clicks an interactive green canvas. Pixel coordinates are converted to yard distances using the selected club's Distance_Bucket as the scale factor. Coordinates are rounded to the nearest whole yard.

### Quick Log
Designed for on-course speed. User taps a club button, then taps a zone on a circular green. Coordinates are randomised within the zone's radius to produce realistic dispersion data:

| Zone | Centre (v, h) | Scatter Radius |
|------|---------------|----------------|
| Centre (Pin) | (0, 0) | 3 yds (min 0.5 from pin) |
| Long | (8, 0) | 2.5 yds |
| Short | (-8, 0) | 2.5 yds |
| Left | (0, -8) | 2.5 yds |
| Right | (0, 8) | 2.5 yds |
| Long-Left | (6, -6) | 2.5 yds |
| Long-Right | (6, 6) | 2.5 yds |
| Short-Left | (-6, -6) | 2.5 yds |
| Short-Right | (-6, 6) | 2.5 yds |
| Missed Green | (18, 0) | 5 yds |

Randomisation uses uniform circular distribution (polar method with `sqrt(random)` for uniform area coverage).

---

## Calculation Engine

### Hypotenuse Distance
```javascript
hypotenuse = Math.sqrt(verticalDistance² + horizontalDistance²)
```

### Distance Bucket Filtering
A shot qualifies for KPI analysis when its hypotenuse distance is within the club's bucket threshold.

### KPI Metrics (per club, within bucket)
- **Avg Absolute Vertical**: `mean(|verticalDistance|)`
- **Avg Absolute Horizontal**: `mean(|horizontalDistance|)`
- **Avg Proximity (Hypotenuse)**: `mean(hypotenuse)`
- **In-Bucket Rate**: qualifying shots / total shots for that club
- **Performance Score**: 0-100% based on proximity relative to benchmark ceiling (2x amateur average)

### Dispersion (Standard Deviation)
Population standard deviation of hypotenuse distances for qualifying shots. Requires minimum 3 shots.

```javascript
mean = sum(hypotenuses) / n
variance = sum((h - mean)² for h in hypotenuses) / n
dispersion = sqrt(variance)
```

### Improvement Opportunity Score
```javascript
score = (gap_to_best_club * 0.6) + (dispersion * 0.4)
```

Used to rank clubs by practice priority.

### Miss Pattern Detection
Compares average vertical bias vs average horizontal bias:
- If |avgV| > |avgH| × 1.5 → vertical dominant (long or short)
- If |avgH| > |avgV| × 1.5 → horizontal dominant (left or right)
- Otherwise → scattered

---

## Canvas Rendering

The app uses two custom canvas-drawn greens (no Chart.js for these):

### Charts Tab Green
- Circular green with outer edge = club's Distance_Bucket
- 5 concentric contour rings at evenly spaced intervals, labelled with yard distances
- Flag (white pole, red flag) and hole at centre
- Shot markers plotted at scaled coordinates with glow effect
- Responsive: re-renders on window resize

### Entry Green (Click-to-Place)
- Same visual style as Charts green
- Interactive: click/touch places a ball marker
- Dashed line from hole to marker
- Coordinate readout updates on hover (desktop) and on tap (mobile)

### Quick Log Zone Green
- CSS-based circular layout with positioned zone buttons
- Dashed 50% contour ring indicating close zone boundary
- Pin emoji at centre

---

## Chart.js Usage

Chart.js is used for two chart types only:

1. **KPI Trend Lines** — mini sparkline charts on each KPI card showing monthly average proximity over time (line chart, no legend, minimal axes)
2. **Handicap Index Chart** — full line chart with colour-coded segments (green = improving, red = worsening, orange = stable), gradient fill, and responsive aspect ratio

---

## Coaching Algorithm (Coach's Corner)

1. Filter shots by rolling window (default 30 days)
2. For each club with 3+ qualifying shots, calculate: avg proximity, avg vertical bias, avg horizontal bias, dispersion
3. Find best performing club (lowest avg proximity)
4. Calculate Improvement Opportunity Score for each club
5. Sort by score descending → top 3 become Priority 1/2/3 recommendations
6. Generate natural-language coaching reasons based on miss pattern
7. Generate up to 3 practice session plans using different club groupings

### Practice Impact Analysis
Correlates handicap changes with dispersion changes between the two most recent handicap entries. Requires 2+ handicap entries and 3+ shots per club within the analysis window.

---

## Responsive Design

| Breakpoint | Behaviour |
|------------|-----------|
| > 1200px | Full layout with sidebar ads (3 per side) |
| 769px – 1200px | Single column, no sidebar ads |
| ≤ 768px | Bottom navigation bar, reduced spacing, mobile-optimised canvas/carousel |
| ≤ 600px | Single-column form grid, smaller fonts |

### Mobile-Specific
- Fixed bottom nav with iOS safe area inset support
- Touch events on all canvases
- Carousel uses 100% slide width (no gap) for full visibility
- Date inputs constrained to container width
- Pin place canvas scales to fit viewport

---

## PWA Architecture

- **manifest.json**: Declares standalone display mode, theme colour, icon references
- **service-worker.js**: Cache-first strategy for app shell + CDN assets
- **Icons**: Generated at runtime via canvas (180px apple-touch-icon, 192px PWA icon) as base64 data URIs — no static PNG files required
- **Offline**: Full functionality after first load (all data is local)

---

## Theme System

Two themes controlled by a CSS custom property override on `body.light`:
- **Dark (default)**: `#030711` background, glassmorphism cards, gradient accents
- **Light**: `#f8faf9` background with green gradient, adjusted borders and shadows

Theme preference persisted in localStorage and applied on page load.

---

## Error Handling

| Scenario | Behaviour |
|----------|-----------|
| localStorage unavailable | Console error, empty dataset |
| Corrupted JSON in localStorage | Console error, empty array |
| Invalid form input | Inline validation error message |
| No shots for selected club | "No data" placeholder message |
| < 3 shots in bucket (advice) | Excluded from ranking, warning shown |
| Geolocation denied (weather) | Widget shows "Enable location" message |
| Weather API failure | Widget shows "Weather unavailable" |

---

## Security Considerations

- No backend, no authentication, no network requests except weather API and CDN loads
- All data stored client-side only
- No user PII collected beyond what they voluntarily enter
- CDN resources loaded over HTTPS
- Service worker scoped to application origin only
