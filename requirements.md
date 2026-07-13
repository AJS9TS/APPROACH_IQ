# Requirements Document

## Introduction

ApproachIQ is a standalone single-file HTML web application that allows golfers to log practice shots, visualize shot dispersion on a golf green canvas, compute per-club KPIs with benchmark comparisons, export charts as images, and receive ranked advice on which clubs need the most practice work. The application uses localStorage for persistence with no cloud backend, and is designed for deployment via GitHub Pages or any static hosting.

## Glossary

- **Tracker**: The ApproachIQ web application
- **Shot**: A single recorded practice shot containing club, vertical distance, horizontal distance, distance into the green, and date
- **Club**: A golf club type (Wedge, 9 Iron, 8 Iron, 7 Iron, 6 Iron, 5 Iron, 3 Wood, Driver)
- **Vertical_Distance**: The vertical distance from the shot landing position to the pin, measured in yards (positive = long, negative = short)
- **Horizontal_Distance**: The horizontal distance from the shot landing position to the pin, measured in yards (positive = right, negative = left)
- **Distance_Into_Green**: The distance from where the shot was played into the green, measured in yards
- **Hypotenuse_Distance**: The straight-line distance from the shot landing position to the pin, calculated as sqrt(Vertical_Distance² + Horizontal_Distance²)
- **Green_Canvas**: A custom-drawn golf green visualization showing shot positions relative to the pin, with contour rings at distance intervals
- **Entry_Green_Canvas**: A separate interactive Green_Canvas rendered inside the Log Shot form, used exclusively for Click-to-Place shot entry
- **Click_to_Place**: A shot entry mode where the user taps or clicks on the Entry_Green_Canvas to set the ball landing position visually; Vertical_Distance and Horizontal_Distance are derived automatically from the click coordinates
- **Coordinate_Readout**: A live display beneath the Entry_Green_Canvas showing the current Vertical_Distance, Horizontal_Distance, and Hypotenuse_Distance values corresponding to the cursor position (hover) or placed marker (click)
- **Entry_Mode**: The active method for inputting shot position data — Manual Entry (typed numeric inputs), Click to Place (canvas tap), or Quick Log (zone-based tap with auto-save)
- **Dispersion**: The standard deviation of Hypotenuse_Distance values for a set of shots, representing grouping consistency
- **Distance_Bucket**: A proximity threshold specific to each Club representing the maximum acceptable result distance (Hypotenuse_Distance) from the pin. Shots finishing within this threshold are considered "in bucket" and qualify for detailed KPI analysis.
- **KPI**: Key Performance Indicator — a computed metric summarizing shot accuracy for a Club within a Distance_Bucket
- **Benchmark**: Reference proximity values for Pro, Good Amateur, and Average Amateur levels per club
- **Toast**: A non-blocking slide-in notification that auto-dismisses after a set duration
- **Rolling_Window**: A time-based filter that includes only shots from the last N days
- **Improvement_Opportunity_Score**: A weighted metric (60% proximity gap to best club + 40% dispersion) used to prioritise which clubs need the most practice
- **Miss_Pattern**: The dominant directional tendency of a club's shots (long, short, left, right, or scattered)
- **PWA**: Progressive Web App — a web application that can be installed to a device home screen and used offline via a service worker and web app manifest
- **Service_Worker**: A background script registered by the Tracker that intercepts network requests and serves cached assets, enabling offline functionality
- **Web_App_Manifest**: A JSON file (`manifest.json`) that defines how the Tracker appears when installed on a device, including name, icons, theme colour, and display mode
- **Handicap_Index**: A numerical measure of a golfer's playing ability, updated periodically as rounds are submitted. Lower values indicate better performance.
- **Handicap_Tracker**: The section of the Tracker that records and charts a user's Handicap_Index entries over time
- **Practice_Impact**: A Coach's Corner analysis that correlates changes in per-club shot dispersion with Handicap_Index movement over the same time period, identifying which clubs likely contributed to handicap improvement or decline
- **Course_Plot**: A dedicated tab for on-course shot logging using real green shapes fetched from OpenStreetMap, allowing users to place pin position and ball landing on the actual green outline for each hole
- **OSM**: OpenStreetMap — a free, open-source, community-built map database. The Tracker queries OSM via the Overpass API to retrieve golf green polygon data tagged with `golf=green`.
- **Overpass_API**: A read-only query API for OpenStreetMap data, used by the Tracker to fetch green polygon coordinates for a selected course
- **Nominatim**: OpenStreetMap's geocoding API used for fast name-based search of golf courses. Returns coordinates that are then used to query Overpass for green polygon data.
- **Green_Polygon**: A set of lat/lon coordinates defining the outline shape of a putting green, sourced from OSM and rendered on canvas as the actual green shape

