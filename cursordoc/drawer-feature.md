# Policy Library Drawer Feature

## Overview
The Policy Library now features a drawer-based sub-window system for Quick Links functionality. When users click on Quick Links buttons (except "Explore"), the corresponding workspace opens as a sliding drawer from the right side of the screen.

## Features

### Drawer Component
- **Location**: `frontend/src/components/PolicyUserInterface/Drawer.tsx`
- **Features**:
  - Slides in from the right side with smooth animation
  - **Glow effect** - Subtle glow effect with brand color (#9e1422) instead of solid borders
  - **Instant close** - drawer disappears immediately when close button is clicked (no close animation)
  - Backdrop overlay with click-to-close functionality
  - Configurable sizes (sm, md, lg, xl, full)
  - **Full width coverage** - covers entire main content area when size="full"
  - Custom scrollbar styling
  - **Styled close button** - brand color with hover effects

### Drawer Content Components
Located in `frontend/src/components/PolicyUserInterface/drawer/`:

1. **MyPoliciesDrawer** - Displays categorized user policies
2. **ApprovalDrawer** - Shows approval requests with status filtering
3. **DeviationDrawer** - Handles deviation request forms
4. **FileViewerDrawer** - File management interface

### Quick Links Integration
- **Explore** button remains as main content area (no drawer)
- **My Policies** - Opens drawer with categorized policy view
- **Approval** - Opens drawer with approval requests table
- **Deviation** - Opens drawer with deviation request form
- **File Viewer** - Opens drawer with file management
- **Publish** - Opens modal (unchanged)

## User Experience
- Smooth slide-in animation from right
- Backdrop click or close button to dismiss
- **Extended coverage** - covers main content area and extends into Quick Links panel area
- Maintains left navigation visibility
- **Enhanced workspace** - provides more space for content display
- Responsive design with different drawer sizes
- Consistent styling with the main application

## Technical Implementation
- State management for drawer type and visibility
- Reusable Drawer component with TypeScript interfaces
- Proper event handling and cleanup
- Integration with existing policy management hooks
- **Adaptive height containers** - Card and Box components use preset minimum heights (800px for VStack, 400px for Card, 350px for CardBody) to ensure proper content coverage
