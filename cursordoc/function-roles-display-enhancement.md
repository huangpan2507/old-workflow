# Function Roles Display Enhancement

## Overview

This document describes the enhancement to Function Roles display across all Policy pages. The change makes the interface more compact by showing Function Role codes instead of full names, with tooltips showing the full names on hover.

## Changes Made

### 1. Interface Updates

Updated `FunctionRole` interface definition in all Policy-related components to include the `code` field:

```typescript
interface FunctionRole {
  id: number
  name: string
  code: string  // Added this field
}
```

**Files Updated:**
- `PolicyUserInterface.tsx`
- `PolicyDetailModal.tsx`
- `PolicyUploadForm.tsx`
- `PolicyRevisionForm.tsx`

### 2. Display Logic Changes

#### PolicyUserInterface (Main Policy Library Page)
- **Function Roles Filter Buttons**: Changed from displaying `role.name` to `role.code`
- **Added Tooltips**: Wrapped buttons with `Tooltip` component showing `role.name` on hover
- **Import Added**: Added `Tooltip` to Chakra UI imports

**Before:**
```tsx
<Button>
  {role.name}  // "Human Resources"
</Button>
```

**After:**
```tsx
<Tooltip label={role.name} placement="top">
  <Button>
    {role.code}  // "HR"
  </Button>
</Tooltip>
```

#### PolicyUploadForm (Policy Creation/Editing)
- **Select Dropdown**: Changed from `{role.name}` to `{role.code} - {role.name}` for better clarity
- **Checkbox Group**: Changed from `{role.name}` to `{role.code}` with tooltip
- **Import Added**: Added `Tooltip` to Chakra UI imports

**Select Options:**
```tsx
<option key={role.id} value={role.id}>
  {role.code} - {role.name}  // "HR - Human Resources"
</option>
```

**Checkboxes:**
```tsx
<Tooltip label={role.name} placement="top">
  <Checkbox value={role.id.toString()}>
    {role.code}  // "HR"
  </Checkbox>
</Tooltip>
```

#### PolicyRevisionForm (Policy Revision)
- **Select Dropdown**: Changed from `{role.name}` to `{role.code} - {role.name}`
- **Import Added**: Added `Tooltip` to Chakra UI imports

**Select Options:**
```tsx
<option key={role.id} value={role.id}>
  {role.code} - {role.name}  // "HR - Human Resources"
</option>
```

### 3. User Experience Improvements

#### Benefits
1. **Space Efficiency**: Function Roles filter buttons take up less horizontal space
2. **Consistent Display**: All Policy pages now use the same display strategy
3. **Accessibility**: Full names are still accessible via tooltips
4. **Better Organization**: Code-based display makes it easier to scan through options

#### Display Strategy by Component Type
- **Filter Buttons**: Show `code` with `name` tooltip
- **Select Dropdowns**: Show `code - name` for clarity
- **Checkboxes**: Show `code` with `name` tooltip

## Technical Implementation

### Tooltip Configuration
```tsx
<Tooltip label={role.name} placement="top">
  <Button>
    {role.code}
  </Button>
</Tooltip>
```

### Consistent Interface
All Policy components now use the same `FunctionRole` interface:
```typescript
interface FunctionRole {
  id: number
  name: string
  code: string
}
```

## Files Modified

1. **PolicyUserInterface.tsx**
   - Updated FunctionRole interface
   - Added Tooltip import
   - Modified filter buttons with tooltips

2. **PolicyDetailModal.tsx**
   - Updated FunctionRole interface

3. **PolicyUploadForm.tsx**
   - Updated FunctionRole interface
   - Added Tooltip import
   - Modified Select options and Checkbox group

4. **PolicyRevisionForm.tsx**
   - Updated FunctionRole interface
   - Added Tooltip import
   - Modified Select options

## Testing

- ✅ TypeScript compilation passes
- ✅ No linting errors
- ✅ All Policy pages use consistent display strategy
- ✅ Tooltips provide full names on hover
- ✅ Interface remains functional and accessible

## Future Considerations

1. **Consistent Tooltip Timing**: Consider standardizing tooltip delay across all components
2. **Mobile Optimization**: Ensure tooltips work well on touch devices
3. **Accessibility**: Consider adding ARIA labels for screen readers
4. **Performance**: Monitor if tooltip rendering affects performance with large lists
