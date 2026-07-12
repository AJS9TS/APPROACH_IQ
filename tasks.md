# Tasks: APPROACH IQ

## Status

The core application is built and deployed. All features described in `requirements.md` are implemented. This document tracks maintenance work, bug fixes, and potential future enhancements.

---

## Completed (Core Build)

- [x] Single-file HTML application with inline CSS and JS
- [x] Shot data entry — Manual, Click-to-Place, and Quick Log modes
- [x] localStorage persistence for shots and handicap data
- [x] CSV bulk import with validation and error reporting
- [x] Golf green canvas visualisation with per-club filtering and month filter
- [x] Chart export as PNG
- [x] KPI dashboard with benchmark comparisons (Standard and Detailed views)
- [x] Trend line charts per club (monthly avg proximity over time)
- [x] Coach's Corner — practice plans, priority recommendations, full club ranking
- [x] Practice Impact analysis (handicap correlation with dispersion changes)
- [x] Handicap Index Tracker with line chart and colour-coded segments
- [x] Dark/Light theme toggle with localStorage persistence
- [x] Toast notifications with undo support
- [x] Reviews carousel with auto-advance and dot navigation
- [x] Bottom navigation bar for mobile with iOS safe area support
- [x] PWA support (manifest, service worker, runtime-generated icons)
- [x] Weather widget (Open-Meteo API + geolocation)
- [x] Streak calendar (monthly practice day tracker)
- [x] Sidebar advertisements (desktop only)
- [x] Responsive design across all breakpoints
- [x] Quick Log zone coordinate randomisation (realistic dispersion)
- [x] Hole-in-one celebration animation
- [x] 18-shot round complete popup

---

## Recent Fixes

- [x] Mobile: Date field overflow on small screens
- [x] Mobile: Pin place canvas not fitting viewport
- [x] Mobile: Reviews carousel only showing half a review
- [x] Quick Log: Coordinates now randomised within zone radius (no more false hole-in-ones)

---

## Known Issues / Backlog

- [ ] Quick Log: "Missed Green" always scatters around (18, 0) — could randomise the angle so misses aren't always "long"
- [ ] Service worker cache: No cache-busting strategy — users may see stale version after updates
- [ ] Accessibility: Canvas-based interactions lack keyboard alternatives and screen reader support
- [ ] Data export: No way to export raw shot data as CSV (only chart PNG export exists)
- [ ] No data backup/restore mechanism beyond manual localStorage manipulation

---

## Potential Future Enhancements

### Short Term
- [ ] Achievement/milestone tracker (e.g. "100 shots logged", "5-day streak", "new personal best proximity")
- [ ] Shot history table with edit/delete per shot
- [ ] Per-session grouping (tag shots belonging to the same practice session)
- [ ] Distance-into-green field (currently removed from the data model but referenced in glossary)

### Medium Term
- [ ] Multi-round score tracking alongside handicap (gross/net scores per round)
- [ ] Club comparison view (overlay two clubs on the same green canvas)
- [ ] Custom club names (allow users to add hybrid, 4 iron, 60-degree wedge, etc.)
- [ ] Data sync via file export/import (JSON backup to device or cloud storage)
- [ ] Landscape mode optimisation for tablets

### Long Term
- [ ] Optional cloud sync (Firebase or similar) for cross-device access
- [ ] Social features (share dispersion charts, compare with friends)
- [ ] Integration with launch monitors (Garmin, Trackman CSV formats)
- [ ] AI-powered coaching insights using shot pattern analysis
- [ ] Strokes Gained calculation against amateur baselines

---

## Development Notes

### How to Deploy
1. Push to GitHub repository
2. Enable GitHub Pages on the `main` branch (root folder)
3. Site is live at `https://<username>.github.io/APPROACH_IQ/`

### How to Test Changes
- Open `index.html` directly in a browser (no build step needed)
- Use browser DevTools for mobile viewport testing
- Test PWA install via Chrome DevTools > Application > Manifest
- Test offline: DevTools > Network > Offline checkbox (after first load)

### Key Design Decisions
- **Single file**: Zero configuration deployment, works anywhere, no dependency management
- **No framework**: Keeps bundle size minimal, loads instantly, no hydration delay
- **Canvas for greens**: Gives full control over the golf-specific visualisation that Chart.js scatter plots couldn't achieve
- **Chart.js for line charts only**: Trend lines and handicap chart benefit from axes, tooltips, and responsiveness that Chart.js provides out of the box
- **Runtime icon generation**: Avoids needing to maintain multiple PNG icon files — canvas draws them on page load
- **Quick Log randomisation**: Prevents misleading KPI data from fixed-coordinate zone taps
