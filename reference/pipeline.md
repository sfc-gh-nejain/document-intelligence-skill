# Pipeline Sub-Skill

Set up automated document processing pipelines using Snowflake streams and tasks.

## When to Use

This sub-skill is triggered after any extraction/parsing/visual analysis flow completes, when the user wants to:
- Automate processing of new documents
- Set up continuous document ingestion
- Build production pipelines for document processing

## Post-Processing Options

**Ask** user what they want to do next:

```
What would you like to do next?
Options:
1. Done - one-time extraction (no pipeline needed)
2. Store results in a Snowflake table
3. Set up a pipeline for continuous processing
```

**Route based on response:**

| User Selection | Action |
|----------------|--------|
| Done | End workflow, display final results |
| Store results | Create table and insert results |
| Set up pipeline | Continue to Pipeline Setup below |

---

## Pipeline Setup

### Pre-Check: File Size and Page Optimization (AI_EXTRACT Only)

**If the user used AI_EXTRACT (Flow A), ALWAYS ask these questions first:**

```
Before setting up your pipeline, I need to understand your document characteristics:

1. Document Size:
   - Do any of your files exceed 125 pages?
   - AI_EXTRACT has a limit of 125 pages per call
   
   Options:
   a. No - all files are within 125 pages
   b. Yes - some files may exceed 125 pages (needs chunking)
   c. Not sure - let me check

2. Page Optimization:
   - Do you need to extract from the entire document, or just specific pages?
   - Extracting fewer pages = faster processing and lower cost
   
   Options:
   a. Extract from entire document (all pages)
   b. Extract from first page only (common for invoices/forms with header info)
   c. Extract from specific page range (e.g., pages 1-5)
   d. Extract from specific pages (e.g., pages 1, 3, 10)
```

**Route based on response:**

| File Size | Page Optimization | Recommended Template |
|-----------|-------------------|---------------------|
| Within 125 pages | All pages | Template A: Simple Extraction |
| Within 125 pages | Specific pages | Template A-2: Page Optimization |
| Exceeds 125 pages | All pages | Template C: Chunking Pipeline |
| Exceeds 125 pages | Specific pages | Template C with page filter |

---

### Step 1: Gather Pipeline Requirements

**Ask** user:

```
What type of pipeline do you need?
Options:
1. Simple extraction pipeline (new files → extract → store)
2. Pipeline with page optimization (extract specific pages only)
3. Pipeline for large documents (>125 pages, needs chunking)
4. Full-document pipeline (preserve context, up to 500 pages)
5. Visual analysis pipeline (charts/diagrams to images → analyze)
```

### Step 2: Configure Pipeline Parameters

**Ask** user:

```
Configure your pipeline:
1. Warehouse name for processing: [default: current warehouse]
2. Schedule frequency: 
   - Every 1 minute
   - Every 5 minutes (recommended)
   - Every 15 minutes
   - Every hour
3. File pattern to match: [default: %.pdf]
```

---

## Pipeline Templates

### Template A: Simple Extraction Pipeline

For documents within page limits (≤125 pages for AI_EXTRACT).

```sql
-- Step 1: Create results table
CREATE TABLE IF NOT EXISTS db.schema.extraction_results (
  result_id INT AUTOINCREMENT,
  file_path STRING,
  file_name STRING,
  extracted_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  -- Add your extraction fields here
  field1 STRING,
  field2 STRING,
  field3 STRING,
  raw_response VARIANT
);

-- Step 2: Create stream on stage
CREATE OR REPLACE STREAM db.schema.doc_stream 
  ON STAGE @db.schema.my_stage;

-- Step 3: Create processing task
CREATE OR REPLACE TASK db.schema.extract_documents_task
  WAREHOUSE = my_warehouse
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('db.schema.doc_stream')
AS
  INSERT INTO db.schema.extraction_results (file_path, file_name, field1, field2, field3, raw_response)
  SELECT 
    relative_path,
    SPLIT_PART(relative_path, '/', -1),
    result:response:field1::STRING,
    result:response:field2::STRING,
    result:response:field3::STRING,
    result
  FROM db.schema.doc_stream,
  LATERAL (
    SELECT AI_EXTRACT(
      file => TO_FILE('@db.schema.my_stage', relative_path),
      responseFormat => {
        'field1': 'What is field1?',
        'field2': 'What is field2?',
        'field3': 'What is field3?'
      }
    ) AS result
  )
  WHERE METADATA$ACTION = 'INSERT'
    AND relative_path LIKE '%.pdf';

-- Step 4: Resume task
ALTER TASK db.schema.extract_documents_task RESUME;
```

