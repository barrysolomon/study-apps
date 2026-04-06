# Crash Course Study Apps

Standalone HTML study apps for rapid-learning any topic. No frameworks, no build step, no server — just open in a browser.

## Apps

| File | What it covers |
|---|---|
| [gke-study-app.html](gke-study-app.html) | Google Kubernetes Engine — cloud-native K8s on GCP |
| [otca-study-app.html](otca-study-app.html) | OpenTelemetry Certified Associate — observability & tracing |
| [index.html](index.html) | Landing page linking to both apps |

## How to use

**Online (GitHub Pages):** Visit the hosted URL — pick an app and start studying.

**Offline:** Download any `.html` file, double-click to open in your browser. Everything works — quiz, flashcards, localStorage persistence. No internet needed.

## Study modes

Each app includes 7 integrated modes: a dashboard with progress stats, lessons with exercises, flashcards with category filtering, quizzes (per-section + final exam), scenario drills, timed rapid-fire recall, and a settings/management page.

On first launch you pick a learner role (e.g. Sales, Engineering, Exec) which tailors the content priority and study path to your needs.

## Cloud sync (optional)

Progress can sync across devices via a Google Sheet. This is entirely optional — the apps work perfectly with just localStorage.

**Setup:** See [google-sheets-sync-setup.md](google-sheets-sync-setup.md) for instructions. The short version: create a Google Sheet, add a ~45-line Apps Script, deploy as a web app, paste the URL into Settings → Cloud Sync in the app.

**Multi-user:** Each person enters their name in the sync settings. This creates a separate row in the Sheet per person, so a team can share one sync URL without overwriting each other's progress.

**Offline resilience:** If you lose connectivity, the app keeps working with localStorage and queues sync changes. When you reconnect, it auto-flushes.

## Building a new course

The repo includes a full template for creating new crash course apps:

| File | Purpose |
|---|---|
| [crash-course-TEMPLATE.html](crash-course-TEMPLATE.html) | Runnable template with placeholder content — search for `TODO` (16 markers) |
| [TEMPLATE-QUICKSTART.md](TEMPLATE-QUICKSTART.md) | How to customize the template — data structures, config reference |

Fill in your flashcards, quiz questions, lessons, and scenarios. Open in a browser. Done.

## Tech details

Each app is a single self-contained HTML file (~90-150 KB) with all CSS and JavaScript inline. No external dependencies, no CDN calls, no build tooling. Works in any modern browser (Chrome, Firefox, Safari, Edge) down to 320px viewport width. Data persists in localStorage keyed by `STORAGE_KEY`.
