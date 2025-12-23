# DECIMER.ai API Documentation

## Overview

DECIMER.ai is a web application built with Laravel (PHP) that provides chemical structure recognition and segmentation services. This document details the API endpoints for uploading files and receiving SMILES (Simplified Molecular Input Line Entry System) output.

**Source Repository**: https://github.com/Steinbeck-Lab/DECIMER.ai

---

## Architecture

The application uses a Laravel backend (PHP) that communicates with Python microservices via socket connections for:
- Chemical structure recognition (DECIMER OCSR)
- Chemical structure segmentation
- Structure classification
- SMILES validation

---

## File Upload Endpoint

### POST /file-upload

**Controller**: `FileUploadController::fileUploadPost()`

**Purpose**: Upload chemical structure images or PDF documents for processing.

**Route**: 
```php
Route::post('/file-upload', [FileUploadController::class, 'fileUploadPost'])
    ->name('file.upload.post');
```

**Request Format**:
- Method: `POST`
- Content-Type: `multipart/form-data`
- Field Name: `file[]` (array of files)

**Supported File Formats**:
- PDF documents
- Images: JPG/JPEG, PNG, WEBP, HEIC

**Processing Logic**:

1. **File Validation**:
   - Checks file extensions
   - Cleans filenames by removing special characters
   - Stores files in `storage/app/public/media/`

2. **PDF Processing**:
   - Converts PDF to images using Python script
   - Command: `python3 ../app/Python/convert_pdf_to_images.py <file_path>`

3. **Image Processing**:
   - Normalizes image format using Python script
   - Command: `python3 ../app/Python/normalise_img_format.py <file_path>`

4. **Output**:
   - Returns image paths as JSON
   - Format: `structure_depiction_img_paths` (JSON encoded array)

**Response Structure**:
```php
[
    'success_message' => 'The file was loaded successfully.',
    'file_name' => 'uploaded_filename.png',
    'img_paths' => '[]',  // For PDF processing
    'structure_depiction_img_paths' => '["storage/media/image1.png", ...]'
]
```

---

## Chemical Structure Segmentation Endpoint

### POST /decimer-segmentation

**Controller**: `DecimerSegmentationController::DecimerSegmentationPost()`

**Purpose**: Segment chemical structures from document pages.

**Route**:
```php
Route::post('/decimer-segmentation', [DecimerSegmentationController::class, 'DecimerSegmentationPost'])
    ->name('decimer.segmentation.post');
```

**Request Format**:
```php
[
    'img_paths' => '["/path/to/page1.png", "/path/to/page2.png"]'
]
```

**Processing**:
1. Calls Python segmentation client
2. Command: `python3 ../app/Python/decimer_segmentation_client.py <json_paths>`
3. Python script connects to segmentation server via socket (ports 65438-65440)

**Response**:
```php
[
    'img_paths' => '["/path/to/page1.png"]',
    'structure_depiction_img_paths' => '["/path/to/structure1.png", "/path/to/structure2.png"]',
    'has_segmentation_already_run' => 'true'
]
```

---

## SMILES Generation Endpoint

### POST /decimer-ocsr

**Controller**: `DecimerController::DecimerOCSRPost()`

**Purpose**: Generate SMILES strings from chemical structure images.

**Route**:
```php
Route::post('/decimer-ocsr', [DecimerController::class, 'DecimerOCSRPost'])
    ->name('decimer.ocsr.post');
```

**Request Format**:
```php
[
    'structure_depiction_img_paths' => '["/path/to/structure1.png", "/path/to/structure2.png"]',
    'img_paths' => '[]',
    'has_segmentation_already_run' => 'true' (optional)
]
```

**Processing Pipeline**:

1. **SMILES Generation**:
   - Command: `python3 ../app/Python/decimer_predictor_client.py <json_paths>`
   - Python script connects to DECIMER OCSR servers via socket (ports 65432-65434)
   - Uses multiprocessing for parallel processing across multiple ports
   - Returns: Array of SMILES strings

