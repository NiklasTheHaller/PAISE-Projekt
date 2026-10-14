# 🦀 Produktspezifikation & Anforderungsanalyse: DocuPulse (Rust Edition)
**Projekt:** Document Processing API – KI-gestützte PDF-zu-JSON Extraktions-Engine in Rust  
**Phase:** 1 – Produktspezifikation & Anforderungsanalyse  
**Technologie-Basis:** **Rust** (Axum, Tokio, Serde)  
**Primäre Zielgruppe:** **Software-Entwickler & API-Integratoren**  
**MVP-Kern:** **PDFs einlesen, analysieren und als validiertes JSON ausgeben**  
**Status:** Freigegeben / MVP-Scope geschärft  
**Datum:** Oktober 2026  

---

## 1. Problemstellung & Projektidee

### 1.1 Ausgangslage & Problemstellung
Entwickler stehen in modernen Software-Architekturen regelmäßig vor der Herausforderung, unstrukturierte PDF-Dateien (Rechnungen, Lieferscheine, Berichte, Verträge) in strukturierte Datensätze (JSON) für Datenbanken, APIs oder Microservices umzuwandeln.

Bestehende Lösungen bereiten Entwicklern erhebliche Probleme:
- **Klassische Python-OCR-Stacks:** Schwerfällig im Deployment (große Docker-Images > 1 GB, C-Bindings wie Tesseract/Poppler), hoher Speicherverbrauch, langsam bei hohem Durchsatz und extrem fehleranfällig gegenüber variierenden PDF-Layouts.
- **Enterprise-Cloud-APIs (AWS Textract, Google Document AI):** Hohe Latenz, undurchsichtige Preismodelle, Vendor-Lock-in und komplizierte SDKs. Oft liefern sie unübersichtliche, tausende Zeilen lange OCR-Geometrie-JSONs statt direkt nutzbarer Business-Daten.
- **Generische LLM-Aufrufe:** Nicht-typsicher, probabilistisch formatierter Markdown-Text statt striktem JSON, keine Typprüfung auf Compilerebene und häufige Timeouts bei großen Payloads.

### 1.2 Projektidee & Lösungsansatz
**DocuPulse (Rust Edition)** ist eine extrem performante, ressourcenschonende und entwicklerzentrierte REST-API, geschrieben in **Rust**. 

Die Engine nimmt PDF-Dokumente entgegen, extrahiert deren Inhalt effizient (Text/visuelle Layer), übergibt diesen an ein KI-Modell zur semantischen Analyse und liefert über **Serde** strikt typisiertes, schema-konformes **JSON** an den Entwickler zurück.

**Warum Rust für dieses Projekt?**
1. **Performance & minimale Latenz:** Keine Garbage-Collection-Pausen, sub-millisekunden Overhead im API-Routing durch das `axum`/`tokio`-Ökosystem.
2. **Speicher- und Typsicherheit:** Keine Null-Pointer-Exceptions, garantierte Thread-Sicherheit und extrem geringer Memory-Footprint (< 30 MB RAM im Leerlauf).
3. **Kompaktes Single-Binary Deployment:** Minimales Docker-Image (Scratch / Alpine), keine Python-Interpreter- oder Dependency-Hölle.
4. **Typsicheres JSON-Parsing:** Das `serde`-Ökosystem garantiert, dass nur valides, schema-konformes JSON die API verlässt.

---

## 2. Zielgruppe & Kern-Anwendungsfälle

### 2.1 Zielgruppen-Definition: Die Entwickler-Community

