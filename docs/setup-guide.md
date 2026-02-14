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
│  │              │    │  AI Agent (Gemini 2.0)    │     │
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
            │ API Call (nur Gemini-Anfrage)
            ▼
┌──────────────────────┐
│  Google Gemini API    │
│  - gemini-2.0-flash   │
│  - text-embedding-004 │
└──────────────────────┘
```

## Zwei Workflows

### 1. Knowledge Base Ingestion (`ngo-tools-kb-ingestion-workflow.json`)

Befüllt die Vektor-Datenbank mit NGO.tools Website-Inhalten.

**Flow:** Manual Trigger → Sitemap fetchen → URLs parsen → Seiten abrufen → HTML→Text → Chunking (1000 Zeichen, 200 Overlap) → Embeddings (Gemini) → pgvector speichern

**Wann ausführen:** Einmal initial, danach bei Content-Änderungen oder per Schedule (z.B. wöchentlich).

### 2. RAG Chatbot (`ngo-tools-chatbot-workflow.json`)

Der eigentliche Chatbot, der auf der Website eingebettet wird.

**Flow:** Chat Trigger → AI Agent (Gemini 2.0 Flash) → durchsucht PGVector Knowledge Base → generiert Antwort → zurück an Widget

**Features:**
- Window Buffer Memory (letzte 10 Nachrichten)
- System Prompt auf Deutsch, strikt auf Knowledge Base begrenzt
- Eskalation an Team bei unbekannten Fragen

## Setup-Schritte

### Schritt 1: pgvector auf Cloudron aktivieren

```sql
-- In der PostgreSQL-Datenbank (Cloudron Terminal)
CREATE EXTENSION IF NOT EXISTS vector;
```

### Schritt 2: Workflows importieren

1. n8n öffnen (n8n.zwiener.it)
2. Einstellungen → Import from File
3. Beide JSON-Dateien importieren

### Schritt 3: Credentials konfigurieren

| Credential | Typ | Wo anlegen |
|-----------|-----|-----------|
| Google Gemini API | API Key | n8n → Credentials → Google Gemini |
| Cloudron PostgreSQL | Postgres Connection | n8n → Credentials → Postgres (Host: localhost oder Cloudron-interner Hostname) |

### Schritt 4: Knowledge Base befüllen

1. Ingestion-Workflow öffnen
2. Sitemap-URL prüfen (https://ngo.tools/sitemap.xml)
3. Manuell ausführen
4. Prüfen ob Vektoren gespeichert wurden

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
      'Hallo! 👋 Ich bin der digitale Assistent von NGO.tools.',
      'Wie kann ich dir helfen? Frag mich zu Funktionen, Preisen oder technischen Themen.'
    ],
    i18n: {
      en: {
        title: 'NGO.tools Hilfe',
        subtitle: 'KI-gestützter Assistent für Vereine',
        footer: '⚡ Powered by KI · Keine persönliche Beratung',
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
| **KI-Kennzeichnung** | System Prompt + Footer im Widget: "Powered by KI" | ✅ |
| **Keine Täuschung** | Bot stellt sich als KI vor, nicht als Mensch | ✅ |
| **Mensch erreichbar** | Eskalation an kontakt@ngo.tools bei Nicht-Wissen | ✅ |
| **Logging** | n8n Execution Logs (automatisch) | ✅ |
| **Risikobewertung** | Limited Risk – keine High-Risk-Anwendung | ✅ |

### DSGVO Compliance

| Anforderung | Umsetzung | Status |
|------------|-----------|--------|
| **Datenminimierung** | Nur Chat-Nachrichten verarbeitet, keine Profile | ✅ |
| **Hosting in EU** | Hetzner DE (Cloudron) | ✅ |
| **Kein Tracking** | Keine Cookies, keine User-IDs, Session-basiert | ✅ |
| **Consent** | Cookie-Banner erweitern: "KI-Chatbot nutzt Google Gemini API" | ⚠️ TODO |
| **Datenschutzerklärung** | Ergänzen: Chatbot-Section mit Datenverarbeitung | ⚠️ TODO |
| **AV-Vertrag Google** | Google Cloud Data Processing Addendum | ⚠️ Prüfen |
| **Löschung** | Session-Daten automatisch löschen (z.B. nach 30 Tagen) | ⚠️ TODO |
| **Auskunftsrecht** | Execution Logs exportierbar über n8n | ✅ |

### Datenschutzerklärung – Textbaustein

> **KI-Chatbot**
> Auf unserer Website setzen wir einen KI-gestützten Chatbot ein, der Fragen zu NGO.tools beantwortet. Der Chatbot nutzt Google Gemini (Google Ireland Ltd.) zur Verarbeitung von Anfragen. Ihre Chat-Nachrichten werden über unseren Server (Hetzner, Deutschland) an die Google Gemini API übermittelt. Es werden keine personenbezogenen Daten gespeichert, sofern Sie diese nicht selbst im Chat eingeben. Die Rechtsgrundlage ist Art. 6 Abs. 1 lit. f DSGVO (berechtigtes Interesse an effizienter Kundenkommunikation). Chat-Sitzungen werden nach 30 Tagen automatisch gelöscht.

## Kosten

| Posten | Kosten |
|--------|--------|
| Hetzner Server | Bereits vorhanden |
| Cloudron + n8n | Bereits vorhanden |
| PostgreSQL + pgvector | Bereits vorhanden (Cloudron Addon) |
| Google Gemini API | Pay-per-use, ~$0.10/1M Input-Token (Flash) |
| **Geschätzte Kosten NGO-Traffic** | **< 5€/Monat** |

## Wartung & Pflege

| Aufgabe | Frequenz |
|---------|----------|
| Knowledge Base aktualisieren | Bei Content-Änderungen oder wöchentlich |
| n8n Execution Logs prüfen | Wöchentlich |
| Chatbot-Antworten stichprobenartig testen | Monatlich |
| Session-Daten bereinigen | Automatisch (30-Tage-Retention) |
| Gemini API Kosten überwachen | Monatlich |
