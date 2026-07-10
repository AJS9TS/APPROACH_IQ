# Design Document: Golf Practice Tracker

## Overview

The Golf Practice Tracker is a single-page React/TypeScript application built with Vite. It allows golfers to log practice shots, visualize shot dispersion via scatter plots, calculate KPIs per club within distance buckets, export charts as PNG images, and receive ranked advice on which clubs need the most practice. All data is persisted in localStorage with no backend dependencies.

## Architecture

The application follows a tab-based single-page architecture with all state managed client-side and localStorage as the persistence layer.

```
┌─────────────────────────────────────────────────────┐
│                   App Shell                          │
│  ┌───────┬───────────┬────────┬──────────┐          │
│  │ Entry │  Charts   │  KPIs  │  Advice  │  (tabs)  │
│  └───────┴───────────┴────────┴──────────┘          │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │             Active View                      │    │
│  │                                             │    │
│  │  (ShotEntryForm | ScatterPlotView |         │    │
│  │   KPIDashboard | AdvicePanel)               │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │         Data Layer (hooks + utils)           │    │
│  │  useShotStorage → localStorage              │    │
│  │  calculateKPIs / calculateDispersion         │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### Data Flow

1. **Entry**: User submits shot → validated → persisted to localStorage → state updated
2. **Display**: Component mounts → reads shots from localStorage via hook → renders view
3. **Compute**: Shots filtered by club + distance bucket → KPIs/dispersion calculated in pure functions
4. **Export**: Chart canvas → `toDataURL('image/png')` → download via anchor element

---

## Data Models

### Core Types

```typescript
// Club enumeration
type Club =
  | 'Wedge'
  | '9 Iron'
  | '8 Iron'
  | '7 Iron'
  | '6 Iron'
  | '5 Iron'
  | '3 Wood'
  | 'Driver';

// A single recorded practice shot
interface Shot {
  id: string;           // UUID for unique identification
  club: Club;
  verticalDistance: number;    // yards, positive = long, negative = short
  horizontalDistance: number;  // yards, positive = right, negative = left
  distanceIntoGreen: number;  // yards, always positive
  date: string;               // ISO 8601 date string (YYYY-MM-DD)
}

// Distance bucket thresholds per club (in yards)
const DISTANCE_BUCKETS: Record<Club, number> = {
  'Wedge': 10,
  '9 Iron': 13,
  '8 Iron': 15,
  '7 Iron': 25,
  '6 Iron': 30,
  '5 Iron': 30,
  '3 Wood': 50,
  'Driver': 55,
};

// KPI results for a single club
interface ClubKPI {
  club: Club;
  avgAbsVerticalDistance: number;
  avgAbsHorizontalDistance: number;
  avgHypotenuseDistance: number;
  shotCount: number;          // number of qualifying shots in bucket
}

// Advice entry for a single club
interface ClubAdvice {
  club: Club;
  dispersion: number;         // standard deviation of hypotenuse distances
  shotCount: number;
}

// Validation result
interface ValidationResult {
  valid: boolean;
  errors: string[];           // list of field names with issues
}
```

### localStorage Schema

All shot data is stored under a single key:

```typescript
// Key: "golf-practice-tracker-shots"
// Value: JSON-serialized Shot[]

interface StorageSchema {
  key: 'golf-practice-tracker-shots';
  value: Shot[];  // array of Shot objects
}
```

On load, the app reads and parses this key. If parsing fails (corrupted data), the app shows an error and initializes with an empty array.

---

## Components and Interfaces

### Component Hierarchy

```
App
├── TabNavigation
│   ├── Tab: "Log Shot"
│   ├── Tab: "Charts"
│   ├── Tab: "KPIs"
│   └── Tab: "Advice"
├── ShotEntryForm
│   ├── ClubSelector (dropdown)
│   ├── NumericInput (verticalDistance)
│   ├── NumericInput (horizontalDistance)
│   ├── NumericInput (distanceIntoGreen)
│   ├── DateInput (defaults to today)
│   ├── SubmitButton
│   └── ValidationErrors
├── ScatterPlotView
│   ├── ClubSelector (filter dropdown)
│   ├── ScatterChart (react-chartjs-2 Scatter)
│   └── ExportButton
├── KPIDashboard
│   ├── ClubKPICard (one per club with data)
│   │   ├── ClubName
│   │   ├── AvgAbsVertical
│   │   ├── AvgAbsHorizontal
│   │   └── AvgHypotenuse
│   └── InsufficientDataMessage
└── AdvicePanel
    ├── RankedClubList
    │   └── ClubAdviceRow (club name + dispersion value)
    └── InsufficientDataMessage (for clubs with < 3 shots)