| Persona | Profil & Kontext | Primäre Pain Points & Erwartungen |
| :--- | :--- | :--- |
| **Backend-Basti** (Rust/Go Backend Dev) | Entwickelt Microservices in performanten Sprachen; baut Schnittstellen für Finanzsysteme. | Hasst langsame Python-Dienste; will ein schnelles Tool als Docker-Container deployen; erwartet saubere REST-Endpunkte mit `curl`-Beispielen und OpenAPI-Specs. |
| **Frontend-Fabi** (TypeScript / React Dev) | Baut Web-Dashboards, in denen Nutzer PDFs hochladen und sofort Formularfelder vorausgefüllt bekommen. | Braucht synchrone, schnelle API-Antworten (< 5s) mit exakten JSON-Feldern; wünscht sich klare Fehlermeldungen bei ungültigen PDFs. |
| **DevOps-Dana** (Platform / Cloud Engineer) | Betreibt Container-Plattformen (K8s, Docker) und achtet auf RAM-Kosten und Stabilität. | Will Single-Binary-Container ohne Python-Overhead; verlangt verlässliche Health-Endpoints und geringen Ressourcenverbrauch. |

### 2.2 Kern-Anwendungsfälle (Core Developer Use Cases)

- **UC-01: Einfacher PDF-zu-JSON Sync-Workflow (MVP Core):**  
  Ein Entwickler sendet via `curl` oder HTTP-Client ein PDF an `POST /api/v1/extract`. Die API parst das PDF, analysiert es mit KI und antwortet direkt mit einem standardisierten JSON-Objekt.
- **UC-02: Schema-Driven Extraktion für individuelle APIs:**  
  Ein Entwickler übergibt neben der PDF-Datei ein eigenes JSON-Schema. Die API instruiert das KI-Modell exakt anhand dieses Schemas und validiert den Rückgabewert mit `serde_json`, bevor er ausgeliefert wird.
- **UC-03: Fehlerdiagnose bei korrupten Dokumenten:**  
  Ein Client lädt ein beschädigtes PDF oder eine Bilddatei mit falschem MIME-Type hoch. Die Rust-Engine fängt den Fehler bereits beim PDF-Parsing ab und liefert sofort HTTP 400 mit einem eindeutigen `error_code` zurück, ohne teure KI-Tokens zu verschwenden.

---

## 3. User Stories (Entwickler-Fokus)

Alle User Stories sind nach dem agilen Standard formuliert und enthalten messbare Akzeptanzkriterien im **Given-When-Then (Gherkin)**-Format.

### US-01: PDF-Upload und synchrone JSON-Extraktion (Must-Have – MVP-Kern)
- **Story:** Als **Entwickler** möchte ich ein PDF-Dokument per `POST /api/v1/extract` als Multipart-Formdata hochladen, um in derselben HTTP-Antwort die extrahierten Daten als valides JSON zu erhalten.
- **Akzeptanzkriterien:**
  - **Given:** Ein valides text- oder layoutbasiertes PDF-Dokument (1–3 Seiten, < 10 MB).
  - **When:** Der Entwickler einen HTTP POST an `/api/v1/extract` sendet.
  - **Then:** Antwortet die API mit HTTP 200 und einem wohlgeformten JSON-Payload (`Content-Type: application/json`).
  - **And:** Der JSON-Body enthält die extrahierten Schlüssel-Wert-Paare sowie Status- und Verarbeitungszeit-Metadaten (`processing_time_ms`).

### US-02: Nativer PDF-Textlayer- und Struktur-Parser in Rust (Must-Have – MVP-Kern)
- **Story:** Als **System-Entwickler** möchte ich, dass die API den Textlayer und Metadaten des PDFs mit einer nativen Rust-Crate (z. B. `lopdf` oder `pdf-extract`) liest, um vor dem KI-Aufruf zu prüfen, ob lesbarer Text vorhanden ist.
- **Akzeptanzkriterien:**
  - **Given:** Ein textbasiertes PDF wird empfangen.
  - **When:** Die Rust-Pipeline das Dokument verarbeitet.
  - **Then:** Wird der Text extrahiert und die Seitenzahl deterministisch im Rust-Speicher ermittelt.
  - **And:** Enthält das PDF keinen lesbaren Text (reiner Scan), schlägt die Pipeline automatisch die visuelle/multimodale Verarbeitungsroute ein.