## Requirements

### Requirement 1: Shot Data Entry

**User Story:** As a golfer, I want to input my practice shot data, so that I can track my performance over time.

#### Acceptance Criteria

1. THE Tracker SHALL provide a form to input a Shot with the following fields: Club, Vertical_Distance (yards), Horizontal_Distance (yards), and date.
2. THE Tracker SHALL support the following Club values: Wedge, 9 Iron, 8 Iron, 7 Iron, 6 Iron, 5 Iron, 3 Wood, and Driver.
3. WHEN a user submits a valid Shot form, THE Tracker SHALL persist the Shot data to localStorage.
4. WHEN a user submits a Shot form with missing required fields, THE Tracker SHALL display a validation error message identifying the missing fields.
5. THE Tracker SHALL allow Vertical_Distance and Horizontal_Distance to accept positive and negative numeric values.
6. THE Tracker SHALL default the date field to the current date.
7. THE Tracker SHALL provide three Entry_Mode options within the Log Shot form: Manual Entry, Click to Place, and Quick Log.

### Requirement 2: Shot Data Persistence

**User Story:** As a golfer, I want my shot data to persist between browser sessions, so that I do not lose my practice history.

#### Acceptance Criteria

1. THE Tracker SHALL store all Shot data in the browser localStorage.
2. WHEN the Tracker loads, THE Tracker SHALL retrieve and display previously saved Shot data from localStorage.
3. IF localStorage is unavailable or corrupted, THEN THE Tracker SHALL display an error message indicating data cannot be loaded and allow the user to start with an empty dataset.

### Requirement 3: CSV Bulk Import

**User Story:** As a golfer, I want to import shot data in bulk from a CSV file, so that I can quickly load a full practice session without entering shots one by one.

#### Acceptance Criteria

1. THE Tracker SHALL provide an "Import CSV" button that opens the browser file picker filtered to .csv files.
2. THE Tracker SHALL parse CSV files with columns in the order: Club, Vertical_Distance, Horizontal_Distance, Date (YYYY-MM-DD).
3. THE Tracker SHALL auto-detect and skip a header row if it contains column name keywords (club, vertical, date).
4. THE Tracker SHALL validate each row individually and import only valid rows.
5. WHEN rows contain invalid data, THE Tracker SHALL skip those rows and report the count of skipped rows with error details.
6. THE Tracker SHALL display a toast notification summarising the import result (shots imported, rows skipped).

### Requirement 4: Golf Green Visualization

**User Story:** As a golfer, I want to see my shots plotted on a visual representation of a golf green, so that I can intuitively understand my shot dispersion relative to the pin.

#### Acceptance Criteria

1. THE Tracker SHALL render a custom canvas-based golf green with the pin at the centre.
2. THE Tracker SHALL draw the green as a circle with the outer edge representing the club's Distance_Bucket threshold (maximum "good" distance).
3. THE Tracker SHALL draw concentric contour rings at evenly spaced distance intervals within the green, labelled with yard distances.
4. THE Tracker SHALL render the pin as a flag (white pole, red flag, dark hole) at the origin.
5. THE Tracker SHALL plot each shot as a coloured marker at the correct position relative to the pin.
6. WHEN a user selects a Club, THE Tracker SHALL display only shots for that club on the green.
7. THE Tracker SHALL provide a month filter to show only shots from a selected month.
8. THE Tracker SHALL dynamically populate the month filter dropdown with all months that contain shot data, sorted newest first, plus an "All Time" default option.
8. THE Tracker SHALL render the canvas responsively, scaling to fit the container width while maintaining a 1:1 aspect ratio at device pixel ratio for crisp rendering.
9. WHEN the browser window is resized or the device orientation changes, THE Tracker SHALL re-render visible canvas charts to fit the new viewport dimensions.