```

---

## Interfaces & Key Functions

### Storage Hook

```typescript
function useShotStorage(): {
  shots: Shot[];
  addShot: (shot: Omit<Shot, 'id'>) => void;
  loadError: string | null;
}
```

Wraps localStorage read/write with error handling. Generates UUID for `id` on add.

### Validation

```typescript
function validateShotInput(input: Partial<Shot>): ValidationResult;
```

Checks: all required fields present, club is valid enum value, distances are valid numbers, date is valid ISO string.

### KPI Calculation

```typescript
function filterShotsByBucket(shots: Shot[], club: Club): Shot[];
function calculateKPIs(shots: Shot[], club: Club): ClubKPI | null;
```

- `filterShotsByBucket`: Returns shots for the given club where `distanceIntoGreen <= DISTANCE_BUCKETS[club]`.
- `calculateKPIs`: Filters shots by bucket, then computes averages. Returns `null` if no qualifying shots.

### Dispersion & Advice

```typescript
function calculateHypotenuse(shot: Shot): number;
function calculateDispersion(shots: Shot[], club: Club): number | null;
function generateAdvice(shots: Shot[]): ClubAdvice[];
```

- `calculateHypotenuse`: `Math.sqrt(v² + h²)` for a single shot.
- `calculateDispersion`: Standard deviation of hypotenuse distances for qualifying shots. Returns `null` if fewer than 3 qualifying shots.
- `generateAdvice`: For each club, calculates dispersion (excluding clubs with < 3 qualifying shots), sorts descending by dispersion value.

### Chart Export

```typescript
function exportChartAsPNG(chartRef: React.RefObject<ChartJS>, club: Club): void;
```

Uses `chartRef.current.toBase64Image()` to get the PNG data, creates a temporary anchor element with the download attribute set to `{club}_scatter_plot.png`, triggers click, and removes the element.

---

## Chart Rendering Approach

Using `react-chartjs-2` with Chart.js:

```typescript
import { Scatter } from 'react-chartjs-2';
import { ChartOptions } from 'chart.js';

