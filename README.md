# AeroMRO-Inspector

> **Hybrid RAG Search Observability & Embedding Space Inspector for Aviation MRO**

항공 정비(MRO) 문서 검색에서 발생하는 **Vocabulary Mismatch**와 Retrieval 품질 문제를 분석하기 위한 검색 관제·검증 플랫폼입니다.

NTSB 항공 사고 조사 보고서와 FAA Service Difficulty Reports(SDR)를 대상으로 **BM25 기반 Lexical Search와 Vector Search를 결합한 Hybrid Retrieval**을 수행하고, 검색 점수·Ranking·Embedding Space를 시각화하여 검색 결과가 왜 선택되었는지 분석할 수 있도록 설계했습니다.

---

## 1. Overview

### Background

항공 MRO 현장에서는 정비사가 사용하는 구어체·현장 용어와 공식 기술 문서에 사용되는 표준 용어 사이에 차이가 존재합니다.

예를 들어 정비사는 특정 증상을 일상적인 표현으로 검색할 수 있지만, 실제 정비 문서에는 다른 기술 용어가 사용될 수 있습니다.

이러한 문제에서는 단순한 Keyword Search만으로는 관련 문서를 충분히 검색하기 어렵고, Vector Search만으로도 정확한 전문 용어 매칭이나 세부 조건을 놓칠 수 있습니다.

**AeroMRO-Inspector**는 이러한 Retrieval 문제를 단순히 해결하는 RAG 애플리케이션이 아니라,

```text
Data
  ↓
Retrieval
  ↓
Ranking
  ↓
Visualization
  ↓
Evaluation
```

전체 검색 과정을 관찰하고 분석하는 **Search Observability Platform**을 목표로 합니다.

### Scope

* NTSB Accident Investigation Reports
* FAA Service Difficulty Reports (SDR)
* Document / Chunk 기반 검색
* BM25 Lexical Search
* Vector Similarity Search
* Hybrid Ranking
* Search Result Inspection
* Embedding Space Visualization
* Retrieval Quality Evaluation

---

## 2. Key Features

### Hybrid Retrieval

* BM25 기반 Keyword / Lexical Search
* Embedding 기반 Vector Search
* 검색 결과 Score Normalization
* BM25 + Vector Hybrid Ranking
* Top-K Retrieval
* Retrieval 결과별 Score Breakdown

### Search Inspector

* Query 입력 및 검색
* BM25 / Vector / Hybrid Score 비교
* Ranking 결과 확인
* Retrieved Chunk 상세 확인
* 검색 결과의 Ranking 변화 분석
* Search Trace 기반 Retrieval 과정 확인

### Embedding Space

* Document Chunk Embedding 생성
* UMAP 기반 2D Projection
* WebGL 기반 대규모 Point Visualization
* Query와 Retrieved Chunk의 공간적 관계 확인
* 검색 결과 Highlight
* Document / Source / Cluster 기반 Filtering

### Search Quality Evaluation

* Evaluation Query Dataset
* Relevance Label
* Recall@K
* Precision@K
* MRR
* nDCG
* Search Latency 측정

### Experiment

* Embedding Model 비교
* Chunk Size / Overlap 비교
* BM25 / Vector Weight 비교
* Retrieval 설정별 결과 비교
* Experiment 결과 기록

### Async Processing

* Document Ingestion Worker
* Embedding Worker
* UMAP Projection Worker
* Evaluation Worker
* Job Status / Retry / Failure 관리

---

## 3. Architecture

### System Architecture

