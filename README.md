# 🗂️ DocManager — Smart PDF Document Manager

A local-first PDF management system built with Python and Streamlit.  
Upload, tag, search, read, and track your documents — all from your browser.

---

## 📌 Table of Contents

-   [Problem Statement](#-problem-statement)
-   [Solution](#-solution)
-   [Features](#-features)
-   [Tech Stack](#-tech-stack)
-   [Project Structure](#-project-structure)
-   [Architecture Overview](#-architecture-overview)
-   [Upload Flow](#-upload-flow)
-   [Search & Read Flow](#-search--read-flow)
-   [Database Design](#-database-design)
-   [Analytics System](#-analytics-system)
-   [Setup & Installation](#-setup--installation)
-   [Environment Variables](#-environment-variables)
-   [Admin Controls](#-admin-controls)
-   [Known Limitations](#-known-limitations)

---

## 🔴 Problem Statement

Managing a personal collection of PDF documents — lecture notes, research papers, study material — is frustrating without a proper tool:

-   **No organisation** — PDFs pile up in folders with no tagging or metadata
-   **No fast search** — finding a specific document means manually browsing file names
-   **No built-in reader** — opening a PDF requires switching to a separate application
-   **No progress tracking** — there is no way to know which documents you have read or how far you got
-   **No usage insight** — no visibility into how often you interact with your documents

These problems compound when a collection grows past a handful of files. The result is a library that exists but cannot be used effectively.

---

## ✅ Solution

DocManager solves all of the above in a single local web app:

-   **Upload** any PDF with tags, a description, and an optional lecture date
-   **Search** instantly by tag (substring match) or by date
-   **Browse** results with auto-generated cover thumbnails
-   **Read** page-by-page directly in the browser — no PDF plugin needed
-   **Track progress** automatically — every page you view is recorded, and reading progress is shown as a percentage
-   **Analyse** usage with a built-in analytics dashboard showing events and per-document reading completion

Everything runs locally. No cloud, no accounts, no dependencies outside your machine.

---

## ✨ Features

Feature

Description

📤 **PDF Upload**

Upload a PDF with tags, description, and optional lecture date

🖼️ **Auto Thumbnail**

Cover image generated from page 0 of each PDF

🔍 **Smart Search**

Search by tag (LIKE) or date (exact). Results shown with thumbnails

📖 **In-Browser Reader**

Page-by-page reading — PDFs pre-rendered to PNG at 2× resolution

📊 **Analytics Dashboard**

Bar chart of app events + per-document reading progress table

🔐 **Admin Panel**

Password-protected full reset of database and file storage

---

## 🛠️ Tech Stack

Technology

Version

Role

**Python**

3.14.3

Core language

**Streamlit**

1.55.0

Web UI — tabs, session state, file uploader

**PyMuPDF (fitz)**

1.27.2.2

PDF rendering — page images + page count

**SQLite**

built-in

Local database — zero config

**Pandas**

2.3.3

Analytics DataFrames

**Pillow**

12.1.1

Image processing

**python-dotenv**

1.2.2

Load `ADMIN_PASSWORD` from `.env`

---

## 📁 Project Structure

```
DocManager/│├── app/│   └── main.py              ← Streamlit UI (entry point)│├── core/│   ├── models.py            ← Document dataclass (maps 1:1 to DB row)│   ├── services.py          ← DocumentService (upload orchestration)│   ├── analytics.py         ← AnalyticsService (event & page tracking)│   ├── reader.py            ← PDFReader (PDF → PNG page images)│   ├── file_manager.py      ← FileManager (save uploaded file to disk)│   └── thumbnail.py         ← ThumbnailGenerator (cover image + page count)│├── db/│   ├── database.py          ← SQLite connection + schema init (3 tables)│   └── repository.py        ← DocumentRepository (INSERT + SELECT queries)│├── data/│   └── documents.db         ← SQLite database file (auto-created on first run)│├── storage/│   ├── pdfs/                ← Uploaded PDFs + per-document page image folders│   └── thumbnails/          ← Cover thumbnail PNGs│├── .env                     ← ADMIN_PASSWORD (do not commit in production)├── requirements.txt         ← Python dependencies└── instructions.txt         ← Setup notes
```

---

## 🏗️ Architecture Overview

The project uses a clean **3-layer architecture**: UI → Service → Data.

```mermaid
graph TD    USER["👤 User (Browser)"] --> UI    subgraph UI["UI Layer — app/main.py"]        STREAMLIT["Streamlit App        Tabs: Upload · Search and View · Analytics        Session State Management"]    end    subgraph SVC["Service Layer — core/"]        DS["DocumentService        upload_document()        search_documents()"]        AS["AnalyticsService        record_page_visit()        get_app_visits()"]        FM["FileManager        save_file()"]        TG["ThumbnailGenerator        generate_thumbnail()        get_total_pages()"]        PR["PDFReader        convert_pdf_to_images()"]        MOD["Document model        core/models.py"]    end    subgraph DATA["Data Layer — db/ + storage/"]        REPO["DocumentRepository        add_document()        search_documents()"]        DB[("SQLite        documents        page_visits        app_visits")]        STORE["storage/pdfs/        raw PDFs + page PNGs"]        THUMB["storage/thumbnails/        cover PNGs"]    end    STREAMLIT --> DS    STREAMLIT --> AS    DS --> FM    DS --> TG    DS --> PR    DS --> MOD    DS --> REPO    AS --> DB    REPO --> DB    FM --> STORE    PR --> STORE    TG --> THUMB
```

---

## 🔄 Upload Flow

When a user uploads a PDF, `DocumentService` runs a 6-step pipeline. Each step is handled by a dedicated class.

```mermaid
flowchart TD    A(["👤 User fills upload form    PDF + tags + description + date"]) --> B    B["app/main.py    Streamlit catches form submit    calls DocumentService.upload_document()"]    B --> C["Step 1 — FileManager.save_file()    Writes file bytes to disk    storage/pdfs/{timestamp}_{name}.pdf"]    C --> D["Step 2 — ThumbnailGenerator.generate_thumbnail()    Opens PDF with PyMuPDF    Renders page 0 at default resolution    Saves to storage/thumbnails/{name}.png"]    D --> E["Step 3 — ThumbnailGenerator.get_total_pages()    Opens PDF    Returns len(doc) — total page count"]    E --> F["Step 4 — PDFReader.convert_pdf_to_images()    Renders ALL pages at 2x resolution    Matrix(2,2) for sharp display    storage/pdfs/{folder}/page_0.png    storage/pdfs/{folder}/page_1.png ..."]    F --> G["Step 5 — Build Document object    name · path · thumbnail_path    tags · description · upload_date    lecture_date · total_pages"]    G --> H["Step 6 — DocumentRepository.add_document()    INSERT INTO documents    Parameterised query — 9 fields"]    H --> I(["✅ Document saved    Ready to search and read"])
```

---

## 🔍 Search & Read Flow

```mermaid
flowchart TD    A(["👤 User enters tag or date    in Search and View tab"]) --> B    B["DocumentService.search_documents()    Delegates to DocumentRepository"]    B --> C["DocumentRepository.search_documents()    Builds dynamic SQL:    SELECT * FROM documents    WHERE tags LIKE '%tag%'    OR lecture_date = 'date'"]    C --> D{Results found?}    D -- No --> E(["🔴 Empty results shown"])    D -- Yes --> F["Display results list    Thumbnail · Name · Tags    Description · Lecture Date"]    F --> G{User clicks Open?}    G -- No --> F    G -- Yes --> H["Set reader_mode = True    selected_doc = doc    current_page = 0    st.rerun()"]    H --> I["Load sorted page images from    storage/pdfs/{folder}/"]    I --> J["Display page image    st.image(img_path)    Prev and Next buttons"]    J --> K["AnalyticsService.record_page_visit()    INSERT INTO page_visits    document_id + page_number + timestamp"]    K --> L["Calculate progress    COUNT DISTINCT page_number    divided by total_pages x 100"]    L --> M["Show progress bar    Progress: X% — n of total pages"]    M --> N{Continue reading?}    N -- Next page --> J    N -- Close Reader --> O(["📕 Reader closed    reader_mode = False"])
```

---

## 🗄️ Database Design

Three tables are created automatically on first run by `init_db()` in `db/database.py`.

```mermaid
erDiagram    documents {        INTEGER id PK        TEXT name        TEXT path        TEXT thumbnail_path        TEXT tags        TEXT description        TEXT upload_date        TEXT lecture_date        INTEGER total_pages    }    page_visits {        INTEGER id PK        INTEGER document_id FK        INTEGER page_number        TEXT timestamp    }    app_visits {        INTEGER id PK        TEXT event_type        TEXT timestamp    }    documents ||--o{ page_visits : "tracks reading of"
```

### Table Purpose

-   **`documents`** — one row per uploaded PDF. Stores all metadata plus paths to the saved file and its thumbnail.
-   **`page_visits`** — one row per page view. `COUNT(DISTINCT page_number)` gives unique pages read, which drives the progress percentage.
-   **`app_visits`** — one row per user action. Recorded events: `upload_click`, `search_click`, `open_document`, `prev_page`, `next_page`, `close_reader`. Powers the usage bar chart.

### Search Query Logic

```sql
-- Tag onlySELECT * FROM documents WHERE tags LIKE '%{tag}%'-- Date onlySELECT * FROM documents WHERE lecture_date = '{date}'-- Both (joined with OR)SELECT * FROM documents WHERE tags LIKE '%{tag}%' OR lecture_date = '{date}'
```

> All queries use `?` parameterised placeholders — not string concatenation — preventing SQL injection.

---

## 📊 Analytics System

The analytics layer is completely separate from document management. `AnalyticsService` writes to and reads from its own two tables independently.

```mermaid
flowchart LR    ACTION["User Action"] --> ETYPE{Event type}    ETYPE --> APPEVENT["App event    upload_click    search_click    open_document    prev_page    next_page    close_reader"]    ETYPE --> PAGEEVENT["Page viewed    document_id    page_number"]    APPEVENT --> APPWRITE["INSERT INTO app_visits    event_type + timestamp"]    PAGEEVENT --> PAGEWRITE["INSERT INTO page_visits    document_id + page_number + timestamp"]    APPWRITE --> CHART["Analytics Tab    Bar chart    Event vs COUNT(*)"]    PAGEWRITE --> PROGRESS["Analytics Tab    Progress table    COUNT DISTINCT pages    divided by total_pages"]
```

### Reading Progress Formula

```
unique_pages = COUNT(DISTINCT page_number) WHERE document_id = Xprogress %   = (unique_pages / total_pages) × 100
```

Revisiting a page does **not** inflate progress — only newly viewed unique pages count.

---

## 🚀 Setup & Installation

### Option A — pip

```bash
# 1. Clone the repogit clone https://github.com/nimowhyca/DocManager.gitcd DocManager# 2. Create a virtual environmentpython -m venv venv# WindowsvenvScriptsactivate# macOS / Linuxsource venv/bin/activate# 3. Install dependenciespip install -r requirements.txt# PyMuPDF sometimes needs a clean install to avoid cache issuespip uninstall pymupdfpip install --no-cache-dir pymupdf# 4. Run the appstreamlit run app/main.py
```

### Option B — uv (recommended for Python 3.14+)

```bash
git clone https://github.com/nimowhyca/DocManager.gitcd DocManageruv inituv pin python 3.14.3uv add -r requirements.txtuv syncstreamlit run app/main.py
```

Open your browser at **[http://localhost:8501](http://localhost:8501)**

---

## 🔐 Environment Variables

The app reads one environment variable from a `.env` file in the project root:

```env
ADMIN_PASSWORD=your_secure_password_here
```

> ⚠️ **Warning:** The `.env` file is currently committed to this repository. In any production or shared environment, add `.env` to `.gitignore` and provide the password via a system environment variable or secrets manager instead.

---

## ⚙️ Admin Controls

The admin panel at the top of the app allows a full system reset. It requires the password from `.env`.

```mermaid
flowchart TD    A(["Admin clicks Clean Database"]) --> B["show_reset = True    Password input field appears"]    B --> C{Password matches    ADMIN_PASSWORD?}    C -- No --> D(["❌ Error shown    Form resets"])    C -- Yes --> E["os.remove data/documents.db"]    E --> F["shutil.rmtree storage/pdfs/"]    F --> G["shutil.rmtree storage/thumbnails/"]    G --> H["os.makedirs storage/pdfs/    os.makedirs storage/thumbnails/"]    H --> I(["✅ System reset complete    Restart the app to continue"])
```

---

## ⚠️ Known Limitations

-   **Security** — `.env` containing a plaintext `ADMIN_PASSWORD` is committed to the repository. Use environment variables in production.
-   **Tags** — stored as a comma-separated string in a single column. Normalising to a separate `tags` table would allow exact multi-tag filtering.
-   **Search logic** — tag and date conditions are joined with `OR`. In most cases `AND` would be more intuitive and precise.
-   **Synchronous upload** — page conversion for large PDFs blocks the entire Streamlit UI thread. A background task queue (Celery, RQ) would fix this.
-   **No error handling** — no `try/except` around file I/O or database calls. A failed write crashes the app silently.
-   **No tests** — `DocumentRepository`, `DocumentService`, and `PDFReader` have no unit or integration tests.
-   **Single user** — SQLite and local file storage are not designed for concurrent users. Multi-user deployment requires PostgreSQL and cloud object storage (e.g. S3).

---

## 📦 Dependencies

```
streamlit==1.55.0pandas==2.3.3PyMuPDF==1.27.2.2pillow==12.1.1python-dotenv==1.2.2
```

---

## 📂 Storage Conventions

Path

Contents

`storage/pdfs/{timestamp}_{name}.pdf`

Raw uploaded PDF file

`storage/pdfs/{timestamp}_{name}/page_N.png`

Per-page PNG images at 2× resolution

`storage/thumbnails/{timestamp}_{name}.png`

Cover thumbnail (page 0, default resolution)

`data/documents.db`

SQLite database (all 3 tables)