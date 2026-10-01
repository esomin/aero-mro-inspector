# AeroMRO-Inspector

## 제품 및 시스템 설계서

---

# 1. 프로젝트 개요

## 1.1 제품명

**AeroMRO-Inspector**

항공 정비(MRO) 도메인의 사고·결함 데이터를 기반으로 하이브리드 검색 및 RAG 시스템의 검색 품질을 분석하고, 검색 결과와 임베딩 공간을 시각적으로 관제하는 전문가용 검색 품질 검증 플랫폼이다.

---

## 1.2 배경

항공 운항 정비 현장에서는 다음과 같은 형태의 문제가 반복적으로 발생한다.

> "착륙하고 나서 오른쪽 엔진 쪽에서 이상한 진동이 느껴졌는데 비슷한 사례가 있었나?"

현장 정비사의 질의는 자연어·구어체에 가깝지만, 실제 기술 문서는 다음과 같이 표준화된 표현을 사용한다.

* Engine vibration
* Right engine
* Abnormal vibration
* Fan blade damage
* Compressor
* Bearing
* Flight phase
* Maintenance action
* Inspection procedure

따라서 사용자의 표현과 문서의 표현 사이에는 상당한 **Vocabulary Mismatch**가 발생한다.

전통적인 키워드 기반 검색(BM25)만 사용할 경우 동일한 의미를 가진 표현을 충분히 연결하지 못할 수 있으며, 반대로 벡터 검색만 사용할 경우 표면적으로 의미가 유사하지만 실제 정비 절차와 관련성이 낮은 문서가 상위에 노출될 수 있다.

AeroMRO-Inspector는 이러한 문제를 해결하기 위해 다음 세 가지 검색 전략을 결합한다.

1. **Lexical Search**
2. **Semantic Vector Search**
3. **Hybrid Re-ranking**

그리고 최종 검색 결과뿐만 아니라,

* 어떤 문서가 검색되었는지
* BM25 점수가 얼마인지
* Vector Similarity가 얼마인지
* 최종 점수가 어떻게 계산되었는지
* 임베딩 공간에서 해당 문서가 어디에 위치하는지

를 함께 확인할 수 있도록 한다.

---

# 2. 제품 목표

## 2.1 핵심 목표

AeroMRO-Inspector의 목표는 "항공 정비 질문에 답하는 챗봇"이 아니다.

핵심 목표는 다음과 같다.

> **RAG 검색 시스템의 검색 품질과 내부 검색 과정을 관찰하고 검증할 수 있는 Search Intelligence Platform 구축**

이를 위해 검색 결과를 단순히 반환하는 대신 검색 과정 자체를 데이터로 만든다.

---

## 2.2 세부 목표

### Goal 1. 검색 정확도 향상

BM25와 Vector Search를 결합하여 어휘가 다른 질의에서도 관련 문서를 검색할 수 있도록 한다.

### Goal 2. 검색 과정의 설명 가능성 확보

각 결과에 대해 다음 정보를 제공한다.

* BM25 Score
* Vector Similarity
* Hybrid Score
* Rank
* Source
* Document Type
* Chunk
* Matching Terms
* Embedding Distance

### Goal 3. 임베딩 공간 관제

수천 개의 문서 청크를 2D 공간으로 축소하여 검색 질의와 관련 문서의 위치 관계를 시각화한다.

### Goal 4. 검색 품질 평가

검색 결과에 대해 Recall@K, Precision@K, MRR, nDCG 등의 평가 지표를 계산한다.

### Goal 5. 비동기 연산 분리

다음과 같은 무거운 작업을 API 서버에서 분리한다.

* Document ingestion
* Text extraction
* Chunking
* Embedding generation
* Vector indexing
* UMAP
* Evaluation
* Batch re-indexing

---

# 3. 제품 사용자

## 3.1 Primary User

### AI / ML Engineer

목적:

* Embedding 모델 비교
* Retrieval 전략 검증
* Chunking 전략 비교
* Hybrid Search weight 조정
* 검색 품질 평가

---

## 3.2 Secondary User

### Backend / Search Engineer

목적:

* 검색 latency 확인
* Vector DB 상태 확인
* indexing 상태 확인
* query pipeline 분석
* retrieval failure 분석

---

## 3.3 Domain Expert

### MRO / Aviation Domain Expert

목적:

* 검색 결과의 실제 관련성 판단
* 검색 질의와 정비 사례 간 관계 검증
* 검색 결과에 대한 relevance labeling

---

# 4. 데이터 소스

## 4.1 NTSB Accident / Incident Reports

미국 국가교통안전위원회(NTSB)의 항공 사고·사건 조사 보고서를 주요 데이터 소스로 활용한다.

수집 가능한 정보:

```text
Report ID
Aircraft
Aircraft Model
Operator
Date
Location
Event Type
Narrative
Findings
Probable Cause
Recommendations
```

---

## 4.2 FAA Service Difficulty Reports

FAA Service Difficulty Reports(SDR)를 활용하여 실제 항공기 결함 및 정비 사례 데이터를 구성한다.

주요 필드:

```text
Aircraft Make
Aircraft Model
Component
Part
Defect
Description
Corrective Action
Date
ATA Chapter
```

---

# 5. 데이터 처리 파이프라인

전체 데이터 흐름은 다음과 같다.

```text
External Data
     │
     ▼
┌───────────────┐
│ Data Ingestion │
└───────┬───────┘
        │
        ▼
┌────────────────┐
│ Data Normalize │
└───────┬────────┘
        │
        ▼
┌────────────────┐
│ Text Chunking  │
└───────┬────────┘
        │
        ▼
┌────────────────┐
│ Embedding      │
│ Generation     │
└───────┬────────┘
        │
        ├───────────────┐
        ▼               ▼
 PostgreSQL         Vector Index
        │               │
        └───────┬───────┘
                ▼
         Search Pipeline
                │
        ┌───────┴────────┐
        ▼                ▼
   BM25 Search      Vector Search
        │                │
        └───────┬────────┘
                ▼
         Hybrid Ranking
                │
                ▼
         Search Results
                │
        ┌───────┴────────┐
        ▼                ▼
   Result Inspector   Embedding Map
```

---

# 6. 시스템 아키텍처

## 6.1 High-Level Architecture

```text
                         ┌─────────────────────┐
                         │     Web Client      │
                         │ React + TypeScript  │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │      API Layer      │
                         │     FastAPI         │
                         └──────────┬──────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 │                  │                  │
                 ▼                  ▼                  ▼
          Search Service      Document Service    Evaluation
                 │                  │                  │
                 ▼                  ▼                  ▼
          Hybrid Search       PostgreSQL          Metrics DB
                 │
        ┌────────┴─────────┐
        ▼                  ▼
    BM25 Index        Vector Index
        │                  │
        └────────┬─────────┘
                 ▼
            Re-ranker
                 │
                 ▼
          Search Response


             Async Processing
                    │
                    ▼
             ┌────────────┐
             │ Job Queue  │
             └─────┬──────┘
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
     Embedding   UMAP     Evaluation
      Worker     Worker     Worker
```

---

# 7. 기술 스택

## 7.1 Frontend

### React

검색 관제 화면과 데이터 시각화를 담당한다.

### TypeScript

검색 결과, 평가 결과, 데이터 모델 등의 타입 안정성을 확보한다.

### React Query

API 상태 및 서버 데이터를 관리한다.

### Deck.gl

수천~수만 개의 embedding point를 WebGL 기반으로 렌더링한다.

### Map / Scatter Visualization

임베딩 공간을 2D 좌표계로 표현한다.

---

# 8. Backend

## 8.1 API Server

**FastAPI**

역할:

* Search API
* Document API
* Evaluation API
* Embedding metadata API
* Job status API

---

## 8.2 Search Engine

검색 요청은 다음 순서로 처리한다.

```text
User Query
    │
    ▼
Query Normalization
    │
    ├───────────────┐
    ▼               ▼
BM25 Search    Vector Search
    │               │
    │               │
    └───────┬───────┘
            ▼
       Score Normalize
            │
            ▼
      Hybrid Ranking
            │
            ▼
       Top-K Results
            │
            ▼
       Result Inspector
```

---

# 9. Hybrid Search

## 9.1 BM25

BM25는 질의에 포함된 키워드와 문서 간 lexical relevance를 계산한다.

예:

```text
Query:
engine vibration right side

Document A:
right engine vibration detected during landing

Document B:
abnormal compressor noise
```

BM25에서는 `engine`, `vibration`, `right` 등의 직접적인 단어가 존재하는 Document A가 높은 점수를 받을 가능성이 높다.

---

## 9.2 Vector Search

