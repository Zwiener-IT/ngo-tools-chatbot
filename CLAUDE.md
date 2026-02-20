# CLAUDE.md — NGO.tools RAG Chatbot

## Projektübersicht

DSGVO- & EU AI Act-konformer RAG Chatbot für [NGO.tools](https://ngo.tools).
Betreiber der Plattform: Help NGO gGmbH (Friedrich Rominger).

| Komponente | Technologie |
|-----------|------------|
| Orchestrierung | n8n (Self-Hosted auf Cloudron) |
| Vektor-DB | PostgreSQL + pgvector (Cloudron Addon) |
| LLM | OpenAI `gpt-4o-mini` (temp 0.3, maxTokens 1024) |
| Embeddings | OpenAI `text-embedding-3-small` |
| Knowledge Base | PGVector, Tabelle `n8n_vectors_ngo_tools`, topK 5 |
| Text Splitter | 1000 chars, 200 overlap |
| Memory | Window Buffer (10 Messages) |
| Frontend | @n8n/chat Widget + Admin UI (Pico CSS) |
| Hosting | Hetzner (DE) – komplett EU |

## Workflow-Registry

| Workflow | n8n ID | Lokale JSON | Webhook-URL | Methode |
|----------|--------|-------------|-------------|---------|
| RAG Chatbot | `PFI5UEgky9nN0U4j` | `workflows/chatbot-rag.json` | `/webhook/ngo-tools-chat/chat` | Chat Trigger |
| KB Ingestion | `D74o4IMPY9yNooJl` | `workflows/kb-ingestion.json` | `/webhook/ngo-tools-ingest` | POST |
| URL Crawler | `8J4wxj2UuFmpXpKY` | `workflows/url-crawler.json` | `/webhook/ngo-tools-crawl` | POST |
| KB Status | `0NWL8G87JV7AylSf` | `workflows/kb-status.json` | `/webhook/ngo-tools-kb-status` | GET |
| KB Delete | `wfgok310jJzLs67L` | `workflows/kb-delete.json` | `/webhook/ngo-tools-kb-delete` | POST |

Alle Webhook-URLs haben das Prefix `https://n8n.zwiener.it`.

## Credential-IDs (n8n.zwiener.it)

| Credential | ID |
|-----------|-----|
| OpenAI API | `XhcWlA6JrLM2ptmU` |
| Postgres | `sd9vc72XPsJhxW8I` |

## Dateistruktur

```
ngo-tools-chatbot/
├── CLAUDE.md              ← diese Datei
├── README.md              ← Projekt-Dokumentation
├── .env.local             ← N8N_API_KEY, N8N_API_URL (git-ignored)
├── workflows/
│   ├── chatbot-rag.json   ← Haupt-Chatbot (PFI5UEgky9nN0U4j)
│   ├── kb-ingestion.json  ← Dokumente in PGVector speichern (D74o4IMPY9yNooJl)
│   ├── url-crawler.json   ← Website crawlen + ingestieren (8J4wxj2UuFmpXpKY)
│   ├── kb-status.json     ← KB-Statistiken abfragen (0NWL8G87JV7AylSf)
│   └── kb-delete.json     ← Chunks nach Source löschen (wfgok310jJzLs67L)
├── admin/
│   └── index.html         ← Admin UI (Pico CSS, Single-File)
├── widget/
│   └── embed.html         ← Chat-Widget Snippet für ngo.tools
├── data/
│   ├── system-prompt.md   ← System Prompt (Single Source of Truth)
│   ├── ngo-tools-docs.md  ← Generierte Referenzdoku (22k Zeilen)
│   └── ngo_docs.json      ← Legacy, git-ignored
├── docs/
│   └── setup-guide.md     ← Setup & Compliance Guide
└── .gitignore
```

## Deploy-Anleitung (n8n API)

```bash
# .env.local laden
source .env.local

# Workflow aktualisieren (PUT braucht name, nodes, connections, settings)
curl -X PUT "$N8N_API_URL/workflows/<WORKFLOW_ID>" \
  -H "X-N8N-API-KEY: $N8N_API_KEY" \
  -H "Content-Type: application/json" \
  -d @workflows/<dateiname>.json

# Workflow aktivieren (separater Call!)
curl -X POST "$N8N_API_URL/workflows/<WORKFLOW_ID>/activate" \
  -H "X-N8N-API-KEY: $N8N_API_KEY"
```

## n8n API Hinweise

- PUT erwartet `name`, `nodes`, `connections`, `settings` — `active` ist read-only bei PUT
- Activation: POST `/workflows/{id}/activate` (nicht PATCH, nicht PUT mit active)
- Webhook `responseMode: "responseNode"` als top-level Parameter (nicht in options)
- PGVector Default Data Loader: keine `dataType`/`jsonData` params, nur `options.metadata`