### US-03: Vordefiniertes Rechnungs-/Beleg-Profil (`invoice`) (Must-Have – MVP-Kern)
- **Story:** Als **Backend-Entwickler** möchte ich mit dem Parameter `profile=invoice` ohne eigenes Schema Standard-Rechnungsdaten auslesen können.
- **Akzeptanzkriterien:**
  - **Given:** Ein PDF einer Rechnung und der Query-/Form-Parameter `profile=invoice`.
  - **When:** Die Extraktion erfolgt.
  - **Then:** Liefert das JSON mindestens: `invoice_number` (String), `date` (ISO 8601), `total_amount` (f64), `currency` (String), `vendor` (Name) und `line_items` (Array von Objekten).

### US-04: Benutzerdefinierte Schema-Vorgabe (Custom JSON-Schema) (Must-Have – MVP-Kern)
- **Story:** Als **Data-Entwickler** möchte ich im Request ein benutzerdefiniertes JSON-Schema mitsenden können, damit die API exakt die von mir spezifizierten Felder extrahiert.
- **Akzeptanzkriterien:**
  - **Given:** Ein valides PDF und ein gültiges JSON-Schema im Parameter `schema`.
  - **When:** Die Extraktion abgeschlossen wird.
  - **Then:** Stimmen die Schlüssel und Datentypen im resultierenden JSON exakt mit dem übergebenen Schema überein.
  - **And:** Bei Schema-Verletzungen antwortet die API mit HTTP 422 (Unprocessable Entity) und einer detaillierten Fehlerliste.