2. **SMILES Validation**:
   - Command: `python3 ../app/Python/check_smiles_validity.py <smiles_json>`
   - Validates each SMILES string
   - Returns: Array of validity status ('valid' or 'invalid')

3. **InChIKey Generation**:
   - Command: `python3 ../app/Python/get_inchikey_list_from_smiles.py <smiles_json>`
   - Generates InChIKeys for valid SMILES
   - Returns: Array of InChIKey strings

4. **Structure Classification**:
   - Command: `python3 ../app/Python/decimer_classifier_client.py <json_paths>`
   - Classifies structures (chemical vs non-chemical)
   - Returns: Array of boolean values

**Response Structure**:
```php
[
    'img_paths' => '[]',
    'structure_depiction_img_paths' => '["/path/to/structure1.png", "/path/to/structure2.png"]',
    'smiles_array' => '["CCO", "CC(=O)O"]',  // SMILES strings
    'validity_array' => '["valid", "valid"]',
    'inchikey_array' => '["LFQSCWFLJHTTHZ-UHFFFAOYSA-N", "QTBSBXVTEAMEQO-UHFFFAOYSA-N"]',
    'classifier_result_array' => '["True", "True"]',
    'has_segmentation_already_run' => 'true',
    'single_image_upload' => 'false'
]
```

**Limitations**:
- Maximum 20 structures per request
- Structures beyond 20 are ignored in processing but padded with empty strings in output

---

## Python Client: SMILES Generation

**File**: `app/Python/decimer_predictor_client.py`

**Function**: Connects to DECIMER OCSR microservices to generate SMILES from images.

**Socket Communication**:
- Host: `supervisor`
- Ports: 65432-65434 (auto-discovery of available ports)
- Protocol: TCP socket
- Message Format: Send image path, receive SMILES string

**Code Structure**:

```python
def send_and_receive(path, port):
    """Send image path to OCSR server and receive SMILES."""
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect(("supervisor", port))
        s.sendall(path.encode("utf-8"))
        data = s.recv(32768)
        return data.decode("utf-8")
```

**Load Balancing**:
- Auto-discovers available ports in range 65432-65434
- Distributes work across multiple OCSR servers using round-robin
- Uses multiprocessing pool for parallel processing

**Usage**:
```bash
python3 decimer_predictor_client.py '["/path/to/structure1.png", "/path/to/structure2.png"]'
```

**Output**:
```json
["CCO", "CC(=O)O"]
```

---

## Complete Workflow

### Scenario 1: Upload Images Directly

1. **Upload File(s)**:
   ```
   POST /file-upload
   Files: [structure1.png, structure2.png]
   ```

2. **Receive Image Paths**:
   ```json
   {
       "structure_depiction_img_paths": "[\"storage/media/structure1.png\", \"storage/media/structure2.png\"]"
   }
   ```

3. **Generate SMILES**:
   ```
   POST /decimer-ocsr
   Body: {
       "structure_depiction_img_paths": "[\"storage/media/structure1.png\", \"storage/media/structure2.png\"]"
   }
   ```

4. **Receive SMILES Output**:
   ```json
   {
       "smiles_array": "[\"CCO\", \"CC(=O)O\"]",
       "validity_array": "[\"valid\", \"valid\"]",
       "inchikey_array": "[\"LFQSCWFLJHTTHZ-UHFFFAOYSA-N\", \"QTBSBXVTEAMEQO-UHFFFAOYSA-N\"]"
   }
   ```

### Scenario 2: Upload PDF Document

1. **Upload PDF**:
   ```
   POST /file-upload
   Files: [document.pdf]
   ```

2. **Receive Page Image Paths**:
   ```json
   {
       "img_paths": "[\"storage/media/page1.png\", \"storage/media/page2.png\"]"
   }
   ```

3. **Segment Structures**:
   ```
   POST /decimer-segmentation
   Body: {
       "img_paths": "[\"storage/media/page1.png\", \"storage/media/page2.png\"]"
   }
   ```