질의를 embedding vector로 변환한다.

```text
Query
 ↓
Embedding Model
 ↓
[0.12, -0.32, 0.81, ...]
 ↓
Vector Similarity
 ↓
Nearest Chunks
```

따라서 다음과 같은 표현도 연결할 수 있다.

```text
"engine shaking"

        ↕

"abnormal engine vibration"
```

---

# 10. Hybrid Score

최종 검색 점수는 두 검색 결과를 결합한다.

기본 모델:

```text
Hybrid Score
=
α × Normalized BM25
+
(1 - α) × Vector Similarity
```

예:

```text
α = 0.4

BM25       = 0.82
Vector     = 0.91

Hybrid
= 0.4 × 0.82
+ 0.6 × 0.91

= 0.874
```

UI에서 α 값을 변경하여 검색 결과가 어떻게 변화하는지 실시간으로 확인할 수 있도록 한다.

---

# 11. Re-ranking

초기 retrieval 단계에서 Top-N을 추출한 뒤 필요할 경우 re-ranker를 적용한다.

```text
BM25 Top 50
       +
Vector Top 50
       ↓
Candidate Merge
       ↓
Top 50
       ↓
Re-ranker
       ↓
Final Top 10
```

이를 통해 retrieval과 ranking을 별도의 단계로 관찰할 수 있다.

---

# 12. Chunking 전략

문서 전체를 하나의 embedding으로 저장하지 않는다.

예:

```text
Original Report
        │
        ├── Chunk 01
        ├── Chunk 02
        ├── Chunk 03
        ├── Chunk 04
        └── ...
```

Chunk metadata:

```json
{
  "document_id": "NTSB-2025-001",
  "chunk_id": "NTSB-2025-001-03",
  "section": "Findings",
  "chunk_index": 3,
  "text": "...",
  "token_count": 384
}
```

---

# 13. Embedding Pipeline

Embedding 생성은 API 요청 과정에서 수행하지 않는다.

```text
Document Created
       ↓
Embedding Job
       ↓
Worker
       ↓
Embedding Model
       ↓
Vector
       ↓
Vector Storage
```

Job 상태:

```text
PENDING
RUNNING
SUCCESS
FAILED
```

---

# 14. UMAP Pipeline

고차원 embedding을 직접 브라우저에 전달하지 않는다.

```text
Embedding
768 dimensions
       ↓
UMAP Worker
       ↓
2D coordinates
       ↓
x, y
       ↓
PostgreSQL
       ↓
Deck.gl
```

예:

```json
{
  "chunk_id": "SDR-000123-04",
  "x": 12.84,
  "y": -5.31,
  "cluster": 7
}
```

---

# 15. Embedding Space Explorer

핵심 화면 중 하나다.

화면에는 수천 개의 chunk가 point 형태로 표시된다.

```text
                  Embedding Space

       · · · · ·         · · ·
     · · · · · · ·     · · · ·
   · · · · · · ·         · ·
                    ★ Query
                 ● Result #1
                ● Result #2

       · · · · ·
        · · · · ·
```

각 point:

* Document Chunk
* Source
* ATA Chapter
* Aircraft
* Similarity
* Search Rank

정보를 가진다.

---

# 16. Query Point

검색 질의를 embedding하여 embedding space에 함께 표시한다.

```text
Query
"right engine vibration after landing"
             │
             ▼
         Embedding
             │
             ▼
        UMAP Transform
             │
             ▼
        Query Point
```

이를 통해 검색 질의 주변에 어떤 문서들이 위치하는지 시각적으로 확인할 수 있다.

---

# 17. Search Inspector

검색 결과를 단순 리스트가 아니라 검색 엔진의 내부 결과로 표현한다.

예:

```text
#1  SDR-001283
────────────────────────────
Hybrid Score       0.923
Vector Similarity  0.951
BM25 Score         0.781

Source             FAA SDR
ATA Chapter        72
Component          Engine

Matched Terms
engine
vibration
right

[View Chunk]
[View Source]
```

---

# 18. Ranking Comparison

사용자가 검색 알고리즘을 비교할 수 있도록 한다.

```text
                BM25      Vector     Hybrid

Result A         #1        #4         #1
Result B         #2        #1         #2
Result C         #8        #2         #3
Result D         #3        #7         #4
```

이를 통해 검색 알고리즘 변경에 따른 ranking 변화가 즉시 드러난다.