### Requirement 5: Chart Export

**User Story:** As a golfer, I want to download my green visualization as an image, so that I can share my shot patterns with a coach or friends.

#### Acceptance Criteria

1. THE Tracker SHALL provide a download button on the green visualization view.
2. WHEN a user clicks the download button, THE Tracker SHALL generate and download the visualization as a PNG image file.
3. THE Tracker SHALL name the exported file using the pattern "{Club}_scatter_plot.png".
4. THE Tracker SHALL render the chart image entirely on the client side without server-side processing.

### Requirement 6: KPI Dashboard

**User Story:** As a golfer, I want to see detailed performance metrics for each club with benchmark comparisons, so that I can objectively measure my accuracy against known standards.

#### Acceptance Criteria

1. THE Tracker SHALL display a large KPI card for each Club that has recorded Shot data.
2. THE Tracker SHALL calculate and display: average absolute Vertical_Distance, average absolute Horizontal_Distance, and average Hypotenuse_Distance per Club within the Distance_Bucket.
3. THE Tracker SHALL apply the following Distance_Bucket thresholds based on result proximity (Hypotenuse_Distance to pin): Wedge 10 yds, 9 Iron 13 yds, 8 Iron 15 yds, 7 Iron 25 yds, 6 Iron 30 yds, 5 Iron 30 yds, 3 Wood 50 yds, Driver 55 yds. A shot qualifies for KPI calculation when its Hypotenuse_Distance is within the club's bucket threshold.
4. THE Tracker SHALL display a performance score ring (0-100%) comparing the user's average proximity to benchmark values (Pro, Good Amateur, Average Amateur).
5. THE Tracker SHALL display benchmark reference values (Pro, Good, Average) for each club.
6. THE Tracker SHALL display an In-Bucket Rate percentage representing the proportion of shots for that club that finished within the Distance_Bucket threshold of the pin (Hypotenuse_Distance ≤ bucket).
7. THE Tracker SHALL display a Miss Tendency bias indicator showing average directional miss (left/right, short/long) on a visual track.
8. THE Tracker SHALL display a status indicator per club: "On track" (≤ good benchmark), "Room to improve" (≤ average benchmark), or "Focus area" (above average).
9. WHEN a Club has no Shot data within the Distance_Bucket, THE Tracker SHALL display a message indicating insufficient data.
10. THE Tracker SHALL provide a month filter to show KPIs for a selected month only.
11. THE Tracker SHALL provide a Standard/Detailed toggle on the KPI page. "Standard" SHALL be the default view.
12. IN Standard view, THE Tracker SHALL display a simplified card per club showing: performance score (ring), average distance to pin, and target distance (Distance_Bucket threshold).
13. IN Detailed view, THE Tracker SHALL display the full KPI card with all metrics as specified in ACs 1-9 above.

### Requirement 7: Trend Line Chart

**User Story:** As a golfer, I want to see how my proximity to the pin has changed over time for each club, so that I can track my improvement.

#### Acceptance Criteria

1. THE Tracker SHALL display a mini trend line chart within each KPI card showing average Hypotenuse_Distance per month over time.
2. THE trend chart SHALL use all-time data regardless of the KPI page month filter, to show full trajectory.
3. THE trend chart SHALL only render when a club has data spanning 2 or more months.
4. THE trend chart SHALL use a smooth line with filled area beneath.

### Requirement 8: Today's Practice & Recommendations (Coach's Corner)

**User Story:** As a golfer, I want a simple, actionable practice plan when I open the app, so that I know exactly what to work on at the range without overthinking it.

#### Acceptance Criteria