4. **Receive Segmented Structure Paths**:
   ```json
   {
       "structure_depiction_img_paths": "[\"storage/media/structure1.png\", \"storage/media/structure2.png\", ...]"
   }
   ```

5. **Generate SMILES** (same as Scenario 1, step 3-4)

---

## Additional Endpoints

### Clipboard Paste

**Endpoint**: `POST /clipboard-paste`

**Controller**: `ClipboardController::store()`

**Purpose**: Paste chemical structure images from clipboard.

### Archive Creation

**Endpoint**: `POST /archive-creation`

**Controller**: `ResultArchiveController::archiveCreationPost()`

**Purpose**: Create downloadable archive of results.

---

## Microservices Architecture

The Laravel application communicates with Python microservices:

1. **DECIMER OCSR Servers** (Ports 65432-65434):
   - Generate SMILES from structure images
   - Multiple instances for load balancing

2. **DECIMER Segmentation Servers** (Ports 65438-65440):
   - Segment chemical structures from document pages
   - Multiple instances for parallel processing

3. **DECIMER Classifier Servers**:
   - Classify structures as chemical/non-chemical

These microservices run as separate processes managed by Supervisor and communicate via TCP sockets.

---

## Key Files Reference

### Controllers
- `app/Http/Controllers/FileUploadController.php` - File upload handling
- `app/Http/Controllers/DecimerController.php` - SMILES generation
- `app/Http/Controllers/DecimerSegmentationController.php` - Structure segmentation

### Python Clients
- `app/Python/decimer_predictor_client.py` - SMILES generation client
- `app/Python/decimer_segmentation_client.py` - Segmentation client
- `app/Python/decimer_classifier_client.py` - Classification client
- `app/Python/check_smiles_validity.py` - SMILES validation
- `app/Python/get_inchikey_list_from_smiles.py` - InChIKey generation
- `app/Python/convert_pdf_to_images.py` - PDF to image conversion
- `app/Python/normalise_img_format.py` - Image format normalization

### Routes
- `routes/web.php` - All application routes

---

## Security & Limitations

### Security Measures
- Filename sanitization removes special characters
- File extension validation
- Old file cleanup (files older than 1 hour are removed)

### Limitations
- Maximum 20 structures processed per OCSR request
- Supported file formats: PDF, PNG, JPG/JPEG, WEBP, HEIC
- Cannot mix PDF and image uploads in single request
- Socket timeout: 1 second for port discovery
- Maximum execution time: 300 seconds for segmentation

---

## Error Handling

**Common Errors**:

1. **Invalid File Format**:
   ```
   'Invalid file! Valid formats: pdf, png, jpg/jpeg, webp, HEIC'
   ```

2. **Mixed Input Types**:
   ```
   'Invalid mixed inputs! Please upload a pdf document or upload chemical structure images.'
   ```

3. **No Structure Images**:
   ```
   'No structure images to process'
   ```

4. **Processing Failures**:
   - Returns empty arrays for failed SMILES generation
   - Pads results with empty strings for structures beyond limit
   - Logs errors to Laravel log files

---

## Example cURL Requests

### Upload Image File
```bash
curl -X POST https://decimer.ai/file-upload \
  -F "file[]=@structure1.png" \
  -F "file[]=@structure2.png"
```

### Generate SMILES
```bash
curl -X POST https://decimer.ai/decimer-ocsr \
  -d 'structure_depiction_img_paths=["storage/media/structure1.png"]' \
  -d 'img_paths=[]'
```

---

## Conclusion

The DECIMER.ai API provides a complete pipeline for:
1. **File Upload** (`/file-upload`) - Upload images or PDFs
2. **Structure Segmentation** (`/decimer-segmentation`) - Extract structures from documents
3. **SMILES Generation** (`/decimer-ocsr`) - Generate SMILES strings with validation

The system uses a microservices architecture with PHP Laravel frontend and Python backend services communicating via TCP sockets for efficient parallel processing.