### Template A-2: Page Optimization Pipeline

For extracting only specific pages to reduce cost and improve speed.

```sql
-- Step 1: Create page extraction stage and procedure
CREATE STAGE IF NOT EXISTS db.schema.optimized_pages_stage
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  COMMENT = 'Stage for extracted pages';

-- Step 2: Create page extraction procedure
CREATE OR REPLACE PROCEDURE db.schema.extract_specific_pages(
    stage_name STRING, 
    file_name STRING, 
    dest_stage_name STRING,
    page_selection VARIANT  -- integer, array of integers, or {'start': N, 'end': M}
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.9'
PACKAGES = ('snowflake-snowpark-python', 'pypdf2')
HANDLER = 'run'
AS
$
from PyPDF2 import PdfReader, PdfWriter
from snowflake.snowpark import Session
from snowflake.snowpark.file_operation import FileOperation
from io import BytesIO
import os

def run(session: Session, stage_name: str, file_name: str, dest_stage_name: str, page_selection) -> dict:
    result = {"status": "success", "source_file": file_name, "output_file": "", "extracted_pages": []}
    
    try:
        file_url = f"{stage_name}/{file_name}"
        session.file.get(file_url, '/tmp/')
        file_path = os.path.join('/tmp/', file_name)
        
        with open(file_path, 'rb') as f:
            pdf_reader = PdfReader(BytesIO(f.read()))
            num_pages = len(pdf_reader.pages)
            
            # Determine pages (convert to 0-indexed)
            if isinstance(page_selection, int):
                pages = [page_selection - 1]
            elif isinstance(page_selection, list):
                pages = [p - 1 for p in page_selection]
            elif isinstance(page_selection, dict):
                start = page_selection.get('start', 1) - 1
                end = min(page_selection.get('end', num_pages), num_pages)
                pages = list(range(start, end))
            
            pages = [p for p in pages if 0 <= p < num_pages]
            
            writer = PdfWriter()
            for page_num in pages:
                writer.add_page(pdf_reader.pages[page_num])
                result["extracted_pages"].append(page_num + 1)
            
            base_name = os.path.splitext(file_name)[0]
            output_filename = f'{base_name}_optimized.pdf'
            output_path = os.path.join('/tmp/', output_filename)
            
            with open(output_path, 'wb') as out:
                writer.write(out)
            
            FileOperation(session).put(f"file://{output_path}", dest_stage_name, auto_compress=False)
            result["output_file"] = output_filename
            
            os.remove(file_path)
            os.remove(output_path)
                
        return result
    except Exception as e:
        return {"status": "error", "message": str(e)}
$;

-- Step 3: Create results table
CREATE TABLE IF NOT EXISTS db.schema.extraction_results (
  result_id INT AUTOINCREMENT,
  file_path STRING,
  file_name STRING,
  pages_extracted ARRAY,
  extracted_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  field1 STRING,
  field2 STRING,
  field3 STRING,
  raw_response VARIANT
);

-- Step 4: Create stream on source stage
CREATE OR REPLACE STREAM db.schema.doc_stream 
  ON STAGE @db.schema.my_stage;

-- Step 5: Create page extraction task
CREATE OR REPLACE TASK db.schema.extract_pages_task
  WAREHOUSE = my_warehouse
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('db.schema.doc_stream')
AS
DECLARE
  file_record RECORD;
BEGIN
  FOR file_record IN (
    SELECT relative_path 
    FROM db.schema.doc_stream 
    WHERE METADATA$ACTION = 'INSERT' AND relative_path LIKE '%.pdf'
  ) DO
    -- Extract first page only (modify page_selection as needed)
    -- Options: 1 (first page), [1, 3, 5] (specific pages), {'start': 1, 'end': 5} (range)
    CALL db.schema.extract_specific_pages(
      '@db.schema.my_stage',
      :file_record.relative_path,
      '@db.schema.optimized_pages_stage',
      1  -- First page only; change to [1,2,3] or {'start':1,'end':5} as needed
    );
  END FOR;
END;

ALTER TASK db.schema.extract_pages_task RESUME;

-- Step 6: Create stream on optimized pages and extraction task
CREATE OR REPLACE STREAM db.schema.optimized_stream 
  ON STAGE @db.schema.optimized_pages_stage;

CREATE OR REPLACE TASK db.schema.extract_from_optimized_task
  WAREHOUSE = my_warehouse
  SCHEDULE = '2 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('db.schema.optimized_stream')
AS
  INSERT INTO db.schema.extraction_results (file_path, file_name, field1, field2, field3, raw_response)
  SELECT 
    relative_path,
    SPLIT_PART(relative_path, '/', -1),
    result:response:field1::STRING,
    result:response:field2::STRING,
    result:response:field3::STRING,
    result
  FROM db.schema.optimized_stream,
  LATERAL (
    SELECT AI_EXTRACT(
      file => TO_FILE('@db.schema.optimized_pages_stage', relative_path),
      responseFormat => {
        'field1': 'What is field1?',
        'field2': 'What is field2?',
        'field3': 'What is field3?'
      }
    ) AS result
  )
  WHERE METADATA$ACTION = 'INSERT';

ALTER TASK db.schema.extract_from_optimized_task RESUME;
```

