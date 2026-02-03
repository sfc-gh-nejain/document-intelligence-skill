# Parsing Sub-Skill

Full content parsing using AI_PARSE_DOCUMENT with page optimization.

## When to Use

This sub-skill is triggered when the user selects:
- "Full text with layout preservation (tables, headers, structure)"
- "Text only via OCR (quick text extraction)"

## AI_PARSE_DOCUMENT Constraints

**Inform the user before proceeding:**

| Constraint | Limit |
|------------|-------|
| Max file size | 50 MB |
| Max pages | 500 per call |
| Output format | Markdown |

## Cost Information

**Show this pricing information to the user for awareness (NOT for decision making):**

```
AI_PARSE_DOCUMENT Pricing (for reference only):

LAYOUT Mode: 3.33 credits per 1,000 pages (~0.00333/page)
OCR Mode: 0.5 credits per 1,000 pages (~0.0005/page)

Note: Choose the mode based on document structure and quality needs, 
      not pricing. See "Parsing Modes" section for guidance.
```

## Supported Formats

PDF, PNG, PPTX/PPT, DOC/DOCX, JPEG/JPG, HTM/HTML, TEXT/TXT, TIF/TIFF, BMP, GIF, WEBP

**Note:** CSV, MD, and EML files are NOT supported by AI_PARSE_DOCUMENT (use AI_EXTRACT instead).

## TO_FILE Syntax (IMPORTANT)

**Always use the fully qualified stage name with @ prefix:**

```sql
TO_FILE('@DB.SCHEMA.STAGE', 'filename.pdf')
```

| Component | Format | Example |
|-----------|--------|---------|
| Stage name | `'@DB.SCHEMA.STAGE'` | `'@MYDB.PUBLIC.DOC_STAGE'` |
| File path | `'relative/path/file.ext'` | `'documents/report.pdf'` |

**Examples:**
```sql
-- Single file
TO_FILE('@MYDB.PUBLIC.DOC_STAGE', 'report.pdf')

-- File in subfolder
TO_FILE('@MYDB.PUBLIC.DOC_STAGE', 'reports/2024/annual_report.pdf')

-- Using variable for batch processing
TO_FILE('@MYDB.PUBLIC.DOC_STAGE', relative_path)
```

**Common mistakes to avoid:**
- ❌ `TO_FILE('@stage', 'file.pdf')` - Missing DB.SCHEMA
- ❌ `TO_FILE('DB.SCHEMA.STAGE', 'file.pdf')` - Missing @ prefix
- ✅ `TO_FILE('@DB.SCHEMA.STAGE', 'file.pdf')` - Correct format

## Parsing Modes

**IMPORTANT: Focus on QUALITY, not price. Choose mode based on document characteristics.**

| Mode | When to Use | Output |
|------|-------------|--------|
| **LAYOUT** | Documents with headings, sections, tables, columns, structured formatting | Markdown with formatting preserved |
| **OCR** | Text-heavy documents where page layout is not relevant | Plain text extraction |

### Mode Selection Guide

**Use LAYOUT Mode when document contains:**
- Headings and subheadings
- Tables or tabular data
- Multi-column layouts
- Structured sections (e.g., forms, reports, contracts)
- Headers and footers that matter
- Bullet points or numbered lists
- Any formatting that conveys meaning

**Use OCR Mode when:**
- Document is purely text with no meaningful structure
- You only need raw text content
- Layout and formatting are not relevant to your use case
- Processing scanned handwritten notes or simple text images

---

## ⚠️ MANDATORY Workflow - Follow ALL Steps

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PARSING WORKFLOW (ALL STEPS REQUIRED)            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Step 1: Page Optimization ──→ Step 2: Display Cost                 │
│  (REQUIRED - Ask user)         (REQUIRED - Show before execution)   │
│                                        │                            │
│                                        ▼                            │
│                              Step 3: Test Single File               │
│                              (REQUIRED - Parse ONE file first)      │
│                                        │                            │
│                                        ▼                            │
│                              Step 4: Display Results                │
│                              (REQUIRED - Show parsed content)       │
│                                        │                            │
│                                        ▼                            │
│                              Step 5: Post-Processing Options        │
│                              (REQUIRED - Ask what to do next)       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**CRITICAL:** Do NOT skip any step. Each step is mandatory for the parsing flow.

---

## Step 1: Page Optimization (REQUIRED)

Before parsing, ask user about page optimization to improve performance and reduce costs.

**Ask** user:
```
Would you like to parse the entire document or specific pages?
Options:
1. Parse entire document
2. Parse specific page range (e.g., pages 1-50)
3. Parse specific pages (e.g., pages 1, 5, 10, 25)
4. I'm not sure - help me decide
```

---

