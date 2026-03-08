# Policy Revision Form Minimal File Upload

## Overview
Updated the PolicyRevisionForm component to use the same minimal file upload control style as the PolicyUploadForm for consistency across the application.

## Changes Made

### 1. Added AttachmentIcon Import
```tsx
import { AttachmentIcon } from '@chakra-ui/icons'
```

### 2. Replaced Complex Upload UI with Minimal Button
**Before (Complex with Drag-and-Drop):**
```tsx
<VStack spacing={3} align="stretch">
  {/* Native file input - visible and directly clickable */}
  <Box>
    <input
      type="file"
      accept=".pdf,.doc,.docx,.txt,.jpg,.jpeg,.png"
      onChange={(e) => {
        console.log("Policy revision native file input changed:", e.target.files)
        setSelectedFile(e.target.files?.[0] || null)
      }}
      style={{
        width: "100%",
        padding: "8px",
        border: "1px solid #e2e8f0",
        borderRadius: "6px",
        fontSize: "14px",
        backgroundColor: "white",
        cursor: "pointer"
      }}
    />
    
    {/* Show selected file name */}
    {selectedFile && (
      <Box fontSize="sm" color="green.600" mt={2}>
        Selected: {selectedFile.name}
      </Box>
    )}
  </Box>
  
  {/* Drag and drop area */}
  <Box
    border="2px dashed"
    borderColor="gray.300"
    borderRadius="md"
    p={6}
    textAlign="center"
    cursor="pointer"
    _hover={{ borderColor: "blue.400", bg: "blue.50" }}
    onDrop={(e) => {
      e.preventDefault()
      const files = e.dataTransfer.files
      if (files.length > 0) {
        console.log("File dropped:", files[0])
        setSelectedFile(files[0])
      }
    }}
    onDragOver={(e) => e.preventDefault()}
    onDragEnter={(e) => e.preventDefault()}
  >
    <VStack spacing={2}>
      <Box fontSize="2xl">📄</Box>
      <Text fontSize="sm" color="gray.600">
        Drag and drop a file here, or click to browse
      </Text>
      <Text fontSize="xs" color="gray.500">
        Supported formats: PDF, DOC, DOCX, TXT
      </Text>
    </VStack>
  </Box>
</VStack>
```

**After (Minimal Button):**
```tsx
<Box>
  {/* Simple File Upload Button */}
  <Button
    as="label"
    htmlFor="file-upload"
    cursor="pointer"
    variant="outline"
    leftIcon={<AttachmentIcon />}
    w="full"
    h="40px"
    fontSize="sm"
    fontWeight="normal"
    borderColor="gray.300"
    _hover={{ 
      borderColor: "gray.400",
      bg: "gray.50"
    }}
  >
    {selectedFile ? selectedFile.name : "Choose File"}
  </Button>
  
  {/* Hidden file input */}
  <input
    id="file-upload"
    type="file"
    onChange={(e) => {
      console.log("Policy revision file input changed:", e.target.files)
      setSelectedFile(e.target.files?.[0] || null)
    }}
    accept=".pdf,.doc,.docx,.txt,.jpg,.jpeg,.png"
    style={{ display: "none" }}
  />
  
  {/* File info */}
  {selectedFile && (
    <Text fontSize="xs" color="gray.500" mt={1}>
      Size: {(selectedFile.size / 1024).toFixed(1)} KB
    </Text>
  )}
</Box>
```

### 3. Simplified File Information Display
**Before (Complex Helper Text):**
```tsx
<FormHelperText>
  Upload the revised policy document. If not provided, the original document will be used.
  {selectedFile && (
    <Text color="green.500" fontSize="sm" mt={1}>
      Selected: {selectedFile.name} ({(selectedFile.size / 1024).toFixed(1)} KB)
    </Text>
  )}
</FormHelperText>
```

**After (Clean Helper Text):**
```tsx
<FormHelperText>
  Upload the revised policy document. If not provided, the original document will be used.
</FormHelperText>
```

## Benefits

1. **Consistency**: Matches the PolicyUploadForm file upload style
2. **Cleaner Interface**: Removes complex drag-and-drop area
3. **Better UX**: Single, clear action button
4. **Simplified Code**: Removed drag-and-drop event handlers
5. **Mobile Friendly**: Works better on touch devices

## Design Consistency

Both PolicyUploadForm and PolicyRevisionForm now use:
- **Same Button Style**: Outline variant with gray border
- **Same Dimensions**: Full width, 40px height
- **Same Hover Effects**: Light gray background
- **Same File Display**: Button text shows filename
- **Same File Info**: Size displayed below button

## Files Modified
- `frontend/src/components/PolicyUserInterface/PolicyRevisionForm.tsx`

## Dependencies Added
- `AttachmentIcon` from Chakra UI icons

## Testing Recommendations

1. Test file selection works correctly
2. Verify button text updates with selected file
3. Test file size display
4. Verify form submission includes file
5. Compare with PolicyUploadForm for consistency
6. Test on mobile devices