1. THE Tracker SHALL display a "Today's Practice" session plan at the top of the Coach's Corner tab, above the priority recommendations.
2. THE Tracker SHALL generate up to 3 different session plans based on the user's data, each focusing on different club groupings:
   - **Session A ("Work on your weaknesses"):** top 2 priority clubs by improvement score.
   - **Session B ("Sharpen the mid-range"):** the 3rd and 4th priority clubs, or a mix if fewer clubs qualify.
   - **Session C ("Contrast drill — worst vs best"):** the highest-priority club paired with the user's strongest club.
3. THE Tracker SHALL display left and right arrow buttons allowing the user to cycle between available session plans. Only the active plan SHALL be visible at a time. The arrows SHALL wrap around (last → first, first → last).
4. EACH session plan SHALL include: warm up with the user's most consistent club (5 shots), focused work on 2 clubs (10 and 8 shots respectively) with plain-English tips based on miss direction, and a 5-shot finish with free club choice.
5. EACH session plan SHALL display an estimated ball count and session duration (~20 minutes), and indicate which data period it is based on.
6. WHEN insufficient data exists (fewer than 3 shots per club in bucket), THE Tracker SHALL display a message prompting the user to log more shots to unlock the plan.
6. THE Tracker SHALL calculate an Improvement Opportunity Score for each qualifying club, weighted 60% on the gap between that club's average proximity and the best performing club, and 40% on dispersion.
2. THE Tracker SHALL identify the top 3 worst performing clubs and present them as Priority 1, 2, and 3 recommendations.
3. EACH recommendation SHALL include a natural-language coaching reason that identifies the dominant miss pattern:
   - IF a club consistently misses long, THE Tracker SHALL advise "take one less club or control swing length."
   - IF a club consistently misses short, THE Tracker SHALL advise "commit to a full swing or take one more club."
   - IF a club consistently misses right, THE Tracker SHALL advise "check alignment and face angle at impact."
   - IF a club consistently misses left, THE Tracker SHALL advise "review grip pressure and swing path."
   - IF a club has a scattered miss pattern, THE Tracker SHALL advise "focus on consistent setup and tempo."
4. EACH recommendation SHALL include contextual commentary comparing the club's performance gap to the user's best club.
5. EACH recommendation SHALL display supporting statistics: average distance to pin, dispersion, improvement score, and shot count.
6. THE Tracker SHALL display a full club ranking below the top 3 recommendations, ordered by improvement opportunity score (highest first).
7. THE Tracker SHALL colour-code rankings: red (top third), amber (middle third), green (bottom third) with visual progress bars.
8. WHEN a Club has fewer than 3 recorded shots within the Distance_Bucket, THE Tracker SHALL exclude that Club from recommendations and display a message indicating more data is needed.
9. THE Tracker SHALL default to a rolling 30-day window for recommendation calculation, reflecting current form.
10. THE Tracker SHALL provide a period selector with options: Last 30 days, Last 60 days, Last 90 days, and All Time.

### Requirement 9: Toast Notifications

**User Story:** As a golfer, I want non-intrusive feedback when I perform actions, so that the app feels responsive without disrupting my flow.

#### Acceptance Criteria

1. THE Tracker SHALL display toast notifications (slide-in, top-right) instead of browser alert dialogs for all user feedback.
2. THE Tracker SHALL support toast types: success (green), error (red), and info (blue).
3. THE Tracker SHALL auto-dismiss toasts after 3-5 seconds.
4. THE Tracker SHALL display an "Undo" action on shot-save toasts that removes the last saved shot when clicked.

### Requirement 10: Undo Last Action

**User Story:** As a golfer, I want to quickly undo my last shot entry if I made a mistake, so that I don't have to clear all data.

#### Acceptance Criteria

1. WHEN a shot is saved, THE Tracker SHALL display a toast with an Undo option.
2. WHEN the user clicks Undo, THE Tracker SHALL remove the most recently added shot from localStorage.
3. THE Tracker SHALL update the shot counter and any visible views after an undo action.

### Requirement 11: Dark/Light Mode

**User Story:** As a golfer, I want to switch between dark and light themes, so that I can use the app comfortably in different lighting conditions (indoors vs outdoor range).

