# BareMetalDB

A vector database built from scratch in C++ — no external ML or database
libraries. Implements **HNSW**, **KD-Tree**, and **Brute Force** search
side by side, plus a full **RAG (Retrieval-Augmented Generation) pipeline**
powered by a local LLM via [Ollama](https://ollama.com).

> Built as an educational deep-dive into how production vector databases
> like Pinecone, Weaviate, and Chroma actually work under the hood.

---

## Features

| Feature | Description |
|---|---|
| **3 Search Algorithms** | HNSW (production-grade approximate NN), KD-Tree (exact, space-partitioning), Brute Force (exact, baseline) — run all three and compare speed |
| **3 Distance Metrics** | Cosine similarity, Euclidean distance, Manhattan distance |
| **16D Demo Dataset** | 20 hand-crafted vectors across 4 categories (CS, Math, Food, Sports) |
| **2D PCA Visualization** | Live scatter plot of the semantic space via power-iteration PCA |
| **Real Document Embeddings** | Paste any text → Ollama embeds it with `nomic-embed-text` (768D) |
| **RAG Pipeline** | Ask a question → HNSW retrieves relevant chunks → local LLM answers grounded in your documents |
| **Full REST API** | Insert, delete, search, benchmark, and inspect the HNSW graph structure over HTTP |
| **Zero external dependencies** | No ML frameworks, no vector DB libraries — every algorithm is implemented from scratch in a single C++ file |

---

## How It Works

```
Your Text
    │
    ▼
Ollama (nomic-embed-text)     ← converts text into a 768-dimensional vector
    │
    ▼
HNSW Index (C++)              ← indexes the vector in a multilayer graph
    │
    ▼
Semantic Search                ← finds nearest neighbors in vector space
    │
    ▼
Ollama (llama3.2)              ← reads retrieved chunks, generates an answer
    │
    ▼
Answer
```

**HNSW (Hierarchical Navigable Small World)** is the same algorithm used by
Pinecone, Weaviate, Chroma, and Milvus. It builds a multilayer graph where
each layer is progressively sparser — searches start at the top layer and
zoom in, achieving **O(log N)** average complexity instead of **O(N)** for
brute force.

---

## Prerequisites

You need three things installed:

1. **MSYS2** (or any C++17-capable compiler) — provides `g++`
2. **Git**
3. **Ollama** — runs the local embedding and generation models

---

## Setup (Windows)

### 1. Install a C++ compiler via MSYS2
1. Download from [msys2.org](https://www.msys2.org) and run the installer (default path `C:\msys64`).
2. Open **MSYS2 UCRT64** from the Start Menu and update:
   ```
   pacman -Syu
   ```
3. Install the compiler:
   ```
   pacman -S mingw-w64-ucrt-x86_64-gcc
   ```
4. Add `C:\msys64\ucrt64\bin` to your **System PATH** (make sure it comes
   before any other MinGW installation you might have).
5. Open a **new** terminal and verify:
   ```
   g++ --version
   ```

### 2. Install Ollama and pull the models
1. Download from [ollama.com](https://ollama.com) and install.
2. Pull the two required models:
   ```
   ollama pull nomic-embed-text
   ollama pull llama3.2
   ```
3. Verify:
   ```
   ollama list
   ```

> **Minimum specs:** 8GB RAM recommended. The two models use ~3GB combined.

### 3. Clone this repository
```
git clone https://github.com/Mukul230-glitxh/BareMetalDB.git
cd BareMetalDB
```

### 4. Compile
```
g++ -std=c++17 -O2 main.cpp -o db -lws2_32
```
`-lws2_32` links Windows socket support, required by the embedded HTTP server.

### 5. Run

**Terminal 1** — start Ollama (skip if it's already running in the system tray):
```
ollama serve
```

**Terminal 2** — start BareMetalDB:
```
.\db.exe
```
Expected output:
```
=== BareMetalDB Engine ===
http://localhost:8080
20 demo vectors | 16 dims | HNSW+KD-Tree+BruteForce
Ollama: ONLINE
  embed model: nomic-embed-text  gen model: llama3.2
```

Open your browser to **http://localhost:8080**.

---

## Using the Application

### Tab 1 — Search (Demo Vectors)
- Type a concept (`binary tree`, `sushi`, `basketball`, `calculus`) and pick
  an algorithm (HNSW / KD-Tree / Brute Force) and a distance metric.
- **Search** returns the nearest neighbors; the matching point highlights on
  the PCA scatter plot.
- **Compare All Algos** runs all three and shows their timing side by side.

### Tab 2 — Documents (Real Embeddings)
- Paste a title and body text. Long documents are automatically split into
  overlapping ~250-word chunks, each embedded and indexed independently.

### Tab 3 — Ask AI (RAG Pipeline)
- Insert some documents first, then ask a question about them.
- Behind the scenes: your question is embedded → HNSW retrieves the top-k
  most relevant chunks → those chunks are sent to `llama3.2` as grounding
  context → the model answers using only that context.

---

## REST API Reference

### Demo Vector Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/search?v=f1,f2,...&k=5&metric=cosine&algo=hnsw` | k-NN search |
| `POST` | `/insert` | Insert a demo vector — body: `{"label":"...","data":[...]}` |
| `DELETE` | `/delete/:id` | Delete by ID |
| `GET` | `/items` | List all demo vectors with PCA coordinates |
| `GET` | `/benchmark?v=...&k=5&metric=cosine` | Compare all 3 algorithms |
| `GET` | `/hnsw-info` | HNSW graph structure and layer stats |
| `GET` | `/stats` | Database statistics |
| `GET` | `/status` | Ollama online/offline + model names |

### Document & RAG Endpoints

| Method | Endpoint | Body | Description |
|---|---|---|---|
| `POST` | `/doc/insert` | `{"title":"...","text":"..."}` | Chunk, embed, and store a document |
| `GET` | `/doc/list` | — | List all stored chunks |
| `DELETE` | `/doc/delete/:id` | — | Delete a chunk |
| `POST` | `/doc/ask` | `{"question":"...","k":3}` | RAG: retrieve + generate |

### Example — search via curl
```bash
curl "http://localhost:8080/search?v=0.9,0.8,0.7,0.6,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1,0.1&k=3&metric=cosine&algo=hnsw"
```

### Example — ask a question via curl
```bash
curl -X POST http://localhost:8080/doc/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is dynamic programming?","k":3}'
```

---

## Project Structure

```
BareMetalDB/
├── main.cpp        ← C++ backend: BruteForce, KD-Tree, HNSW, REST API, RAG pipeline
├── httplib.h       ← Single-header HTTP server library (cpp-httplib)
├── index.html      ← Frontend: PCA scatter plot, search UI, chat interface
└── README.md       ← This file
```

### Architecture (`main.cpp`)

```
BruteForce      O(N·d)      Exact, baseline
KDTree          O(log N)*   Exact, axis-aligned partitioning (*degrades in high dimensions)
HNSW            O(log N)    Approximate, multilayer small-world graph

VectorDB        Unified interface over all 3 (16D demo vectors)
DocumentDB      HNSW-only index for real Ollama embeddings (768D)
OllamaClient    HTTP client → /api/embeddings + /api/generate
```

---

## Algorithm Deep Dive

### HNSW (Hierarchical Navigable Small World)
Nodes are inserted into a multilayer graph. Each node is randomly assigned
a maximum layer via an exponentially decaying distribution. Layer 0 holds
every node with many short-range connections; higher layers hold
exponentially fewer nodes with longer-range connections.

- **Insert:** Start at the top layer, greedily walk to the nearest node,
  drop a layer, repeat. From the node's assigned max layer down to 0, run a
  beam search (`efConstruction`) and connect to the `M` nearest neighbors
  bidirectionally, pruning any neighbor that exceeds `M` connections.
- **Search:** Same greedy top-down descent, then a beam search with width
  `ef` at layer 0.
- **Why it's fast:** The upper layers act as a highway — you reach the
  right neighborhood in a few hops, then refine at layer 0.

### KD-Tree
Binary space partitioning. Each node splits space along one dimension,
cycling through dimensions with tree depth. Search prunes entire subtrees
when the closest possible point on the far side of a split can't beat the
current best candidate.

**Weakness:** In high dimensions, almost every point sits near the
boundary of the search hypersphere, so pruning stops helping — this is why
KD-Tree degrades toward brute-force performance at 768D, and why HNSW is
used for the real embedding search in this project.

---

## Common Issues

| Problem | Fix |
|---|---|
| `Ollama: OFFLINE` in header | Run `ollama serve` in a terminal |
| Embedding takes forever | Ollama is loading the model on first use — wait ~1–2 min |
| `g++: command not found` | Add `C:\msys64\ucrt64\bin` to your PATH |
| `undefined reference to WSA...` | Missing `-lws2_32` flag |
| Port 8080 already in use | `netstat -ano \| findstr 8080` then `taskkill /PID <pid> /F` |
| LLM answers are slow | Normal on CPU (10–30s). Try a smaller model — see below |

### Use a smaller/faster LLM
```
ollama pull llama3.2:1b
```
Then update the `genModel` string in `main.cpp`'s `OllamaClient` constructor
call to `"llama3.2:1b"`, recompile, and restart.

---

## Known Limitations

- **HNSW deletion is not supported.** Proper HNSW deletion requires
  tombstoning and periodic graph repair, which is out of scope for this
  project — deleted vectors are removed from Brute Force/KD-Tree results
  but may still be traversed internally by the HNSW graph.
- **JSON parsing is hand-rolled**, not a general-purpose parser — it works
  for this project's fixed request/response shapes but isn't meant to
  handle arbitrary JSON.

---

## License

MIT — use this however you want.
