# Test Document Usage Guide

## Overview

This directory contains the manual test execution plan for the X in multiple formats:

1. **manual-test-checklist.md** - Markdown format (source document)
2. **manual-test-checklist-word.html** - HTML format (ready for Word conversion)
3. **generate_word_doc.py** - Python script to regenerate HTML from Markdown

## Converting to Word Document

### Method 1: Direct HTML to Word (Recommended)

1. Open `manual-test-checklist-word.html` in any web browser (Chrome, Firefox, Edge, etc.)
2. The document will display with proper formatting
3. **Option A - Save as Word:**
   - Press `Ctrl+S` (or `Cmd+S` on Mac) to save
   - Choose "Web Page, Complete" format
   - Open the saved HTML file in Microsoft Word
   - Word will automatically convert it to .docx format
   - Save as .docx file

4. **Option B - Copy and Paste:**
   - Select all content in the browser (`Ctrl+A` or `Cmd+A`)
   - Copy (`Ctrl+C` or `Cmd+C`)
   - Open Microsoft Word
   - Paste (`Ctrl+V` or `Cmd+V`)
   - Word will preserve most formatting
   - Save as .docx file

### Method 2: Using Online Converters

1. Open `manual-test-checklist-word.html` in a browser
2. Use online HTML to DOCX converters:
   - https://www.zamzar.com/convert/html-to-docx/
   - https://convertio.co/html-docx/
   - https://cloudconvert.com/html-to-docx
3. Upload the HTML file and download the converted Word document

### Method 3: Using Pandoc (Command Line)

If you have Pandoc installed:

```bash
pandoc manual-test-checklist-word.html -o manual-test-checklist.docx
```

## Document Features

### Fillable Fields

The Word document includes fillable fields (underlined spaces) for:
- Test execution dates
- Tester names
- Test results (Pass/Fail/Partial/Skip)
- Actual results
- Notes and observations
- Issue tracking
- Sign-off information

### Test Status Symbols

Use these symbols to mark test status:
- ✅ Pass - Test case passed successfully
- ❌ Fail - Test case failed
- ⚠️ Partial - Test case partially passed
- ⏭️ Skip - Test case skipped (with reason)
- 🔄 Retest - Test case requires retesting

### Document Structure

1. **Document Information** - Project details and approval fields
2. **Test Execution Summary** - Overview table for all phases
3. **Test Status Legend** - Explanation of status symbols
4. **Phase 1-15** - Detailed test cases for each module
5. **Test Execution Summary** - Results tracking tables
6. **Issue Tracking** - Critical, Major, and Minor issues
7. **Sign-off Section** - Approval and completion signatures

## Using the Document

### For Testers

1. Open the Word document
2. Fill in document information at the top
3. Navigate to the relevant test phase
4. For each test case:
   - Read the test objective and preconditions
   - Follow the test steps carefully
   - Record actual results
   - Mark status (✅/❌/⚠️/⏭️/🔄)
   - Add notes if needed
   - Fill in tester name and date

### For Test Leads

1. Review completed test cases
2. Update the Test Execution Summary table
3. Track issues in the Issues Found sections
4. Calculate pass rates
5. Complete sign-off section when testing is complete

## Regenerating the HTML Document

If you update the Markdown source file, regenerate the HTML:

```bash
cd /home/song/workspace/xproject/foundation/cursordocs
python3 generate_word_doc.py
```

This will update `manual-test-checklist-word.html` with the latest content from `manual-test-checklist.md`.

## Tips for Word Editing

1. **Preserve Formatting**: When copying/pasting, use "Keep Source Formatting" option
2. **Page Breaks**: The document includes page breaks between phases for better printing
3. **Tables**: All tables are formatted and can be edited directly in Word
4. **Fillable Fields**: Click on underlined fields to add text
5. **Status Symbols**: Copy and paste status symbols from the legend section

## Printing

The document is formatted for A4 paper size:
- Margins: 1 inch (2.54 cm) on all sides
- Font: Calibri 11pt (standard Word font)
- Page breaks between major sections
- Tables formatted for readability

## Version Control

- **Version 1.0** - Initial release with 288 test cases covering 15 test phases
- Update the version number in the Document Information section when making changes

## Support

For questions or issues with the test document:
1. Check this README first
2. Review the test cases for clarity
3. Contact the test team lead

---

**Last Updated**: 2024
**Document Owner**: Test Team