#### Acceptance Criteria

1. THE Tracker SHALL provide a theme toggle button (moon/sun icon) accessible from any view.
2. WHEN the user clicks the toggle, THE Tracker SHALL switch between dark mode and light mode colour schemes.
3. THE Tracker SHALL persist the user's theme preference in localStorage.
4. THE Tracker SHALL apply the saved theme preference on page load.

### Requirement 12: Reviews Carousel

**User Story:** As a visitor, I want to see testimonials from other golfers, so that I can trust the app's value before investing time in it.

#### Acceptance Criteria

1. THE Tracker SHALL display a carousel of 5 five-star user reviews on the home page below the main content.
2. THE carousel SHALL auto-advance every 5 seconds.
3. THE carousel SHALL provide clickable dot indicators for manual navigation.
4. EACH review SHALL display: star rating, quote text, author name, and handicap badge.

### Requirement 13: Branding and Visual Design

**User Story:** As a product, the app should have strong visual branding that communicates quality and professionalism.

#### Acceptance Criteria

1. THE Tracker SHALL be branded as "APPROACH IQ" with the tagline "Precision Practice Intelligence for the Modern Golfer".
2. THE Tracker SHALL display a custom SVG logo depicting a stylised golf course (green, flag, bunker, water hazard).
3. THE Tracker SHALL use a dark-mode-first design with glassmorphism cards, gradient accents, and animated background elements.
4. THE Tracker SHALL display a live shot counter and active clubs counter in the hero section.
5. THE Tracker SHALL display sidebar advertisements (3 per side) for golf equipment on screens wider than 1200px, hidden on mobile.
6. THE Tracker SHALL display a branded footer.
7. THE Tracker SHALL label the five navigation tabs as: "⛳ Log", "🗺️ Course Plot", "📊 Charts", "📈 KPIs", and "🎯 Coach's Corner".
8. ON screens wider than 768px, THE Tracker SHALL display navigation as a pill-style tab bar centred below the hero section.
9. ON screens 768px wide or narrower, THE Tracker SHALL hide the top tab bar and display a fixed bottom navigation bar pinned to the bottom of the viewport, with each tab showing an icon and a short label. The bottom nav SHALL respect the iOS safe area inset so it does not overlap the iPhone home indicator.
10. WHEN a tab is switched on mobile, THE Tracker SHALL scroll the page to the top with a smooth animation.

### Requirement 14: Application Architecture

**User Story:** As a developer, I want the application to be a single HTML file with no build dependencies, so that it can be deployed anywhere with zero configuration.

#### Acceptance Criteria

1. THE Tracker SHALL be built as a single HTML file containing all CSS and JavaScript inline.
2. THE Tracker SHALL load Chart.js from a CDN for trend line charts.
3. THE Tracker SHALL load Google Fonts (Inter, Space Grotesk) from a CDN.
4. THE Tracker SHALL operate entirely client-side with no backend server dependencies.
5. THE Tracker SHALL function in modern web browsers (Chrome, Firefox, Safari, Edge — latest two major versions).
6. THE Tracker SHALL be deployable on GitHub Pages by renaming to index.html and pushing to a repository.

### Requirement 17: Handicap Index Tracker

**User Story:** As a golfer, I want to track my handicap index over time alongside my practice data, so that I can see whether my range work is translating into real score improvements.

#### Acceptance Criteria

