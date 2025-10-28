# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Latent Scope is a tool for visualizing and exploring datasets through latent spaces. It combines a Python backend (Flask) with a React frontend to provide a workflow for embedding, projecting (UMAP), clustering (HDBSCAN), labeling (LLM), and exploring unstructured text data.

**Core Philosophy:**
- All data stored in flat files (parquet, h5, json) - no databases
- Everything indexed by row position in the input dataset
- All intermediate steps and parameters preserved for reproducibility
- Designed for research and experimentation

## Development Setup

### Python Development
```bash
# Install in development mode
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -e .

# Run the server
ls-serve ~/latent-scope-data
```

Requires Python 3.12 (recommended). The `-e` flag installs in editable mode so code changes are immediately reflected.

### Web Development
```bash
cd web
npm install
npm run dev  # Runs on http://localhost:5174, calls API at :5001
```

The React dev server hot-reloads on changes. It communicates with the Flask backend at localhost:5001.

### Building for Distribution
```bash
# Build and test in a clean environment
./build.sh 0.6.0  # Replace with actual version number

# This will:
# 1. Build the React app (npm run production)
# 2. Bundle it into the Python package
# 3. Create a wheel in dist/
# 4. Install it in a test venv for verification
```

The build process (defined in setup.py) runs `npm run production` in the web directory and copies the built assets into `latentscope/web/dist/` before packaging.

## Architecture

### Python Backend Structure

**latentscope/server/** - Flask application
- `app.py` - Main Flask app setup and dataset routes
- `jobs.py` - Subprocess management for long-running CLI scripts; polls progress
- `search.py` - Nearest neighbor search using embeddings
- `tags.py` - User-created tags/selections in the UI
- `datasets.py` - Dataset CRUD and metadata operations
- `admin.py` - Admin operations and bulk imports
- `bulk.py` - Bulk operations on datasets

**latentscope/scripts/** - CLI commands (also called via jobs.py from UI)
- `ingest.py` - Convert CSV to parquet, setup metadata
- `embed.py` - Generate embeddings using various models
- `umapper.py` - Dimensionality reduction with UMAP
- `cluster.py` - Clustering with HDBSCAN
- `label_clusters.py` - Auto-label clusters using LLMs
- `scope.py` - Tie together embed/umap/cluster/labels into explorable config
- `export_plot.py` - Export visualizations
- `sae.py` - Sparse autoencoder operations

Each script has both a Python interface and CLI entry point (defined in setup.py entry_points).

**latentscope/models/** - Model abstraction layer
- `embedding_models.json` - Configuration for all embedding models
- `chat_models.json` - Configuration for all LLM models
- `providers/` - Provider implementations (transformers, openai, cohere, mistralai, voyageai, togetherai)

The provider system abstracts different APIs/local models behind uniform interfaces via `get_embedding_model(id)` and `get_chat_model(id)`.

**latentscope/util/** - Configuration and utilities
- Handles dotenv for API keys and DATA_DIR configuration

### Frontend Structure (web/src/)

**pages/** - React pages (routing via react-router-dom)
- Home - List datasets and scopes
- Setup - Configure scopes (run the pipeline)
- Explore - Interactive visualization and data exploration
- Jobs/Job - Monitor running processes

**components/** - Reusable React components
Notable: Uses regl-scatterplot for high-performance WebGL scatter plots

### Data Directory Structure

Each dataset gets a directory in `DATA_DIR` (e.g., ~/latent-scope-data/dataset-name/):
```
dataset-name/
├── input.parquet              # Source data
├── meta.json                  # Dataset metadata
├── embeddings/
│   ├── embedding-001.h5       # Vectors in HDF5
│   └── embedding-001.json     # Parameters used
├── umaps/
│   ├── umap-001.parquet       # 2D coordinates
│   ├── umap-001.json          # Parameters
│   └── umap-001.png           # Thumbnail
├── clusters/
│   ├── clusters-001.parquet   # Cluster assignments
│   ├── clusters-001-labels-001.parquet  # LLM labels
│   └── clusters-001.json      # Parameters
├── scopes/
│   └── scopes-001.json        # Combines embed+umap+cluster+labels
└── tags/
    └── ❤️.indices             # User selections from UI
```

All files use standard formats (parquet, json, h5) for easy external access.

## Common Commands

### CLI Workflow (Alternative to Web UI)
```bash
# Initialize data directory with optional API keys
ls-init ~/latent-scope-data --openai_key=XXX --mistral_key=YYY

# Ingest a CSV
ls-ingest-csv "my-dataset" ~/Downloads/data.csv

# List available models
ls-list-models

# Embed the data
ls-embed my-dataset "text_column" transformers-intfloat___e5-small-v2 ""

# Reduce dimensions with UMAP
ls-umap my-dataset embedding-001 25 0.1

# Cluster the 2D points
ls-cluster my-dataset umap-001 5 5

# Label clusters with an LLM
ls-label my-dataset "text_column" cluster-001 openai-gpt-3.5-turbo ""

# Create a scope (explorable configuration)
ls-scope my-dataset embedding-001 umap-001 cluster-001 cluster-001-labels-001 "My Scope" "Description"

# Start the server
ls-serve ~/latent-scope-data
```

### Web Linting
```bash
cd web
npm run lint
```

## Key Design Patterns

**Index-Based Architecture**: All operations reference the input dataset by row index. Embeddings, UMAP coordinates, clusters, and tags all store indices into the original input.parquet, never duplicating the source text.

**Jobs System**: Long-running operations (embed, umap, cluster) are run as subprocesses by jobs.py. The subprocess output is captured and written to jobs/*.json files, which the frontend polls to show progress.

**Metadata Everywhere**: Each operation (embedding, umap, cluster) produces both data files and a corresponding .json metadata file with parameters used. This enables reproducibility and comparison of different configurations.

**Scopes**: A "scope" is a named configuration combining one choice each of: embedding, umap, cluster, and labels. Multiple scopes can exist for the same dataset, allowing instant switching between different parameter sets in the UI.

**Model Providers**: New models are added by:
1. Adding configuration to models/embedding_models.json or chat_models.json
2. Implementing the provider interface if it's a new API (see models/providers/)

## Environment Configuration

Create a `.env` file in the project root (see `.env.example`):
- `LATENT_SCOPE_DATA` - Path to data directory (required)
- API keys for various services (optional, enables those models):
  - `OPENAI_API_KEY`, `OPENAI_BASE_URL`
  - `VOYAGE_API_KEY`
  - `TOGETHER_API_KEY`
  - `COHERE_API_KEY`
  - `MISTRAL_API_KEY`

The `ls-init` command creates this file, or it can be edited manually.

## Important Notes

- Python 3.12 is the recommended and tested version
- The web interface requires Node.js for development
- Mobile is not supported (regl-scatterplot limitations)
- No test suite currently exists - manual testing via web UI or CLI
- Git branch should start with `claude/` for automated pushes to work