---

# 19. Search Evaluation

검색 품질은 정성적으로만 평가하지 않는다.

## 19.1 Recall@K

관련 문서가 Top-K 안에 얼마나 포함되는지 측정한다.

```text
Recall@10
=
Retrieved Relevant Documents
/
Total Relevant Documents
```

---

## 19.2 Precision@K

Top-K 결과 중 실제 관련 문서 비율을 측정한다.

---

## 19.3 MRR

첫 번째 relevant result가 얼마나 상위에 등장하는지를 측정한다.

```text
MRR =
1 / rank(first relevant result)
```

---

## 19.4 nDCG

검색 결과의 순서와 relevance level을 함께 평가한다.

Relevance:

```text
0 = Not Relevant
1 = Related
2 = Relevant
3 = Highly Relevant
```

---

# 20. Evaluation Dataset

검색 품질 검증을 위해 Query / Relevant Document dataset을 구성한다.

예:

```json
{
  "query": "engine vibration after landing",
  "relevant_chunks": [
    "SDR-0012-03",
    "NTSB-2031-05"
  ]
}
```

이를 기준으로 검색 알고리즘을 반복 평가한다.

---

# 21. Evaluation Dashboard

대시보드에서는 다음 지표를 제공한다.

```text
Retrieval Quality

Recall@5       0.81
Recall@10      0.89
MRR            0.76
nDCG@10        0.84

Latency

P50            82 ms
P95            174 ms
P99            310 ms
```

---

# 22. Experiment Management

검색 파라미터 변경을 실험 단위로 저장한다.

예:

```text
Experiment #18

Embedding:
model-A

Chunk Size:
512

Chunk Overlap:
64

BM25 Weight:
0.4

Vector Weight:
0.6

Re-ranker:
Enabled
```

결과:

```text
Recall@10
0.82 → 0.89

MRR
0.71 → 0.78
```

---

# 23. 데이터 모델

## 23.1 documents

```sql
documents
---------
id
source
external_id
title
document_type
aircraft_model
operator
event_date
content
created_at
updated_at
```

---

## 23.2 chunks

```sql
chunks
------
id
document_id
chunk_index
section
content
token_count
created_at
```

---

## 23.3 embeddings

```sql
embeddings
----------
id
chunk_id
model
dimension
vector
created_at
```

Vector column은 PostgreSQL 기반 vector extension을 활용할 수 있다.

---

## 23.4 embedding_points

```sql
embedding_points
----------------
chunk_id
x
y
cluster
projection_version
```

---

## 23.5 search_logs

```sql
search_logs
-----------
id
query
bm25_weight
vector_weight
top_k
latency_ms
created_at
```

---

## 23.6 search_results

```sql
search_results
--------------
id
search_id
chunk_id
rank
bm25_score
vector_score
hybrid_score
rerank_score
```

---

## 23.7 evaluation_queries

```sql
evaluation_queries
------------------
id
query
description
created_at
```

---

## 23.8 relevance_labels

```sql
relevance_labels
----------------
query_id
chunk_id
relevance
annotator
created_at
```

---

# 24. API 설계

## 24.1 Search

```http
POST /api/v1/search
```

Request:

```json
{
  "query": "engine vibration after landing",
  "top_k": 10,
  "bm25_weight": 0.4,
  "vector_weight": 0.6,
  "rerank": true
}
```

Response:

```json
{
  "query": "...",
  "latency_ms": 84,
  "results": [
    {
      "rank": 1,
      "chunk_id": "SDR-123-04",
      "bm25_score": 0.82,
      "vector_score": 0.94,
      "hybrid_score": 0.89
    }
  ]
}
```

---

# 25. Document API

```http
GET    /api/v1/documents
GET    /api/v1/documents/{id}
POST   /api/v1/documents
DELETE /api/v1/documents/{id}
```

---

# 26. Embedding API

```http
GET /api/v1/embeddings
GET /api/v1/embeddings/{chunk_id}
```

---

# 27. Visualization API

```http
GET /api/v1/embedding-space
```

Query:

```text
?projection_version=v3
&cluster=all
```

Response:

```json
{
  "points": [
    {
      "chunk_id": "123",
      "x": 12.42,
      "y": -4.32,
      "cluster": 4
    }
  ]
}
```

---

# 28. Evaluation API

```http
POST /api/v1/evaluations/run
GET  /api/v1/evaluations/{id}
GET  /api/v1/evaluations/{id}/results
```