### Template B: Parsing Pipeline

For full-text extraction with AI_PARSE_DOCUMENT.

```sql
-- Step 1: Create results table
CREATE TABLE IF NOT EXISTS db.schema.parsed_documents (
  doc_id INT AUTOINCREMENT,
  file_path STRING,
  file_name STRING,
  total_pages INT,
  content TEXT,
  parsed_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Step 2: Create stream on stage
CREATE OR REPLACE STREAM db.schema.parse_stream 
  ON STAGE @db.schema.my_stage;

-- Step 3: Create processing task
CREATE OR REPLACE TASK db.schema.parse_documents_task
  WAREHOUSE = my_warehouse
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('db.schema.parse_stream')
AS
  INSERT INTO db.schema.parsed_documents (file_path, file_name, total_pages, content)
  SELECT 
    relative_path,
    SPLIT_PART(relative_path, '/', -1),
    parsed_result:pageCount::INT,
    parsed_result:content::STRING
  FROM db.schema.parse_stream,
  LATERAL (
    SELECT AI_PARSE_DOCUMENT(
      TO_FILE('@db.schema.my_stage', relative_path),
      {'mode': 'LAYOUT'}
    ) AS parsed_result
  )
  WHERE METADATA$ACTION = 'INSERT'
    AND relative_path LIKE '%.pdf';

-- Step 4: Resume task
ALTER TASK db.schema.parse_documents_task RESUME;
```

### Template C: Chunking Pipeline (Large Documents)

For documents exceeding 125 pages that need to be split into chunks.

