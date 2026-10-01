# Maik

**Maik — Market And Intelligence Knowledge**

Maik is an agent-oriented knowledge system designed to organize, process, and query large volumes of market information in an incremental, modular, and LLM-provider-independent way.

The core idea is simple:

> Maik does not need to own an AI model. It provides storage, state, tools, and workflow contracts; the agent connected through MCP performs the reasoning.

The project starts with a layered document base and can later evolve into event consolidation, market memory, and intelligence.

---

## Initial Objective

The first version of Maik should allow:

1. the user to freely organize a file base inside a `raw` layer;
2. Maik to detect which contents are still pending processing;
3. an external agent, connected through the **Worker MCP**, to read those materials;
4. the agent itself to generate a structured Markdown representation;
5. Maik to store that representation inside the `processed` layer while preserving the same hierarchy as `raw`;
6. users and agents to query one or multiple knowledge bases through indexes;
7. the whole process to work without requiring an LLM API embedded inside Maik.

---

# Principles

## 1. Raw is controlled by the user

The `raw` layer contains the original materials.

The user can:

- add files;
- create and modify folders;
- move files;
- rename files;
- define their own hierarchy;
- add contents manually or through automations/skills.

Example:

```text
raw/
├── Topic 1/
│   ├── Subtopic 1/
│   │   ├── Files 1/
│   │   │   └── report.pdf
│   │   └── Files 2/
│   │       └── strategy.pdf
│   └── Subtopic 2/
│       └── insight.pdf
└── Topic 2/
    └── Subtopic 2/
        └── report.pdf
```

The structure created by the user already carries semantic value and must be preserved.

---

## 2. Processed mirrors Raw

The `processed` layer contains normalized Markdown representations.

```text
processed/
├── Topic 1/
│   ├── Subtopic 1/
│   │   ├── Files 1/
│   │   │   └── report.md
│   │   └── Files 2/
│   │       └── strategy.md
│   └── Subtopic 2/
│       └── insight.md
└── Topic 2/
    └── Subtopic 2/
        └── report.md
```

The `processed` hierarchy is derived from `raw`.

The user may inspect this layer, but it should be treated conceptually as **read-only**.

---

## 3. The agent generates the Markdown

Maik does not need to intellectually convert a document into knowledge.

The workflow is:

```text
RAW
 ↓
Maik detects pending content
 ↓
Worker MCP
 ↓
External Agent
 ↓
Agent interprets the material
 ↓
Agent generates Markdown
 ↓
Worker MCP
 ↓
PROCESSED
```

The agent may run on different providers:

```text
                 MAIK

              Worker MCP
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     ChatGPT    Claude   Local LLM
```

Maik is not tied to a specific provider.

---

## 4. MCP provides capabilities; Skills define behavior

Maik is conceptually divided into:

| Component | Responsibility |
|---|---|
| **Maik Core** | filesystem, catalog, workspaces, IDs, hashes, states, and indexes |
| **Maik User MCP** | day-to-day interaction with the knowledge base |
| **Maik Worker MCP** | processing of pending jobs |
| **Maik Skills** | instructions and workflows used by agents |

Principle:

> **Core stores. MCP exposes. Skills guide. The agent reasons.**

---

# Initial Architecture

```text
                       LLM PROVIDER
                ChatGPT / Claude / other
                           │
                       MAIK SKILLS
                           │
                ┌──────────┴──────────┐
                │                     │
           USER MCP              WORKER MCP
                │                     │
                └──────────┬──────────┘
                           │
                       MAIK CORE
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
      RAW              PROCESSED           INDEX
                                             │
                                          SQLite
```

---

# Workspaces

Maik organizes knowledge into **workspaces**.

Each workspace represents an independent knowledge base.

Examples:

```text
Market
Investments
University Research
Technology
```

A workspace contains at least:

```text
Workspace
├── id
├── name
├── storage_mode
├── root_path
└── configuration
```

---