---

# 29. Job API

```http
GET /api/v1/jobs
GET /api/v1/jobs/{id}
```

Job 상태를 frontend에서 polling 또는 WebSocket/SSE로 관찰할 수 있도록 한다.

---

# 30. Frontend IA

전체 화면 구조:

```text
AeroMRO Inspector

├── Dashboard
│
├── Search
│   ├── Query
│   ├── Results
│   └── Inspector
│
├── Embedding Space
│   ├── 2D Visualization
│   ├── Filters
│   └── Point Inspector
│
├── Documents
│   ├── Document List
│   └── Document Detail
│
├── Evaluation
│   ├── Metrics
│   ├── Queries
│   └── Experiments
│
└── Jobs
    ├── Running
    └── History
```

---

# 31. Dashboard

Dashboard는 시스템 상태를 한눈에 보여준다.

### System

```text
Documents       12,438
Chunks          48,291
Embeddings      48,291
Indexed         100%
```

### Retrieval

```text
Recall@10       89.2%
MRR             78.4%
nDCG@10         84.1%
```

### Infrastructure

```text
API P95         174ms
Search P95      142ms
Worker Queue    3
```

---

# 32. Search 화면

상단:

```text
┌─────────────────────────────────────────────┐
│ Search query                                │
│ engine vibration after landing        [RUN] │
└─────────────────────────────────────────────┘
```

검색 옵션:

```text
BM25 Weight       ─────●────
Vector Weight     ─────────●
Top K             10
Re-ranker         ON
```

하단:

```text
┌───────────────┬─────────────────────────────┐
│ Search Result │ Result Inspector             │
│               │                              │
│ #1            │ Score Breakdown              │
│ #2            │ BM25                         │
│ #3            │ Vector                       │
│ #4            │ Hybrid                       │
│               │                              │
└───────────────┴─────────────────────────────┘
```

---

# 33. Embedding Space 화면

```text
┌──────────────────────────────────────────────┐
│ Embedding Space                              │
│                                              │
│             · · ·                             │
│       · · ·       · ·                         │
│             ★ QUERY                           │
│          ● ● ●                                │
│                                              │
│  Filters                                     │
│  Source       [ALL]                          │
│  ATA Chapter  [ALL]                          │
│  Cluster      [ALL]                          │
└──────────────────────────────────────────────┘
```

Interaction:

* Zoom
* Pan
* Hover
* Click
* Box Select
* Cluster Filter
* Search Result Highlight

---

# 34. Query-to-Document Visualization

검색을 실행하면 관련 결과를 embedding map에 강조 표시한다.

```text
Query
  ★
  │
  ├── Result #1
  ├── Result #2
  ├── Result #3
  └── Result #4
```

검색 결과의 rank에 따라 시각적 강조 정도를 달리한다.

---

# 35. Document Detail

문서 상세 화면:

```text
Document
─────────────────────────

Source
FAA SDR

Aircraft
B737

ATA
72 - Engine

Component
Engine Fan

─────────────────────────

Original Text

...

─────────────────────────

Chunks

Chunk #01
Chunk #02
Chunk #03
```

각 chunk에서 embedding과 검색 결과를 확인할 수 있다.

---

# 36. Search Trace

검색 요청 하나를 기준으로 전체 pipeline을 추적한다.

```text
Query
  ↓ 4ms
Normalization
  ↓ 12ms
BM25
  ↓ 31ms
Vector Search
  ↓ 18ms
Merge
  ↓ 7ms
Re-ranking
  ↓ 12ms
Response
```

이를 통해 latency bottleneck을 파악할 수 있다.

---

# 37. 비동기 Worker Architecture

API Server가 무거운 작업을 직접 수행하지 않는다.

```text
                    API
                     │
                     ▼
                  Job Queue
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   Embedding       UMAP       Evaluation
    Worker         Worker        Worker
```

각 worker는 독립적으로 scaling할 수 있다.

---

# 38. Worker 작업

## Embedding Worker

```text
Input:
Chunk ID

Process:
Load chunk
→ Tokenize
→ Embedding
→ Store vector
→ Update status
```

---

## UMAP Worker

```text
Load embeddings
→ Fit UMAP
→ Generate coordinates
→ Store projection
```

---

## Evaluation Worker

```text
Load evaluation queries
→ Run Search
→ Compare Ground Truth
→ Calculate metrics
→ Store result
```