```sql
-- Step 1: Create stages
CREATE STAGE IF NOT EXISTS db.schema.chunks_stage
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  COMMENT = 'Stage for document chunks';

-- Step 2: Create chunking stored procedure
CREATE OR REPLACE PROCEDURE db.schema.split_document_into_chunks(
    stage_name STRING, 
    file_name STRING, 
    dest_stage_name STRING,
    chunk_size INT DEFAULT 125
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.9'
PACKAGES = ('snowflake-snowpark-python', 'pypdf2')
HANDLER = 'run'
AS
$
from PyPDF2 import PdfReader, PdfWriter
from snowflake.snowpark import Session
from snowflake.snowpark.file_operation import FileOperation
from io import BytesIO
import os
import math

def run(session: Session, stage_name: str, file_name: str, dest_stage_name: str, chunk_size: int = 125) -> dict:
    result = {
        "status": "success",
        "source_file": file_name,
        "total_pages": 0,
        "chunks_created": 0,
        "chunk_files": []
    }
    
    try:
        file_url = f"{stage_name}/{file_name}"
        get_result = session.file.get(file_url, '/tmp/')
        file_path = os.path.join('/tmp/', file_name)
        
        if not os.path.exists(file_path):
            return {"status": "error", "message": f"File {file_name} not found"}

        with open(file_path, 'rb') as f:
            pdf_data = f.read()
            pdf_reader = PdfReader(BytesIO(pdf_data))
            num_pages = len(pdf_reader.pages)
            result["total_pages"] = num_pages
            
            num_chunks = math.ceil(num_pages / chunk_size)
            result["chunks_created"] = num_chunks
            
            base_name = os.path.splitext(file_name)[0]
            
            for chunk_num in range(num_chunks):
                start_page = chunk_num * chunk_size
                end_page = min((chunk_num + 1) * chunk_size, num_pages)
                
                writer = PdfWriter()
                for page_num in range(start_page, end_page):
                    writer.add_page(pdf_reader.pages[page_num])
                
                chunk_filename = f'{base_name}_chunk_{chunk_num + 1}_pages_{start_page + 1}_{end_page}.pdf'
                chunk_file_path = os.path.join('/tmp/', chunk_filename)
                
                with open(chunk_file_path, 'wb') as output_file:
                    writer.write(output_file)

                FileOperation(session).put(
                    f"file://{chunk_file_path}",
                    dest_stage_name,
                    auto_compress=False
                )
                
                result["chunk_files"].append({
                    "chunk_number": chunk_num + 1,
                    "filename": chunk_filename,
                    "start_page": start_page + 1,
                    "end_page": end_page,
                    "page_count": end_page - start_page
                })
                
                os.remove(chunk_file_path)
            
            os.remove(file_path)
                
        return result
        
    except Exception as e:
        return {"status": "error", "message": str(e)}
$;

-- Step 3: Create tracking tables
CREATE TABLE IF NOT EXISTS db.schema.document_chunks (
  chunk_id INT AUTOINCREMENT,
  source_file STRING,
  chunk_file STRING,
  chunk_number INT,
  start_page INT,
  end_page INT,
  page_count INT,
  chunked_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  processed BOOLEAN DEFAULT FALSE
);

CREATE TABLE IF NOT EXISTS db.schema.extraction_results (
  result_id INT AUTOINCREMENT,
  source_file STRING,
  chunk_file STRING,
  chunk_number INT,
  file_name STRING,
  extracted_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  field1 STRING,
  field2 STRING,
  field3 STRING,
  raw_response VARIANT
);

-- Step 4: Create aggregated results view
CREATE OR REPLACE VIEW db.schema.extraction_results_aggregated AS
SELECT 
  source_file,
  ARRAY_AGG(OBJECT_CONSTRUCT(
    'chunk_number', chunk_number,
    'field1', field1,
    'field2', field2,
    'field3', field3
  )) WITHIN GROUP (ORDER BY chunk_number) AS chunk_results,
  MAX(CASE WHEN chunk_number = 1 OR chunk_number IS NULL THEN field1 END) AS field1,
  MAX(CASE WHEN chunk_number = 1 OR chunk_number IS NULL THEN field2 END) AS field2,
  COUNT(*) AS total_chunks,
  MIN(extracted_at) AS first_extracted_at,
  MAX(extracted_at) AS last_extracted_at
FROM db.schema.extraction_results
GROUP BY source_file;

-- Step 5: Create chunking task
CREATE OR REPLACE STREAM db.schema.doc_source_stream 
  ON STAGE @db.schema.my_stage;

CREATE OR REPLACE TASK db.schema.chunk_documents_task
  WAREHOUSE = my_warehouse
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('db.schema.doc_source_stream')
AS
DECLARE
  file_record RECORD;
  chunk_result VARIANT;
BEGIN
  FOR file_record IN (
    SELECT relative_path 
    FROM db.schema.doc_source_stream 
    WHERE METADATA$ACTION = 'INSERT' 
      AND relative_path LIKE '%.pdf'
  ) DO
    LET page_count INT := (
      SELECT AI_PARSE_DOCUMENT(
        TO_FILE('@db.schema.my_stage', :file_record.relative_path),
        {'mode': 'OCR', 'page_filter': [{'start': 0, 'end': 1}]}
      ):pageCount::INT
    );
    
    IF (page_count > 125) THEN
      chunk_result := (
        CALL db.schema.split_document_into_chunks(
          '@db.schema.my_stage',
          :file_record.relative_path,
          '@db.schema.chunks_stage',
          125
        )
      );
      
      INSERT INTO db.schema.document_chunks (source_file, chunk_file, chunk_number, start_page, end_page, page_count)
      SELECT 
        :file_record.relative_path,
        f.value:filename::STRING,
        f.value:chunk_number::INT,
        f.value:start_page::INT,
        f.value:end_page::INT,
        f.value:page_count::INT
      FROM TABLE(FLATTEN(input => :chunk_result:chunk_files)) f;
    ELSE
      INSERT INTO db.schema.document_chunks (source_file, chunk_file, chunk_number, start_page, end_page, page_count)
      VALUES (:file_record.relative_path, :file_record.relative_path, 1, 1, :page_count, :page_count);
    END IF;
  END FOR;
END;

ALTER TASK db.schema.chunk_documents_task RESUME;

-- Step 6: Create extraction task (processes chunks)
CREATE OR REPLACE STREAM db.schema.chunks_to_process_stream 
  ON TABLE db.schema.document_chunks
  SHOW_INITIAL_ROWS = TRUE;

CREATE OR REPLACE TASK db.schema.extract_from_chunks_task
  WAREHOUSE = my_warehouse
  SCHEDULE = '2 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('db.schema.chunks_to_process_stream')
AS
  INSERT INTO db.schema.extraction_results (source_file, chunk_file, chunk_number, file_name, field1, field2, field3, raw_response)
  SELECT 
    dc.source_file,
    dc.chunk_file,
    dc.chunk_number,
    SPLIT_PART(dc.chunk_file, '/', -1),
    result:response:field1::STRING,
    result:response:field2::STRING,
    result:response:field3::STRING,
    result
  FROM db.schema.chunks_to_process_stream dc,
  LATERAL (
    SELECT AI_EXTRACT(
      file => TO_FILE(
        CASE 
          WHEN dc.chunk_file = dc.source_file THEN '@db.schema.my_stage'
          ELSE '@db.schema.chunks_stage'
        END, 
        dc.chunk_file
      ),
      responseFormat => {
        'field1': 'What is field1?',
        'field2': 'What is field2?',
        'field3': 'What is field3?'
      }
    ) AS result
  )
  WHERE dc.METADATA$ACTION = 'INSERT';

ALTER TASK db.schema.extract_from_chunks_task RESUME;
```

