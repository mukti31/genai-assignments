# AI QA Knowledge Assistant — Ingestion Architecture & Implementation Playbook

## 0. Purpose and scope

This document is the implementation baseline produced from the user's stated requirements and the completed 20-question discovery.

**Scope: Ingestion only.**

The first version accepts uploaded:
- PDF
- DOCX
- XLSX
- TXT

The architecture is intentionally extensible for later source adapters such as Jira, ADO, TestRail, Zephyr/Xray, Figma, Swagger, GitHub/Bitbucket, meeting recordings, Confluence/wiki, defect systems, and release-note systems.

### Mandatory correctness rules

1. Do not invent business requirements, test cases, defects, relationships, or source values.
2. Distinguish source-derived facts from AI suggestions/interpretations.
3. If evidence is insufficient, store/return **Insufficient evidence**.
4. Every important extracted conclusion must retain source provenance.
5. No environment-specific value is hard-coded. Database names, connection strings, API keys, model names, bucket names, queue names, limits, and paths come from environment configuration.
6. `INGESTION_COMPLETE` is allowed only after the verification gate succeeds.

---

# 1. Discovery decisions

| # | Decision | Selected answer |
|---|---|---|
| 1 | Initial inputs | Uploaded PDF/DOCX/XLSX/TXT |
| 2 | MongoDB storage | Extracted content + embeddings + metadata; structured entities when reliable |
| 3 | Original file storage | Local filesystem in development; AWS S3 in deployment |
| 4 | Trigger | Asynchronous ingestion |
| 5 | Queue | Redis + BullMQ |
| 6 | Vector storage | VectorStore abstraction; MongoDB Vector Search first |
| 7 | Chunking | Structure-aware hybrid chunking |
| 8 | Embeddings | Mistral primary + OpenAI fallback |
| 9 | Structured parsing | Rule-based first + LLM fallback |
| 10 | LLM | Groq only |
| 11 | Entities | Requirement, Acceptance Criteria, Test Case, Defect |
| 12 | Provenance | Full provenance chain |
| 13 | Validation | Layered validation + extensible security-scan boundary |
| 14 | Duplicate/versioning | Content hash + document versions |
| 15 | Failure handling | Retry + controlled partial processing |
| 16 | Verification | Full verification + retrieval smoke test |
| 17 | Relationships | Evidence-backed relationships only |
| 18 | Isolation | Project/application based |
| 19 | UI | Upload + status + verification details |
| 20 | API architecture | REST controllers + service layer |

These are project decisions, not assumptions.

---

# 2. Target journey

```text
FILE UPLOAD
    |
    v
PROJECT VALIDATION
    |
    v
FILE + CONTENT VALIDATION
    |
    v
HASH / DUPLICATE / VERSION CHECK
    |
    v
STORE ORIGINAL FILE
    |
    v
CREATE INGESTION JOB
    |
    v
REDIS + BULLMQ
    |
    v
INGESTION WORKER
    |
    +--> Extract PDF/DOCX/XLSX/TXT
    |
    +--> Clean / normalize
    |
    +--> Rule-based parsing
    |       |
    |       +--> Requirement
    |       +--> Acceptance Criteria
    |       +--> Test Case
    |       +--> Defect
    |
    +--> Groq fallback when deterministic parsing is insufficient
    |
    +--> Validate evidence + provenance
    |
    +--> Structure-aware chunking
    |
    +--> Mistral embeddings
    |       |
    |       +--> failure -> OpenAI fallback
    |
    +--> MongoDB persistence
    |
    +--> Verification
            |
            +--> storage
            +--> extraction
            +--> entities
            +--> chunks
            +--> embeddings
            +--> provenance
            +--> retrieval smoke test
            |
            v
       INGESTION_COMPLETE
```