## Storage Modes

The Core must support two storage modes.

### Managed

Maik chooses and manages the directory itself.

Example:

```text
~/.maik/
└── workspaces/
    └── market/
        ├── raw/
        └── processed/
```

This is intended for users who do not want to manually configure storage.

### External

The user or the MCP provides a directory.

Example:

```text
PathToWorkspace/
├── raw/
└── processed/
```

This directory may later live on:

- a local disk;
- a synchronized folder;
- OneDrive;
- Google Drive;
- NAS;
- a network volume;
- another compatible backend.

The Core must not depend on fixed absolute paths.

---

# Document Identity

Maik separates three concepts:

```text
ID   → logical identity of the document
HASH → identity of the content
PATH → current location and context
```

## ID

Remains stable even if the file is moved or renamed.

## Hash

Allows Maik to detect whether the content itself has changed.

SHA-256 can be used initially.

## Path

Represents the current location inside the hierarchy defined by the user.


The path also acts as free semantic context.

---

# Synchronization Rules

## New File

```text
new file
   ↓
register in catalog
   ↓
status = pending
```

## Moved File

```text
path changed
hash unchanged
   ↓
keep ID
   ↓
update path
   ↓
move corresponding .md
   ↓
do not reprocess
```

## Renamed File

Same rule as a moved file.

## Modified Content

```text
same or different path
different hash
   ↓
content updated
   ↓
mark as pending
   ↓
reprocess
```

## Removed File

The Core must detect removal and update its catalog.

The policy for removing the corresponding `processed` representation can be defined during implementation.

---

# Internal Catalog

Maik uses SQLite as its operational catalog.

The database is not the source of truth for the actual content.

It may store information such as:

```text
documents
---------
id
workspace_id
raw_path
processed_path
content_hash
mime_type
status
created_at
updated_at
claimed_at
completed_at
worker
error
```

Initial states:

```text
pending
processing
completed
failed
```

The catalog must be rebuildable from existing files.

---

# User MCP

The **User MCP** is used during normal interaction with Maik.

Planned initial capabilities:

```text
list_workspaces
create_workspace
get_workspace

add_content
list_content
move_content
remove_content

search
read_processed
status
```

The user/agent writes only to the `raw` layer.

The `processed` layer is derived through Maik's workflow.

---

# Worker MCP

The **Worker MCP** is used by agents responsible for processing.

Planned initial capabilities:

```text
get_pending_documents
claim_document
read_raw_document
save_processed_document
complete_document
fail_document
release_document
```

Workflow:

```text
get_pending
    ↓
claim
    ↓
read_raw_document
    ↓
agent processes
    ↓
save_processed_document
    ↓
complete
```

The agent does not write directly to the filesystem.

The Core calculates the correct destination in `processed`.

---

# Skills

The project may distribute a set of skills that teach different agents how to operate Maik.

Structure:

```text
skills/
├── maik-add-content/
│   └── SKILL.md
├── maik-process-document/
│   └── SKILL.md
├── maik-query/
│   └── SKILL.md
├── maik-research-and-store/
│   └── SKILL.md
└── maik-organize/
    └── SKILL.md
```

## maik-process-document

Responsible for guiding the worker through the transformation:

```text
raw document
     ↓
structured Markdown
```

Expected rules:

- preserve the original content;
- do not add external facts;
- avoid unsupported inference;
- preserve numbers, dates, and tables;
- preserve references;
- structure the document for both human and machine reading;
- keep a link to the original material.

---

# Processed Markdown Format

Suggested initial format:

```markdown
---
maik_id: 01...
workspace_id: ...
raw_path: Topic/report.pdf
content_hash: ...
processed_at: 2026-09-30T21:00:00
---

# Document Title

## Content

Structured content from the source document.

## Tables

Relevant tables, when available.

## References

References present in the original material.
```

The schema may evolve and therefore should be versioned.

---

# Indexes

Maik should have two levels of indexes.