// Chart configuration for a given club's shots
function getScatterConfig(clubShots: Shot[]): {
  data: { datasets: [{ data: Array<{ x: number; y: number }> }] };
  options: ChartOptions<'scatter'>;
} {
  return {
    data: {
      datasets: [{
        label: 'Shots',
        data: clubShots.map(s => ({
          x: s.horizontalDistance,
          y: s.verticalDistance,
        })),
        backgroundColor: 'rgba(59, 130, 246, 0.6)',
      }],
    },
    options: {
      scales: {
        x: { title: { display: true, text: 'Horizontal Distance (yards)' } },
        y: { title: { display: true, text: 'Vertical Distance (yards)' } },
      },
      plugins: {
        annotation: {
          annotations: {
            pin: { type: 'point', xValue: 0, yValue: 0 },  // origin marker
          },
        },
      },
    },
  };
}
```

The pin at (0, 0) is rendered using the chartjs-plugin-annotation or as an additional dataset point with distinct styling.

---

## Export Mechanism

1. Each `ScatterPlotView` holds a `ref` to the Chart.js canvas instance.
2. On "Download PNG" click:
   - Call `chartRef.current.toBase64Image()` → returns a `data:image/png;base64,...` string.
   - Create an `<a>` element with `href` set to the data URL.
   - Set `download` attribute to `{club}_scatter_plot.png` (club name as-is, spaces included).
   - Programmatically click the anchor, then remove it from the DOM.
3. No server interaction required.

---

## Advice Algorithm

### Dispersion Calculation

For a given club:

1. Filter all shots for that club where `distanceIntoGreen <= DISTANCE_BUCKETS[club]`.
2. If fewer than 3 qualifying shots, exclude from ranking.
3. For each qualifying shot, compute `hypotenuse = sqrt(verticalDistance² + horizontalDistance²)`.
4. Compute the standard deviation of the hypotenuse values:
   - `mean = sum(hypotenuses) / n`
   - `variance = sum((h - mean)² for h in hypotenuses) / n`
   - `dispersion = sqrt(variance)` (population standard deviation)
5. Rank clubs by dispersion descending — highest dispersion = most inconsistent = needs most practice.

### Standard Deviation Formula (Population)

```typescript
function standardDeviation(values: number[]): number {
  const n = values.length;
  if (n === 0) return 0;
  const mean = values.reduce((sum, v) => sum + v, 0) / n;
  const variance = values.reduce((sum, v) => sum + (v - mean) ** 2, 0) / n;
  return Math.sqrt(variance);
}
```

---

## Error Handling

| Scenario | Behavior |
|----------|----------|
| localStorage unavailable | Show error banner, operate with empty in-memory array |
| Corrupted JSON in localStorage | Show error message, reset to empty dataset |
| Invalid form input | Inline validation errors listing missing/invalid fields |
| No shots for selected club | Show "No data recorded for this club" in chart view |
| Fewer than 3 shots in bucket (advice) | Exclude from ranking, show "Need more data" note |
| No qualifying shots in bucket (KPIs) | Show "Insufficient data" message for that club |

---

## Testing Strategy

- **Unit tests**: Verify specific examples (form defaults, chart axis config, origin marker) and edge cases (corrupted localStorage, empty bucket, zero shots for a club).
- **Property-based tests**: Validate universal properties for pure computation functions (KPI calculations, dispersion, bucket filtering, validation logic, persistence round-trips). Minimum 100 iterations per property.
- **Test framework**: Vitest with fast-check for property-based testing.
- **Coverage focus**: Pure utility functions (calculation, filtering, validation) are the primary PBT targets. UI rendering is covered by example-based tests.

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Shot persistence round-trip

*For any* valid Shot object, storing it via `addShot` and then reading all shots from localStorage SHALL return an array containing that exact shot (matching all field values).

**Validates: Requirements 1.3, 2.1, 2.2**

### Property 2: Validation identifies missing fields

*For any* subset of required Shot fields that are omitted or invalid, the `validateShotInput` function SHALL return a `ValidationResult` where `valid` is `false` and `errors` contains exactly the names of the missing/invalid fields.

**Validates: Requirements 1.4**

### Property 3: Club selection filters shots correctly

*For any* set of shots across multiple clubs and any selected club, the scatter plot data SHALL contain only shots belonging to the selected club.

**Validates: Requirements 3.4**

### Property 4: Export filename follows naming pattern

*For any* Club value, the exported PNG filename SHALL equal `"{Club}_scatter_plot.png"` where `{Club}` is the exact club name string.

**Validates: Requirements 4.3**

### Property 5: Distance bucket filtering correctness

*For any* club and any set of shots, `filterShotsByBucket` SHALL return exactly those shots where the club matches AND `distanceIntoGreen <= DISTANCE_BUCKETS[club]`, and no others.

**Validates: Requirements 5.4**

### Property 6: KPI calculation correctness

*For any* non-empty set of shots within a club's distance bucket, the calculated KPIs SHALL equal: `avgAbsVerticalDistance = mean(|verticalDistance|)`, `avgAbsHorizontalDistance = mean(|horizontalDistance|)`, and `avgHypotenuseDistance = mean(sqrt(verticalDistance² + horizontalDistance²))`.

**Validates: Requirements 5.1, 5.2, 5.3**

### Property 7: Dispersion calculation correctness

*For any* set of 3 or more shots within a club's distance bucket, the calculated dispersion SHALL equal the population standard deviation of their hypotenuse distances (`sqrt(verticalDistance² + horizontalDistance²)`).

**Validates: Requirements 6.1**

### Property 8: Advice ranking is sorted by dispersion descending

*For any* set of clubs each with 3 or more qualifying shots, the advice ranking SHALL be ordered from highest dispersion to lowest dispersion, with no inversions.

**Validates: Requirements 6.2**

### Property 9: Clubs with fewer than 3 qualifying shots excluded from advice

*For any* club with fewer than 3 shots within its distance bucket, that club SHALL NOT appear in the advice ranking output.

**Validates: Requirements 6.4**