```text
┌────────────────────────────────────────────────────────────────────┐
│                         AeroMRO-Inspector                          │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────── Frontend ──────────────────────────────┐
│                         React / TypeScript                         │
│                                                                    │
│  Dashboard  │  Search Inspector  │  Embedding Space  │ Evaluation │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ HTTP API
                               ▼
┌──────────────────────────── Backend ───────────────────────────────┐
│                           FastAPI                                  │
│                                                                    │
│  Search API │ Document API │ Evaluation API │ Experiment API      │
└───────────────┬───────────────────────┬────────────────────────────┘
                │                       │
                ▼                       ▼
┌─────────────────────────┐   ┌──────────────────────────────────────┐
│      Retrieval          │   │          Async Workers               │
│                         │   │                                      │
│  ┌───────┐  ┌────────┐ │   │ Ingestion │ Embedding │ UMAP │ Eval │
│  │ BM25  │  │ Vector │ │   └──────────────────────────────────────┘
│  └───┬───┘  └───┬────┘ │
│      └──────┬────┘      │
│             ▼           │
│      Hybrid Ranking     │
└─────────────┬───────────┘
              │
              ▼
┌────────────────────────────────────────────────────────────────────┐
│                         PostgreSQL                                 │
│                                                                    │
│ Documents │ Chunks │ Embeddings │ Search Logs │ Evaluations       │
│ Experiments │ Jobs │ Embedding Points                           │
└────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```mermaid
flowchart TD
    A[NTSB Reports] --> C[Data Ingestion]
    B[FAA SDR] --> C

    C --> D[Normalization]
    D --> E[Chunking]
    E --> F[Document / Chunk Storage]

    E --> G[Embedding Worker]
    G --> H[Vector Index]

    Q[User Query] --> I[Search API]

    I --> J[BM25 Search]
    I --> K[Vector Search]

    J --> L[Score Normalization]
    K --> L

    L --> M[Hybrid Ranking]
    M --> N[Top-K Results]

    N --> O[Search Inspector]

    H --> P[UMAP Worker]
    P --> R[Embedding Space]
    R --> O

    N --> S[Evaluation]
    S --> T[Metrics]
```

### Retrieval Flow

```text
User Query
    │
    ├───────────────┐
    ▼               ▼
 BM25            Embedding
 Search            │
    │              ▼
    │          Vector Search
    │               │
    └───────┬───────┘
            ▼
    Score Normalization
            │
            ▼
      Hybrid Ranking
            │
            ▼
         Top-K
            │
            ├── Search Inspector
            ├── Search Trace
            └── Evaluation
```

---

## 4. Tech Stack

| Area            | Technology               | Purpose                              |
| --------------- | ------------------------ | ------------------------------------ |
| Frontend        | React                    | Web Application                      |
| Frontend        | TypeScript               | Type-safe application development    |
| Data Fetching   | TanStack Query           | Server State / API Management        |
| Visualization   | Deck.gl / WebGL          | Embedding Space Visualization        |
| Backend         | FastAPI                  | REST API / Search Service            |
| Language        | Python                   | Retrieval / Data Processing          |
| Search          | BM25                     | Lexical Retrieval                    |
| Vector Search   | PostgreSQL + pgvector    | Embedding Search                     |
| Embedding       | Sentence Embedding Model | Semantic Representation              |
| Projection      | UMAP                     | High-dimensional → 2D Projection     |
| Database        | PostgreSQL               | Document / Search / Evaluation Data  |
| Worker          | Python Worker            | Heavy Async Processing               |
| Container       | Docker                   | Development / Deployment Environment |
| Reverse Proxy   | Nginx                    | API / Frontend Routing               |
| Testing         | Pytest / Vitest          | Backend / Frontend Testing           |
| Version Control | Git / GitHub             | Source Code Management               |

---

## 5. Project Structure

```text
aero-mro-inspector/
│
├── apps/
│   │
│   ├── web/                         # React Frontend
│   │   ├── src/
│   │   │   ├── components/          # Shared UI Components
│   │   │   ├── features/
│   │   │   │   ├── search/          # Search Inspector
│   │   │   │   ├── embedding/       # Embedding Space
│   │   │   │   ├── documents/       # Document Explorer
│   │   │   │   ├── evaluation/      # Evaluation UI
│   │   │   │   └── experiments/     # Experiment UI
│   │   │   ├── pages/               # Application Pages
│   │   │   ├── api/                 # API Client
│   │   │   └── app/                 # Application Configuration
│   │   └── package.json
│   │
│   └── api/                         # FastAPI Backend
│       ├── app/
│       │   ├── api/                 # API Routes
│       │   ├── domain/              # Domain Models / Business Logic
│       │   ├── retrieval/           # BM25 / Vector / Hybrid Search
│       │   ├── embedding/           # Embedding Pipeline
│       │   ├── evaluation/          # Retrieval Evaluation
│       │   ├── experiments/         # Experiment Management
│       │   ├── documents/           # Document Processing
│       │   └── core/                # Configuration / Infrastructure
│       └── tests/
│
├── workers/                         # Async Processing Workers
│   ├── ingestion/                   # Data Ingestion
│   ├── embedding/                   # Embedding Generation
│   ├── projection/                  # UMAP Projection
│   └── evaluation/                  # Evaluation Jobs
│
├── packages/
│   ├── schemas/                     # Shared Data Schemas
│   └── config/                      # Shared Configuration
│
├── data/
│   ├── raw/                         # Raw Source Data
│   ├── processed/                   # Normalized Data
│   └── evaluation/                  # Evaluation Dataset
│
├── infrastructure/
│   ├── docker/                      # Docker Configuration
│   └── nginx/                       # Reverse Proxy Configuration
│
├── scripts/
│   ├── ingest.py                    # Data Ingestion
│   ├── embed.py                     # Embedding Generation
│   └── evaluate.py                  # Evaluation
│
├── docker-compose.yml
├── .env.example
├── Makefile
├── README.md
└── LICENSE
```

### Dependency Flow

```text
┌──────────────┐
│   web        │
│ React        │
└──────┬───────┘
       │ HTTP
       ▼