1. THE Tracker SHALL display a Handicap Index Tracker section between the reviews carousel and the footer, visible on all screen sizes.
2. THE Tracker SHALL display the most recent handicap entry as the "current HCP" prominently in the tracker header.
3. THE Tracker SHALL render a line graph showing handicap index over time, with a background colour distinct from both the dark mode background (`#050a12`) and light mode background (`#f8faf9`). In dark mode the chart background SHALL be `#1e2a3a`; in light mode it SHALL be `#e8edf5`. The Y axis SHALL have a fixed range of 0 (bottom) to 30 (top), so that a lower (improving) handicap trends downward on the chart.
4. EACH line segment and data point on the handicap chart SHALL be colour-coded based on the direction of change between consecutive entries: green (`#10b981`) when the handicap decreases by more than 0.2 (improving), red (`#f43f5e`) when it increases by more than 0.2 (worsening), and orange (`#f97316`) when the change is within ±0.2 (stable).
5. WHEN the user clicks the "+ Log Handicap" button, THE Tracker SHALL display a modal popout containing: a numeric input for the handicap value, a date input defaulting to today, and a save button.
6. THE modal SHALL also display a scrollable history list of all previously logged entries, each showing the handicap value, date, and a delete button.
7. WHEN fewer than 2 entries exist, THE Tracker SHALL display a "no data" prompt in place of the chart, instructing the user to log their first entry.
8. WHEN 2 or more entries exist, THE Tracker SHALL render the line chart and display summary statistics: lowest handicap, highest handicap, total change (first to latest), and total entry count.
9. THE Tracker SHALL persist all handicap entries to localStorage under a dedicated key, independent of shot data.
10. THE modal SHALL close when the user clicks the overlay background or a close button.
11. THE Tracker SHALL provide a "Clear Last" button on the handicap tracker that removes the most recent entry. Repeated clicks SHALL continue removing entries until none remain, at which point all handicap data is cleared.

### Requirement 18: Mobile Carousel Sizing

**User Story:** As a mobile user, I want the reviews carousel to be readable and well-proportioned on a small screen.

#### Acceptance Criteria

1. ON screens 768px wide or narrower, THE Tracker SHALL reduce review card padding, font sizes, and section margins so the carousel fits comfortably without overflow or cramped text.

### Requirement 16: Progressive Web App (PWA) Support

**User Story:** As a golfer, I want to install APPROACH IQ on my phone's home screen so that it feels like a native app and works without an internet connection.

#### Acceptance Criteria

1. THE Tracker SHALL include a Web App Manifest (`manifest.json`) declaring the app name, short name, description, theme colour, background colour, display mode (`standalone`), and icon references.
2. THE Tracker SHALL register a service worker that caches the app shell (index.html, manifest.json, icons) and CDN assets (Chart.js) on first load, enabling offline use.
3. WHEN the app is used offline after the first load, THE Tracker SHALL serve cached assets and remain fully functional for shot logging and viewing data.
4. THE Tracker SHALL declare `apple-mobile-web-app-capable`, `apple-mobile-web-app-status-bar-style` (black-translucent), and `apple-mobile-web-app-title` meta tags to support installation via Safari on iOS.
5. THE Tracker SHALL provide an `apple-touch-icon` (180×180px) that matches the brand logo tile, generated at runtime via canvas and injected as a base64 data URI, displayed when the app is saved to an iOS home screen.
6. THE Tracker SHALL provide a PWA icon at 192×192px matching the brand logo tile design (gradient rounded square with golf course artwork), generated at runtime via canvas and injected as a base64 data URI.
7. THE Tracker SHALL set `theme-color` to `#10b981` so the browser chrome adopts the brand green colour when the app is open.

### Requirement 15: Click-to-Place Shot Entry

**User Story:** As a golfer, I want to tap where my ball landed on a visual green instead of typing numbers, so that logging shots on the range feels fast and intuitive.

#### Acceptance Criteria

