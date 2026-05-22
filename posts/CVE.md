# CVE KB

A high-signal CVE knowledge base construction and retrieval project for automated penetration testing Agents and RAG systems.

Instead of dumping PoCs, blog posts, and vulnerability articles wholesale into a vector store, this project organizes public PoC signals, NVD metadata, detection templates, and attack surface clues into structured CVE cards optimized for LLM retrieval and reasoning.

The full large database is currently set as the default:

- Year range: `2018-2026`
- Default knowledge base directory: `out-prod-2018plus`
- Default vector store directory: `vectordb-2018plus`
- Offline mode preferred by default

## Use Cases

- CVE retrieval backend for automated penetration testing Agents
- Fast vulnerability lookup during competitions / CTF / real engagements
- Vulnerability knowledge recall for cloud security, Kubernetes, AI infra, identity systems, CI/CD
- Reverse lookup of candidate CVEs from target port, path, header, version, and product fingerprints
- Pulling exploit / detection / chain clues from CVE search results

## Current Capabilities

- Build structured CVE records from `PoC-in-GitHub`
- Generate three chunk types: `overview / exploit / detection`
- Generate BM25 retrieval index
- Generate attack chain relationship graph
- Generate attack surface fingerprint database
- Supports `bm25 / vector / hybrid` retrieval modes
- Supports offline local vector store
- Supports offline `NVD feed`
- Supports offline `nuclei-templates`
- Supports fine-grained tags:
  - `cloud`
  - `k8s`
  - `ai_infra`
  - `identity`
  - `cicd`

## Why Not a Typical CVE Vector Store

Common problems with the naive approach:

- Raw text is noisy; README and article content pollutes embeddings
- Searching by product name tends to semantically drift to unrelated vulnerabilities
- Hard to preserve structured clues like version, port, path, header, and payload
- Difficult for Agents to subsequently reason about "is this exploitable", "how to probe", "how to chain"

This project's goal is to turn each CVE into a high-signal retrieval card. Core fields include:

- `products / vendors`
- `affected_versions`
- `vuln_types`
- `asset_tags / domain_tags`
- `cloud_tags / k8s_tags / ai_infra_tags / identity_tags / cicd_tags`
- `attack_surface`
- `exploit_recipe`
- `detection_query`
- `search_terms`
- `retrieval_text`

## Repository Structure

### Unified Entry Point

- `kb.py`
  - The recommended entry point
  - Provides `build / query / ask / target / exploit / detect / chain / fingerprint / ingest`
  - Points to the `2018-2026` large database by default

### Build Entry Point

- `build_cve_kb.py`
  - Lower-level build CLI
  - Suitable for passing more granular parameters

### Retrieval Entry Point

- `retriever.py`
  - Local retriever
  - Supports:
    - `bm25`
    - `vector`
    - `hybrid`
    - Natural language `ask`
    - Structured `target`
    - Fingerprint matching
    - exploit / detection / chain

### Vector Ingestion

- `ingest.py`
  - Embeds `chunks.jsonl` and writes to FAISS
  - Supports local embedding

### Bridge

- `bridge.py`
  - Converts `records.jsonl` into a format better suited for Agent / Playground consumption

### Manual Curation Template

- `curated_overrides.example.json`
  - Manual overrides for high-value CVEs before a competition

### Core Code Directory

- `cve_kb_builder/builder.py`
  - Main pipeline
  - Responsible for integrating PoC, NVD, README, references, nuclei templates, etc. into the final knowledge base

- `cve_kb_builder/taxonomy.py`
  - Tag inference, rules, attack surface, and exploit recipe derivation

- `cve_kb_builder/sources.py`
  - Data source reading and enrichment
  - Currently supports:
    - `PoC-in-GitHub`
    - Online NVD / EPSS / KEV
    - Offline `NVD feed`
    - GitHub README
    - Deep reference crawling
    - Offline `nuclei-templates`

- `cve_kb_builder/indexing.py`
  - BM25, chain graph, and fingerprint generation

- `cve_kb_builder/models.py`
  - Data structure definitions

