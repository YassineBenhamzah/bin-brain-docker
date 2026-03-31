# Phase 3 Journal: OCR Node.js Service (Tesseract & Poppler)

**Date:** March 30, 2026

---

## What We Built

A robust, local-first OCR microservice in Node.js that processes both images and native PDFs using Tesseract.js and Poppler without crashing or requiring heavy C++ compilation on Windows/Alpine Docker.

---

## Technical Hurdles & Architectural Decisions

### 1. File Changes Weren't Syncing to the Running Container
**Problem:** In Phase 1, the `ocr` container was built using `COPY . .` and ran `node index.js`. Any changes made locally in VS Code didn't reflect until a full container rebuild.
**Solution:** We added volume binding in `docker-compose.yml`:
```yaml
    volumes:
      - documents:/var/documents
      - ./ocr:/app
      - /app/node_modules
```

### 2. Node Watch Not Working on Windows/Docker
**Problem:** Even after adding volumes, Node's built-in watcher (`node --watch index.js`) didn't detect file changes. Windows file-save events often fail to trigger `inotify` inside Alpine containers.
**Solution:** Replaced native `--watch` with Nodemon's **legacy polling**:
```yaml
    command: npx -y nodemon -L index.js
```

### 3. Tesseract Crashing on PDFs
**Problem:** Tesseract only processes raster images (PNG/JPG). Passing a PDF directly threw an unhandled Rejection: `Error in pixReadStream: Pdf reading is not supported` and crashed the server.
**Initial Plan:** Use a pure JS package like `pdf-img-convert` to rasterize the PDF in memory.
**Secondary Problem:** `pdf-img-convert` silently requires `canvas`, which relies on `node-pre-gyp`. Attempting to install it triggered complex C++ build failures on Windows.
**Final Solution:** Abandoned native compilation entirely. Upgraded the Alpine Docker container to install `poppler-utils` via `apk`, and used Node's built-in `child_process.execSync` to run `pdftoppm` blazingly fast against the container's file system.

---

## Step 1: Upgraded Dockerfile
Added the native Linux utility `poppler-utils` directly to Alpine, entirely bypassing Node/NPM compilation errors.

**File:** `ocr/Dockerfile`
```dockerfile
FROM node:20-alpine
WORKDIR /app
RUN apk add --no-cache poppler-utils
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

## Step 2: Wrote the Intelligent OCR Engine
Created a `/process` POST route that inspects the file extension. Images go straight to Tesseract. PDFs are seamlessly flattened into PNGs via shell command first, then sent to Tesseract.

**File:** `ocr/index.js`
```javascript
const express = require('express');
const Tesseract = require('tesseract.js');
const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

const app = express();
app.use(express.json());

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

app.post('/process', async (req, res) => {
  const { filePath } = req.body;

  if (!filePath || !fs.existsSync(filePath)) {
    return res.status(404).json({ error: 'file_not_found', attempted_path: filePath });
  }

  try {
    let imageToProcess = filePath;
    let tempImage = null;
    const extension = path.extname(filePath).toLowerCase();

    // If PDF, use Poppler to grab the first page
    if (extension === '.pdf') {
      console.log(`Converting PDF to image...`);
      execSync(`pdftoppm -f 1 -l 1 -png "${filePath}" /tmp/pdf_page`);
      
      tempImage = '/tmp/pdf_page-1.png';
      imageToProcess = tempImage;
    }

    console.log(`Starting OCR...`);
    const { data: { text, confidence } } = await Tesseract.recognize(imageToProcess, 'eng');
    
    // Clean up temporary PNG
    if (tempImage && fs.existsSync(tempImage)) {
      fs.unlinkSync(tempImage);
    }

    console.log(`OCR complete! Confidence: ${confidence}`);
    res.json({ text, confidence });

  } catch (err) {
    console.error('OCR failed:', err);
    res.status(500).json({ error: 'ocr_failed', details: err.message });
  }
});

app.listen(3000, () => console.log('OCR service running on port 3000'));
```

## Step 3: End-to-End Verification
Tested via Postman:
**POST** `http://localhost:3001/process`
```json
{
  "filePath": "/var/documents/fa26bb61-8826-44d6-b7ae-2a76ecbfb668.pdf"
}
```

**Result:** ✅ Success (HTTP 200)
Successfully stripped the French resume text from the PDF with 87% confidence rating.

---

## Phase 3 Status: ✅ COMPLETE