## Step 2: Display Cost BEFORE Execution (REQUIRED)

### ⚠️ MANDATORY: Show Cost to User - Do NOT Skip

**You MUST show this cost information to the user BEFORE running AI_PARSE_DOCUMENT:**

```
Before I run AI_PARSE_DOCUMENT, here's the cost information:

AI_PARSE_DOCUMENT Pricing:
- LAYOUT Mode: 3.33 credits per 1,000 pages (~0.00333/page)
- OCR Mode: 0.5 credits per 1,000 pages (~0.0005/page)

For your document ([X] pages) in [LAYOUT/OCR] mode:
- Estimated cost: [calculated cost] credits

Note: Pricing shown for awareness only. Mode selection should be 
based on document structure, not cost.

Shall I proceed with parsing?
```

**Do NOT skip this step.** Always calculate and show the estimated cost based on:
1. Number of pages to be processed
2. Selected mode (LAYOUT or OCR)

---

## Step 3: Test Single File First (REQUIRED)

### ⚠️ MANDATORY: Parse ONE File First Before Batch Processing

**Always test with a single representative file first.** This ensures the parsing mode and page options are correct.

### Based on Page Optimization Choice:

#### If "Parse entire document"

```sql
-- Test: Parse single file with page_split for easier processing
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@db.schema.stage', '<test_file.pdf>'),
  {'mode': 'LAYOUT', 'page_split': true}
) AS parsed_result;
```

#### If "Parse specific page range"

First ask for the range:
```
What page range would you like to parse?
Example: 1-50 (parses pages 1 through 50)
Note: Page numbers are 1-indexed for your input, converted to 0-indexed internally.
```

```sql
-- Test: Parse specific range (page_filter uses 0-indexed pages)
-- For user input "1-50", use start: 0, end: 50
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@db.schema.stage', '<test_file.pdf>'),
  {'mode': 'LAYOUT', 'page_filter': [{'start': 0, 'end': 50}]}
) AS parsed_result;
```

#### If "Parse specific pages"

First ask for page numbers:
```
Which specific pages would you like to parse?
Example: 1, 5, 10, 25 (parses only these pages)
Note: Page numbers are 1-indexed for your input, converted to 0-indexed internally.
```

```sql
-- Test: Parse specific pages (0-indexed internally)
-- For user input "1, 5, 10, 25", use [0, 4, 9, 24]
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@db.schema.stage', '<test_file.pdf>'),
  {'mode': 'LAYOUT', 'page_filter': [0, 4, 9, 24]}
) AS parsed_result;
```

#### If "I'm not sure"

Provide guidance:
```
Here's how to decide:

- Parse entire document: Best for full text extraction, RAG pipelines, 
  or when you need all content. Works well for documents ≤500 pages.

- Parse specific page range: Best when you know the relevant section 
  (e.g., "chapters 1-3 are on pages 1-75"). Faster and cheaper.

- Parse specific pages: Best for extracting just a table of contents, 
  specific chapters, or known important pages.

Tips:
- First page often has title, author, date
- Last few pages often have references, appendix
- Table of contents usually in first 5 pages

Would you like me to first get the page count so you can decide?
```

If user wants page count:
```sql
-- Quick page count check (minimal processing)
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@db.schema.stage', '<test_file.pdf>'),
  {'mode': 'OCR', 'page_filter': [{'start': 0, 'end': 1}]}
):pageCount AS total_pages;
```

---

## Step 4: Display Results to User (REQUIRED)

### ⚠️ MANDATORY: Show Parsed Content - Do NOT Skip

**You MUST display the parsing results to the user before proceeding.**

### Extract and display content:

```sql
SELECT 
  parsed_result:content::STRING AS full_text,
  parsed_result:pageCount::INT AS total_pages
FROM (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@db.schema.stage', '<test_file.pdf>'),
    {'mode': 'LAYOUT'}
  ) AS parsed_result
);
```

### For page-split results, show individual pages:

```sql
SELECT 
  page.value:index::INT AS page_num,
  page.value:content::STRING AS page_content
FROM (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@db.schema.stage', '<test_file.pdf>'),
    {'mode': 'LAYOUT', 'page_split': true}
  ) AS parsed_result
),
LATERAL FLATTEN(input => parsed_result:pages) page;
```

### Present results to user:

```
Here are the parsing results for your test file:

📄 File: [filename]
📊 Total Pages: [X]
📝 Mode: [LAYOUT/OCR]

--- Parsed Content Preview ---
[Show first 500-1000 characters of content]
...

Are you satisfied with these results?
Options:
1. Yes, proceed to next steps
2. No, try different mode (switch LAYOUT ↔ OCR)
3. No, adjust page selection
```

**If user is not satisfied:** Return to Step 1 or Step 3 based on their choice.