┌──────────────┐
│     api      │
│   FastAPI    │
└──────┬───────┘
       │
       ├──────────────┐
       ▼              ▼
┌──────────────┐ ┌──────────────┐
│  retrieval   │ │   workers    │
└──────┬───────┘ └──────┬───────┘
       │                │
       └───────┬────────┘
               ▼
        ┌─────────────┐
        │ PostgreSQL  │
        │ + pgvector  │
        └─────────────┘
```

Dependency direction:

```text
Frontend
   ↓
API
   ↓
Domain / Retrieval / Evaluation
   ↓
Infrastructure / Database
```

---

## 6. Getting Started

### Prerequisites

Before starting the project, install the following:

* Git
* Docker
* Docker Compose
* Node.js 20+
* Python 3.11+
* npm / pnpm

Verify the environment:

```bash
git --version
docker --version
docker compose version
node --version
python --version
```

### Installation & Run

#### 1. Clone Repository

```bash
git clone https://github.com/<your-account>/aero-mro-inspector.git
cd aero-mro-inspector
```

#### 2. Configure Environment

```bash
cp .env.example .env
```

Configure required environment variables:

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/aeromro
EMBEDDING_MODEL=<embedding-model>
API_PORT=8000
WEB_PORT=3000
```

#### 3. Start Infrastructure

```bash
docker compose up -d postgres
```

Check running containers:

```bash
docker compose ps
```

#### 4. Install Backend Dependencies

```bash
cd apps/api

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

#### 5. Run Database Migration

```bash
alembic upgrade head
```

#### 6. Start Backend

```bash
uvicorn app.main:app --reload --port 8000
```

The API will be available at:

```text
http://localhost:8000
```

API documentation:

```text
http://localhost:8000/docs
```

#### 7. Install Frontend Dependencies

Open another terminal:

```bash
cd apps/web
npm install
```

#### 8. Start Frontend

```bash
npm run dev
```

The web application will be available at:

```text
http://localhost:3000
```

---

### Data Ingestion

Load source documents:

```bash
python scripts/ingest.py
```

Generate embeddings:

```bash
python scripts/embed.py
```

Generate Embedding Space projection:

```bash
python scripts/project.py
```

Run retrieval evaluation:

```bash
python scripts/evaluate.py
```

---

### Testing

Backend tests:

```bash
cd apps/api
pytest
```

Frontend tests:

```bash
cd apps/web
npm test
```

Build frontend:

```bash
npm run build
```

Run the complete development environment:

```bash
docker compose up
```

---

## Project Status

> **MVP — In Development**

Current MVP focuses on the three core capabilities:

```text
┌─────────────────────────────────────────┐
│          AeroMRO-Inspector MVP          │
├─────────────────────────────────────────┤
│                                         │
│  01  Hybrid Search                      │
│      BM25 + Vector Search               │
│                                         │
│  02  Search Inspector                   │
│      Score + Ranking + Retrieval Trace  │
│                                         │
│  03  Embedding Space                    │
│      UMAP + WebGL Visualization         │
│                                         │
└─────────────────────────────────────────┘
```

The system is designed to be extended with retrieval evaluation, experiment management, reranking, asynchronous job orchestration, and additional observability capabilities.
