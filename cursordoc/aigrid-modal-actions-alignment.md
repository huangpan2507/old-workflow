## AI Grid Modal Actions Alignment

### Summary
- Standardised the `Rename Grid Space` and `Delete Grid Space` modals so their action buttons share a right-aligned toolbar that mirrors Chakra UI best practices.
- Ensures Cancel, Save, and Delete buttons always render horizontally with Delete anchored to the far right, improving clarity on desktop and mobile widths.

### Changes
- Updated `frontend/src/components/AIGrid/AIGridSidebar.tsx` to:
  - Import and use `ModalFooter` for both modals instead of stacking buttons inside `ModalBody`.
  - Wrap buttons in a responsive `HStack` (`justify="flex-end"`, `flexWrap="wrap"`) guaranteeing Delete remains the final action while supporting narrower screens.
  - Reset rename modal state on Cancel while keeping the new layout consistent with other AI Grid dialogs.
- Documented the UX update in `README.md` under **AI Grid Rich Text Editing**.

### Verification
1. Open AI Grid sidebar, trigger rename modal; observe Cancel and Save aligned on a single row, Save disabled state unchanged.
2. Trigger delete confirmation modal; Cancel and Delete appear on the same row with Delete rightmost, including when resizing browser to mobile width.
3. Confirm no Chakra UI warnings or layout regressions appear in browser console.





