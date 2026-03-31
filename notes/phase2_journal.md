# Phase 2 Journal: Laravel Upload API & Database

**Date:** March 28, 2026

---

## What We Built

A working file upload pipeline: upload a PDF via API → file saved to shared Docker volume → database row created with `status = pending`.

---

## Step 1: Created the Document Model + Migration

**Command:**
```bash
docker exec -it lfe-laravel-1 php artisan make:model Document -m
```

This generated:
- `laravel/app/Models/Document.php`
- `laravel/database/migrations/2026_03_28_153110_create_documents_table.php`

---

## Step 2: Defined the Database Table

**File:** `laravel/database/migrations/2026_03_28_153110_create_documents_table.php`

```php
public function up(): void
{
    Schema::create('documents', function (Blueprint $table) {
        $table->id();
        $table->string('original_filename');
        $table->string('stored_filename');
        $table->enum('status', ['pending', 'processing', 'done', 'failed'])->default('pending');
        $table->longText('ocr_text')->nullable();
        $table->float('confidence_score')->nullable();
        $table->string('failure_reason')->nullable();
        $table->timestamps();
    });
}
```

**Ran migration:**
```bash
docker exec -it lfe-laravel-1 php artisan migrate
```

---

## Step 3: Updated the Document Model

**File:** `laravel/app/Models/Document.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Document extends Model
{
    protected $fillable = [
        'original_filename',
        'stored_filename',
        'status',
        'ocr_text',
        'confidence_score',
        'failure_reason',
    ];
}
```

---

## Step 4: Created the Controller

**Command:**
```bash
docker exec -it lfe-laravel-1 php artisan make:controller DocumentController
```

**File:** `laravel/app/Http/Controllers/DocumentController.php`

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\Document;
use Illuminate\Support\Str;

class DocumentController extends Controller
{
    public function store(Request $request)
    {
        $request->validate([
            'pdf' => 'required|mimes:pdf|max:10240',
        ]);

        $file = $request->file('pdf');
        $storedFilename = Str::uuid()->toString() . '.pdf';

        $file->move('/var/documents', $storedFilename);

        $document = Document::create([
            'original_filename' => $file->getClientOriginalName(),
            'stored_filename' => $storedFilename,
            'status' => 'pending',
        ]);

        return response()->json($document, 201);
    }

    public function index(Request $request)
    {
        $query = Document::query();

        if ($request->has('q')) {
            $query->where('ocr_text', 'LIKE', '%' . $request->q . '%');
        }

        return response()->json($query->latest()->get());
    }

    public function show($id)
    {
        $document = Document::findOrFail($id);
        return response()->json($document);
    }
}
```

---

## Step 5: Installed API Routes

**Command (to create `routes/api.php` in Laravel 11):**
```bash
docker exec -it lfe-laravel-1 php artisan install:api
```

**File:** `laravel/routes/api.php`

```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\DocumentController;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');

Route::post('/documents', [DocumentController::class, 'store']);
Route::get('/documents', [DocumentController::class, 'index']);
Route::get('/documents/{id}', [DocumentController::class, 'show']);
```

---

## Step 6: Tested the Upload

**Postman request:**
- Method: `POST`
- URL: `http://localhost:8000/api/documents`
- Body: `form-data` → key: `pdf` (type: File) → value: any PDF

**Result:** ✅ Success
- File stored in Docker volume: `fa26bb61-8826-44d6-b7ae-2a76ecbfb668.pdf`
- Database row created with `status = pending`

**Verification commands:**
```bash
# Check the file exists on disk
docker exec -it lfe-laravel-1 ls -la /var/documents

# Check the database row
docker exec -it lfe-laravel-1 php artisan tinker --execute="echo json_encode(App\Models\Document::all(), JSON_PRETTY_PRINT);"
```

---

## API Reference (So Far)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/documents` | Upload a PDF (form-data, key: `pdf`) |
| GET | `/api/documents` | List all documents (supports `?q=search`) |
| GET | `/api/documents/{id}` | Get a single document by ID |

---

## Key Concepts Learned

- **Shared Docker Volume:** `/var/documents` is a named volume mounted in both the `laravel` and `ocr` containers. Files uploaded by Laravel are accessible by the OCR service — no HTTP file transfer needed.
- **UUID Filenames:** We rename uploaded files to UUIDs to prevent filename collisions and security issues (never trust user filenames).
- **`$fillable`:** Laravel requires you to explicitly list which fields can be mass-assigned via `Document::create()`.

---

## Phase 2 Status: ✅ COMPLETE
