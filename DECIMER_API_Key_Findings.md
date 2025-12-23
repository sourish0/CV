# DECIMER.ai API - Key Findings Summary

## 🎯 Main Question: Where is the file upload API and how to get SMILES output?

## ✅ Answer Summary

### File Upload API Endpoint
**URL**: `POST /file-upload`

**Location in Code**: 
- Controller: `app/Http/Controllers/FileUploadController.php`
- Method: `fileUploadPost()`
- Route: Defined in `routes/web.php`

**Input**: Multipart form data with files
```php
Field: file[] (array of files)
Supported: PDF, PNG, JPG/JPEG, WEBP, HEIC
```

**Output**: JSON with image paths
```json
{
    "structure_depiction_img_paths": "[\"storage/media/structure1.png\", ...]"
}
```

---

### SMILES Output API Endpoint
**URL**: `POST /decimer-ocsr`

**Location in Code**:
- Controller: `app/Http/Controllers/DecimerController.php`
- Method: `DecimerOCSRPost()`
- Python Client: `app/Python/decimer_predictor_client.py`
- Servers: Socket connections to ports 65432-65434

**Input**: Structure image paths
```json
{
    "structure_depiction_img_paths": "[\"storage/media/structure1.png\"]"
}
```

**Output**: SMILES strings with metadata
```json
{
    "smiles_array": "[\"CCO\", \"CC(=O)O\"]",
    "validity_array": "[\"valid\", \"valid\"]",
    "inchikey_array": "[\"LFQSCWFLJHTTHZ-UHFFFAOYSA-N\", ...]"
}
```

---

## 🔄 Complete Workflow

```
1. Upload File
   POST /file-upload
   → Files: [structure.png]
   → Response: structure_depiction_img_paths

2. Generate SMILES
   POST /decimer-ocsr
   → Input: structure_depiction_img_paths
   → Response: smiles_array, validity_array, inchikey_array
```

---

## 💡 Key Technical Details

### Architecture
- **Frontend**: Laravel (PHP)
- **Backend**: Python microservices
- **Communication**: TCP Socket connections
- **Parallel Processing**: Multiple OCSR servers (ports 65432-65434)

### Socket Communication Flow
```python
# decimer_predictor_client.py
1. Connect to socket (supervisor:65432-65434)
2. Send: image_path (UTF-8 encoded)
3. Receive: SMILES string (max 32KB)
4. Parallel processing with multiprocessing.Pool
```

### Load Balancing
- Auto-discovers available ports
- Round-robin distribution
- Parallel processing across multiple servers

---

## 📊 API Workflow Diagram

```
┌──────────────┐
│ Upload Image │  POST /file-upload
└──────┬───────┘
       │
       ▼
┌──────────────────────────────┐
│ Normalize & Store            │
│ storage/media/structure.png  │
└──────┬───────────────────────┘
       │
       ▼
┌──────────────┐
│Generate SMILES│ POST /decimer-ocsr
└──────┬───────┘
       │
       ▼
┌──────────────────────────────┐
│ Python Socket Client          │
│ → OCSR Server (ports 65432-34)│
│ → Send: image_path            │
│ → Recv: SMILES                │
└──────┬───────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Post-processing               │
│ - Validate SMILES             │
│ - Generate InChIKey           │
│ - Classify structure          │
└──────┬───────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Final Output                  │
│ {smiles_array, validity, ... }│
└──────────────────────────────┘
```

---

## 🚀 Quick Test Examples

### Test Upload
```bash
curl -X POST https://decimer.ai/file-upload \
  -F "file[]=@structure.png"
```

### Test SMILES Generation
```bash
curl -X POST https://decimer.ai/decimer-ocsr \
  -d 'structure_depiction_img_paths=["storage/media/structure.png"]'
```

---

## 📁 Key Files in Repository

### Controllers (PHP - Laravel)
1. **FileUploadController.php** - Handles file uploads
2. **DecimerController.php** - Handles SMILES generation
3. **DecimerSegmentationController.php** - Handles PDF segmentation

### Python Clients
1. **decimer_predictor_client.py** - Socket client for SMILES
2. **check_smiles_validity.py** - Validates SMILES
3. **get_inchikey_list_from_smiles.py** - Generates InChIKeys
4. **decimer_classifier_client.py** - Classifies structures

### Routes
- **routes/web.php** - All HTTP routes defined here

---

## ⚠️ Important Limitations

1. **Maximum 20 structures** per OCSR request
2. Cannot mix PDF and images in single upload
3. Socket timeout: 1 second
4. Max execution time: 300 seconds (segmentation)
5. Old files cleaned after 1 hour

---

## 📚 Documentation Files Created

1. **DECIMER_API_Documentation.md** - Complete comprehensive guide
2. **DECIMER_API_Quick_Reference.md** - Quick reference with diagrams
3. **DECIMER_API_Key_Findings.md** - This summary document

---

## ✨ Conclusion

**Question**: Find the API point where the file is getting uploaded and the output SMILES we are getting

**Answer**: 
- **Upload API**: `POST /file-upload` (FileUploadController)
- **SMILES API**: `POST /decimer-ocsr` (DecimerController)
- **Communication**: PHP → Python socket client → OCSR servers → SMILES output
- **Output Format**: JSON with smiles_array, validity_array, inchikey_array

All details documented in the accompanying markdown files!
