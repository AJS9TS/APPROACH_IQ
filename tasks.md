# Implementation Plan: Golf Practice Tracker

## Overview

Build a standalone React/TypeScript/Vite single-page application for logging golf practice shots, visualizing shot dispersion via scatter plots, computing per-club KPIs within distance buckets, exporting charts as PNG, and providing ranked practice advice. All data persists in localStorage. Testing uses Vitest with fast-check for property-based tests on pure utility functions.

## Tasks

- [ ] 1. Project scaffolding and core types
  - [ ] 1.1 Scaffold Vite + React + TypeScript project
    - Initialize a new Vite project with the React-TS template in a dedicated directory (e.g., `golf-practice-tracker/`)
    - Install dependencies: `react`, `react-dom`, `chart.js`, `react-chartjs-2`, `chartjs-plugin-annotation`, `uuid`
    - Install dev dependencies: `vitest`, `fast-check`, `@testing-library/react`, `@testing-library/jest-dom`, `jsdom`
    - Configure `vitest` in `vite.config.ts` (jsdom environment)
    - _Requirements: 7.1, 7.2_

  - [ ] 1.2 Define data models and constants
    - Create `src/types/shot.ts` with `Club` type, `Shot` interface, `ClubKPI` interface, `ClubAdvice` interface, `ValidationResult` interface
    - Create `src/constants/distanceBuckets.ts` with `DISTANCE_BUCKETS` record and `CLUBS` array
    - _Requirements: 1.2, 5.4_

- [ ] 2. Data persistence layer
  - [ ] 2.1 Implement `useShotStorage` custom hook
    - Create `src/hooks/useShotStorage.ts`
    - Read shots from localStorage key `"golf-practice-tracker-shots"` on mount
    - Implement `addShot` that generates UUID, appends to array, writes to localStorage
    - Handle corrupted JSON: catch parse errors, set `loadError` message, initialize empty array
    - Handle localStorage unavailable: set `loadError`, operate with in-memory array
    - _Requirements: 2.1, 2.2, 2.3, 1.3_

  - [ ]* 2.2 Write property test for shot persistence round-trip
    - **Property 1: Shot persistence round-trip**
    - Generate arbitrary valid Shot objects using fast-check; verify that storing via addShot and reading back returns the exact shot data
    - **Validates: Requirements 1.3, 2.1, 2.2**

- [ ] 3. Validation logic
  - [ ] 3.1 Implement `validateShotInput` function
    - Create `src/utils/validation.ts`
    - Check all required fields present: club, verticalDistance, horizontalDistance, distanceIntoGreen, date
    - Validate club is a valid Club enum value
    - Validate distances are finite numbers
    - Validate date is a valid ISO 8601 date string (YYYY-MM-DD)
    - Return `{ valid: boolean, errors: string[] }` with field names of invalid/missing fields
    - _Requirements: 1.4, 1.5_

  - [ ]* 3.2 Write property test for validation
    - **Property 2: Validation identifies missing fields**
    - Generate partial shot inputs with random subsets of fields omitted/invalid; verify errors array contains exactly the missing/invalid field names
    - **Validates: Requirements 1.4**

- [ ] 4. KPI calculation utilities
  - [ ] 4.1 Implement bucket filtering and KPI calculation
    - Create `src/utils/kpiCalculations.ts`
    - Implement `filterShotsByBucket(shots, club)`: filter shots matching club AND `distanceIntoGreen <= DISTANCE_BUCKETS[club]`
    - Implement `calculateKPIs(shots, club)`: filter by bucket, compute `avgAbsVerticalDistance`, `avgAbsHorizontalDistance`, `avgHypotenuseDistance`; return null if no qualifying shots
    - Implement `calculateHypotenuse(shot)`: `Math.sqrt(verticalDistance² + horizontalDistance²)`
    - _Requirements: 5.1, 5.2, 5.3, 5.4_

  - [ ]* 4.2 Write property test for bucket filtering
    - **Property 5: Distance bucket filtering correctness**
    - Generate arbitrary shots across clubs; verify filterShotsByBucket returns exactly shots matching club AND distance threshold
    - **Validates: Requirements 5.4**

  - [ ]* 4.3 Write property test for KPI calculation
    - **Property 6: KPI calculation correctness**
    - Generate non-empty sets of shots within a bucket; verify computed averages match manual calculation of mean(|v|), mean(|h|), mean(sqrt(v²+h²))
    - **Validates: Requirements 5.1, 5.2, 5.3**

- [ ] 5. Dispersion and advice utilities
  - [ ] 5.1 Implement dispersion and advice functions
    - Create `src/utils/adviceCalculations.ts`
    - Implement `standardDeviation(values)`: population std dev formula
    - Implement `calculateDispersion(shots, club)`: filter by bucket, compute hypotenuses, return population std dev; return null if fewer than 3 qualifying shots
    - Implement `generateAdvice(shots)`: for each club, calculate dispersion (skip clubs with < 3 qualifying shots), sort descending by dispersion
    - _Requirements: 6.1, 6.2, 6.3, 6.4_

  - [ ]* 5.2 Write property test for dispersion calculation
    - **Property 7: Dispersion calculation correctness**
    - Generate sets of 3+ shots within a bucket; verify calculated dispersion equals population standard deviation of hypotenuse distances
    - **Validates: Requirements 6.1**

  - [ ]* 5.3 Write property test for advice ranking order
    - **Property 8: Advice ranking is sorted by dispersion descending**
    - Generate multiple clubs each with 3+ qualifying shots; verify output is sorted highest-to-lowest dispersion
    - **Validates: Requirements 6.2**

  - [ ]* 5.4 Write property test for advice exclusion rule
    - **Property 9: Clubs with fewer than 3 qualifying shots excluded from advice**
    - Generate clubs with varying shot counts; verify clubs with < 3 qualifying shots are absent from the ranking
    - **Validates: Requirements 6.4**