## Local Index

Each workspace maintains its own index.

It may contain:

```text
document_id
path
filename
title
metadata
status
```

## Global Index

The Core maintains a lightweight view of all workspace indexes.

Example:

```text
Global Index
├── Market → local index
├── Investments → local index
└── Technology → local index
```

The global index does not need to duplicate document contents.

---

# Federated Search

Maik should support searching:

- one workspace;
- a set of workspaces;
- all accessible workspaces.

Conceptual example:

```text
search(
    query="Relevant query...",
    workspace_ids=["technology", "investments"]
)
```

or:

```text
search(
    query="Relevant query...",
    workspace_ids=None
)
```

where `None` means all authorized workspaces.

Flow:

```text
Query
  ↓
Global / Local Index
  ↓
Candidate Documents
  ↓
Workspace
  ↓
Processed Markdown
```

The contents remain physically separated.

---

# Progressive Search

Maik should always prefer the cheapest mechanism before using a more expensive one.

Initial order:

```text
1. path / folder structure
2. filename
3. indexes
4. metadata
5. Markdown content
6. semantic search in the future
```

The MVP does not depend on embeddings or a vector database.

---

# Initial Project Structure

```text
maik/
├── core/
│   ├── workspace.py
│   ├── catalog.py
│   ├── scanner.py
│   ├── sync.py
│   ├── search.py
│   └── storage/
│
├── user_mcp/
│
├── worker_mcp/
│
├── skills/
│   ├── maik-add-content/
│   ├── maik-process-document/
│   ├── maik-query/
│   └── maik-research-and-store/
│
├── tests/
│
└── pyproject.toml
```

Managed workspaces may be stored outside the application repository.

---

# V0.1 Scope

The first version should implement only the document core.

```text
Workspace
   ↓
RAW
   ↓
Pending Jobs
   ↓
Worker MCP
   ↓
External Agent
   ↓
Markdown
   ↓
PROCESSED
   ↓
Indexes
   ↓
Federated Search
```

## V0.1 Deliverables

- workspace creation and management;
- `managed` and `external` storage;
- `raw` + `processed` structure;
- SQLite catalog;
- identification through ID, hash, and path;
- detection of new, moved, renamed, and modified documents;
- simple processing queue;
- User MCP;
- Worker MCP;
- `.md` generation by the external agent;
- local indexes;
- global index;
- federated search across workspaces;
- initial set of skills.

---

# Out of Scope for V0.1

The first version does not include:

- event consolidation;
- entity resolution;
- knowledge graph;
- trend detection;
- market signals;
- business impact analysis;
- recommendations;
- vector search;
- embeddings;
- large-scale crawling;
- automatic monitoring of the entire web.

These components belong to later Maik layers.

---

# Planned Evolution

```text
Layer 1 — RAW
Original evidence

        ↓

Layer 2 — PROCESSED
Structured Markdown representation

        ↓

Layer 3 — EVENTS
Event extraction and consolidation

        ↓

Layer 4 — MARKET MEMORY
Relationships, history, and temporal context

        ↓

Layer 5 — MARKET INTELLIGENCE
Patterns, signals, and changes

        ↓

Layer 6 — BUSINESS IMPACT
Relationship between market developments
and a specific organization's context
```

The same worker architecture can be reused at every layer:

```text
Layer N
   ↓
Pending Jobs
   ↓
Worker MCP
   ↓
Agent + Skill
   ↓
Layer N+1
```

---

# Vision

Maik should work as a knowledge infrastructure independent of the model being used.

It does not try to replace the agent.

It provides the agent with:

- persistent memory;
- organization;
- tools;
- state;
- work queues;
- indexes;
- provenance;
- federated access to knowledge.

In its initial form:

> **Maik turns user-organized files into a persistent, searchable Markdown memory processed by external agents through MCP.**

In the future, the same foundation can evolve into a market brain capable of consolidating events, connecting signals, and supporting intelligence workflows.
