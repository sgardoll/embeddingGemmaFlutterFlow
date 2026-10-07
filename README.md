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

---

## Architecture Flowchart

```mermaid
flowchart TB
    subgraph "SETUP PHASE (One-time)"
        A[App Start] --> B{Model Downloaded?}
        B -->|No| C[downloadEmbeddingModel]
        C -->|"url: model URL"| D[Model Downloaded to Device]
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

**Imported automatically in FlutterFlow Library** 

---

### Custom Actions

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
findTopMatches(query, documents, topK) → List<VectorDocumentStruct>
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | String | Yes | Search query text |
| `documents` | List&lt;VectorDocumentStruct&gt; | Yes | Documents to search through |
| `topK` | int | No | Number of results (default: 5) |

| Returns | Description |
|---------|-------------|
| `List<VectorDocumentStruct>` | Top K most similar documents, ranked by similarity |

**FlutterFlow Usage:**
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
        App->>downloadEmbeddingModel: Call with model URL
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
| Download model | `downloadEmbeddingModel` | modelUrl, tokenizerUrl | "Success" or "Error: ..." |
| Convert text to vectors | `processDocumentsToVectors` | List&lt;String&gt; | List&lt;VectorDocumentStruct&gt; |
| Save to database | `saveVectorsToDb` | List&lt;VectorDocumentStruct&gt; | void |
| Search similar docs | `findTopMatches` | query, docs, topK | List&lt;VectorDocumentStruct&gt; |
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
