# 🎯 MVP-Scope & Abgrenzung: DocuPulse

**Projekt:** DocuPulse – Leichtgewichtige Document Processing API (PDF-zu-JSON)  
**Phase:** 1 – Produktspezifikation & Anforderungsanalyse  
**Zielgruppe:** Software-Entwickler & API-Integratoren  
**Stand:** November 2026  

---

## 1. Zielsetzung des MVPs

Das Minimum Viable Product (MVP) von **DocuPulse** verfolgt ein klares, innerhalb eines Semesters realistisch umsetzbares Ziel:

> **Einen vollständigen, Ende-zu-Ende demonstrierbaren API-Durchstich bereitzustellen, der textbasierte PDF-Dokumente über einen HTTP-Endpunkt entgegennimmt, semantisch mittels KI analysiert und als strikt validiertes, typsicheres JSON zurückgibt.**

Statt ein überladenes Gesamtsystem mit eigenem Web-Frontend, Benutzerverwaltung und Datenbank-Persistenz zu entwerfen, konzentriert sich das MVP zu 100 % auf die **Developer Experience (DX)** und **Zuverlässigkeit der Kern-Pipeline („PDF rein → JSON raus“)**.

---

## 2. Demonstrierbarer Kern-Anwendungsfall (End-to-End MVP Flow)

Das MVP deckt folgenden vollständigen Anwendungsfall ab:

1. **API-Request:** Ein Entwickler (oder ein aufrufender Microservice) sendet ein textbasiertes PDF-Dokument per `POST /api/v1/extract` (`multipart/form-data`) an den DocuPulse-Server. Dabei gibt er entweder das Standardprofil `profile=invoice` oder ein eigenes JSON-Schema (`schema={...}`) an.
2. **Lokale Vorprüfung & Textextraktion (Pre-Processing):**
   - Der Server prüft Dateigröße (≤ 10 MB) und PDF-Header (`%PDF-`).
   - Der Textlayer sowie die Seitenzahl werden lokal im Arbeitsspeicher aus dem PDF extrahiert.
   - Ist die Datei korrupt oder enthält sie keinen extrahierbaren Textlayer (reiner Bild-Scan), bricht der Server **vor** dem externen KI-Aufruf kontrolliert mit HTTP `400 Bad Request` bzw. `422 Unprocessable Entity` ab (Kostenschutz).
3. **Semantische KI-Extraktion:** Der extrahierte Text wird zusammen mit den Schema-Vorgaben asynchron an die angebundene **Google Gemini API (Gemini Flash)** gesendet.
4. **Strikte Schema- & Typ-Validierung:** Die Rohantwort des Modells wird gegen die Zielstruktur (vordefiniertes `Invoice`-Struct bzw. übergebenes JSON-Schema) validiert. Ungültige Antworten oder Markdown-Artefakte werden abgefangen.
5. **JSON-Response & Speichermbereinigung:** Der Client erhält synchron eine HTTP-`200`-Antwort mit dem validierten JSON-Dokument und Laufzeit-Metadaten (`processing_time_ms`, `pages`). Alle PDF-Daten im RAM werden unmittelbar nach Request-Ende verworfen (Zero Data Retention).

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Entwickler / Client
    participant API as DocuPulse API (Rust/Axum)
    participant Parser as Lokaler PDF-Parser
    participant LLM as Google Gemini API (Flash)

    Dev->>API: POST /api/v1/extract (file.pdf + profile="invoice" | schema)
    API->>Parser: Magic-Byte-Check (%PDF-) & Textextraktion im RAM
    alt Ungültiges PDF oder leerer Textlayer (Scan)
        Parser-->>API: Validierungsfehler
        API-->>Dev: HTTP 400 / 422 (JSON Error – kein KI-Call / 0 Kosten)
    else Valides, textbasiertes PDF
        Parser-->>API: Extrahierter Text + Seitenzahl
        API->>LLM: Strukturierter Prompt + Text + Ziel-Schema
        LLM-->>API: Roh-JSON
        API->>API: Strikte Validierung (Serde / JSON-Schema)
        API-->>Dev: HTTP 200 OK (Validiertes JSON + Metadaten)
    end
