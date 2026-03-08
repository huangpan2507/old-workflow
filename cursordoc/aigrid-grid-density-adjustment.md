# AIGrid Grid Density Adjustment

## Summary
- Canvas users can now resize grid items with granular increments in both vertical and horizontal directions without exposing neighboring charts.
- Introduced a runtime grid density of `4`, meaning each historic grid unit is now subdivided into four smaller units.

## Implementation Details
- Added `GRID_DENSITY` helper utilities in `frontend/src/components/AIGrid/AIGridLayout.tsx` to scale node and layout dimensions as they enter and leave the React Grid Layout layer.
- Runtime config multiplies breakpoint columns (`12 → 48`, `10 → 40`, etc.) and divides row height (`100px → 25px`) so drag/resize interactions move in one-quarter steps.
- Legacy `node_list` and `canvas_metadata.layouts` are automatically scaled on load using the stored `canvas_metadata.grid_density` value (defaults to `1` when missing).
- Persistence payloads now stamp `canvas_metadata.grid_density = 4`, ensuring future clients know which scale the saved coordinates use.

## Compatibility Notes
- Old grid spaces load exactly as before but gain the finer drag/resize precision immediately; no manual migration required.
- If a future client decides to change the density again, it can rescale layouts by comparing its runtime density with the stored `grid_density`.

## Testing & Verification
- Verified that legacy spaces render with the same physical dimensions after scaling.
- Dragged and resized multiple ECharts dashboard disks to confirm the first chart can occupy the full viewport without leaking the second chart.
- Confirmed manual saves persist the higher-resolution coordinates and retain them after a reload.