BullMQ provides worker execution and retry mechanisms; its documentation states that workers process queued jobs and failed jobs can be retried. [BullMQ Workers](https://docs.bullmq.io/guide/workers) and [BullMQ retrying jobs](https://docs.bullmq.io/guide/jobs/retrying-job).

Mistral's embeddings API accepts text input and a model and returns numerical embedding vectors; its API also supports an optional output dimension where available. [Mistral Embeddings API](https://docs.mistral.ai/api/endpoint/embeddings).

MongoDB Vector Search supports semantic search over stored embeddings and supports filtering on indexed fields, which is relevant to project-level isolation. [MongoDB Vector Search](https://www.mongodb.com/docs/vector-search/).

---

# 3. Logical architecture

```text
React UI
   |
   v
Express REST API
   |
   +--> Project Service
   +--> Upload Service
   +--> Ingestion Job Service
   |
   v
BullMQ / Redis
   |
   v
Ingestion Worker
   |
   +--> FileStorage abstraction
   |      +--> Local
   |      +--> S3
   |
   +--> Validation
   +--> Extractors
   |      +--> PDF
   |      +--> DOCX
   |      +--> XLSX
   |      +--> TXT
   |
   +--> Cleaning
   +--> Rule parsers
   +--> Groq parser
   +--> Provenance
   +--> Chunking
   +--> EmbeddingService
   |      +--> Mistral
   |      +--> OpenAI fallback
   |
   +--> Repository layer
   |      +--> MongoDB
   |      +--> VectorStore abstraction
   |
   +--> Verification
   |
   v
MongoDB
   +--> documents
   +--> versions
   +--> jobs
   +--> chunks
   +--> requirements
   +--> acceptance_criteria
   +--> test_cases
   +--> defects
   +--> relationships
   +--> provenance
   +--> verification_results

Original file:
Local filesystem in development
AWS S3 in deployment
```

AWS's current JavaScript SDK documentation describes S3 access through the SDK and recommends configuring authentication rather than embedding credentials in application code. [AWS SDK for JavaScript v3](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/getting-started-nodejs.html).

---

# 4. Repository structure

```text
ai-qa-knowledge-assistant/
|
+-- apps/
|   +-- api/
|   |   +-- src/
|   |       +-- server.ts
|   |       +-- app.ts
|   |       +-- config/
|   |       +-- routes/
|   |       +-- controllers/
|   |       +-- middleware/
|   |       +-- workers/
|   |
|   +-- web/
|       +-- src/
|           +-- app/
|           +-- features/
|           +-- components/
|           +-- services/
|
+-- modules/
|   +-- ingestion/
|   |   +-- domain/
|   |   +-- application/
|   |   +-- infrastructure/
|   |   +-- extractors/
|   |   +-- parsers/
|   |   +-- chunking/
|   |   +-- validation/
|   |   +-- verification/
|   |
|   +-- retrieval/
|   |   +-- domain/
|   |   +-- application/
|   |   +-- infrastructure/
|   |
|   +-- knowledge/
|       +-- domain/
|       +-- repositories/
|
+-- packages/
|   +-- config/
|   +-- logging/
|   +-- shared/
|   +-- schemas/
|
+-- tests/
|   +-- unit/
|   +-- integration/
|   +-- contract/
|   +-- fixtures/
|   +-- e2e/
|
+-- docs/
|   +-- architecture/
|   +-- runbooks/
|
+-- .env
+-- example.env
+-- .gitignore
+-- package.json
+-- tsconfig.json
```

The `modules/ingestion` module must not depend on UI code. Retrieval is a sibling module and can be implemented later without moving ingestion.

---

# 5. Environment configuration

Create `example.env`:

```env
NODE_ENV=
PORT=

MONGODB_URI=
MONGODB_DATABASE=
MONGODB_VECTOR_INDEX_NAME=

REDIS_URL=
BULLMQ_QUEUE_NAME=
BULLMQ_WORKER_CONCURRENCY=
BULLMQ_MAX_ATTEMPTS=
BULLMQ_BACKOFF_DELAY_MS=

FILE_STORAGE_PROVIDER=
LOCAL_STORAGE_ROOT=
AWS_REGION=
AWS_S3_BUCKET=
AWS_S3_PREFIX=

MAX_UPLOAD_BYTES=
ALLOWED_EXTENSIONS=
ALLOWED_MIME_TYPES=

MISTRAL_API_KEY=
MISTRAL_BASE_URL=
MISTRAL_EMBEDDING_MODEL=
MISTRAL_OUTPUT_DIMENSION=

OPENAI_API_KEY=
OPENAI_BASE_URL=
OPENAI_EMBEDDING_MODEL=

GROQ_API_KEY=
GROQ_BASE_URL=
GROQ_MODEL=

LLM_PARSER_ENABLED=
LLM_PARSER_MAX_INPUT_TOKENS=

EMBEDDING_TIMEOUT_MS=
LLM_TIMEOUT_MS=
EXTERNAL_API_MAX_RETRIES=

LOG_LEVEL=
LOG_FORMAT=

INGESTION_VERSION=
PARSER_VERSION=
CHUNKER_VERSION=
```

Do not copy production secrets into `example.env`.

Use a validated configuration object so application services receive configuration rather than reading `process.env` throughout the codebase.

---

# 6. Core status model

```text
UPLOADED
VALIDATING
VALIDATED
QUEUED
PROCESSING
EXTRACTING
PARSING
CHUNKING
EMBEDDING
PERSISTING
VERIFYING
INGESTION_COMPLETE
PARTIALLY_PROCESSED
FAILED
```

Centralize state transitions. Services must not invent arbitrary states.

---

# 7. Data model

## Project

```text
Project
- _id
- projectKey
- name
- status
- createdAt
- updatedAt
```

## Document

```text
Document
- _id
- projectId
- originalFileName
- fileType
- mimeType
- contentHash
- storageProvider
- storageKey
- currentVersionId
- status
- createdAt
- updatedAt
```

## DocumentVersion

```text
DocumentVersion
- _id
- documentId
- versionNumber
- contentHash
- sourceLocation
- parserVersion
- chunkerVersion
- ingestionVersion
- status
- createdAt
```

## IngestionJob

```text
IngestionJob
- _id
- projectId
- documentId
- documentVersionId
- bullJobId
- status
- currentStage
- progress
- attempt
- startedAt
- completedAt
- errorCode
- errorMessage
```

## Chunk

```text
Chunk
- _id
- projectId
- documentId
- documentVersionId
- chunkIndex
- text
- section
- page
- sheet
- location
- entityType
- entityId
- embedding
- embeddingProvider
- embeddingModel
- embeddingDimension
- provenanceId
```

## Requirement

```text
Requirement
- _id
- projectId
- documentVersionId
- sourceChunkIds[]
- requirementId
- title
- description
- acceptanceCriteriaIds[]
- sourceEvidence[]
- extractionMethod
```

`requirementId` is populated only when supported by source evidence.

## Acceptance Criteria

```text
AcceptanceCriteria
- _id
- projectId
- documentVersionId
- sourceChunkIds[]
- requirementId
- text
- sourceEvidence[]
- extractionMethod
```

## Test Case

```text
TestCase
- _id
- projectId
- documentVersionId
- sourceChunkIds[]
- testCaseId
- title
- preconditions
- steps
- expectedResult
- requirementIds[]
- sourceEvidence[]
- extractionMethod
```

## Defect

```text
Defect
- _id
- projectId
- documentVersionId
- sourceChunkIds[]
- defectId
- summary
- description
- rootCause
- resolution
- requirementIds[]
- testCaseIds[]
- sourceEvidence[]
- extractionMethod
```

## Relationship

```text
Relationship
- _id
- projectId
- fromEntityType
- fromEntityId
- relationshipType
- toEntityType
- toEntityId
- evidenceChunkIds[]
- relationshipStatus
- extractionMethod
```

Relationship status should distinguish:

```text
DOCUMENTED
INSUFFICIENT_EVIDENCE
```

Semantic similarity alone must not be persisted as a documented business relationship.

---

# 8. Provenance

```text
Provenance
- _id
- projectId
- documentId
- documentVersionId
- sourceFileName
- page
- section
- paragraph
- table
- sheet
- cellRange
- sourceText
- sourceChunkId
- extractionMethod
- parserVersion
- extractedAt
```

Every important extracted field should point to evidence.

If a reliable source location/value cannot be established:

```text
evidenceStatus = INSUFFICIENT_EVIDENCE
```

---

# 9. Phase-by-phase implementation

## Phase 0 — Bootstrap

### Build

Create the repository structure, TypeScript configuration, configuration loader, logger, shared schemas, and test framework.

### Files

```text
package.json
tsconfig.json
.env
example.env
.gitignore

apps/api/src/server.ts
apps/api/src/app.ts

packages/config/
packages/logging/
packages/shared/
packages/schemas/

modules/ingestion/
modules/retrieval/
modules/knowledge/

tests/
```

### API

`GET /health`

### Run

```text
npm install
npm run build
npm run test
npm run dev
```

### Gate

Proceed only when:

- build succeeds
- tests run
- API starts
- `/health` responds
- missing required environment configuration fails clearly

### Failure scenarios

- missing `.env`
- invalid configuration
- unavailable port
- invalid type in environment variable

---

## Phase 1 — MongoDB + Redis

### Build

Create MongoDB and Redis connection modules.

### Files

```text
modules/knowledge/infrastructure/mongodb/
apps/api/src/infrastructure/redis/
```

### API

Extend `/health` to report:

```text
api
mongodb
redis
```

### Gate

- MongoDB connects
- Redis connects
- unavailable dependency produces controlled health failure
- no secrets are returned

BullMQ requires Redis as its backend and its worker model supports asynchronous job processing. [BullMQ Quick Start](https://docs.bullmq.io/quick-start).

---

## Phase 2 — Project boundary

### Build

Project creation and lookup.

### APIs

```text
POST /api/projects
GET  /api/projects/:projectId
```

### Gate

Create one project and retrieve it.

Reject ingestion requests without a valid project.

---

## Phase 3 — File storage abstraction

### Build

```text
FileStorage
  +-- LocalFileStorage
  +-- S3FileStorage
```

Methods:

```text
put()
get()
delete()
exists()
getMetadata()
```

Provider is selected through `.env`.

### Gate

Upload a fixture and confirm the bytes are retrievable.

---

## Phase 4 — Upload + validation

### API

```text
POST /api/projects/:projectId/documents
```

### Pipeline

```text
project validation
-> extension
-> size
-> MIME/content
-> hash
-> duplicate/version
-> storage
-> document record
-> queue job
```

### Tests

- valid PDF
- valid DOCX
- valid XLSX
- valid TXT
- unsupported file
- oversized file
- invalid MIME/content
- corrupt file
- exact duplicate
- changed version

### Gate

No invalid file reaches the extraction worker.

---

## Phase 5 — BullMQ job orchestration

### Build

```text
IngestionQueue
IngestionWorker
```

Job payload:

```text
projectId
documentId
documentVersionId
```

Do not place full file contents into the queue payload.

Configure retry count, backoff, and concurrency from `.env`.

### Gate

Upload creates a job, worker consumes it, status is persisted, and transient failure retry works.

BullMQ documents automatic retries through job `attempts` and backoff strategies. [Retrying failing jobs](https://docs.bullmq.io/guide/retrying-failing-jobs).

---

## Phase 6 — Extraction

### Build

```text
DocumentExtractor
  +-- PdfExtractor
  +-- DocxExtractor
  +-- XlsxExtractor
  +-- TxtExtractor
```

Common output:

```text
ExtractedDocument
- rawText
- pages[]
- sections[]
- paragraphs[]
- tables[]
- sheets[]
```

Do not invent page/section/cell locations that the source extractor cannot provide.

### Gate

Each supported format produces a valid intermediate representation or a controlled extraction error.

---

## Phase 7 — Cleaning

### Build

Deterministic normalization:

- line ending normalization
- safe whitespace normalization
- extraction artifact removal
- preserve identifiers
- preserve labels
- preserve business punctuation
- preserve table meaning

Do not rewrite business meaning.

### Gate

Raw extraction and cleaned extraction can be compared for representative fixtures.

---

## Phase 8 — Rule-based structured parsing

### Build

```text
RequirementParser
AcceptanceCriteriaParser
TestCaseParser
DefectParser
```

Priority:

```text
explicit labels
-> headings
-> table structure
-> IDs
-> deterministic patterns
```

Example:

```text
TC-231 | Address validation | Expected: AddressLine2 is optional
```

may produce fields directly supported by that source.

Do not generate a root cause or expected result that does not exist in the source.

### Gate

Known fixture values match expected parser output.

---

## Phase 9 — Groq fallback parser

### Build

Use Groq only when deterministic parsing cannot reliably structure the source.

### Contract

Require schema-valid structured output containing evidence.

Conceptually:

```json
{
  "field": "expectedResult",
  "value": "...",
  "evidence": "...",
  "evidenceStatus": "SUPPORTED"
}
```

Unsupported output becomes:

```text
INSUFFICIENT_EVIDENCE
```

### Gate

- malformed Groq response is rejected
- unsupported field is not persisted as fact
- source evidence is retained
- parser method is recorded

---

## Phase 10 — Structure-aware chunking

### Build

Prefer:

```text
section
-> paragraph
-> table/record
-> configured size limit
-> configured overlap
```

Keep Requirement/Test Case/Defect records together where possible.

All chunk limits and overlap values come from environment configuration.

### Gate

Inspect representative chunks manually and confirm:

- no unexplained content loss
- meaningful boundaries
- source location retained
- entity association retained

---

## Phase 11 — Embedding service

### Build

```text
EmbeddingService
  +-- MistralEmbeddingProvider
  +-- OpenAIEmbeddingProvider
```

Selected behavior:

```text
Mistral
  |
  +-- success -> use vector
  |
  +-- failure -> OpenAI fallback
```

Persist:

```text
provider
model
dimension
createdAt
sourceChunkId
```

Mistral's official API documents the embeddings endpoint, model selection, text input, and output-dimension option where supported. [Mistral Embeddings API](https://docs.mistral.ai/api/endpoint/embeddings).

### Critical verification

The MongoDB Vector Search index dimension must match the vector dimension used by the stored embeddings. MongoDB explicitly documents that the embedding model determines vector dimensions and that the dimension must be specified in the vector index. [MongoDB Vector Search](https://www.mongodb.com/docs/vector-search/).

---

## Phase 12 — MongoDB persistence

### Repositories

```text
ProjectRepository
DocumentRepository
DocumentVersionRepository
IngestionJobRepository
ChunkRepository
RequirementRepository
AcceptanceCriteriaRepository
TestCaseRepository
DefectRepository
RelationshipRepository
ProvenanceRepository
VerificationRepository
```

### Persistence sequence

```text
document/version
-> extraction/provenance
-> entities
-> relationships
-> chunks
-> embeddings
-> verification
```

Use transactions only where the actual consistency requirement justifies them; do not assume separate MongoDB writes are automatically atomic.

### Gate

Read back every expected record from MongoDB.

---

## Phase 13 — MongoDB Vector Search

### Build

Create a vector index for the chunk embedding field.

Include project filtering fields in the vector-search design.

MongoDB documents that Vector Search supports semantic retrieval and pre-filtering on indexed fields, making project-scoped retrieval feasible. [MongoDB Vector Search](https://www.mongodb.com/docs/vector-search/).

### Gate

Run a known semantic smoke query against an ingested fixture and verify that the expected source chunk is retrievable.

---

## Phase 14 — Evidence-backed relationships

Supported relationship types initially:

```text
Requirement -> AcceptanceCriteria
Requirement -> TestCase
Defect -> Requirement
Defect -> TestCase
```

Only persist a documented relationship when the source provides evidence.

Do not convert semantic similarity into a factual relationship.

---

## Phase 15 — Verification engine

### Checks

1. Document exists.
2. Version exists.
3. Content hash matches.
4. Extraction completed.
5. Extracted content exists.
6. Entities satisfy schemas.
7. Entity evidence exists.
8. Chunks exist and preserve provenance.
9. Embeddings exist.
10. Provider/model/dimension are recorded.
11. MongoDB records can be read back.
12. Retrieval smoke test returns the newly ingested source.

### Completion rule

```text
All required checks pass
        |
        v
INGESTION_COMPLETE
```

Otherwise:

```text
PARTIALLY_PROCESSED
or
FAILED
```

---

## Phase 16 — Status APIs

```text
GET /api/projects/:projectId/documents
GET /api/projects/:projectId/documents/:documentId
GET /api/projects/:projectId/ingestion-jobs/:jobId
GET /api/projects/:projectId/documents/:documentId/verification
```

Expose:

- current status
- current stage
- progress
- version
- chunk count
- entity counts
- embedding status
- verification status
- errors
- timestamps

Never expose credentials.

---

## Phase 17 — React UI

Create:

```text
apps/web/src/features/ingestion/
  UploadPage
  UploadForm
  IngestionStatus
  VerificationSummary
  ErrorSummary
```

Journey:

```text
Project
-> File
-> Upload
-> Job
-> Processing
-> Extraction
-> Entities/chunks
-> Embeddings
-> Verification
-> INGESTION_COMPLETE
```

### Gate

A developer can visually prove the backend ingestion state without inspecting MongoDB directly.

---

## Phase 18 — Logging + errors

Every ingestion log should include, where available:

```text
requestId
jobId
projectId
documentId
documentVersionId
stage
errorCode
duration
```

Error categories:

```text
VALIDATION_ERROR
STORAGE_ERROR
EXTRACTION_ERROR
PARSING_ERROR
LLM_ERROR
EMBEDDING_ERROR
DATABASE_ERROR
QUEUE_ERROR
VERIFICATION_ERROR
```

Never log API keys, access tokens, connection strings, or unnecessary sensitive source content.

---

## Phase 19 — Testing

### Unit tests

- environment validation
- file validation
- hashing
- regex utilities
- cleaning
- parsers
- chunking
- provenance
- schemas
- embedding provider selection
- retry classification

### Integration tests

```text
upload -> storage
upload -> queue
worker -> extractor
worker -> parser
worker -> chunker
worker -> embedding
worker -> MongoDB
worker -> verification
```

### Fixtures

```text
tests/fixtures/
  requirement.pdf
  testcase.docx
  defects.xlsx
  notes.txt
```

Fixture information must be explicitly created test data, not fabricated production evidence.

### End-to-end acceptance path

```text
UPLOAD
-> VALIDATE
-> QUEUE
-> EXTRACT
-> PARSE
-> CHUNK
-> EMBED
-> STORE
-> VERIFY
-> RETRIEVE
-> INGESTION_COMPLETE
```

---

# 10. Final ingestion readiness checklist

## Configuration

- [ ] All environment-dependent values come from `.env`.
- [ ] `example.env` exists.
- [ ] No secret is hard-coded.
- [ ] Startup validates configuration.

## Upload

- [ ] PDF works.
- [ ] DOCX works.
- [ ] XLSX works.
- [ ] TXT works.
- [ ] Unsupported files are rejected.
- [ ] Size validation works.
- [ ] MIME/content validation works.
- [ ] Exact duplicates are detected.
- [ ] Changed documents create versions.

## Storage

- [ ] Local storage works in development.
- [ ] S3 adapter works for deployment.
- [ ] Original bytes can be recovered.
- [ ] Storage metadata is persisted.

## Queue

- [ ] BullMQ job is created.
- [ ] Worker consumes the job.
- [ ] Retry works for transient failures.
- [ ] Failed jobs are visible.
- [ ] Concurrency is configurable.

## Extraction

- [ ] PDF extraction works.
- [ ] DOCX extraction works.
- [ ] XLSX extraction works.
- [ ] TXT extraction works.
- [ ] Source locations are preserved where available.

## Parsing

- [ ] Requirement parser works.
- [ ] Acceptance Criteria parser works.
- [ ] Test Case parser works.
- [ ] Defect parser works.
- [ ] Rule parser runs before Groq.
- [ ] Groq response is schema validated.
- [ ] Unsupported values become `Insufficient evidence`.

## Chunking

- [ ] Structure-aware chunking works.
- [ ] Limits come from configuration.
- [ ] Provenance survives chunking.

## Embeddings

- [ ] Mistral works.
- [ ] OpenAI fallback works.
- [ ] Provider/model/dimension are recorded.
- [ ] MongoDB vector index dimension matches embeddings.

## MongoDB

- [ ] Documents persist.
- [ ] Versions persist.
- [ ] Entities persist.
- [ ] Evidence-backed relationships persist.
- [ ] Chunks persist.
- [ ] Embeddings persist.
- [ ] Provenance persists.
- [ ] Verification results persist.

## Verification

- [ ] Storage verification works.
- [ ] Extraction verification works.
- [ ] Entity verification works.
- [ ] Chunk verification works.
- [ ] Embedding verification works.
- [ ] Retrieval smoke test works.
- [ ] Only verified jobs become `INGESTION_COMPLETE`.

## UI

- [ ] Upload works.
- [ ] Status is visible.
- [ ] Current stage is visible.
- [ ] Counts are visible.
- [ ] Verification result is visible.
- [ ] Errors are visible.

---

# 11. Deliberately outside the first-version ingestion scope

These are future adapters, not assumptions for Version 1:

- Jira
- ADO
- TestRail
- Zephyr
- Xray
- Figma
- Swagger/API live ingestion
- GitHub
- Bitbucket
- meeting recordings
- Confluence
- wiki connectors
- external defect databases
- release-note systems

Developer repositories remain non-vector according to the stated requirement and are not part of the first uploaded-file ingestion implementation.

---

# 12. Implementation order

```text
Phase 0  Bootstrap
Phase 1  MongoDB + Redis
Phase 2  Project boundary
Phase 3  File storage
Phase 4  Upload + validation
Phase 5  BullMQ
Phase 6  Extraction
Phase 7  Cleaning
Phase 8  Rule parsing
Phase 9  Groq fallback
Phase 10 Chunking
Phase 11 Embeddings
Phase 12 Persistence
Phase 13 Vector index
Phase 14 Relationships
Phase 15 Verification
Phase 16 Status APIs
Phase 17 React UI
Phase 18 Logging/errors
Phase 19 Testing
Phase 20 Final readiness
```

**Gate rule:** Do not proceed until the current phase's verification gate passes.

---

# 13. Technology evidence

- Mistral Embeddings API: https://docs.mistral.ai/api/endpoint/embeddings
- MongoDB Vector Search: https://www.mongodb.com/docs/vector-search/
- MongoDB `$vectorSearch`: https://www.mongodb.com/docs/manual/reference/operator/aggregation/vectorsearch/
- BullMQ Workers: https://docs.bullmq.io/guide/workers
- BullMQ retries: https://docs.bullmq.io/guide/retrying-failing-jobs
- AWS SDK for JavaScript v3: https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/getting-started-nodejs.html

These sources establish technology capabilities only; they are not business requirements.

---

# 14. Final architecture rule

```text
             SOURCE EVIDENCE
                    |
                    v
                INGESTION
                    |
        +-----------+-----------+
        |           |           |
     EXTRACT      PARSE       CHUNK
        |           |           |
        +-----------+-----------+
                    |
                EMBEDDING
                    |
                    v
                 MONGODB
                    |
               VERIFICATION
                    |
            +-------+-------+
            |               |
         FAIL/PARTIAL   COMPLETE
            |               |
            v               v
       visible error  INGESTION_COMPLETE
```

**Core correctness principle:**

> The system may transform source information, but it must not silently transform an unsupported AI inference into an organizational fact.
