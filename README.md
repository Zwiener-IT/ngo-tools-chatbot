# NGO.tools RAG Chatbot

DSGVO- & EU AI Act-konformer KI-Chatbot für [NGO.tools](https://ngo.tools) – den All-in-One Werkzeugkasten für Vereine und Nonprofits.

## Stack

| Komponente | Technologie |
|-----------|------------|
| Orchestrierung | n8n (Self-Hosted auf Cloudron) |
| Vektor-DB | PostgreSQL + pgvector (Cloudron Addon) |
| LLM | Google Gemini 2.0 Flash |
| Embeddings | Google text-embedding-004 |
| Frontend | @n8n/chat Widget |
| Hosting | Hetzner (DE) – komplett EU |

## Architektur

```
Website (ngo.tools)
  └── @n8n/chat Widget
        └── HTTPS Webhook
              └── n8n (Cloudron)
                    ├── AI Agent (Gemini 2.0 Flash)
                    ├── PGVector Knowledge Base
                    └── Window Buffer Memory
```

## Workflows

### 1. `workflows/chatbot-rag.json`
Der Haupt-Chatbot – beantwortet Nutzerfragen basierend auf der Knowledge Base.

### 2. `workflows/kb-ingestion.json`
Befüllt die Vektor-Datenbank: Sitemap crawlen → HTML parsen → Chunking → Embeddings → pgvector.

## Quick Start

1. **pgvector aktivieren**
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   ```

2. **Workflows importieren**
   - n8n öffnen → Import from File → beide JSON-Dateien laden

3. **Credentials konfigurieren**
   - Google Gemini API Key
   - PostgreSQL Connection (Cloudron)

4. **Knowledge Base befüllen**
   - KB Ingestion Workflow manuell ausführen

5. **Widget einbetten**
   - HTML-Snippet aus `widget/embed.html` in die Website einfügen

## Compliance

### EU AI Act (Limited Risk)
- KI-Kennzeichnung im Widget und System Prompt
- Eskalation an menschliches Team
- Automatisches Logging via n8n Executions

### DSGVO
- Hosting komplett in DE (Hetzner)
- Keine User-Tracking, Session-basiert
- Datenschutzerklärung-Textbaustein in `docs/setup-guide.md`
- AV-Vertrag mit Google erforderlich

## Kosten

| Posten | Kosten |
|--------|--------|
| Infrastruktur | 0€ (bestehender Cloudron-Server) |
| Gemini API | ~5€/Monat (geschätzt für NGO-Traffic) |

## Dokumentation

Vollständige Anleitung: [docs/setup-guide.md](docs/setup-guide.md)

## Lizenz

MIT

---

Erstellt von [Zwiener IT](https://zwiener.it) – KI-Strategieberatung für Mittelstand und NGOs.
