# Canvas Text Editing & Run Modal Guide

## Overview

This document explains the November 2025 enhancements applied to the Canvas workspace (`/aigrid` route). Analysts can now edit rendered disk content inline via a TipTap editor and trigger Standard Mode executions without leaving the workspace.

## Rich Text Editing Workflow

1. Enter edit mode using the floating pen icon.
2. Open the grid item menu (`⋮`) and select **编辑文本**.
3. A TipTap toolbar is rendered inside the disk card. Supported commands: bold/italic/underline, bullet & numbered lists, alignment, inline highlights, table insertion and undo/redo.
4. Edit text directly in the embedded editor; all styling is reflected instantly.
5. Use **Save** to persist changes (calls `handleTextEditCommit`) or **Cancel** to discard the draft.
6. Leaving edit mode closes any active editor session automatically.

### Persistence Details

- `conclusion_detail` now stores a TipTap JSON payload: `{"format":"tiptap-json","version":1,"doc":{...}}`.
- During save, `serializeTipTapContent` writes the JSON string while `handleTextEditCommit` persists it through `GridSpaceService`.
- Legacy Markdown strings remain untouched; `parseRichTextContent` detects the format and either renders sanitized TipTap HTML or falls back to `react-markdown`.
- If autosave fails, users receive Chakra toasts and can retry after resolving the underlying issue.

## Run Modal Behaviour

1. In edit mode, open the grid item menu and select **运行**.
2. A centered modal appears (`AIGridRunModal`) embedding `ReportDiskStandardView`.
3. Analysts can upload files or provide URLs. The modal uses the same pipeline as Standard Mode, including progress states and error feedback.
4. After successful completion:
   - `DiskService.getDiskExecutionResults` refreshes the disk’s `latestResult`.
   - Autosave persists the refreshed TipTap/Markdown payload.
5. Closing the modal without completion leaves the original disk result untouched.

## Testing Notes

- `frontend/src/components/AIGrid/tiptap/TipTapEditor.tsx` handles toolbar interactions; smoke-test via Storybook or manual grid editing.
- `frontend/src/components/AIGrid/AIGridRunModal.tsx` asserts modal lifecycle (completion + close handlers).
- Run `npm run build` to ensure TypeScript + Vite can compile the TipTap extensions.

## Related Files

- `frontend/src/components/AIGrid/AIGridItem.tsx`
- `frontend/src/components/AIGrid/AIGridLayout.tsx`
- `frontend/src/components/AIGrid/AIGridRunModal.tsx`
- `frontend/src/components/AIGrid/tiptap/*`
- `frontend/src/services/gridSpaceService.ts`


