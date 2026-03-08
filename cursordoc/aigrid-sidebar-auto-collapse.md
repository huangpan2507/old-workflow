## Canvas Sidebar Auto-Collapse

### Summary
- Align the Canvas navigation behavior with Circuits so the management sidebar auto-collapses on entry and can be re-expanded via the existing header button.
- Reuse the same persistence logic so the collapsed state continues to round trip through `localStorage` and restores when leaving Canvas.

### Changes
- `frontend/src/routes/_layout.tsx`
  - Introduced `autoCollapsePaths` helper shared by initialization and the route change effect.
  - Treat `/aigrid` the same as `/workbench` when forcing collapse on entry and restoring the previous state on exit.
  - Included `/aigrid` in `managementModePages` to guarantee consistent management layout controls.
- `frontend/src/components/Common/Sidebar.tsx`
  - Reused the same auto-collapse guard for menu clicks so navigating to `/aigrid` stores the collapsed preference before routing.
- Updated `README.md` Canvas section with the new behavior.

### Verification
1. From any non-collapsed page, click Canvas in the sidebar; the main sidebar collapses immediately and the header shows the expand button (same as Circuits).
2. Toggle the header expand button; state persists via `localStorage` and survives page refresh.
3. Navigate away from Canvas; the sidebar restores to the pre-collapse state.