---

# 39. 장애 처리

Worker 실패 시 다음 상태를 기록한다.

```text
FAILED
```

그리고:

```text
retry_count
last_error
failed_at
```

을 저장한다.

재시도 정책:

```text
Retry 1 → 10 sec
Retry 2 → 30 sec
Retry 3 → 2 min
```

3회 실패 이후 Dead Letter Queue로 이동한다.

---

# 40. 캐싱 전략

검색 결과는 동일 query에 대해 일정 시간 캐싱할 수 있다.

Cache Key:

```text
hash(
 query
 +
 bm25_weight
 +
 vector_weight
 +
 top_k
 +
 embedding_model
)
```

단, 검색 알고리즘 설정이나 embedding model이 변경되면 cache를 무효화한다.

---

# 41. Observability

시스템 자체도 관측 가능해야 한다.

수집 대상:

```text
API latency
Search latency
DB latency
Vector search latency
Embedding worker latency
Queue length
Job failure
Error rate
```

---

# 42. Logging

검색 요청마다 최소한 다음을 기록한다.

```json
{
  "request_id": "req_123",
  "query": "engine vibration",
  "top_k": 10,
  "bm25_weight": 0.4,
  "vector_weight": 0.6,
  "latency_ms": 82
}
```

이를 통해 검색 결과를 재현할 수 있도록 한다.

---

# 43. Security

공개 데이터 기반 프로젝트이므로 인증 구조는 MVP에서 최소화할 수 있다.

다만 API 수준에서는 다음을 고려한다.

* CORS
* Rate Limit
* Input validation
* SQL injection 방어
* Query length 제한
* API error masking

---

# 44. 데이터 품질 검증

외부 데이터 ingestion 시 다음을 검증한다.

```text
Required Field
Duplicate
Encoding
Date Format
Empty Content
Invalid Aircraft Code
Invalid ATA Code
```

검증 실패 데이터는 별도의 quarantine 영역에 저장한다.

---

# 45. 검색 실패 분석

AeroMRO-Inspector의 차별화 기능으로 검색 실패를 별도로 분석한다.

예:

```text
Query:
"engine shaking after landing"

Expected:
SDR-123

Actual:
SDR-912
SDR-821
SDR-733
```

시스템은 다음을 보여준다.

```text
Why was SDR-123 missed?

BM25
  → lexical mismatch

Vector
  → similarity 0.61

Candidate Rank
  → #43

Final Rank
  → #37
```

이를 통해 검색 품질 문제의 원인을 추적할 수 있다.

---

# 46. Vocabulary Mismatch 분석

질의와 문서에서 사용되는 단어를 비교한다.

예:

```text
Query
"engine shaking"

Document
"abnormal vibration of powerplant"
```

Semantic Similarity:

```text
0.91
```

Lexical Match:

```text
0.12
```

이를 시각적으로 보여주면 hybrid retrieval의 필요성을 직접 증명할 수 있다.

---

# 47. Search Ablation

검색 알고리즘을 단계별로 끄고 성능을 비교할 수 있도록 한다.

```text
Experiment A
BM25 only

Experiment B
Vector only

Experiment C
BM25 + Vector

Experiment D
BM25 + Vector + Re-ranker
```

각 결과를 동일한 evaluation dataset에서 비교한다.

---

# 48. MVP 범위

초기 버전에서는 다음 기능만 구현한다.

### Data

* NTSB 데이터 ingestion
* FAA SDR 데이터 ingestion
* Normalization
* Chunking

### Search

* BM25
* Vector Search
* Hybrid Search
* Top-K

### Embedding

* Embedding generation
* Vector storage
* UMAP projection

### UI

* Search
* Search Inspector
* Embedding Space
* Document Detail

### Evaluation

* Recall@K
* MRR
* nDCG

---

# 49. Phase 2

다음 기능을 추가한다.

* Re-ranking
* Experiment management
* Query labeling
* Search failure analysis
* Search trace
* Job monitoring
* Advanced filters

---

# 50. Phase 3

고도화 기능:

* Multiple embedding model comparison
* Automated evaluation
* Query clustering
* Semantic anomaly detection
* Dataset versioning
* Retrieval regression testing
* Model/version benchmark

---

# 51. MVP 성공 기준

## Functional