## Output Files

After building, the following files are produced:

- `records.jsonl`
  - Primary records

- `chunks.jsonl`
  - Chunks for embedding / RAG

- `bm25_docs.jsonl`
  - BM25 documents

- `bm25_index.json`
  - BM25 index

- `chain_graph.json`
  - Attack chain relationships

- `fingerprints.jsonl`
  - Attack surface / fingerprint database

- `summary.json`
  - Build summary

## Default Database

The current default is the large database:

- `out-prod-2018plus`
- `vectordb-2018plus`

This database covers:

- `2018`
- `2019`
- `2020`
- `2021`
- `2022`
- `2023`
- `2024`
- `2025`
- `2026`

Current local full build results:

- `7533` CVE records
- `22599` chunks

## Quick Start

### 1. Prepare Data Sources

Clone `PoC-in-GitHub` locally:

```bash
git clone https://github.com/nomi-sec/PoC-in-GitHub.git source_repo
```

For offline enrichment, also prepare:

```bash
git clone https://github.com/projectdiscovery/nuclei-templates offline_feeds/nuclei-templates
```

Place offline NVD feeds in:

- `offline_feeds/nvd/`

File name format examples:

- `nvdcve-2.0-2024.json.gz`
- `nvdcve-2.0-2025.json.gz`

### 2. Build Full Local Database

```bash
python kb.py build --nvd-feed-dir offline_feeds/nvd --nuclei-templates-dir offline_feeds/nuclei-templates
```

Defaults to building:

- `2018-2026`
- Output to `out-prod-2018plus`

### 3. Build Local Vector Store

```bash
python kb.py ingest --provider local
```

By default, reads:

- `out-prod-2018plus/chunks.jsonl`

And writes to:

- `vectordb-2018plus`

## Most Common Commands

### Standard Query

```bash
python kb.py query "Kubernetes ingress admission controller rce" --mode hybrid --provider local
```

### Natural Language Ask

```bash
python kb.py ask "Target is a port 443 admin portal, looks like a firewall or gateway, login page with remote access entry, version around 10.2.4, looking for unauthenticated command execution"
```

### Structured Target Lookup

```bash
python kb.py target --product keycloak --version 24.0.3 --path /realms/ --text "oidc token authorization bypass"
```

### Exploit Chunk

```bash
python kb.py exploit CVE-2024-3400
```

### Detection Chunk

```bash
python kb.py detect CVE-2024-3400
```

### Chain

```bash
python kb.py chain CVE-2024-3400
```

### Fingerprint

```bash
python kb.py fingerprint "port 443 open, remote access login portal, firewall gateway"
```

## Retrieval Modes

- `bm25`
  - Fastest
  - Best for queries with clear product names, paths, ports, or headers

- `vector`
  - Better for natural language and semantic queries

- `hybrid`
  - Recommended
  - BM25 handles hard signals; vector handles semantic coverage

## Offline Enrichment Sources

To ensure older CVEs also retain version, path, port, and detection clues:

- Offline `NVD feed`
  - Supplements:
    - `products`
    - `vendors`
    - `affected_versions`
    - `CPE`

- Offline `nuclei-templates`
  - Supplements:
    - `paths`
    - `headers`
    - `request_examples`
    - Some ports
    - `detection_query`

## Content Not Recommended for GitHub

The following should generally not be pushed to the source repository:

- `source_repo/`
- `offline_feeds/`
- `out-*`
- `vectordb*`
- `*.log`
- `__pycache__/`
- `.pytest_cache/`
- `.claude/`

## Installing Vector Dependencies

To run the local vector store, install:

```bash
pip install langchain-core langchain-community faiss-cpu fastembed pyyaml
```

For online embedding, additionally install:

```bash
pip install langchain-openai
```

## Summary

This is a CVE retrieval backend for automated penetration testing Agents and RAG systems — not a generic archive repository.

It emphasizes:

- High-signal structured records
- Offline capability
- Version / port / path / header / exploit clues
- Designed for LLMs to continue reasoning and acting on the retrieved data
