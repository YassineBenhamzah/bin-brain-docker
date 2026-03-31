# Phase 4 Journal: Automatic Background OCR

**Date:** March 30, 2026

---

## What We Built

We completely removed the manual step from the architecture. When a user uploads a PDF, Laravel instantly returns a `201 Created` response. In the absolute background, a dedicated Docker container grabs the file, sends it over the internal Docker network to the Node.js OCR microservice, and saves the extracted payload back into the MySQL database.

---

## Step 1: The Queue Job

Created a Laravel Job to securely isolate the OCR HTTP network call from the user's web request.

**Command:**
```bash
docker exec -it lfe-laravel-1 php artisan make:job ProcessDocumentJob
```

**File:** `laravel/app/Jobs/ProcessDocumentJob.php`
```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use App\Models\Document;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class ProcessDocumentJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $documentId;

    public function __construct(int $documentId)
    {
        $this->documentId = $documentId;
    }

    public function handle(): void
    {
        $document = Document::find($this->documentId);
        if (!$document) return;

        $document->update(['status' => 'processing']);
        $filePath = '/var/documents/' . $document->stored_filename;

        try {
            Log::info("Sending document {$document->id} to OCR service");
            
            // Wait up to 300 seconds for massive PDFs
            $response = Http::timeout(300)->post('http://ocr:3000/process', [
                'filePath' => $filePath
            ]);

            if ($response->successful()) {
                $data = $response->json();
                $document->update([
                    'ocr_text' => $data['text'],
                    'confidence_score' => $data['confidence'],
                    'status' => 'done'
                ]);
            } else {
                $document->update([
                    'status' => 'failed',
                    'failure_reason' => $response->json('error') ?? 'OCR returned an error'
                ]);
            }
        } catch (\Exception $e) {
            Log::error("OCR connection failed: " . $e->getMessage());
            $document->update([
                'status' => 'failed',
                'failure_reason' => 'Could not connect to OCR service'
            ]);
        }
    }
}
```

## Step 2: Triggering the Job

Connected the Queue Job into the existing upload controller. 

**File:** `laravel/app/Http/Controllers/DocumentController.php`
```php
ProcessDocumentJob::dispatch($document->id);
```

## Step 3: Upgrading DevOps (Dedicated Docker Worker)

**Problem:** Running `php artisan queue:work` manually in a terminal is fragile. If the terminal closes, the application breaks.
**Solution:** Added a dedicated, headless `worker` container to `docker-compose.yml`. It uses the exact same `laravel` image but runs the queue listener 24/7 in the background alongside MySQL and the APIs.

**File:** `docker-compose.yml`
```yaml
  # --- NEW: BACKGROUND QUEUE WORKER ---
  worker:
    build: ./laravel
    command: php artisan queue:work
    environment:
      DB_HOST: mysql
      DB_DATABASE: binbrain
      DB_USERNAME: laravel
      DB_PASSWORD: secret
    volumes:
      - documents:/var/documents
      - ./laravel:/var/www
    depends_on:
      - mysql
      - laravel
```

## Post-Phase Status Check

We monitored the logs while uploading a test PDF:
```text
  2026-03-30 15:30:37 App\Jobs\ProcessDocumentJob ...................... RUNNING
  2026-03-30 15:30:41 App\Jobs\ProcessDocumentJob ...................... 3s DONE
```
With the newly upgraded "Warm Boot" Tesseract pool (built in late Phase 3), the background job cleared start-to-finish across multiple Docker containers in **3 seconds flat**. 

## Phase 4 Status: ✅ COMPLETE