### Template D: Full-Document Pipeline (AI_PARSE_DOCUMENT + AI_COMPLETE)

For documents that must be processed as a whole with full context preservation. Supports up to 500 pages.

```sql
-- Step 1: Create results table
CREATE TABLE IF NOT EXISTS db.schema.extraction_results_full_doc (
  result_id INT AUTOINCREMENT,
  file_path STRING,
  file_name STRING,
  total_pages INT,
  extraction_method STRING DEFAULT 'AI_PARSE_DOCUMENT + AI_COMPLETE',
  extracted_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  -- Add your extraction fields here
  field1 STRING,
  field2 STRING,
  field3 VARIANT,
  raw_response VARIANT
);

-- Step 2: Create stream and task
CREATE OR REPLACE STREAM db.schema.full_doc_stream 
  ON STAGE @db.schema.my_stage;

CREATE OR REPLACE TASK db.schema.full_document_extraction_task
  WAREHOUSE = my_warehouse
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('db.schema.full_doc_stream')
AS
  INSERT INTO db.schema.extraction_results_full_doc (
    file_path, file_name, total_pages, field1, field2, field3, raw_response
  )
  SELECT 
    relative_path,
    SPLIT_PART(relative_path, '/', -1),
    parsed_result:pageCount::INT,
    PARSE_JSON(extraction_result):field1::STRING,
    PARSE_JSON(extraction_result):field2::STRING,
    PARSE_JSON(extraction_result):field3,
    PARSE_JSON(extraction_result)
  FROM (
    SELECT 
      relative_path,
      AI_PARSE_DOCUMENT(
        TO_FILE('@db.schema.my_stage', relative_path),
        {'mode': 'LAYOUT'}
      ) AS parsed_result,
      AI_COMPLETE(
        'claude-3-5-sonnet',
        'Extract the following information from this document.

DOCUMENT:
' || AI_PARSE_DOCUMENT(
          TO_FILE('@db.schema.my_stage', relative_path),
          {'mode': 'LAYOUT'}
        ):content::STRING || '

Extract: field1, field2, field3',
        {
          'response_format': {
            'type': 'json',
            'schema': {
              'type': 'object',
              'properties': {
                'field1': {'type': 'string'},
                'field2': {'type': 'string'},
                'field3': {'type': 'object'}
              },
              'required': ['field1', 'field2']
            }
          },
          'max_tokens': 8192
        }
      ) AS extraction_result
    FROM db.schema.full_doc_stream
    WHERE METADATA$ACTION = 'INSERT'
      AND relative_path LIKE '%.pdf'
  );

ALTER TASK db.schema.full_document_extraction_task RESUME;
```

