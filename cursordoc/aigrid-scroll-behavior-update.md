## Context
- Component: `frontend/src/components/AIGrid/AIGridLayout.tsx`
- Issue: Dragging multiple applications caused `document` horizontal scroll because `scrollIntoView` tried to center new grid items horizontally while the layout container had `overflow-x: hidden`. Left sidebar (“SPACES / MY AGENTS”) was pushed outside viewport.

## Change Summary
- Updated the post-drop auto-focus logic to keep the existing vertical centering but limit horizontal adjustment to `inline: "nearest"`. This confines scrolling to the grid container without touching the global document.

## Expected Behavior
- Repeated drops keep the sidebar fixed in place.
- Vertical scrolling remains smooth when newly added items are out of view.
- Horizontal drag/resize interactions stay unaffected and remain smooth.

## Testing Notes
- Manual reasoning/test plan:
  - Drop several applications in different positions; observe that the sidebar stays fixed.
  - Ensure vertical centering still occurs for new items and no extra horizontal scrollbar appears.





