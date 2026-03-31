# Phase 5: The Next.js Frontend Dashboard

Now that the entire backend engine (Laravel + Node.js OCR + Queue Worker) is flawlessly automated, we need to build the graphical interface so a user can actually interact with the system seamlessly in their web browser.

## The Goal
Build a gorgeous, interactive Next.js Dashboard running on `http://localhost:3003` that talks directly to your Laravel API running on `http://localhost:8000`.

## Proposed Features

### 1. Fix Laravel CORS
Because your Next.js application runs on a different port (`3003`) than your Laravel backend (`8000`), the browser's security safeguards will block React from communicating with Laravel. 
- **Action:** Before writing any frontend code, we must inject an explicit CORS (Cross-Origin Resource Sharing) exception in Laravel's configuration to allow `http://localhost:3003`.

### 2. High-End UI Component
We will rip out the default Next.js template in `nextjs/app/page.tsx` and replace it with a dynamic React component powered by TailwindCSS.
- **Upload Zone:** A clean, stylish file selector to upload PDFs and images.
- **Live Notification:** When the file is submitted, the UI instantly throws an HTTP `POST` to Laravel.
- **The Dashboard Grid:** A beautiful grid displaying all historic documents.

### 3. Real-Time Polling Magic
Since the OCR engine takes anywhere from 2 to 15 seconds to run asynchronously in the background, the UI cannot afford to freeze and wait.
- **Action:** We will build a React `useEffect` hook that silently pings `GET /api/documents` every 3 seconds in the background. 
- **User Experience:** You will literally watch the dashboard badge change from `pending` ➔ `processing` ➔ `done` in real-time, and the extracted text will magically appear on the screen without ever pressing the browser refresh button!

## User Review Required

> [!TIP]
> This completes the core user journey connecting all 4 of your Docker containers together.

Are you ready to dive into the frontend code to see your beautiful project finally come to life visually?