### Template E: Visual Analysis Pipeline

For processing images or PDFs containing charts, blueprints, diagrams.

```sql
-- Step 1: Create stages
CREATE STAGE IF NOT EXISTS db.schema.images_stage
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  COMMENT = 'Stage for converted images';

-- Step 2: Create PDF-to-image conversion procedure
CREATE OR REPLACE PROCEDURE db.schema.convert_pdf_to_images(
    stage_name STRING, 
    file_name STRING, 
    dest_stage_name STRING,
    dpi INT DEFAULT 200,
    image_format STRING DEFAULT 'PNG',
    specific_pages ARRAY DEFAULT NULL
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.9'
PACKAGES = ('snowflake-snowpark-python', 'pdf2image', 'pillow')
HANDLER = 'run'
AS
$
from pdf2image import convert_from_bytes
from snowflake.snowpark import Session
from snowflake.snowpark.file_operation import FileOperation
from io import BytesIO
import os

def run(session: Session, stage_name: str, file_name: str, dest_stage_name: str, 
        dpi: int = 200, image_format: str = 'PNG', specific_pages: list = None) -> dict:
    result = {
        "status": "success",
        "source_file": file_name,
        "total_pages": 0,
        "images_created": 0,
        "image_files": []
    }
    
    try:
        file_url = f"{stage_name}/{file_name}"
        get_result = session.file.get(file_url, '/tmp/')
        file_path = os.path.join('/tmp/', file_name)
        
        if not os.path.exists(file_path):
            return {"status": "error", "message": f"File {file_name} not found in {stage_name}"}

        with open(file_path, 'rb') as f:
            pdf_data = f.read()
        
        if specific_pages:
            images = convert_from_bytes(
                pdf_data, 
                dpi=dpi, 
                fmt=image_format.lower(),
                first_page=min(specific_pages),
                last_page=max(specific_pages)
            )
            page_indices = [p - min(specific_pages) for p in specific_pages]
            images = [images[i] for i in page_indices if i < len(images)]
        else:
            images = convert_from_bytes(pdf_data, dpi=dpi, fmt=image_format.lower())
        
        result["total_pages"] = len(images)
        
        base_name = os.path.splitext(file_name)[0]
        ext = image_format.lower()
        if ext == 'jpeg':
            ext = 'jpg'
        
        for i, image in enumerate(images):
            page_num = specific_pages[i] if specific_pages else i + 1
            
            image_filename = f'{base_name}_page_{page_num}.{ext}'
            image_file_path = os.path.join('/tmp/', image_filename)
            
            image.save(image_file_path, format=image_format.upper())
            
            if not os.path.exists(image_file_path):
                return {"status": "error", "message": f"Failed to create image {image_filename}"}

            FileOperation(session).put(
                f"file://{image_file_path}",
                dest_stage_name,
                auto_compress=False
            )
            
            result["image_files"].append({
                "page_number": page_num,
                "filename": image_filename,
                "format": image_format.upper()
            })
            result["images_created"] += 1
            
            os.remove(image_file_path)
        
        os.remove(file_path)
            
        return result
        
    except Exception as e:
        return {"status": "error", "message": str(e)}
$;

-- Step 3: Create results table
CREATE TABLE IF NOT EXISTS db.schema.visual_analysis_results (
  result_id INT AUTOINCREMENT,
  image_path STRING,
  source_pdf STRING,
  page_number INT,
  analysis_type STRING,
  analysis_result VARIANT,
  analyzed_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Step 4: Create stream and task for images
CREATE OR REPLACE STREAM db.schema.images_stream 
  ON STAGE @db.schema.images_stage;

CREATE OR REPLACE TASK db.schema.analyze_images_task
  WAREHOUSE = my_warehouse
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('db.schema.images_stream')
AS
  INSERT INTO db.schema.visual_analysis_results (image_path, analysis_type, analysis_result)
  SELECT 
    relative_path,
    'visual_analysis',
    PARSE_JSON(AI_COMPLETE(
      'claude-3-5-sonnet',
      [
        {
          'role': 'user',
          'content': [
            {'type': 'image', 'image_url': {'url': TO_FILE('@db.schema.images_stage', relative_path)}},
            {'type': 'text', 'text': 'Analyze this image. Extract all visible data, text, and key information. Return as JSON.'}
          ]
        }
      ],
      {'max_tokens': 4096}
    ))
  FROM db.schema.images_stream
  WHERE METADATA$ACTION = 'INSERT'
    AND (relative_path LIKE '%.png' OR relative_path LIKE '%.jpg');

ALTER TASK db.schema.analyze_images_task RESUME;
```