1. THE Tracker SHALL provide a toggle in the Log Shot form with two options: "✏️ Manual Entry" (default) and "📍 Click to Place". Only the active mode's input controls SHALL be visible at any time.
2. IN Click to Place mode, THE Tracker SHALL render an Entry_Green_Canvas inside the Log Shot form, visually identical in style to the Charts tab green (circular green, contour rings labelled with yard distances, flag and hole at centre).
3. THE Entry_Green_Canvas SHALL scale its coordinate system to the selected Club's Distance_Bucket, so that the outer ring always represents the bucket boundary for the chosen club. WHEN the user changes the Club selector, THE Entry_Green_Canvas SHALL re-render with the updated scale.
4. WHEN a user clicks or taps anywhere on the Entry_Green_Canvas, THE Tracker SHALL compute Vertical_Distance and Horizontal_Distance by converting the pixel offset from the canvas centre using the current scale (pixels per yard), rounded to the nearest whole integer yard.
5. WHEN a position is placed, THE Tracker SHALL render a coloured ball marker with a glow effect at the clicked position, and draw a dashed line from the hole to the marker.
6. THE Tracker SHALL display a Coordinate_Readout panel beneath the Entry_Green_Canvas showing: Vertical_Distance (signed, labelled +long / -short), Horizontal_Distance (signed, labelled +right / -left), and Hypotenuse_Distance ("X.X yds to pin").
7. ON desktop, THE Coordinate_Readout SHALL update in real time as the cursor moves over the Entry_Green_Canvas (hover preview), before any click is made.
8. WHEN the user clicks to place a ball, THE Coordinate_Readout SHALL lock to the placed coordinates and the hint text SHALL update to indicate the user can tap again to adjust.
9. WHEN the user submits the form in Click to Place mode without first placing a ball on the Entry_Green_Canvas, THE Tracker SHALL display a validation error: "Tap the green to place your ball first."
10. AFTER a successful save in Click to Place mode, THE Tracker SHALL reset the Entry_Green_Canvas (clear the marker), reset the Coordinate_Readout to its empty state, and clear the Distance_Into_Green field.
11. THE Entry_Green_Canvas SHALL support touch events on mobile devices with the same placement behaviour as mouse clicks on desktop.
12. THE Club and date fields SHALL remain visible and shared between both entry modes; switching Entry_Mode SHALL not clear values already entered in those fields.

### Requirement 20: Quick Log Mode (On-Course Entry)

**User Story:** As a weekend golfer, I want to rapidly log my approach shots during a round with minimal taps, so that I can track all 18 holes without slowing down play.

#### Acceptance Criteria

1. THE Tracker SHALL provide a third Entry_Mode option labelled "⚡ Quick Log" in the Log Shot form toggle, alongside Manual and Click to Place.
2. IN Quick Log mode, THE Tracker SHALL display a grid of 8 large club buttons (W, 9i, 8i, 7i, 6i, 5i, 3W, DR) instead of a dropdown selector. The user taps one to select the club for the next shot.
3. THE selected club button SHALL be visually highlighted with a green active state until a different club is tapped.
4. IN Quick Log mode, THE Tracker SHALL display a circular zone-based green with 9 tap zones: centre (pin), long, short, left, right, long-left, long-right, short-left, short-right — plus a "Missed Green" button outside the circle for shots that missed the green entirely.
5. WHEN the user taps a zone, THE Tracker SHALL immediately auto-save the shot using the selected club, the zone's mapped coordinates, and today's date. No separate submit button is required.
6. THE Tracker SHALL map zones to approximate yard coordinates with randomised scatter: centre (~0,~0 within 3 yd radius, minimum 0.5 yds from pin), cardinal directions (~±8,~0 or ~0,~±8 within 2.5 yd radius), diagonals (~±6,~±6 within 2.5 yd radius), and missed green (~18,~0 within 5 yd radius). Each tap SHALL generate a unique randomised point within the zone's radius to produce realistic dispersion data rather than identical fixed coordinates.
7. WHEN a zone is tapped without a club selected, THE Tracker SHALL display a warning: "Pick a club first."
8. AFTER each auto-save, THE Tracker SHALL reset the club selection (deselect the active club button) so the user must pick a club again for the next shot. This prevents accidental logging with the wrong club.
8. AFTER each auto-save, THE Tracker SHALL display a brief flash confirmation showing the club and zone (e.g. "✓ 7 Iron — long left") that auto-dismisses after 2 seconds.
9. THE Tracker SHALL display a running shot counter showing total shots logged in the current Quick Log session.
10. THE Quick Log mode SHALL NOT require the user to enter a date — it SHALL default to today's date automatically.
11. IN Quick Log mode, THE Tracker SHALL hide the shared club dropdown, date field, Save/Import/Clear buttons, and CSV hint — only the big club grid and zone green SHALL be visible.
12. WHEN the shot counter reaches 18, THE Tracker SHALL display a popup offering two options: "Save & End Round" (resets the counter to 0 and shows a confirmation toast) or "Keep Going" (dismisses the popup and allows continued logging for consecutive rounds).
13. THE Tracker SHALL display an "Undo" button next to the shot counter that removes the most recently logged shot, decrements the counter, and shows a confirmation toast.
14. THE zone green SHALL display a dashed contour ring at 50% radius indicating the boundary between the inner "close" zones and the outer zones, labelled with the approximate distance (~3 yds) to give the user a visual sense of scale.