### US-05: Strikte Typprüfung & JSON-Garantie via Serde (Must-Have – MVP-Kern)
- **Story:** Als **API-Konsument** möchte ich die Garantie haben, dass jede erfolgreiche Serverantwort 100% syntaktisch valides JSON ist und keine Markdown-Codeblocks (z. B. ` ```json `) enthält.
- **Akzeptanzkriterien:**
  - **Given:** Die Rohantwort des KI-Modells.
  - **When:** Der Rust-Service die Daten vor dem Senden deserialisiert (`serde_json::from_str`) und re-serialisiert.
  - **Then:** Wird sichergestellt, dass fehlerhafte KI-Ausgaben vor der Client-Rückgabe abgefangen und repariert oder als Fehler deklariert werden.

### US-06: Hohe Performance & minimaler Memory-Footprint (Must-Have)
- **Story:** Als **DevOps-Ingenieur** möchte ich einen in Rust kompilierten Service betreiben, der minimalen Arbeitsspeicher benötigt und schnelle Antwortzeiten bietet.
- **Akzeptanzkriterien:**
  - **Given:** Ein laufender DocuPulse API Service in einem Linux-Container.
  - **When:** Der Service im Leerlauf läuft bzw. 50 synchrone Requests verarbeitet.
  - **Then:** Bleibt der RAM-Bedarf des Prozesses typischerweise unter 50 MB (ausgenommen temporäre PDF-Buffer).
  - **And:** Der Routing- und Preprocessing-Overhead von Rust (ohne KI-Netzwerklatenz) liegt unter 20 ms.

### US-07: Robuste Fehlerbehandlung für korrupte PDFs & falsche Formate (Must-Have)
- **Story:** Als **Entwickler** möchte ich bei ungültigen Eingaben (z. B. kaputte PDF-Bytes, Passwortschutz, Nicht-PDF-Dateien) sofort verständliche Fehlermeldungen erhalten.
- **Akzeptanzkriterien:**
  - **Given:** Ein Client sendet eine korrupte Datei oder ein geschütztes PDF.
  - **When:** Der Rust PDF-Parser einen Fehler wirft.
  - **Then:** Bricht die Anfrage sofort ab (kein teurer KI-Call) und liefert HTTP 400 mit JSON: `{"error": "INVALID_PDF", "message": "PDF header or xref table corrupted"}`.

### US-08: Zero Data Retention (DSGVO-Datenschutz) (Must-Have)
- **Story:** Als **Entwickler** möchte ich sicher sein, dass übermittelte PDF-Dateien nur im flüchtigen RAM verarbeitet und niemals unverschlüsselt auf einer Festplatte persistiert werden.
- **Akzeptanzkriterien:**
  - **Given:** Ein eingehender PDF-Request.
  - **When:** Die Extraktion beendet oder fehlgeschlagen ist.
  - **Then:** Werden alle Puffer im Rust-Arbeitsspeicher freigegeben (RAII-Prinzip / Drop-Semantik). Es verbleiben keine temporären Dateien im Dateisystem.

### US-09: Interaktive OpenAPI / Swagger-Dokumentation (Should-Have)
- **Story:** Als **Entwickler** möchte ich unter `/swagger-ui` eine interaktive Dokumentation aufrufen und Beispieldokumente direkt im Browser testen können.
- **Akzeptanzkriterien:**
  - **Given:** Ein Browser-Aufruf an `/swagger-ui`.
  - **When:** Die Seite lädt.
  - **Then:** Wird die vollständige OpenAPI 3.1 Dokumentation gerendert, generiert via Rust-Macros (z. B. `utoipa`).

### US-10: Asynchrone Job-Verarbeitung für mehrseitige PDFs (Should-Have)
- **Story:** Als **Entwickler** möchte ich umfangreichere PDFs (> 5 Seiten) über `POST /api/v1/jobs` asynchron einreichen und den Status per `GET /api/v1/jobs/{id}` pollen können.
- **Akzeptanzkriterien:**
  - **Given:** Ein PDF mit mehr als 5 Seiten.
  - **When:** Der Request an den Job-Endpunkt gesendet wird.
  - **Then:** Liefert die API HTTP 202 mit einer eindeutigen `job_id` (UUID v4) zurück.
  - **And:** Der Status wechselt von `queued` -> `processing` -> `completed`.

---

## 4. Detaillierte Anforderungen

### 4.1 Funktionale Anforderungen (FA)

| ID | Kategorie | Anforderungsbeschreibung | Priorität |
| :--- | :--- | :--- | :--- |
| **FA-01** | Ingestion | Entgegennahme von PDF-Dokumenten via HTTP `multipart/form-data` bis zu 15 MB Dateigröße. | Must |
| **FA-02** | Validation | Überprüfung der Datei-Magic-Bytes (`%PDF-`) im Rust-Handler vor Weiterverarbeitung. | Must |
| **FA-03** | PDF-Parsing | Auslesen von nativem Text, Metadaten und Seitenzahlen mit einer nativen Rust-Crate (`lopdf`/`pdf-extract`). | Must |
| **FA-04** | KI-Inferenz | Asynchrone Anbindung eines KI-Modells (z. B. Gemini Flash API / OpenAI-kompatibler Endpunkt) via `reqwest`. | Must |
| **FA-05** | Profil-Extraktion | Implementierung des festen Schemas `invoice` zur Extraktion von Rechnungsnummer, Datum, Summen, Währung, Posten. | Must |
| **FA-06** | Custom Schema | Dynamische Schema-Extraktion über ein vom Client mitgesendetes JSON-Schema. | Must |
| **FA-07** | JSON-Ausgabe | Ausgabe des Ergebnisses als strikt validiertes JSON über `serde_json` mit HTTP 200. | Must |
| **FA-08** | Error-Handling | Einheitliches Fehlerformat für Client-Fehler (400 Bad PDF, 413 Too Large, 422 Invalid Schema) als JSON. | Must |
| **FA-09** | Ephemeral RAM | Vollständiges Verwerfen von Dokumentendaten aus dem RAM nach Abschluss des Requests (Zero Disk Footprint). | Must |
| **FA-10** | Metrics | Rückgabe von Metadaten im Response: `page_count`, `processing_time_ms`, `extracted_fields_count`. | Should |
| **FA-11** | Async Queue | Asynchroner Verarbeitungs-Endpunkt `POST /api/v1/jobs` mit In-Memory / Tokio-MPSC-Queue für größere PDFs. | Should |
| **FA-12** | OpenAPI Docs | Automatisch generierte Swagger-UI (`utoipa`) unter `/swagger-ui`. | Should |
| **FA-13** | Grounding | Angabe der Seitenzahl (`page_number`) für extrahierte Entitäten im JSON-Output. | Should |
| **FA-14** | Webhook | Push-Benachrichtigung an eine Client-URL bei Fertigstellung eines asynchronen Jobs. | Could |

### 4.2 Nichtfunktionale Anforderungen (NFA)

| ID | Merkmal | Metrik / Vorgabe | Priorität |
| :--- | :--- | :--- | :--- |
| **NFA-01** | **Tech-Stack** | Implementierung der gesamten Serverlogik in **Rust** (Edition 2021+) mit `axum` und `tokio`. | Must |
| **NFA-02** | **Memory Footprint** | Maximal 50 MB RSS-Speicherverbrauch der Anwendung im Leerlauf; kein unkontrollierter RAM-Leak bei Last. | Must |
| **NFA-03** | **Pre-Processing Latenz** | Rust-seitige Vorverarbeitung (PDF Header Check & Text-Extraktion) in unter 50 ms für Dokumente bis 5 Seiten. | Must |
| **NFA-04** | **JSON-Striktheit** | 100 % Garantie für valides JSON. Keine Markdown-Artefakte oder syntaktische Fehler in 200er Antworten. | Must |
| **NFA-05** | **Sicherheit** | Keine Pufferüberläufe (Rust Memory Safety); keine Speicherung von Kundendaten auf persistenten Datenträgern. | Must |
| **NFA-06** | **Container-Größe** | Kompaktes Docker-Image (< 80 MB) basierend auf Alpine/Debian-Slim oder Distroless ohne Python-Abhängigkeiten. | Must |
| **NFA-07** | **Concurrency** | Fähigkeit, dank `tokio` bis zu 100 parallele synchrone Requests auf einem Single-Core-System ohne Deadlocks zu multiplexen. | Should |
| **NFA-08** | **Code-Qualität** | Compiler-Warnungsfreier Code (`cargo clippy -- -D warnings`), 100% formatierter Code (`cargo fmt`). | Should |

---

## 5. Priorisierung nach MoSCoW

```
+-----------------------------------------------------------------------------------+
| MUST HAVE (MVP-Scope: "PDF lesen -> JSON ausgeben")                               |
| - Axum/Tokio HTTP-Server in Rust                                                  |
| - POST /api/v1/extract (Multipart PDF Upload bis 15 MB)                           |
| - PDF-Validierung & nativer Text-/Metadaten-Extraktor (Rust-Crates)               |
| - KI-Anbindung (Async Reqwest an Multimodal/Text LLM API)                         |
| - Standard-Profil 'invoice' für Rechnungs-JSON                                     |
| - Benutzerdefiniertes Schema (Custom JSON-Schema Parameter)                       |
| - Strikte Serde-Validierung & Fehlerbehandlung (JSON-Antworten)                   |
| - Flüchtiger In-Memory Lebenszyklus (Zero Data Retention)                          |
+-----------------------------------------------------------------------------------+
| SHOULD HAVE (Erweiterung / Version 1.1)                                           |
| - Asynchrone Job-Queue (POST /jobs + GET /jobs/{id}) für mehrseitige Dokumente   |
| - OpenAPI / Swagger-UI via utoipa                                                 |
| - Seitenreferenzen (Grounding: page_number pro Feld)                              |
| - Detaillierte Performance-Metadaten (Tokens, Parsing-Dauer)                      |
+-----------------------------------------------------------------------------------+
| COULD HAVE (Ausblick)                                                             |
| - Webhook Callbacks für asynchrone Jobs                                           |
| - CLI-Binary-Modus: `docupulse extract sample.pdf --schema schema.json`           |
| - Lokale OCR-Fallback-Engine via Tesseract-Rust-Bindings                         |
+-----------------------------------------------------------------------------------+
| WON'T HAVE FOR MVP                                                                |
| - Benutzerverwaltung, Web-Frontend oder Datenbankpersistenz                       |
| - Eigene neuronale Netze trainieren                                               |
| - Unterstützung von Nicht-PDF-Formaten (DOCX, Excel)                              |
+-----------------------------------------------------------------------------------+
```

---

## 6. MVP-Scope Definition

### 6.1 Im MVP enthalten (In-Scope)
1. **Rust REST-Service:** Auf Basis von `axum` und `tokio`.
2. **Eingabe:** Ausschließlich **PDF-Dateien** (`application/pdf`).
3. **Ausgabe:** Ausschließlich wohlgeformtes, typsicheres **JSON** (`application/json`).
4. **Verarbeitungspipeline:**
   - PDF-Bytes im Speicher empfangen -> Magic-Bytes prüfen.
   - Textlayer extrahieren (falls vorhanden).
   - KI-Modell mit Prompt + Text/PDF-Payload aufrufen.
   - Antwort via `serde_json` prüfen und bereinigen.
   - JSON-Ergebnis an Client zurücksenden.
5. **Konfiguration:** `profile=invoice` oder dynamisches `schema={...}`.

### 6.2 Nicht im MVP enthalten (Explicitly Out-of-Scope)
- Keine Persistenz in relationalen Datenbanken (Postgres, MySQL).
- Kein Benutzer-Login / JWT-Auth (nur optionaler API-Key).
- Keine Bildformate (JPEG/PNG) im ersten Schritt (Fokus rein auf PDF).
- Kein Frontend (reine Developer-API für Curl / HTTP-Clients).

---

## 7. Widerspruchs- und Unklarheitenprüfung (Rust & PDF Spezifisch)

Die Prüfung auf technische Widersprüche und Fallstricke im Rust-Ökosystem ergab folgende Punkte:

### Konflikt 1: Native PDF-Text-Extraktion vs. Gescannte PDFs (Image-Only)
- **Problem:** Wenn das PDF ein Scan ohne OCR-Textlayer ist, liefert ein nativer Textparser (`pdf-extract`) einen leeren String zurück.
- **Auflösung:** Die Rust-Pipeline prüft die Länge des extrahierten Textes. Ist der Text leer oder < 50 Zeichen bei mehrseitigen Dokumenten, wird das PDF als Binärdatei direkt an die Multimodal Vision API des KI-Modells übergeben (Base64-kodiert), statt einen leeren Prompt zu senden.

### Konflikt 2: Blocking I/O beim PDF-Parsing in Tokios Async Runtime
- **Problem:** Viele PDF-Parsing-Bibliotheken in C oder Rust sind synchron und CPU-intensiv. Werden sie direkt in einem `async fn` Handler ausgeführt, blockieren sie die Tokio-Worker-Threads (Thread Starvation).
- **Auflösung:** CPU-lastiges PDF-Parsing wird in Tokio über `tokio::task::spawn_blocking` gekapselt, sodass die asynchrone I/O-Event-Loop für andere HTTP-Clients uneingeschränkt responsiv bleibt.

### Konflikt 3: Strikte Serde-Typisierung vs. dynamische Custom-Schemas
- **Problem:** Rust ist statisch typisiert. Ein vordefiniertes Schema wie `Invoice` lässt sich perfekt als `struct Invoice` definieren. Ein vom Nutzer zur Laufzeit übergebenes JSON-Schema hat jedoch keinen festen Rust-Typ zur Compile-Zeit.
- **Auflösung:** Duale Architektur in Serde:
  - Für Standardprofile (`invoice`) existiert ein stark typisiertes Rust-Struct (`#[derive(Serialize, Deserialize)]`).
  - Für Custom-Schemas wird `serde_json::Value` in Kombination mit der Crate `jsonschema` verwendet, um den dynamischen JSON-Output zur Laufzeit gegen das Schema des Clients zu validieren.
