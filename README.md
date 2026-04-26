# VectorMind AI — Vector Database Engine in C++

A production-inspired vector database built from scratch in C++, with a live web UI, REST API, and a local RAG pipeline. Built to understand how **Pinecone**, **Weaviate**, and **Chroma** work under the hood.

---

## What It Does

- Implements **3 search algorithms** side-by-side: HNSW, KD-Tree, and Brute Force
- Supports **3 distance metrics**: Cosine similarity, Euclidean, Manhattan
- Visualizes **semantic clusters** in real-time via a 2D PCA scatter plot
- Embeds real text using **Ollama** (`nomic-embed-text`, 768D vectors)
- Answers questions via a **RAG pipeline**: HNSW retrieval → local LLM (`llama3.2`)
- Exposes a full **REST API** for insert, search, delete, and benchmark operations

---

## How the RAG Pipeline Works

```
Your Text
   │
   ▼
Ollama (nomic-embed-text)     →  converts text to a 768D vector
   │
   ▼
HNSW Index (C++)              →  indexes the vector in a multilayer graph
   │
   ▼
Semantic Search               →  finds nearest neighbor chunks
   │
   ▼
Ollama (llama3.2)             →  generates an answer from retrieved context
   │
   ▼
Answer
```

---

## Why HNSW?

HNSW (Hierarchical Navigable Small World) is the algorithm behind Pinecone, Weaviate, Chroma, and Milvus.

It builds a **multilayer graph** — upper layers act as a highway for fast navigation, lower layers zoom in for precision. This achieves **O(log N)** search complexity vs O(N) for brute force.

KD-Trees degrade in high dimensions (curse of dimensionality). HNSW's graph-based traversal avoids this, making it the industry standard for 100D+ embeddings.

---

## Prerequisites

- **MSYS2** — C++ compiler (Windows)
- **Git**
- **Ollama** — runs local AI models (8GB RAM recommended)

---

## Setup (Windows)

### 1. Install MSYS2

Download from [msys2.org](https://www.msys2.org). Keep the default install path (`C:\msys64`).

Open **MSYS2 UCRT64** from the Start Menu and run:

```bash
pacman -Syu
pacman -S mingw-w64-ucrt-x86_64-gcc
```

Add `C:\msys64\ucrt64\bin` to your Windows **System PATH** (`Win+R` → `sysdm.cpl` → Advanced → Environment Variables).

Verify:
```bash
g++ --version   # should print GCC 15.x.x
```

### 2. Install Git

Download from [git-scm.com](https://git-scm.com/download/win). Default settings are fine.

### 3. Install Ollama

Download from [ollama.com](https://ollama.com). After install, pull the two required models:

```bash
ollama pull nomic-embed-text   # ~274 MB — embedding model
ollama pull llama3.2           # ~2 GB  — language model
```

### 4. Clone and Compile

```bash
git clone https://github.com/YOUR_USERNAME/VectorMindAI.git
cd VectorMindAI

g++ -std=c++17 -O2 main.cpp -o db -lws2_32
```

Compilation takes 10–20 seconds and produces `db.exe`.

### 5. Run

```bash
# Terminal 1 (if Ollama isn't already running)
ollama serve

# Terminal 2
./db
```

Open **http://localhost:8080** in your browser.

---

## Using the App

### Tab 1 — Search (Demo Vectors)

20 pre-loaded 16D semantic vectors across 4 categories (CS, Math, Food, Sports).

- Type any concept: `binary tree`, `sushi`, `basketball`, `calculus`
- Choose algorithm (HNSW / KD-Tree / Brute Force) and distance metric
- Click **⚡ SEARCH** — results appear with distances, matching point highlighted on the scatter plot
- Click **▶ COMPARE ALL ALGOS** to benchmark all 3 algorithms simultaneously

The scatter plot projects all vectors to 2D via PCA — you can see semantic clusters form visually.

### Tab 2 — Documents (Real Embeddings)

Embed any text using Ollama's `nomic-embed-text` model (768D).

- Paste lecture notes, articles, or any text
- Long documents are auto-split into overlapping 250-word chunks
- Each chunk is embedded and stored in a dedicated HNSW index

### Tab 3 — Ask AI (RAG Pipeline)

Requires documents inserted in Tab 2.

1. Your question is embedded (768D vector)
2. HNSW retrieves the 3 most semantically similar chunks
3. Chunks are passed as context to `llama3.2`
4. The answer streams in with a typewriter effect

Click the context chips to see exactly which chunks the model used.

---

## REST API

### Demo Vectors

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/search?v=f1,f2,...&k=5&metric=cosine&algo=hnsw` | K-NN search |
| `POST` | `/insert` | Insert a vector |
| `DELETE` | `/delete/:id` | Delete by ID |
| `GET` | `/benchmark?v=...&k=5&metric=cosine` | Compare all 3 algorithms |
| `GET` | `/hnsw-info` | Graph structure and layer stats |
| `GET` | `/stats` | Database statistics |

### Documents & RAG

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| `POST` | `/doc/insert` | `{"title":"...","text":"..."}` | Embed and store document |
| `GET` | `/doc/list` | — | List all stored documents |
| `DELETE` | `/doc/delete/:id` | — | Delete a chunk |
| `POST` | `/doc/ask` | `{"question":"...","k":3}` | RAG: retrieve + generate |
| `GET` | `/status` | — | Ollama status and model info |

**Example:**
```bash
# Search
curl "http://localhost:8080/search?v=0.9,0.8,0.7,0.6,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1&k=3&metric=cosine&algo=hnsw"

# Ask
curl -X POST http://localhost:8080/doc/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is dynamic programming?","k":3}'
```

---

## Project Structure

```
VectorDB/
├── main.cpp       ← C++ backend: HNSW, KD-Tree, Brute Force, REST API, RAG
├── httplib.h      ← Single-header HTTP library (cpp-httplib)
├── index.html     ← Frontend: PCA scatter plot, chat UI, benchmark view
└── README.md
```

### Core Components

| Component | Complexity | Notes |
|-----------|------------|-------|
| `BruteForce` | O(N·d) | Exact, used as baseline |
| `KDTree` | O(log N) | Exact, works well up to ~20D |
| `HNSW` | O(log N) | Approximate, scales to 768D+ |
| `DocumentDB` | — | HNSW-only index for Ollama embeddings |
| `OllamaClient` | — | HTTP client for `/api/embeddings` and `/api/generate` |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `Ollama: OFFLINE` in header | Run `ollama serve` in a terminal |
| `g++: command not found` | Add `C:\msys64\ucrt64\bin` to Windows PATH |
| Port 8080 already in use | `netstat -ano \| findstr 8080` → `taskkill /PID <pid> /F` |
| LLM responses are slow | Normal on CPU — switch to `llama3.2:1b` for ~3× faster answers |

**Using a faster model:**
```bash
ollama pull llama3.2:1b
```
Then update `main.cpp`:
```cpp
std::string genModel = "llama3.2:1b";
```
Recompile and restart.

---

## License

MIT