- [ ] 6. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Shot entry form component
  - [ ] 7.1 Implement `ShotEntryForm` component
    - Create `src/components/ShotEntryForm.tsx`
    - Render form with: Club dropdown (all 8 clubs), numeric inputs for verticalDistance/horizontalDistance/distanceIntoGreen, date input defaulting to today
    - On submit: call `validateShotInput`, if valid call `addShot` from the storage hook, if invalid display inline error messages
    - Allow negative values for vertical and horizontal distance inputs
    - Clear form on successful submission
    - _Requirements: 1.1, 1.2, 1.4, 1.5, 1.6_

- [ ] 8. Scatter plot chart view
  - [ ] 8.1 Implement `ScatterPlotView` component
    - Create `src/components/ScatterPlotView.tsx`
    - Render club selector dropdown to filter shots by club
    - Render Chart.js Scatter chart via `react-chartjs-2` with horizontal distance on X-axis, vertical distance on Y-axis
    - Mark pin at origin (0,0) using a distinct dataset point or annotation plugin
    - Hold a ref to the Chart.js instance for export
    - Show "No data recorded for this club" when selected club has no shots
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ] 8.2 Implement chart export functionality
    - Create `src/utils/chartExport.ts` with `exportChartAsPNG(chartRef, club)` function
    - Use `chartRef.current.toBase64Image()` to get PNG data URL
    - Create temporary anchor element with `download` attribute set to `{club}_scatter_plot.png`
    - Trigger click and remove element
    - Add "Download PNG" button to `ScatterPlotView` that calls this function
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ]* 8.3 Write property test for club selection filtering
    - **Property 3: Club selection filters shots correctly**
    - Generate shots across multiple clubs; verify scatter data for a selected club contains only that club's shots
    - **Validates: Requirements 3.4**

  - [ ]* 8.4 Write property test for export filename pattern
    - **Property 4: Export filename follows naming pattern**
    - For each Club value, verify the generated filename equals `"{Club}_scatter_plot.png"`
    - **Validates: Requirements 4.3**

- [ ] 9. KPI dashboard view
  - [ ] 9.1 Implement `KPIDashboard` component
    - Create `src/components/KPIDashboard.tsx`
    - For each club with qualifying data, render a card showing: club name, avg absolute vertical distance, avg absolute horizontal distance, avg hypotenuse distance, shot count
    - For clubs with no qualifying shots in their bucket, show "Insufficient data" message
    - Use `calculateKPIs` utility for each club
    - _Requirements: 5.1, 5.2, 5.3, 5.5, 5.6_

- [ ] 10. Advice panel view
  - [ ] 10.1 Implement `AdvicePanel` component
    - Create `src/components/AdvicePanel.tsx`
    - Use `generateAdvice` utility to get ranked list
    - Display ranked clubs with dispersion values (highest first = needs most practice)
    - Show "Need more data" note for clubs with fewer than 3 qualifying shots
    - _Requirements: 6.1, 6.2, 6.3, 6.4_

- [ ] 11. App shell and tab navigation
  - [ ] 11.1 Implement tab navigation and app shell
    - Create `src/components/TabNavigation.tsx` with tabs: "Log Shot", "Charts", "KPIs", "Advice"
    - Update `src/App.tsx` to render `TabNavigation` and conditionally render the active view component based on selected tab
    - Default to "Log Shot" tab on load
    - Wire `useShotStorage` hook at the App level and pass shots/addShot to child components
    - _Requirements: 7.1, 7.2_

- [ ] 12. Seed data and integration
  - [ ] 12.1 Create seed data utility for development/demo
    - Create `src/utils/seedData.ts` with a function that generates realistic sample shots across all 8 clubs (at least 5 shots per club with varied distances)
    - Add a "Load Sample Data" button (visible in dev mode or as a UI option) that populates localStorage with the seed data
    - Ensure seed data covers different distance buckets so KPIs, charts, and advice all render meaningful content
    - _Requirements: 2.1, 5.4, 6.4_

  - [ ] 12.2 Wire all components together and verify end-to-end flow
    - Ensure shot entry persists and immediately reflects in Charts, KPIs, and Advice tabs
    - Ensure localStorage error handling displays appropriate banner messages
    - Verify scatter plot renders correctly with pin at origin
    - Verify export generates correctly named PNG file
    - Add basic responsive styling for mobile and desktop viewports
    - _Requirements: 2.2, 2.3, 3.3, 4.3, 7.3_

- [ ] 13. Final checkpoint
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties from the design document
- Unit tests validate specific examples and edge cases
- All pure utility functions (validation, KPI calculation, dispersion, bucket filtering) are primary PBT targets
- UI components are tested via example-based tests with Testing Library

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["2.1", "3.1", "4.1", "5.1"] },
    { "id": 3, "tasks": ["2.2", "3.2", "4.2", "4.3", "5.2", "5.3", "5.4"] },
    { "id": 4, "tasks": ["7.1", "8.1", "8.2", "9.1", "10.1"] },
    { "id": 5, "tasks": ["8.3", "8.4", "11.1", "12.1"] },
    { "id": 6, "tasks": ["12.2"] }
  ]
}
```