---

## Pipeline Management

### Monitor Pipeline Status

```sql
-- Check task history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
  TASK_NAME => 'extract_documents_task',
  SCHEDULED_TIME_RANGE_START => DATEADD('hour', -24, CURRENT_TIMESTAMP())
))
ORDER BY SCHEDULED_TIME DESC;

-- Check stream status
SHOW STREAMS LIKE 'doc%';

-- View pending files in stream
SELECT * FROM db.schema.doc_stream;
```

### Pause/Resume Pipeline

```sql
-- Pause task
ALTER TASK db.schema.extract_documents_task SUSPEND;

-- Resume task
ALTER TASK db.schema.extract_documents_task RESUME;
```

### Modify Pipeline Schedule

```sql
-- Change schedule to every 15 minutes
ALTER TASK db.schema.extract_documents_task 
  SET SCHEDULE = '15 MINUTE';
```

---

## Troubleshooting

### Task not running
- Check if task is resumed: `SHOW TASKS LIKE 'task_name';`
- Check if stream has data: `SELECT COUNT(*) FROM stream_name;`
- Verify warehouse is available

### Stream shows no data
- Refresh stage directory: `ALTER STAGE @stage_name REFRESH;`
- Check file pattern matches: `SELECT * FROM DIRECTORY(@stage_name);`

### Extraction errors
- Check task history for error messages
- Verify file format is supported
- Check file size and page limits
