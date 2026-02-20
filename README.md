# NGO.tools RAG Chatbot

DSGVO- & EU AI Act-konformer KI-Chatbot für [NGO.tools](https://ngo.tools) – den All-in-One Werkzeugkasten für Vereine und Nonprofits.

## Stack

| Komponente | Technologie |
|-----------|------------|
| Orchestrierung | n8n (Self-Hosted auf Cloudron) |
| Vektor-DB | PostgreSQL + pgvector (Cloudron Addon) |
| LLM | OpenAI `gpt-4o-mini` |
| Embeddings | OpenAI `text-embedding-3-small` |
| Frontend | @n8n/chat Widget + Admin UI |
| Hosting | Hetzner (DE) – komplett EU |

## Architektur

```
Website (ngo.tools)
  └── @n8n/chat Widget
        └── HTTPS Webhook
              └── n8n (Cloudron)
                    ├── AI Agent (OpenAI gpt-4o-mini)
                    ├── PGVector Knowledge Base
                    └── Window Buffer Memory (10 Messages)

Admin UI (admin/index.html)
  ├── Dokumente hochladen → KB Ingestion Workflow
  ├── URLs crawlen → URL Crawler Workflow
  ├── Status abfragen → KB Status Workflow
  └── Chunks löschen → KB Delete Workflow
```

## Workflows

| # | Workflow | n8n ID | Lokale JSON | Beschreibung |
|---|---------|--------|-------------|-------------|
| 1 | RAG Chatbot | `PFI5UEgky9nN0U4j` | `workflows/chatbot-rag.json` | Haupt-Chatbot — beantwortet Nutzerfragen via RAG |
| 2 | KB Ingestion | `D74o4IMPY9yNooJl` | `workflows/kb-ingestion.json` | Dokumente chunken + embedden → pgvector speichern |
| 3 | URL Crawler | `8J4wxj2UuFmpXpKY` | `workflows/url-crawler.json` | Website crawlen (Sitemap oder Einzel-URL) + in KB speichern |
| 4 | KB Status | `0NWL8G87JV7AylSf` | `workflows/kb-status.json` | Knowledge Base Statistiken (Chunks, Quellen, Typen) |
| 5 | KB Delete | `wfgok310jJzLs67L` | `workflows/kb-delete.json` | Chunks nach Source aus KB löschen |

## Quick Start

1. **pgvector aktivieren**
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   ```

2. **Credentials in n8n anlegen**
   - OpenAI API Key
   - PostgreSQL Connection (Cloudron)

3. **Workflows importieren**
   - n8n → Import from File → alle 5 JSON-Dateien aus `workflows/` laden
   - Oder: Admin UI unter `admin/index.html` öffnen für komfortablere Verwaltung

4. **Knowledge Base befüllen**
   - Admin UI → Tab "URL Crawlen" → Sitemap `https://ngo.tools/sitemap.xml` crawlen
   - Oder: Tab "Dokumente" → Freitext/JSON/PDF hochladen

5. **Widget einbetten**
   - HTML-Snippet aus `widget/embed.html` in die Website einfügen

## Admin UI

Die Admin UI (`admin/index.html`) bietet:
- **Dokumente**: Freitext, JSON oder PDF/TXT/MD hochladen → automatische Ingestion
- **URL Crawlen**: Sitemap oder Einzel-URLs crawlen → HTML → Text → Embeddings
- **Status**: Übersicht aller KB-Einträge mit Suche, Filter, Sortierung + Löschen

## Daten

| Datei | Beschreibung |
|-------|-------------|
| `data/system-prompt.md` | System Prompt für den Chatbot (Single Source of Truth) |
| `data/ngo-tools-docs.md` | Generierte Referenzdokumentation (~22k Zeilen) |

## Compliance

### EU AI Act (Limited Risk)
- KI-Kennzeichnung im Widget und System Prompt
- Eskalation an menschliches Team
- Automatisches Logging via n8n Executions

### DSGVO
- Hosting komplett in DE (Hetzner)
- Keine User-Tracking, Session-basiert
- Datenschutzerklärung-Textbaustein in `docs/setup-guide.md`
- AV-Vertrag mit OpenAI erforderlich

## Kosten

| Posten | Kosten |
|--------|--------|
| Infrastruktur | 0 EUR (bestehender Cloudron-Server) |
| OpenAI API | ~5 EUR/Monat (geschätzt für NGO-Traffic) |

## Dokumentation

Vollständige Anleitung: [docs/setup-guide.md](docs/setup-guide.md)

## Lizenz

MIT

---

Erstellt von [Zwiener IT](https://zwiener.it) – KI-Strategieberatung für Mittelstand und NGOs.
