# DECIMER.ai API Quick Reference

## Key API Endpoints Summary

### 1. File Upload Endpoint
**URL**: `POST /file-upload`  
**Purpose**: Upload chemical structure images or PDF documents  
**Input**: Multipart form data with `file[]` field  
**Output**: Image paths as JSON  

```
Input: [structure.png, structure2.png]
   ↓
Processing: File validation, normalization
   ↓
Output: structure_depiction_img_paths JSON array
```

### 2. Segmentation Endpoint (for PDFs)
**URL**: `POST /decimer-segmentation`  
**Purpose**: Extract chemical structures from document pages  
**Input**: Page image paths  
**Output**: Segmented structure image paths  

```
Input: img_paths = ["page1.png", "page2.png"]
   ↓
Python Socket Client → Segmentation Server (ports 65438-65440)
   ↓
Output: structure_depiction_img_paths = ["struct1.png", "struct2.png", ...]
```

### 3. SMILES Generation Endpoint ⭐
**URL**: `POST /decimer-ocsr`  
**Purpose**: Generate SMILES strings from structure images  
**Input**: Structure image paths  
**Output**: SMILES strings + validation + InChIKeys  

```
Input: structure_depiction_img_paths = ["struct1.png", "struct2.png"]
   ↓
Python Socket Client → OCSR Servers (ports 65432-65434)
   ↓
Parallel Processing: Load balanced across multiple servers
   ↓
Output: 
  - smiles_array = ["CCO", "CC(=O)O"]
  - validity_array = ["valid", "valid"]
  - inchikey_array = ["LFQSCWFL...", "QTBSBXVT..."]
  - classifier_result_array = ["True", "True"]
```

---

## Complete Workflow Diagram

### Direct Image Upload → SMILES
```
┌─────────────────────┐
│   Upload Images     │
│  POST /file-upload  │
│  [img1.png, img2]   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ FileUploadController│
│  - Validate files   │
│  - Normalize format │
│  - Store in media   │
└──────────┬──────────┘
           │
           ▼
structure_depiction_img_paths: ["storage/media/img1.png", ...]
           │
           ▼
┌─────────────────────┐
│  Generate SMILES    │
│ POST /decimer-ocsr  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  DecimerController  │
│   - Call Python     │
│     OCSR client     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────┐
│   Python Socket Communication   │
│                                 │
│  decimer_predictor_client.py    │
│      ↓                          │
│  Connect to ports 65432-65434   │
│      ↓                          │
│  Send: image_path               │
│  Recv: SMILES string            │
│      ↓                          │
│  Parallel processing Pool       │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────┐
│  Post-Processing    │
│  - Validate SMILES  │
│  - Generate InChIKey│
│  - Classify struct  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────┐
│         Final Output            │
│                                 │
│  smiles_array: ["CCO", ...]     │
│  validity_array: ["valid", ...] │
│  inchikey_array: ["LFQS...", ...]│
│  classifier_result_array: [...]  │
└─────────────────────────────────┘
```

### PDF Upload → Segmentation → SMILES
```
┌─────────────────────┐
│    Upload PDF       │
│  POST /file-upload  │
│   [document.pdf]    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  PDF → Images       │
│  Python: convert    │
│  _pdf_to_images.py  │
└──────────┬──────────┘
           │
           ▼
img_paths: ["page1.png", "page2.png", ...]
           │
           ▼
┌─────────────────────┐
│  Segment Structures │
│POST/decimer-segment │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────┐
│  Python Socket: Segmentation    │
│                                 │
│  decimer_segmentation_client.py │
│      ↓                          │
│  Connect to ports 65438-65440   │
│      ↓                          │
│  Send: page_paths               │
│  Recv: structure_paths          │
└──────────┬──────────────────────┘
           │
           ▼
structure_depiction_img_paths: ["struct1.png", ...]
           │
           ▼
┌─────────────────────┐
│  Generate SMILES    │
│  (same as above)    │
└─────────────────────┘
```

---

## Socket Communication Architecture