### Requirement 21: Course Plot (On-Course with Real Green Shapes)

**User Story:** As a golfer playing a round, I want to log my approach shots onto actual green shapes for each hole, so that my data accurately reflects whether I hit the green and how close I finished relative to the real pin position.

#### Acceptance Criteria

1. THE Tracker SHALL provide a "🗺️ Course Plot" tab accessible from both the desktop tab bar and the mobile bottom navigation bar.
2. THE Course Plot tab SHALL display a search bar allowing the user to search for golf courses by name via the Nominatim geocoding API (appending "golf" to the query for relevance).
3. WHEN the user types 3 or more characters, THE Tracker SHALL query Nominatim for locations matching the search term (debounced by 500ms) and display up to 10 results in a dropdown, each showing the name and approximate location.
4. WHEN the user selects a course from the search results, THE Tracker SHALL fetch all Green_Polygon data (tagged `golf=green`) and tee box data (tagged `golf=tee`) within a 1.5 km radius of the course centre via the Overpass_API. IF greens have `ref` tags (hole numbers) in the range 1-18, THE Tracker SHALL filter to only those greens, discarding greens from neighbouring courses.
5. THE Tracker SHALL cache fetched course data (green polygons, course name, coordinates) in localStorage for offline use on future visits.
6. IF cached data exists for a selected course, THE Tracker SHALL load from cache without making a network request.
7. WHEN course data is loaded, THE Tracker SHALL display the round UI showing: course name, current hole indicator with prev/next navigation, club selection grid, green canvas, and status bar.
8. THE green canvas SHALL render the actual Green_Polygon shape for the current hole, with a darker fringe area surrounding it and a scale indicator showing yard distance. THE Tracker SHALL orient each green so the approach direction (tee-to-green) is from bottom to top. WHEN tee box data is available for a hole, THE Tracker SHALL use the tee-to-green bearing for rotation; otherwise it SHALL fall back to rotating the green's longest axis to vertical.
9. THE round progression SHALL follow a 3-step flow per hole: (1) select club, (2) tap pin position on the green, (3) tap ball landing position (on or off the green area).
10. AFTER the ball position is tapped, THE Tracker SHALL auto-save the shot (calculating Vertical_Distance and Horizontal_Distance as the difference between ball and pin positions in yards) and auto-advance to the next hole after a brief delay.
11. THE Tracker SHALL display a flash confirmation after each saved shot showing the club used and distance from pin.
12. THE Tracker SHALL provide a "Skip Hole" button that advances to the next hole without logging a shot.
13. THE Tracker SHALL provide an "Undo" button that removes the most recently logged shot from the current round and decrements the shot counter.
14. THE Tracker SHALL provide a "Change Course" button that returns to the course search view.
15. THE Tracker SHALL convert Green_Polygon lat/lon coordinates to local yard-based coordinates using equirectangular approximation, centred on each green's centroid.
16. THE green canvas SHALL render responsively, scaling to fit the container width on all screen sizes including mobile.
17. THE Tracker SHALL sort fetched greens by their OSM `ref` tag (hole number) when available, falling back to geographic position (south-to-north latitude) when ref tags are absent.
18. WHEN the final hole is reached and a shot is saved, THE Tracker SHALL display a "Round complete" message with the total shots logged.
19. THE Tracker SHALL display pin placement as a flag marker (white pole, red flag, hole) and ball placement as a green glowing marker with a dashed line connecting them.
20. ALL shots logged via Course Plot SHALL be persisted using the same localStorage Shot format as other entry modes, ensuring they appear in Charts, KPIs, and Coach's Corner analysis.