```text
[ ] External data ingestion
[ ] Chunk generation
[ ] Embedding generation
[ ] Vector indexing
[ ] BM25 search
[ ] Hybrid search
[ ] Search inspector
[ ] UMAP projection
[ ] Deck.gl visualization
[ ] Evaluation metrics
```

---

## Performance

목표 기준:

```text
Search P95 < 500ms

Initial embedding:
Async processing

Embedding visualization:
60 FPS 수준의 기본 interaction 유지

API:
P95 < 300ms
```

실제 목표치는 데이터 규모와 인프라 사양에 따라 조정한다.

---

# 52. 프로젝트 디렉토리

```text
aeromro-inspector/
│
├── apps/
│   ├── web/
│   │   ├── src/
│   │   │   ├── features/
│   │   │   │   ├── search/
│   │   │   │   ├── embedding/
│   │   │   │   ├── documents/
│   │   │   │   └── evaluation/
│   │   │   ├── components/
│   │   │   └── api/
│   │   └── package.json
│   │
│   └── api/
│       ├── app/
│       │   ├── api/
│       │   ├── domain/
│       │   ├── services/
│       │   ├── repositories/
│       │   └── main.py
│       └── requirements.txt
│
├── workers/
│   ├── embedding/
│   ├── projection/
│   └── evaluation/
│
├── infrastructure/
│   ├── docker/
│   ├── postgres/
│   └── nginx/
│
├── scripts/
│   ├── ingestion/
│   ├── evaluation/
│   └── benchmark/
│
└── docs/
    ├── architecture.md
    ├── search.md
    └── evaluation.md
```

---

# 53. 개발 순서

## Step 1 — Data Layer

```text
NTSB / FAA
    ↓
Normalize
    ↓
PostgreSQL
```

---

## Step 2 — Chunking

```text
Document
 ↓
Section
 ↓
Chunk
 ↓
Metadata
```

---

## Step 3 — Embedding

```text
Chunk
 ↓
Embedding Worker
 ↓
Vector DB
```

---

## Step 4 — Retrieval

```text
Query
 ↓
BM25
 +
Vector
 ↓
Hybrid
```

---

## Step 5 — Evaluation

```text
Ground Truth
 ↓
Evaluation Runner
 ↓
Recall / MRR / nDCG
```

---

## Step 6 — Visualization

```text
Embeddings
 ↓
UMAP
 ↓
Deck.gl
```

---

## Step 7 — Inspector

검색 결과와 scoring breakdown을 연결한다.

---

# 54. 핵심 기술적 의사결정

## Decision 1

### 검색과 생성의 분리

본 프로젝트는 초기 MVP에서 LLM 생성 응답을 핵심 기능으로 두지 않는다.

이유:

제품의 핵심 문제는 "답변 생성"보다 **retrieval quality 검증**에 있기 때문이다.

---

## Decision 2

### API와 Worker 분리

Embedding과 UMAP은 CPU/GPU 리소스를 많이 사용할 수 있으므로 API 서버와 분리한다.

결과적으로:

```text
API
= Fast / Stateless

Worker
= Heavy / Async
```

구조를 유지한다.

---

## Decision 3

### Vector Search만 사용하지 않음

항공 정비 도메인에서는 특정 부품명, ATA code, part number 등의 정확한 lexical matching도 중요하다.

따라서:

```text
Semantic Search
+
Lexical Search
```

구조를 사용한다.

---

## Decision 4

### Embedding을 설명 가능한 데이터로 취급

Embedding을 단순 검색 내부 구현으로 숨기지 않고 다음과 연결한다.

```text
Query
    ↓
Embedding
    ↓
Nearest Chunks
    ↓
Ranking
    ↓
Evaluation
```

이를 통해 검색 시스템을 하나의 관측 가능한 데이터 파이프라인으로 만든다.

---

# 55. 최종 사용자 시나리오

## Scenario 1 — 정비 질의 검색

사용자가 입력한다.

> "right engine vibration after landing"

시스템은:

```text
Query
 ↓
BM25
 ↓
Vector Search
 ↓
Hybrid
 ↓
Top 10
```

결과를 반환한다.

---

## Scenario 2 — 검색 결과 검증

사용자가 #1 결과를 선택한다.

화면:

```text
Hybrid Score      0.91
BM25              0.78
Vector            0.95

Rank              #1

Source            FAA SDR
ATA               72
Component         Engine
```

그리고 원문 chunk를 확인한다.