```

---

## 3. Im MVP enthalten (In-Scope / Must-Have)

Folgende Funktionen bilden den verbindlichen Umfang für das MVP:

| Bereich | Enthaltene Funktionalität (MVP In-Scope) | Referenz |
| :--- | :--- | :--- |
| **HTTP-Endpunkt** | Synchroner REST-Endpunkt `POST /api/v1/extract` (`multipart/form-data`) sowie ein Health-Check `GET /health`. | `US-01`, `FA-01` |
| **Eingabeformat** | Textbasierte PDF-Dateien (`application/pdf`) bis max. 10 MB und max. 5 Seiten pro synchronem Request. | `US-02`, `FA-02`, `FA-03` |
| **Standard-Profil (`invoice`)** | Sofort nutzbares Extraktionsprofil für Rechnungen (`invoice_number`, `date`, `vendor`, `currency`, `total_amount`, `tax_amount`, `line_items`). | `US-03`, `FA-05` |
| **Custom JSON-Schema** | Übergabe eines benutzerdefinierten JSON-Schemas im Request zur flexiblen Datenextraktion für beliebige Dokumenttypen. | `US-04`, `FA-06` |
| **Ausgabe & Validierung** | 100 % valides, typsicheres JSON (`application/json`) durch serverseitige Deserialisierung und Schema-Prüfung vor dem Senden. | `US-05`, `FA-07`, `NFA-03` |
| **Fehlerbehandlung** | Einheitliche JSON-Fehlermeldungen mit klaren HTTP-Statuscodes (`400`, `413`, `422`, `502`, `504`) ohne Absturz des Servers. | `US-06`, `FA-08` |
| **Datenschutz** | Reine In-Memory-Verarbeitung ohne Speicherung von Dokumenten auf der Festplatte oder in einer Datenbank. | `FA-09`, `NFA-04` |

---

## 4. Nicht im MVP enthalten (Out-of-Scope & Spätere Erweiterungen)

Um den Projektrahmen im Semester zuverlässig einzuhalten, werden folgende Features **bewusst aus dem MVP ausgeschlossen** und in spätere Ausbaustufen (Post-MVP) bzw. komplett außerhalb des Projekts verlagert:

### 4.1 Geplante Erweiterungen nach dem MVP (Should-Have / Could-Have)
- **Multimodaler Scan-/OCR-Fallback (`Should-Have`, US-07):** Verarbeitung gescannter PDFs ohne Textlayer über Vision-Payloads oder lokale OCR. *Begründung:* Im MVP werden reine Bild-Scans nach der Textlayer-Prüfung mit einer klaren Fehlermeldung (`EMPTY_TEXT_LAYER`) abgewiesen, um Komplexität und Token-Kosten gering zu halten.
- **Interaktive OpenAPI-/Swagger-UI (`Should-Have`, US-08):** Automatisch generierte Browser-Dokumentation unter `/swagger-ui`.
- **Asynchrone Job-Queue (`Should-Have`, US-09):** `POST /api/v1/jobs` und Status-Polling für große Dokumente (> 5 Seiten).
- **Quellen-Grounding & Webhooks (`Could-Have`, US-10):** Seitenreferenzen pro extrahiertem Feld und Push-Webhooks.

### 4.2 Explizit ausgeschlossen (Won't-Have)
- **Kein grafisches Web-Frontend (GUI):** DocuPulse ist eine reine Backend-API für Entwickler (Nutzung via `curl`, Postman oder Client-Code).
- **Keine Datenbank-Persistenz:** Keine Speicherung von Dokumenten, Extraktionsergebnissen oder Nutzerkonten in SQL/NoSQL-Datenbanken.
- **Keine Benutzer- und Rechteverwaltung:** Kein OAuth/JWT-Login-System, kein Billing-Portal.
- **Keine anderen Dateiformate:** Keine Unterstützung für Word (`.docx`), Excel (`.xlsx`), E-Mails (`.eml`) oder lose Bilddateien (`.jpg`, `.png`).
- **Kein Training eigener KI-Modelle:** Es wird ausschließlich ein vortrainiertes Sprachmodell über eine externe API genutzt.

---

## 5. Externe APIs, Abhängigkeiten & Kostenbetrachtung

Gemäß den Rahmenbedingungen nutzt DocuPulse für die semantische Textanalyse einen externen KI-Service. Abhängigkeiten und Kosten sind wie folgt kalkuliert und abgesichert:

### 5.1 Externe API: Google Gemini API (`gemini-2.5-flash`)
- **Zweck:** Semantische Analyse des lokal extrahierten PDF-Textes und Überführung in die geforderte JSON-Struktur (via *Structured Outputs* / `response_schema`).
- **Kostenmodell im Semesterprojekt:**
  - Für Entwicklung, Tests und Demonstration im Semester wird der **kostenlose Google AI Studio Free-Tier** genutzt (**0,00 € Kosten**).
  - Selbst im Paid-Tier lägen die Kosten bei `gemini-2.5-flash` im Bereich von Bruchteilen eines Cents pro Dokument (ca. 1.000–2.000 Input-Tokens pro einseitiger PDF-Rechnung).
- **Kostenschutz-Mechanismen in unserer Architektur:**
  1. **Lokaler Pre-Check vor dem API-Aufruf:** Korrupte Dateien, falsche Dateiformate, leere PDFs oder unzulässige JSON-Schemas werden lokal in Rust abgefangen, **bevor** ein einziger Token verbraucht wird (`US-02`, `US-06`).
  2. **Striktes Größen- und Seitenlimit:** Synchron akzeptierte PDFs sind im MVP auf max. 10 MB bzw. 5 Seiten begrenzt (`FA-01`).
  3. **Text-Extraktion statt Bild-Upload im MVP:** Da im MVP nur der extrahierte Textlayer (wenige Kilobyte) statt hochauflösender Seitenbilder an die KI-API gesendet wird, sinkt der Token-Verbrauch um ca. 80–90 %.

### 5.2 Umgang mit technischen Abhängigkeiten & Risiken
- **API-Key-Verwaltung:** Der API-Schlüssel (`GEMINI_API_KEY`) wird ausschließlich über Umgebungsvariablen geladen und niemals im Quellcode oder Git-Repository gespeichert.
- **Timeouts & Fehlertoleranz:** Externe HTTP-Aufrufe an die Gemini API erhalten ein festes Timeout (15 Sekunden). Antwortet die externe API nicht rechtzeitig oder mit einem Rate-Limit (`HTTP 429`), gibt DocuPulse kontrolliert `HTTP 502 Bad Gateway` bzw. `HTTP 504 Gateway Timeout` im standardisierten JSON-Fehlerformat zurück.
- **Testbarkeit ohne API-Kosten:** Für automatisierte Unit- und Integrationstests wird die KI-Client-Schnittstelle hinter einem Rust-Trait abstrahiert, sodass Tests mit deterministischen Mock-Antworten (ohne Netzwerkzugriff und ohne API-Key) ausgeführt werden können.

---

## 6. Definition of Done (MVP)

Das MVP gilt in der Implementierungsphase als erfolgreich abgeschlossen, wenn:
1. Der Server lokal oder als Container startet und unter `GET /health` mit `HTTP 200` antwortet.
2. Ein Entwickler eine Beispiel-Rechnung (`rechnung.pdf`) per `curl` mit `profile=invoice` an `POST /api/v1/extract` senden kann und ein vollständig typisiertes JSON-Ergebnis erhält.
3. Ein Entwickler ein beliebiges textbasiertes PDF zusammen mit einem eigenen `schema` senden kann und ein gegen dieses Schema validiertes JSON zurückerhält.
4. Ungültige Dateien (z. B. umbenannte `.txt`-Datei, beschädigtes PDF oder Scan ohne Textlayer) ohne externen KI-Aufruf mit einem klaren JSON-Fehler (`HTTP 400`/`422`) abgewiesen werden.
5. Automatisierte Tests (`cargo test`) diese Kernpfade reproduzierbar prüfen.