**If user is satisfied:** Proceed to Step 5.

---

## Step 5: Post-Processing Options (REQUIRED)

### ⚠️ MANDATORY: Ask User What to Do Next - Do NOT Skip

**You MUST ask the user what they want to do after successful parsing.**

**Ask** user:
```
Your document has been parsed successfully! What would you like to do next?

Options:
1. Done - I only needed this one-time parsing
2. Store results - Save parsed content to a Snowflake table
3. Batch process - Parse multiple documents with same settings
4. Set up pipeline - Create automated processing with streams and tasks
5. Further analysis - Use parsed text for RAG, search, or AI analysis
```

**Based on user choice:**

- **Done:** Parsing flow complete. No further action needed.
- **Store results / Batch process / Set up pipeline:** → Load `reference/pipeline.md` for templates
- **Further analysis:** → Discuss RAG pipeline setup, full-text search, or AI_COMPLETE analysis

---

## Reference: Batch Processing Templates

### Process multiple documents

```sql
-- Enable directory table
ALTER STAGE @db.schema.stage SET DIRECTORY = (ENABLE = TRUE);
ALTER STAGE @db.schema.stage REFRESH;

-- Parse all PDFs in stage
SELECT 
  relative_path AS file_path,
  AI_PARSE_DOCUMENT(
    TO_FILE('@db.schema.stage', relative_path),
    {'mode': 'LAYOUT'}
  ):content::STRING AS parsed_content
FROM DIRECTORY(@db.schema.stage)
WHERE relative_path LIKE '%.pdf';
```

### Store parsed results

```sql
-- Create table for parsed documents
CREATE TABLE IF NOT EXISTS db.schema.parsed_documents (
  doc_id INT AUTOINCREMENT,
  file_path STRING,
  file_name STRING,
  total_pages INT,
  content TEXT,
  parsed_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Insert parsed content
INSERT INTO db.schema.parsed_documents (file_path, file_name, total_pages, content)
SELECT 
  relative_path,
  SPLIT_PART(relative_path, '/', -1),
  parsed_result:pageCount::INT,
  parsed_result:content::STRING
FROM (
  SELECT 
    relative_path,
    AI_PARSE_DOCUMENT(
      TO_FILE('@db.schema.stage', relative_path),
      {'mode': 'LAYOUT'}
    ) AS parsed_result
  FROM DIRECTORY(@db.schema.stage)
  WHERE relative_path LIKE '%.pdf'
);
```

---

## Large Document Handling (>500 pages)

For documents exceeding 500 pages, parse in chunks:

```sql
-- Parse document in chunks
CREATE OR REPLACE TEMPORARY TABLE doc_chunks AS
SELECT 
  page.value:index::INT AS page_num,
  page.value:content::STRING AS content
FROM (
  SELECT PARSE_JSON(AI_PARSE_DOCUMENT(
    TO_FILE('@stage', 'large_doc.pdf'),
    {'mode': 'LAYOUT', 'page_filter': [{'start': 0, 'end': 500}], 'page_split': true}
  )) AS parsed
  UNION ALL
  SELECT PARSE_JSON(AI_PARSE_DOCUMENT(
    TO_FILE('@stage', 'large_doc.pdf'),
    {'mode': 'LAYOUT', 'page_filter': [{'start': 500, 'end': 1000}], 'page_split': true}
  )) AS parsed
) chunks,
LATERAL FLATTEN(input => chunks.parsed:pages) page;
```

---

## Common Use Cases

### RAG Pipeline Preparation

```sql
-- Parse and chunk for RAG
WITH parsed AS (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@stage', 'manual.pdf'),
    {'mode': 'LAYOUT', 'page_split': true}
  ) AS result
)
SELECT 
  page.value:index::INT AS chunk_id,
  page.value:content::STRING AS chunk_text,
  LENGTH(page.value:content::STRING) AS chunk_length
FROM parsed,
LATERAL FLATTEN(input => result:pages) page;
```

### Full-Text Search Preparation

```sql
-- Extract searchable text
INSERT INTO db.schema.document_search_index (doc_id, file_name, searchable_text)
SELECT 
  doc_id,
  file_name,
  LOWER(content) AS searchable_text
FROM db.schema.parsed_documents;
```

---

## Supported Languages

Arabic, Bengali, Burmese, Cebuano, Chinese, Czech, Dutch, English, French, German, Hebrew, Hindi, Indonesian, Italian, Japanese, Khmer, Korean, Lao, Malay, Persian, Polish, Portuguese, Russian, Spanish, Tagalog, Thai, Turkish, Urdu, Vietnamese

---

For detailed AI_PARSE_DOCUMENT function options, see `reference/ai-parse-doc-and-ai-complete.md`.