---

## Scenario 3 — 임베딩 공간 확인

사용자가 "Embedding Space"를 연다.

Query point와 검색 결과가 강조된다.

```text
             ● Result #4

        ● Result #2

               ★ Query

           ● Result #1
```

사용자는 검색 결과가 embedding 공간에서 어떻게 분포하는지 확인할 수 있다.

---

## Scenario 4 — 검색 알고리즘 비교

사용자가 BM25 Weight를 변경한다.

```text
Before

#1 A
#2 B
#3 C


After

#1 C
#2 A
#3 D
```

검색 결과 ranking 변화가 즉시 반영된다.

---

## Scenario 5 — Retrieval Regression

새로운 embedding model을 적용한다.

```text
Model A

Recall@10   0.84
MRR         0.71


Model B

Recall@10   0.89
MRR         0.78
```

동일한 evaluation dataset을 사용해 모델 변경 전후를 비교한다.

---

# 56. 프로젝트의 핵심 가치

AeroMRO-Inspector는 단순한 RAG 애플리케이션이 아니다.

전체 구조는 다음과 같다.

```text
                DOMAIN DATA
                     │
                     ▼
              DATA PIPELINE
                     │
                     ▼
              EMBEDDING SPACE
                     │
                     ▼
             ┌───────────────┐
             │    SEARCH     │
             │               │
             │ BM25          │
             │ Vector        │
             │ Hybrid        │
             │ Re-ranking    │
             └───────┬───────┘
                     │
                     ▼
                INSPECTION
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
      VISUALIZATION          METRICS
          │                     │
          ▼                     ▼
      Embedding Map       Recall / MRR
                          nDCG / Latency
```

따라서 프로젝트의 핵심은

**Data → Retrieval → Ranking → Visualization → Evaluation**

으로 이어지는 하나의 관측 가능한 검색 시스템을 구축하는 것이다.

---

# 57. 포트폴리오에서 강조할 핵심 엔지니어링 포인트

이 프로젝트에서 가장 중요한 것은 기능 개수보다 **검색 시스템을 어떻게 설계하고 관측 가능하게 만들었는가**이다.

핵심 포인트는 다음과 같이 정리한다.

### 01. Hybrid Retrieval Architecture

BM25와 Vector Search를 결합하여 lexical mismatch와 semantic similarity를 동시에 처리한다.

### 02. Async Heavy Computing

Embedding, UMAP, Evaluation을 worker로 분리하여 API latency와 무거운 연산을 격리한다.

### 03. Search Observability

검색 결과뿐만 아니라 BM25 / Vector / Hybrid score를 모두 노출하여 ranking 과정을 추적한다.

### 04. Embedding Space Visualization

수천 개의 고차원 embedding을 UMAP으로 축소하고 WebGL 기반으로 시각화한다.

### 05. Retrieval Evaluation

Recall@K, MRR, nDCG 기반으로 검색 품질을 정량적으로 검증한다.

### 06. Experiment-driven Search

검색 가중치, embedding model, chunking 등의 변경을 실험 단위로 관리하여 retrieval regression을 검증한다.

---

# 58. 한 줄 프로젝트 정의

> **AeroMRO-Inspector는 항공 사고·결함 데이터를 기반으로 Hybrid Retrieval의 검색 결과와 Embedding Space를 시각적으로 검증하고, Retrieval Quality를 정량적으로 평가할 수 있는 RAG Search Observability Platform이다.**

---

# 59. 포트폴리오용 핵심 메시지

### 문제

현장 질의와 기술 문서 사이의 vocabulary mismatch 때문에 keyword search만으로는 관련 정비 사례를 안정적으로 찾기 어렵다.

### 해결

BM25와 Vector Search를 결합한 Hybrid Retrieval과 Re-ranking pipeline을 구축했다.

### 차별점

검색 결과만 제공하지 않고 **검색 점수, ranking 과정, embedding 공간, evaluation metrics를 하나의 화면에서 관찰**할 수 있도록 설계했다.

### 엔지니어링

Embedding과 UMAP 등 무거운 연산은 비동기 worker로 분리하고, WebGL 기반 visualization으로 대규모 embedding point를 브라우저에서 탐색할 수 있도록 구성했다.

### 결과

검색 시스템을 단순한 RAG 응답기가 아니라 **측정·비교·검증할 수 있는 검색 품질 관제 시스템**으로 확장했다.