```
┌──────────────────────────────────────────────────┐
│           Laravel Application (PHP)              │
│                                                  │
│  Controllers:                                    │
│  - FileUploadController                          │
│  - DecimerController                             │
│  - DecimerSegmentationController                 │
└────────────┬─────────────────────────────────────┘
             │
             │ exec() Python scripts
             ▼
┌──────────────────────────────────────────────────┐
│         Python Client Scripts                    │
│                                                  │
│  - decimer_predictor_client.py                   │
│  - decimer_segmentation_client.py                │
│  - decimer_classifier_client.py                  │
└────────┬─────────────────────────────────────────┘
         │
         │ TCP Socket Connection
         │ (Host: "supervisor")
         ▼
┌──────────────────────────────────────────────────┐
│      Python Microservices (Servers)              │
│                                                  │
│  OCSR Servers:       Ports 65432-65434           │
│  - decimer_predictor_server.py                   │
│  - Load balanced across 3 instances              │
│                                                  │
│  Segmentation:       Ports 65438-65440           │
│  - decimer_segmentation_server.py                │
│                                                  │
│  Classifier:                                     │
│  - decimer_classifier_server.py                  │
│                                                  │
│  Other Services:                                 │
│  - STOUT predictor (Name → SMILES)               │
└──────────────────────────────────────────────────┘
```

---

## Request/Response Examples

### Example 1: Direct Image Upload
**Request:**
```http
POST /file-upload HTTP/1.1
Content-Type: multipart/form-data

file[]: structure1.png
file[]: structure2.png
```

**Response:**
```json
{
    "success_message": "The file was loaded successfully.",
    "structure_depiction_img_paths": "[\"storage/media/structure1.png\",\"storage/media/structure2.png\"]"
}
```

### Example 2: Generate SMILES
**Request:**
```http
POST /decimer-ocsr HTTP/1.1
Content-Type: application/x-www-form-urlencoded

structure_depiction_img_paths=["storage/media/structure1.png","storage/media/structure2.png"]
```

**Response:**
```json
{
    "smiles_array": "[\"CCO\",\"CC(=O)O\"]",
    "validity_array": "[\"valid\",\"valid\"]",
    "inchikey_array": "[\"LFQSCWFLJHTTHZ-UHFFFAOYSA-N\",\"QTBSBXVTEAMEQO-UHFFFAOYSA-N\"]",
    "classifier_result_array": "[\"True\",\"True\"]",
    "structure_depiction_img_paths": "[\"storage/media/structure1.png\",\"storage/media/structure2.png\"]"
}
```

---

## Python Socket Client Code Example

```python
import sys
import json
import socket
from multiprocessing import Pool
from itertools import cycle

def send_and_receive(path, port):
    """Send image path to OCSR server and receive SMILES."""
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect(("supervisor", port))
        s.sendall(path.encode("utf-8"))
        data = s.recv(32768)
        return data.decode("utf-8")

# Load balance across ports
available_ports = [65432, 65433, 65434]
ports = cycle(available_ports)
paths = ["/path/to/struct1.png", "/path/to/struct2.png"]

# Create work tuples
starmap_tuples = [(path, next(ports)) for path in paths]

# Process in parallel
with Pool(len(paths)) as p:
    SMILES = p.starmap(send_and_receive, starmap_tuples)

print(json.dumps(SMILES))
# Output: ["CCO", "CC(=O)O"]
```

---

## Important Notes

### ⚠️ Limitations
- **Maximum 20 structures** per OCSR request
- Structures beyond 20 are ignored
- Cannot mix PDF and image uploads in single request

### 🔒 Security
- Filenames sanitized (special chars removed)
- File extensions validated
- Old files auto-cleaned (1 hour retention)

### ⚡ Performance
- Parallel processing via multiprocessing
- Load balancing across multiple OCSR servers
- Socket timeout: 1 second for discovery
- Max execution time: 300 seconds (segmentation)

### 📊 Data Flow Summary
```
Upload → Normalize → (Segment if PDF) → OCSR → Validate → InChIKey → Output
```

---

## Testing with cURL

### Test File Upload
```bash
curl -X POST https://decimer.ai/file-upload \
  -F "file[]=@/path/to/structure.png"
```

### Test SMILES Generation
```bash
curl -X POST https://decimer.ai/decimer-ocsr \
  -d 'structure_depiction_img_paths=["storage/media/structure.png"]' \
  -d 'img_paths=[]'
```

---

## Key Takeaways

1. **File Upload** (`/file-upload`): Entry point for images/PDFs
2. **Segmentation** (`/decimer-segmentation`): Extract structures from PDFs
3. **SMILES Generation** (`/decimer-ocsr`): Main endpoint for chemical recognition
4. **Socket Architecture**: Python servers handle heavy ML processing
5. **Parallel Processing**: Multiple instances for scalability
6. **Complete Pipeline**: Upload → Segment → Recognize → Validate → Output

**Main Output**: SMILES strings with validity, InChIKeys, and classification results
