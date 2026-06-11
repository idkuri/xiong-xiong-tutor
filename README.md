# 熊熊 (Bear-learn) 🐻

A privacy-first, offline study companion for Windows. Upload PDFs and 熊熊 turns them
into flashcards, quizzes, and an adaptive teddy-bear tutor — all running locally via
Ollama. See [PRD.md](PRD.md) for the full product spec.

## Architecture

```
frontend/   React + TypeScript + Tailwind + Zustand (Vite)
backend/    Python FastAPI — PyMuPDF, chunking, BGE-M3 embeddings,
            ChromaDB + SQLite FTS5 hybrid RAG, tutor engine
```

The desktop shell (Tauri) is planned; development runs as Vite dev server + local API.
The backend listens on `127.0.0.1:8731`; the frontend dev server proxies `/api` to it.

## Prerequisites

- Windows 10/11, Python 3.11+, Node 20+
- [Ollama](https://ollama.com) installed, with models:
  `ollama pull bge-m3` (embeddings, required) and at least one generation model
  (e.g. `ollama pull qwen3:4b`; `qwen2.5:14b-instruct-q4_k_m` recommended on 12 GB GPUs)

## Setup

```powershell
# backend
cd backend
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt

# frontend
cd ..\frontend
npm install
```

## Run

```powershell
.\start.ps1          # PowerShell
```

```bash
./start.sh           # Git Bash / MSYS
```

Both start Ollama, the backend, and the frontend together. They pick the right Ollama GPU backend automatically: **CUDA for NVIDIA**,
**Vulkan for AMD** on Windows (and clears `HSA_OVERRIDE_GFX_VERSION` if present).

or manually:

```powershell
# NVIDIA (CUDA) — do not set OLLAMA_VULKAN
Remove-Item Env:OLLAMA_VULKAN -ErrorAction SilentlyContinue
# AMD on Windows — Vulkan instead of ROCm
$env:OLLAMA_VULKAN = "1"
Remove-Item Env:HSA_OVERRIDE_GFX_VERSION -ErrorAction SilentlyContinue
ollama serve                                 # if not already running
cd backend; .venv\Scripts\python run.py      # API on :8731
cd frontend; npm run dev                     # UI on :5173
```

Open http://localhost:5173.

## Configuration

- `backend/config.json` — chunking heuristics, retrieval N/K, context window
  (created with defaults on first run; tunable without code changes)
- Generation model is selectable in **Settings** inside the app
- All data lives in `backend/data/` (SQLite + ChromaDB + uploads); delete it to reset

## Building the Windows executable

Prerequisites: Rust (MSVC toolchain) + Visual Studio Build Tools (C++ workload).

**One command** (builds everything and creates a shareable zip):

```powershell
.\build-distributable.ps1
```

Output: `dist\熊熊-0.1.0-windows-x64.zip` — users unzip and run the setup `.exe` inside.

Manual steps (if you prefer):

```powershell
# 1. Freeze the backend into a single exe
cd backend
.venv\Scripts\pyinstaller --noconfirm bearn-backend.spec

# 2. Place it where Tauri expects the sidecar (target-triple suffix required)
Copy-Item dist\bearn-backend.exe `
    ..\frontend\src-tauri\binaries\bearn-backend-x86_64-pc-windows-msvc.exe

# 3. Build the app + NSIS installer
cd ..\frontend
npx tauri build
```

The NSIS installer is at `frontend/src-tauri/target/release/bundle/nsis/熊熊_0.1.0_x64-setup.exe`.
The standalone app is at `frontend/src-tauri/target/release/Bearn.exe` (the binary
keeps an ASCII name via `mainBinaryName`; the app displays as 熊熊).
The packaged app starts the backend sidecar automatically, starts Ollama if it
isn't running, and stores user data in `%APPDATA%\Bearn`. On first launch, a
setup wizard downloads and installs Ollama (if needed) and pulls the required
models (`bge-m3` for embeddings and the default generation model).

## Development notes

- `backend/make_test_pdf.py` generates a small sample PDF for testing the pipeline
- API docs available at http://127.0.0.1:8731/docs while the backend is running
