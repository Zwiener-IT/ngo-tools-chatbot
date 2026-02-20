# NGO.tools RAG Chatbot – Setup & Compliance Guide

## Architektur-Übersicht

```
┌──────────────────────────────────────────────────────┐
│  ngo.tools Website                                    │
│  ┌──────────────────┐                                │
│  │  @n8n/chat Widget │◄── JavaScript Snippet          │
│  └────────┬─────────┘                                │
└───────────┼──────────────────────────────────────────┘
            │ HTTPS (Webhook)
            ▼
┌──────────────────────────────────────────────────────┐
│  Hetzner Server (Cloudron)                            │
│                                                       │
│  ┌─────────────┐    ┌──────────────────────────┐     │
│  │   n8n        │───►│  RAG Chatbot Workflow     │     │
│  │  (n8n.       │    │                          │     │
│  │  zwiener.it) │    │  Chat Trigger            │     │
│  │              │    │    ↓                      │     │
│  │              │    │  AI Agent (gpt-4o-mini)   │     │
│  │              │    │    ↓              ↑       │     │
│  │              │    │  PGVector ◄── Embeddings  │     │
│  │              │    │  (Knowledge Base)         │     │
│  └─────────────┘    └──────────────────────────┘     │
│                                                       │
│  ┌─────────────────────────────────────┐             │
│  │  PostgreSQL (Cloudron Addon)         │             │
│  │  + pgvector Extension                │             │
│  │  Tabelle: n8n_vectors_ngo_tools      │             │
│  └─────────────────────────────────────┘             │
└──────────────────────────────────────────────────────┘
            │
            │ API Call
            ▼
┌──────────────────────┐
│  OpenAI API           │
│  - gpt-4o-mini        │
│  - text-embedding-    │
│    3-small            │
└──────────────────────┘
```

## Workflows

| # | Workflow | n8n ID | Lokale JSON | Beschreibung |
|---|---------|--------|-------------|-------------|
| 1 | RAG Chatbot | `PFI5UEgky9nN0U4j` | `workflows/chatbot-rag.json` | Haupt-Chatbot via @n8n/chat Widget |
| 2 | KB Ingestion | `D74o4IMPY9yNooJl` | `workflows/kb-ingestion.json` | Dokumente chunken + embedden → pgvector |
| 3 | URL Crawler | `8J4wxj2UuFmpXpKY` | `workflows/url-crawler.json` | Website crawlen (Sitemap/Einzel-URL) + in KB speichern |
| 4 | KB Status | `0NWL8G87JV7AylSf` | `workflows/kb-status.json` | KB-Statistiken (Chunks, Quellen, Typen) |
| 5 | KB Delete | `wfgok310jJzLs67L` | `workflows/kb-delete.json` | Chunks nach Source aus KB löschen |

### 1. RAG Chatbot (`workflows/chatbot-rag.json`)

Der Haupt-Chatbot — beantwortet Nutzerfragen basierend auf der Knowledge Base.

**Flow:** Chat Trigger → AI Agent (OpenAI gpt-4o-mini) → durchsucht PGVector Knowledge Base → generiert Antwort → zurück an Widget

**Features:**
- Window Buffer Memory (letzte 10 Nachrichten)
- System Prompt auf Deutsch, strikt auf Knowledge Base begrenzt
- Eskalation an Team bei unbekannten Fragen

### 2. KB Ingestion (`workflows/kb-ingestion.json`)

Nimmt Dokumente via Webhook entgegen und speichert sie als Vektoren.

**Flow:** Webhook (POST) → Format Documents → Delete Existing → Wait → Store in PGVector (Chunking + Embeddings)

### 3. URL Crawler (`workflows/url-crawler.json`)

Crawlt Webseiten und speichert den Inhalt in der Knowledge Base.

**Flow:** Webhook (POST) → Route Mode (Sitemap/Single) → Fetch → HTML to Text → Delete Existing → Wait → Store in PGVector

### 4. KB Status (`workflows/kb-status.json`)

Liefert Statistiken zur Knowledge Base als JSON.

**Flow:** Webhook (GET) → Summary Query → Detail Query → Combine → Respond

### 5. KB Delete (`workflows/kb-delete.json`)

Löscht alle Chunks einer bestimmten Quelle.

**Flow:** Webhook (POST, `{source: "..."}`) → Delete Chunks → Respond

## Admin UI

Die Admin UI (`admin/index.html`) bietet eine komfortable Oberfläche für alle KB-Operationen:

1. **Dokumente hochladen**: Freitext, JSON oder PDF/TXT/MD Upload
2. **URL Crawlen**: Sitemap oder Einzel-URLs crawlen
3. **Status**: Übersicht aller KB-Einträge mit Suche, Filter, Sortierung + Löschen

**Öffnen:** Direkt als lokale HTML-Datei im Browser öffnen. Benötigt n8n API Key (wird im LocalStorage gespeichert).

## Setup-Schritte

### Schritt 1: pgvector auf Cloudron aktivieren

```sql
-- In der PostgreSQL-Datenbank (Cloudron Terminal)
CREATE EXTENSION IF NOT EXISTS vector;
```

### Schritt 2: Workflows importieren

1. n8n öffnen (n8n.zwiener.it)
2. Einstellungen → Import from File
3. Alle 5 JSON-Dateien aus `workflows/` importieren

