# Policy Preview (Office → PDF) Implementation

## Overview
- Backend exposes `GET /policy-library/preview/{object_name}` to return a presigned PDF URL for inline preview.
- Supported inputs: Word (`.doc`, `.docx`), Excel (`.xls`, `.xlsx`), PowerPoint (`.ppt`, `.pptx`).
- If source is already `.pdf`, the endpoint returns an inline presigned URL directly.

## Backend
- Service: `backend/app/services/document_preview_service.py`
  - Downloads source from the `policy-library-files` bucket to a temp file
  - Runs `soffice --headless --convert-to pdf --outdir <tmp>`
  - Uploads the generated PDF back to MinIO under `previews/<basename>.pdf`
  - Returns a presigned URL (inline) via `PolicyLibraryMinIOService.get_presigned_url_for_preview`
  - Basic caching: reuses existing `previews/<basename>.pdf` if present

### Requirements
- LibreOffice (`soffice`) available in the backend runtime image/container
- Fonts installed to ensure correct rendering for Chinese and corporate fonts

## Frontend
- Component: `frontend/src/components/PolicyUserInterface/shared/FilePreviewModal.tsx`
  - For Office files, calls `/policy-library/preview/{object_name}` to get `preview_url`
  - Renders with `react-pdf` (PDF.js) for in-app viewing
  - Fallback to download / open-in-new-tab when preview fails

## Notes
- Animations/macros are not preserved in PDF
- Excel pagination is handled by LibreOffice defaults; tune via future export options if needed
- Consider adding freshness checks comparing `last_modified` between source and preview for strict cache invalidation in future
