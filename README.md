# FlutterFlow Gemma Embeddings Library - Usage Guide

## Overview

This library enables **on-device text embeddings** using Google's Gemma model for RAG (Retrieval-Augmented Generation) capabilities in FlutterFlow apps.

![tmpmotg68ar](https://github.com/user-attachments/assets/40337c57-e1fa-457e-9d25-7d81c5a8b5d3)

---

## Get the exported source

```bash
git clone https://github.com/sgardoll/embeddingGemmaFlutterFlow.git
cd embeddingGemmaFlutterFlow
```

This checkout contains the exported Flutter app and custom code. Cloning it does not import a library into the FlutterFlow editor. No verified FlutterFlow library share link or Marketplace listing is supplied here; the setup below uses the exported source.

### Prerequisites and platform limits

- Install a [Flutter SDK](https://docs.flutter.dev/install) whose bundled Dart satisfies the `>=3.0.0 <4.0.0` constraint in [pubspec.yaml](pubspec.yaml).
- The export has Android and iOS project folders. [Android configuration](android/app/build.gradle) sets compile, target and minimum SDK to **36**; [iOS configuration](ios/Podfile) targets **16.0.0**. Set up the [Android toolchain](https://docs.flutter.dev/platform-integration/android/setup), or [Xcode/iOS toolchain on macOS](https://docs.flutter.dev/platform-integration/ios/setup), for the target you intend to investigate. These are source settings, not device compatibility results.
- `flutter_gemma:` is blank/unpinned in the manifest and no `pubspec.lock` is committed. Review the [plugin author's current instructions](https://pub.dev/packages/flutter_gemma) against this export before resolving dependencies. On 7 October 2026, the package page marks `flutter_gemma` discontinued and points to `flutter_edge_ai`; this export still uses the old package and calls `FlutterGemma.initialize()` without arguments. No compatible package version, migration or successful build is established here.
- A `web/` folder is present, but [SQLiteManager.initialize()](lib/backend/sqlite/sqlite_manager.dart) returns early on web without initializing its database. Desktop initialization code is present in [sqfliteFfiInit](lib/custom_code/actions/sqflite_ffi_init.dart), but no desktop project folders are supplied. Neither establishes a working web or desktop embedding demo.

### Inspect and configure the exported demo

1. Review [lib/main.dart](lib/main.dart). It initializes the Flutter binding, calls `sqfliteFfiInit()` and then `initializeGemma()` before SQLite initialization and `runApp`. Keep this startup ordering; Gemma must be initialized before downloading or embedding.
2. Set `modelUrl` and `tokenizerUrl` in [lib/library_values.dart](lib/library_values.dart) to a matching embedding model and tokenizer you can access. The checked-in values use Gecko's `.tflite` and `sentencepiece.model` URLs shown below. Model files are downloaded at runtime, not bundled by this guide; network access and local storage are needed for installation. Endpoint availability and device execution have not been verified.
3. Inspect [StartDownload](lib/demo/start_download/start_download_widget.dart), the initial page selected by [the router](lib/flutter_flow/nav/nav.dart). It passes both URLs to `downloadEmbeddingModel` and stores the returned string in `downloadResult`, then navigates **without checking for `"Success"`**. Reaching the next page does not prove installation succeeded. When wiring these actions into your own app, handle `"Error: ..."` and continue only after `"Success"`.
4. The action order is `initializeGemma()` → `downloadEmbeddingModel(modelUrl, tokenizerUrl)` → `processDocumentsToVectors(List<String>)` → `saveVectorsToDb(List<VectorDocumentStruct>)`. Search consumes vectors in memory with `findTopMatches(query, documents, topK, threshold)`; the exported search page passes `5` and `0.7` for the last two arguments. **Persistence has a source gap:** [saveVectorsToDb](lib/custom_code/actions/save_vectors_to_db.dart) currently writes to an empty SQL table name (`INSERT OR REPLACE INTO ""`), not the documented `embeddings` table. A working end-to-end save/search workflow is not established by this checkout.

After checking dependency compatibility and addressing any source gaps in your own work, Flutter's [CLI](https://docs.flutter.dev/reference/flutter-cli) provides the following local setup/run commands from the clone directory. Replace `DEVICE_ID` with the Android or iOS target listed by `flutter devices`:

```bash
flutter pub get
flutter devices
flutter run -d DEVICE_ID
```

These commands describe an attempted exported-app run, not a verified build. If resolution, initialization or saving fails, retain the error and report it in a repository issue; do not treat the action diagrams as proof of execution. This guide does not supply a FlutterFlow editor import route or a dependency/source repair.

---

## Architecture Flowchart

The diagrams illustrate action wiring after initialization. They do not describe the exported demo's download error handling or establish that its persistence path works; see the source gaps above.

```mermaid
flowchart TB
    subgraph "SETUP PHASE (One-time)"
        A[App Start] --> INIT[initializeGemma]
        INIT --> B{Model Downloaded?}
        B -->|No| C[downloadEmbeddingModel]
        C -->|"modelUrl and tokenizerUrl"| D[Model Downloaded to Device]
        B -->|Yes| E[Ready]
        D --> E
    end

    subgraph "INGESTION PHASE (Index Documents)"
        E --> F[User has documents to index]
        F --> G["processDocumentsToVectors(documents)"]
        G -->|"List&lt;String&gt;"| H[GemmaEmbedderWrapper]
        H -->|"generates embeddings"| I["List&lt;VectorDocumentStruct&gt;"]
        I --> J["saveVectorsToDb(vectors)"]
        J --> K[(SQLite Database)]
    end

    subgraph "QUERY PHASE (Search)"
        L[User enters search query] --> M["findTopMatches(query, documents, topK)"]
        M -->|"embed query"| H
        M -->|"cosine similarity"| N[Ranked Results]
        N --> O["Top K VectorDocumentStruct[]"]
    end

    style A fill:#e1f5fe
    style E fill:#c8e6c9
    style K fill:#fff3e0
    style O fill:#f3e5f5
```

---

## Component Reference

### Data Types

#### `VectorDocumentStruct` (Custom Data Type)

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Unique identifier (auto-generated if empty) |
| `text` | String | Original document text |
| `vector` | List&lt;double&gt; | Embedding vector (768 dimensions for Gemma) |
| `metadata` | String | Optional metadata (e.g., source, category) |

The type is included in [the exported source](lib/backend/schema/structs/vector_document_struct.dart). Automatic import into the FlutterFlow editor is not established by cloning this repository.

---

### Custom Actions

#### `initializeGemma` (startup prerequisite)

`initializeGemma()` takes no arguments and awaits `FlutterGemma.initialize()`; it has no result string or local error handler. The exported `main.dart` already awaits it before `runApp`. For your own action wiring, await it once at startup before any other Gemma operation, including the setup chains below.

#### 1. `downloadEmbeddingModel`

**Purpose:** Downloads and installs the embedding model AND tokenizer. Required before any embedding operations.

```
downloadEmbeddingModel(modelUrl, tokenizerUrl) → String
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelUrl` | String | Yes | URL to the embedding model (.tflite file) |
| `tokenizerUrl` | String | Yes | URL to the tokenizer (sentencepiece.model) |

| Returns | Description |
|---------|-------------|
| `"Success"` | Model and tokenizer installed successfully |
| `"Error: ..."` | Installation failed with error message |

**Recommended URLs (Gecko - NO authentication required):**
```
Model:     https://huggingface.co/litert-community/Gecko-110m-en/resolve/main/Gecko_256_quant.tflite
Tokenizer: https://huggingface.co/litert-community/Gecko-110m-en/resolve/main/sentencepiece.model
```

**FlutterFlow Usage:**
```
┌─────────────────────────────────────────────────┐
│ Action: downloadEmbeddingModel                   │
├─────────────────────────────────────────────────┤
│ modelUrl: "https://huggingface.co/litert-       │
│   community/Gecko-110m-en/resolve/main/         │
│   Gecko_256_quant.tflite"                       │
│                                                  │
│ tokenizerUrl: "https://huggingface.co/litert-   │
│   community/Gecko-110m-en/resolve/main/         │
│   sentencepiece.model"                          │
│                                                  │
│ Output Variable Name: downloadResult             │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ Conditional: downloadResult == "Success"         │
│   ├─ True:  Navigate to Main Page               │
│   └─ False: Show Error Snackbar                 │
└─────────────────────────────────────────────────┘
```

---

#### 2. `processDocumentsToVectors`

**Purpose:** Converts a list of text documents into embedding vectors.

```
processDocumentsToVectors(documents) → List<VectorDocumentStruct>
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `documents` | List&lt;String&gt; | Yes | List of text documents to embed |

| Returns | Description |
|---------|-------------|
| `List<VectorDocumentStruct>` | Documents with their embedding vectors |

**FlutterFlow Usage:**
```
┌─────────────────────────────────────────────────┐
│ Action: processDocumentsToVectors                │
├─────────────────────────────────────────────────┤
│ documents: [                                     │
│   "The quick brown fox jumps over the lazy dog",│
│   "Machine learning is a subset of AI",         │
│   "Flutter is a cross-platform framework"       │
│ ]                                                │
│                                                  │
│ Output Variable Name: vectorDocs                 │
└─────────────────────────────────────────────────┘
```

---

#### 3. `saveVectorsToDb`

**Purpose:** Persists vector documents to the local SQLite database.

```
saveVectorsToDb(vectors) → void
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `vectors` | List&lt;VectorDocumentStruct&gt; | Yes | Vector documents to save |

**FlutterFlow Usage:**
```
┌─────────────────────────────────────────────────┐
│ Action: saveVectorsToDb                          │
├─────────────────────────────────────────────────┤
│ vectors: vectorDocs  (from previous action)      │
└─────────────────────────────────────────────────┘
```

---

#### 4. `findTopMatches`

**Purpose:** Searches for the most similar documents to a query using cosine similarity.

```
findTopMatches(query, documents, topK, threshold) → List<VectorDocumentStruct>
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | String | Yes | Search query text |
| `documents` | List&lt;VectorDocumentStruct&gt; | Yes | Documents to search through |
| `topK` | int | No | Number of results (default: 5) |
| `threshold` | double | Yes | Minimum cosine similarity; the exported demo passes `0.7` |

| Returns | Description |
|---------|-------------|
| `List<VectorDocumentStruct>` | Top K most similar documents, ranked by similarity |

**FlutterFlow Usage:** Supply the required `threshold` argument as well as the values illustrated below.
```
┌─────────────────────────────────────────────────┐
│ Action: findTopMatches                           │
├─────────────────────────────────────────────────┤
│ query: searchTextField.text                      │
│ documents: appState.allVectorDocs               │
│ topK: 5                                          │
│                                                  │
│ Output Variable Name: searchResults              │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ Update Page State: results = searchResults       │
│ Then: Rebuild ListView with results              │
└─────────────────────────────────────────────────┘
```

---

### Custom Widgets

#### `GenerateEmbeddings`

**Purpose:** Pre-built UI widget that handles the entire embedding + save workflow with progress indication.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `width` | double | No | Widget width |
| `height` | double | No | Widget height |
| `documents` | List&lt;String&gt; | No | Documents to process |
| `onComplete` | Action(int) | No | Callback with count of saved vectors |
| `onError` | Action(String) | No | Callback with error message |

**FlutterFlow Usage:**
```
┌─────────────────────────────────────────────────┐
│ Custom Widget: GenerateEmbeddings                │
├─────────────────────────────────────────────────┤
│ width: MediaQuery.sizeOf(context).width          │
│ height: 150                                      │
│ documents: pageState.documentsToEmbed            │
│ onComplete: [                                    │
│   Show Snackbar: "Saved ${count} vectors!"      │
│   Navigate to Search Page                        │
│ ]                                                │
│ onError: [                                       │
│   Show Snackbar: "Error: ${error}"              │
│ ]                                                │
└─────────────────────────────────────────────────┘
```

---

## Complete Workflow Diagrams

### Flow 1: Initial Setup (App First Launch)

This is intended wiring: initialize Gemma first, then supply both model and tokenizer URLs. The exported StartDownload page currently lacks this success/error branch.

```mermaid
sequenceDiagram
    participant User
    participant App
    participant downloadEmbeddingModel
    participant Device Storage

    User->>App: Opens app first time
    App->>App: Check if model exists
    alt Model not found
        App->>User: Show "Downloading Model..." screen
        App->>downloadEmbeddingModel: Call with model and tokenizer URLs
        downloadEmbeddingModel->>Device Storage: Download ~300MB model
        Device Storage-->>downloadEmbeddingModel: Success
        downloadEmbeddingModel-->>App: "Success"
        App->>User: Navigate to Home
    else Model exists
        App->>User: Navigate to Home directly
    end
```

### Flow 2: Document Ingestion

```mermaid
sequenceDiagram
    participant User
    participant FlutterFlow Page
    participant processDocumentsToVectors
    participant GemmaEmbedderWrapper
    participant saveVectorsToDb
    participant SQLite

    User->>FlutterFlow Page: Enters documents or loads from API
    FlutterFlow Page->>processDocumentsToVectors: List<String> documents
    
    loop For each document
        processDocumentsToVectors->>GemmaEmbedderWrapper: getEmbedding(text)
        GemmaEmbedderWrapper-->>processDocumentsToVectors: List<double> vector
    end
    
    processDocumentsToVectors-->>FlutterFlow Page: List<VectorDocumentStruct>
    FlutterFlow Page->>saveVectorsToDb: vectors
    saveVectorsToDb->>SQLite: Batch INSERT
    SQLite-->>saveVectorsToDb: Done
    saveVectorsToDb-->>FlutterFlow Page: Complete
    FlutterFlow Page->>User: "Saved X documents!"
```

### Flow 3: Semantic Search

```mermaid
sequenceDiagram
    participant User
    participant Search Page
    participant findTopMatches
    participant GemmaEmbedderWrapper
    participant Results List

    User->>Search Page: Types "machine learning basics"
    User->>Search Page: Taps Search button
    Search Page->>findTopMatches: query, documents, topK=5
    
    findTopMatches->>GemmaEmbedderWrapper: getEmbedding(query)
    GemmaEmbedderWrapper-->>findTopMatches: queryVector
    
    findTopMatches->>findTopMatches: Calculate cosine similarity for all docs
    findTopMatches->>findTopMatches: Sort by similarity descending
    findTopMatches->>findTopMatches: Take top K
    
    findTopMatches-->>Search Page: Top 5 VectorDocumentStruct[]
    Search Page->>Results List: Update with results
    Results List->>User: Display ranked results
```

---

## FlutterFlow Action Chains

### Chain 1: Setup Flow

`initializeGemma()` must already have completed at app startup before this page-load chain runs.

```
┌─────────────────────────────────────────────────────────────────┐
│                        ON PAGE LOAD                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 1. Custom Action: downloadEmbeddingModel                  │   │
│  │    modelUrl: "https://huggingface.co/litert-community/   │   │
│  │              Gecko-110m-en/resolve/main/                  │   │
│  │              Gecko_256_quant.tflite"                      │   │
│  │    tokenizerUrl: "https://huggingface.co/litert-community│   │
│  │              /Gecko-110m-en/resolve/main/                 │   │
│  │              sentencepiece.model"                         │   │
│  │    Output: downloadResult                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 2. Conditional: downloadResult == "Success"               │   │
│  │    ┌─────────────────┐    ┌─────────────────────────┐    │   │
│  │    │ TRUE            │    │ FALSE                    │    │   │
│  │    │ Navigate to     │    │ Show Snackbar:          │    │   │
│  │    │ HomePage        │    │ "Download failed"       │    │   │
│  │    └─────────────────┘    └─────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Chain 2: Ingest Documents Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    ON BUTTON TAP: "Index Documents"              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 1. Custom Action: processDocumentsToVectors               │   │
│  │    documents: pageState.documentsList                     │   │
│  │    Output: vectorDocs                                     │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 2. Custom Action: saveVectorsToDb                         │   │
│  │    vectors: vectorDocs                                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 3. Update App State                                       │   │
│  │    appState.allVectors = vectorDocs                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 4. Show Snackbar: "Indexed ${vectorDocs.length} docs!"   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Chain 3: Search Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    ON BUTTON TAP: "Search"                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 1. Custom Action: findTopMatches                          │   │
│  │    query: searchTextField.text                            │   │
│  │    documents: appState.allVectors                         │   │
│  │    topK: 5                                                │   │
│  │    Output: searchResults                                  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 2. Update Page State                                      │   │
│  │    pageState.results = searchResults                      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 3. ListView automatically rebuilds with new results       │   │
│  │    (bound to pageState.results)                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Alternative: Using GenerateEmbeddings Widget

Instead of chaining `processDocumentsToVectors` + `saveVectorsToDb`, use the widget:

```
┌─────────────────────────────────────────────────────────────────┐
│                         PAGE LAYOUT                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Column                                                    │   │
│  │  ├─ Text: "Add your documents below"                     │   │
│  │  ├─ TextField: documentsInput (multiline)                │   │
│  │  └─ CustomWidget: GenerateEmbeddings                     │   │
│  │       ├─ documents: documentsInput.text.split('\n')      │   │
│  │       ├─ onComplete: [                                   │   │
│  │       │    Show Snackbar: "Saved ${count} vectors!"     │   │
│  │       │    Navigate to SearchPage                        │   │
│  │       │  ]                                               │   │
│  │       └─ onError: [                                      │   │
│  │            Show Snackbar: "Error: ${error}"             │   │
│  │          ]                                               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Database Schema

This is the documented storage layout. The current `saveVectorsToDb` action targets an empty table name; the save route needs a source repair before these examples can be treated as a working persistence workflow.

The SQLite database stores vectors in the `embeddings` table:

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT (PK) | Unique document identifier |
| `content` | TEXT | Original document text |
| `embedding` | BLOB | Binary-encoded float64 vector |
| `metadata` | TEXT | Optional JSON metadata |

---

## Quick Reference Card

| Task | Action/Widget | Input | Output |
|------|---------------|-------|--------|
| Initialize plugin | `initializeGemma` | None | Await completion before other Gemma actions |
| Download model | `downloadEmbeddingModel` | modelUrl, tokenizerUrl | "Success" or "Error: ..." |
| Convert text to vectors | `processDocumentsToVectors` | List&lt;String&gt; | List&lt;VectorDocumentStruct&gt; |
| Save to database | `saveVectorsToDb` | List&lt;VectorDocumentStruct&gt; | void |
| Search similar docs | `findTopMatches` | query, docs, topK, threshold | List&lt;VectorDocumentStruct&gt; |
| All-in-one UI | `GenerateEmbeddings` widget | documents, callbacks | Built-in UI |

---

## Common Patterns

### Pattern A: Load Documents from API, Then Index

```
1. API Call → Get List<String> documents
2. processDocumentsToVectors(documents) → vectors
3. saveVectorsToDb(vectors)
4. Update App State with vectors for searching
```

### Pattern B: User Input Documents

```
1. User types in TextField (one doc per line)
2. Split by newline → List<String>
3. Use GenerateEmbeddings widget OR action chain
```

### Pattern C: Hybrid Search Page

```
1. On Page Load: Load vectors from App State (or query SQLite)
2. On Search: findTopMatches(query, loadedVectors, 10)
3. Display results in ListView
```

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Empty embeddings | Model not downloaded | Call `downloadEmbeddingModel` first |
| Slow processing | Large documents | Chunk documents into smaller pieces |
| Search returns empty | No vectors in memory | Load vectors from DB or App State first |
| App crashes on embed | Insufficient memory | Use smaller batch sizes |