### Schritt 3: Credentials konfigurieren

| Credential | Typ | n8n Credential ID |
|-----------|-----|-------------------|
| OpenAI API Key | API Key | `XhcWlA6JrLM2ptmU` |
| Cloudron PostgreSQL | Postgres Connection | `sd9vc72XPsJhxW8I` |

### Schritt 4: Knowledge Base befüllen

1. Admin UI öffnen (`admin/index.html`)
2. n8n API Key eingeben
3. Tab "URL Crawlen" → Sitemap-URL `https://ngo.tools/sitemap.xml` eingeben → "Sitemap" wählen → Crawlen
4. Optional: Tab "Dokumente" → zusätzliche Inhalte hochladen

### Schritt 5: Chat-Widget auf Website einbetten

```html
<!-- Im <head> oder vor </body> auf ngo.tools -->
<link href="https://cdn.jsdelivr.net/npm/@n8n/chat/dist/style.css" rel="stylesheet" />
<script type="module">
  import { createChat } from 'https://cdn.jsdelivr.net/npm/@n8n/chat/dist/chat.bundle.es.js';

  createChat({
    webhookUrl: 'https://n8n.zwiener.it/webhook/ngo-tools-chat/chat',
    mode: 'window',
    chatInputKey: 'chatInput',
    chatSessionKey: 'sessionId',
    showWelcomeScreen: true,
    initialMessages: [
      'Hallo! Ich bin der digitale Assistent von NGO.tools.',
      'Wie kann ich dir helfen? Frag mich zu Funktionen, Preisen oder technischen Themen.'
    ],
    i18n: {
      en: {
        title: 'NGO.tools Hilfe',
        subtitle: 'KI-gestützter Assistent für Vereine',
        footer: 'KI-Assistent · Keine persönliche Beratung',
        getStarted: 'Neue Unterhaltung',
        inputPlaceholder: 'Stelle deine Frage...',
      },
    },
    theme: {
      primaryColor: '#4F46E5',
      secondaryColor: '#F3F4F6',
    }
  });
</script>
```

## DSGVO & EU AI Act Compliance

### EU AI Act (Limited Risk – Transparenzpflichten)

| Anforderung | Umsetzung | Status |
|------------|-----------|--------|
| **KI-Kennzeichnung** | System Prompt + Footer im Widget: "KI-Assistent" | done |
| **Keine Täuschung** | Bot stellt sich als KI vor, nicht als Mensch | done |
| **Mensch erreichbar** | Eskalation an kontakt@ngo.tools bei Nicht-Wissen | done |
| **Logging** | n8n Execution Logs (automatisch) | done |
| **Risikobewertung** | Limited Risk – keine High-Risk-Anwendung | done |

### DSGVO Compliance

| Anforderung | Umsetzung | Status |
|------------|-----------|--------|
| **Datenminimierung** | Nur Chat-Nachrichten verarbeitet, keine Profile | done |
| **Hosting in EU** | Hetzner DE (Cloudron) | done |
| **Kein Tracking** | Keine Cookies, keine User-IDs, Session-basiert | done |
| **Consent** | Cookie-Banner erweitern: "KI-Chatbot nutzt OpenAI API" | TODO |
| **Datenschutzerklärung** | Ergänzen: Chatbot-Section mit Datenverarbeitung | TODO |
| **AV-Vertrag OpenAI** | OpenAI Data Processing Addendum | TODO |
| **Löschung** | Session-Daten automatisch löschen (z.B. nach 30 Tagen) | TODO |
| **Auskunftsrecht** | Execution Logs exportierbar über n8n | done |

### Datenschutzerklärung – Textbaustein

> **KI-Chatbot**
> Auf unserer Website setzen wir einen KI-gestützten Chatbot ein, der Fragen zu NGO.tools beantwortet. Der Chatbot nutzt OpenAI (OpenAI, L.L.C., USA) zur Verarbeitung von Anfragen. Ihre Chat-Nachrichten werden über unseren Server (Hetzner, Deutschland) an die OpenAI API übermittelt. Es werden keine personenbezogenen Daten gespeichert, sofern Sie diese nicht selbst im Chat eingeben. Die Rechtsgrundlage ist Art. 6 Abs. 1 lit. f DSGVO (berechtigtes Interesse an effizienter Kundenkommunikation). Chat-Sitzungen werden nach 30 Tagen automatisch gelöscht.

## Kosten

| Posten | Kosten |
|--------|--------|
| Hetzner Server | Bereits vorhanden |
| Cloudron + n8n | Bereits vorhanden |
| PostgreSQL + pgvector | Bereits vorhanden (Cloudron Addon) |
| OpenAI API | Pay-per-use, gpt-4o-mini ~$0.15/1M Input, text-embedding-3-small ~$0.02/1M |
| **Geschätzte Kosten NGO-Traffic** | **< 5 EUR/Monat** |

## Wartung & Pflege

| Aufgabe | Frequenz |
|---------|----------|
| Knowledge Base aktualisieren | Bei Content-Änderungen oder wöchentlich |
| n8n Execution Logs prüfen | Wöchentlich |
| Chatbot-Antworten stichprobenartig testen | Monatlich |
| Session-Daten bereinigen | Automatisch (30-Tage-Retention) |
| OpenAI API Kosten überwachen | Monatlich |
