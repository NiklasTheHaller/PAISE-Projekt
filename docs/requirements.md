# 📋 Produktspezifikation & Anforderungsanalyse: DocuPulse

**Projekt:** DocuPulse – Leichtgewichtige Document Processing API (PDF-zu-JSON)  
**Phase:** 1 – Produktspezifikation & Anforderungsanalyse  
**Primäre Zielgruppe:** Software-Entwickler & API-Integratoren  
**MVP-Kern:** Textbasierte PDFs einlesen, semantisch analysieren und als strikt validiertes JSON ausgeben  
**Stand:** November 2026  

---

## 1. Problemverständnis & Projektziel

### 1.1 Ausgangslage & Problemstellung
Software-Entwickler stehen in modernen Anwendungen regelmäßig vor der Aufgabe, unstrukturierte oder semi-strukturierte PDF-Dokumente (z. B. Rechnungen, Lieferscheine, Bescheinigungen, Berichte) in strukturierte JSON-Datensätze für Datenbanken, Microservices oder Web-Formulare umzuwandeln.

Bestehende Ansätze weisen gravierende Nachteile auf:
- **Klassische RegEx-/OCR-Pipelines:** Sind extrem fragil gegenüber Layout-Änderungen unterschiedlicher Aussteller und erfordern oft große, speicherhungrige Laufzeitumgebungen (> 1 GB Container-Images).
- **Große Enterprise-Cloud-Suiten (AWS Textract, Google Document AI):** Haben komplexe SDKs, intransparente Preismodelle und liefern oft tausende Zeilen lange Geometrie-/Bounding-Box-Daten statt direkt nutzbarer Fachobjekte.
- **Direkte LLM-Aufrufe durch den Entwickler:** Liefern probabilistischen Freitext oder in Markdown eingebettete Codeblöcke (` ```json ... ``` `), bieten keine harte Schema-Validierung vor der Weiterverarbeitung und verschwenden API-Tokens, wenn fehlerhafte oder leere PDFs ungeprüft hochgeladen werden.

### 1.2 Projektziel & Lösungsansatz
**DocuPulse** ist eine schlanke, entwicklerzentrierte REST-API nach dem Prinzip **„PDF rein → validiertes JSON raus“**:
1. Entwickler übergeben ein PDF-Dokument zusammen mit einem Standardprofil (`profile=invoice`) oder einem eigenen JSON-Schema (`schema={...}`).
2. DocuPulse prüft die Datei lokal im Arbeitsspeicher auf Gültigkeit und liest den Textlayer aus, **bevor** externe KI-Ressourcen beansprucht werden.
3. Ein angebundenes Sprachmodell (Google Gemini Flash) übernimmt die semantische Zuordnung der Textinhalte in die gewünschte Zielstruktur.
4. Der Server validiert die KI-Antwort strikt gegen das Ziel-Schema und liefert garantiert wohlgeformtes JSON zurück.

---

## 2. Zielgruppe & Kern-Anwendungsfälle

### 2.1 Zielgruppen-Personas (Software-Entwickler)

| Persona | Rolle & Kontext | Bedürfnisse & Erwartungen an DocuPulse |
| :--- | :--- | :--- |
| **Backend-Basti** | Backend-/Microservice-Entwickler (FinTech / ERP) | Will Rechnungen und Belege ohne eigenen PDF-Parser zuverlässig in typsichere JSON-Objekte umwandeln; erwartet eindeutige HTTP-Statuscodes und Schema-Treue. |
| **Frontend-Fabi** | Fullstack- / TypeScript-Entwickler | Baut ein Web-Dashboard, in dem Nutzer ein PDF hochladen, um Formularfelder automatisch vorzubefüllen; braucht schnelle synchrone Antworten (< 5 s) und ein eigenes JSON-Schema pro Formular. |
| **DevOps-Dana** | Platform- / Cloud-Engineer | Betreibt Container-Umgebungen; fordert einen zustandslosen Dienst ohne Datenbank-Abhängigkeit, mit minimalem RAM-Verbrauch (< 50 MB) und datenschutzkonformer In-Memory-Verarbeitung. |

### 2.2 Kern-Anwendungsfälle (Use Cases)
- **UC-01 – Standardisierte Rechnungsextraktion (`profile=invoice`):** Ein Entwickler sendet eine PDF-Rechnung an `POST /api/v1/extract` mit `profile=invoice` und erhält sofort ein standardisiertes JSON mit Rechnungsnummer, Datum, Lieferant, Währung, Netto-/Steuer-/Bruttobetrag und Rechnungspositionen (`line_items`).
- **UC-02 – Schema-gesteuerte Individualextraktion (`schema`):** Ein Entwickler übergibt neben dem PDF ein eigenes JSON-Schema (z. B. für ein Teilnahmezertifikat oder ein Bestellformular). DocuPulse extrahiert genau die definierten Pflicht- und Optionalfelder und prüft deren Datentypen.
- **UC-03 – Frühe Fehler- und Kostenabwehr (Fail-Fast):** Ein Client sendet eine beschädigte Datei, ein falsches Dateiformat oder ein ungültiges JSON-Schema. DocuPulse weist den Request lokal innerhalb weniger Millisekunden mit einer klaren JSON-Fehlermeldung ab, ohne externe KI-Tokens zu verbrauchen.

---

## 3. User Stories & Akzeptanzkriterien

Alle 10 User Stories sind aus Sicht der Zielgruppe formuliert, nach **MoSCoW** priorisiert (6 Must-Have, 3 Should-Have, 1 Could-Have) und mit überprüfbaren **Given-When-Then**-Akzeptanzkriterien versehen.

### Must-Have User Stories (MVP-Scope)

#### US-01: Synchroner PDF-Upload und JSON-Extraktion (`Must-Have`)
- **Story:** Als **Entwickler** möchte ich ein PDF-Dokument per `POST /api/v1/extract` (`multipart/form-data`) hochladen können, um direkt in derselben HTTP-Antwort die extrahierten Daten als valides JSON zu erhalten.
- **Akzeptanzkriterien:**
  - **Given:** Ein valides, textbasiertes PDF-Dokument (1–5 Seiten, Dateigröße ≤ 10 MB) und ein gültiger Parameter `profile` oder `schema`.
  - **When:** Der Client einen `POST`-Request an `/api/v1/extract` sendet.
  - **Then:** Antwortet die API mit `HTTP 200 OK` und dem Header `Content-Type: application/json`.
  - **And:** Der JSON-Body enthält die Felder `status` (`"success"`), `document` (`filename`, `pages`), `data` (extrahierte Nutzdaten) sowie `processing_time_ms` (Integer > 0).

#### US-02: Lokale PDF-Validierung & Textlayer-Extraktion (`Must-Have`)
- **Story:** Als **API-Betreiber** möchte ich, dass hochgeladene Dateien lokal auf gültige PDF-Struktur und vorhandenen Textlayer geprüft werden, bevor ein externer KI-Aufruf erfolgt, um unnötige API-Kosten und Latenzen zu vermeiden.
- **Akzeptanzkriterien:**
  - **Given:** Ein eingehender Datei-Upload an `/api/v1/extract`.
  - **When:** Der Server die Datei entgegennimmt.
  - **Then:** Prüft der Server lokal die PDF-Magic-Bytes (`%PDF-`), ermittelt die exakte Seitenzahl und extrahiert den enthaltenen Textlayer im Arbeitsspeicher.
  - **And:** Enthält ein PDF im MVP keinen auslesbaren Textlayer (z. B. reiner Bild-Scan mit < 20 Zeichen Text), bricht der Server ohne KI-Aufruf mit `HTTP 422` und `error_code: "EMPTY_TEXT_LAYER"` ab.

#### US-03: Vordefiniertes Extraktionsprofil für Rechnungen (`profile=invoice`) (`Must-Have`)
- **Story:** Als **Backend-Entwickler** möchte ich beim Upload lediglich `profile=invoice` angeben können, um ohne eigenes Schema sofort strukturierte Rechnungsdaten zu erhalten.
- **Akzeptanzkriterien:**
  - **Given:** Ein textbasiertes Rechnungs-PDF und der Form-Parameter `profile=invoice`.
  - **When:** Die Extraktion erfolgreich durchgeführt wird.
  - **Then:** Enthält das Objekt `data` exakt die typisierten Felder `invoice_number` (String), `date` (String im ISO-8601-Format `YYYY-MM-DD`), `vendor` (String), `currency` (3-stelliger ISO-Code, z. B. `"EUR"`), `total_amount` (Number), `tax_amount` (Number oder `null`) und `line_items` (Array aus Objekten mit `description`, `quantity`, `unit_price`, `total`).

#### US-04: Dynamische Extraktion mit benutzerdefiniertem JSON-Schema (`schema`) (`Must-Have`)
- **Story:** Als **Fullstack-Entwickler** möchte ich im Request ein eigenes JSON-Schema übergeben können, damit die API genau die für meinen Anwendungsfall benötigten Felder und Datentypen aus dem PDF extrahiert.
- **Akzeptanzkriterien:**
  - **Given:** Ein textbasiertes PDF und ein valides JSON-Schema (Draft 7 / 2020-12) im Form-Feld `schema`.
  - **When:** Die API das Dokument analysiert.
  - **Then:** Entspricht das zurückgegebene `data`-Objekt vollständig den im Schema definierten `properties`, Datentypen (`string`, `number`, `boolean`, `array`, `object`) und `required`-Feldern.
  - **And:** Ist das vom Client übergebene `schema` selbst kein valides JSON-Schema, antwortet die API sofort (ohne KI-Aufruf) mit `HTTP 400 Bad Request` und `error_code: "INVALID_SCHEMA"`.

#### US-05: Strikte JSON- und Schema-Garantie ohne Markdown-Artefakte (`Must-Have`)
- **Story:** Als **API-Konsument** möchte ich mich darauf verlassen können, dass jede `HTTP 200`-Antwort zu 100 % syntaktisch und strukturell valides JSON ist und niemals ungeparste LLM-Markdown-Blöcke (` ```json `) enthält.
- **Akzeptanzkriterien:**
  - **Given:** Eine Rohantwort des externen KI-Modells nach der Dokumentanalyse.
  - **When:** Der Server die Antwort vor dem Zurücksenden an den Client prüft.
  - **Then:** Wird die Antwort serverseitig deserialisiert und gegen die Zielstruktur (`Invoice`-Typ bzw. übergebenes `schema`) validiert.
  - **And:** Falls die KI-Antwort das Schema verletzt oder nicht als JSON geparst werden kann, liefert der Server niemals kaputtes JSON mit `HTTP 200`, sondern antwortet kontrolliert mit `HTTP 422 Unprocessable Entity` (`error_code: "SCHEMA_VALIDATION_FAILED"`).

#### US-06: Strukturierte Fehlerdiagnose bei ungültigen Eingaben (`Must-Have`)
- **Story:** Als **Entwickler** möchte ich bei fehlerhaften Requests (defektes PDF, Passwortschutz, Datei zu groß, fehlende Parameter) ein einheitliches JSON-Fehlerobjekt mit sprechendem Fehlercode erhalten, um Fehler im Client-Code sauber behandeln zu können.
- **Akzeptanzkriterien:**
  - **Given:** Ein Request mit einer Nicht-PDF-Datei (z. B. `.png` oder `.txt`), einem passwortgeschützten/korrupten PDF oder einer Datei > 10 MB.
  - **When:** Der Request von der API verarbeitet wird.
  - **Then:** Bricht die Verarbeitung vor dem KI-Aufruf ab und liefert `HTTP 400` (ungültiges PDF / Parameter) bzw. `HTTP 413` (Datei > 10 MB).
  - **And:** Der Response-Body folgt immer dem Schema `{"status": "error", "error_code": "<CODE>", "message": "<Detailbeschreibung>"}`.

---

### Should-Have & Could-Have User Stories (Erweiterungen nach dem MVP)

#### US-07: Multimodaler Vision-Fallback für gescannte PDFs (`Should-Have`)
- **Story:** Als **Entwickler** möchte ich auch gescannte PDFs ohne Textlayer verarbeiten können, indem die API bei leerem Textlayer automatisch auf eine visuelle/multimodale KI-Analyse umschaltet.
- **Akzeptanzkriterien:**
  - **Given:** Ein valides PDF, dessen lokaler Textlayer leer ist (reiner Bild-Scan).
  - **When:** Der Request mit aktiviertem Vision-Fallback verarbeitet wird.
  - **Then:** Übergibt der Server das Dokument als Base64-kodiertes PDF-Payload an die multimodale KI-Schnittstelle und liefert das extrahierte JSON mit dem zusätzlichen Metadaten-Feld `"extraction_mode": "vision"` zurück.

#### US-08: Interaktive OpenAPI- / Swagger-Dokumentation (`Should-Have`)
- **Story:** Als **Entwickler** möchte ich unter `GET /swagger-ui` eine interaktive OpenAPI-Dokumentation im Browser öffnen können, um Endpunkte, Schemas und Fehlercodes ohne zusätzliche Tools zu testen.
- **Akzeptanzkriterien:**
  - **Given:** Ein laufender DocuPulse-Server.
  - **When:** Ein Entwickler `/swagger-ui` im Browser oder `/api-docs/openapi.json` per HTTP aufruft.
  - **Then:** Wird eine valide OpenAPI-3.x-Spezifikation inklusive aller Request-/Response-Modelle ausgeliefert.

#### US-09: Asynchrone Job-Queue für große, mehrseitige PDFs (`Should-Have`)
- **Story:** Als **Backend-Entwickler** möchte ich größere PDF-Dokumente (> 5 Seiten) über `POST /api/v1/jobs` asynchron einreichen und das Ergebnis über `GET /api/v1/jobs/{job_id}` abrufen können, um HTTP-Timeouts zu vermeiden.
- **Akzeptanzkriterien:**
  - **Given:** Ein PDF-Dokument mit bis zu 25 Seiten.
  - **When:** Das Dokument an `POST /api/v1/jobs` gesendet wird.
  - **Then:** Antwortet die API innerhalb von < 100 ms mit `HTTP 202 Accepted` und einer eindeutigen `job_id` (UUID v4), über die der Status (`queued`, `processing`, `completed`, `failed`) abgefragt werden kann.

#### US-10: Quellen-Grounding & Webhook-Benachrichtigung (`Could-Have`)
- **Story:** Als **Entwickler** möchte ich optional zu jedem extrahierten Wert die Fundstelle (`page_number`) erhalten oder bei asynchronen Jobs per Webhook-URL über die Fertigstellung benachrichtigt werden.
- **Akzeptanzkriterien:**
  - **Given:** Ein Request mit dem optionalen Parameter `include_grounding=true` bzw. `webhook_url`.
  - **When:** Die Extraktion abgeschlossen ist.
  - **Then:** Enthält die Antwort die jeweilige Seitenzahl der Fundstelle bzw. sendet das Ergebnis per HTTP-POST an die angegebene Webhook-URL.

---

## 4. Funktionale und Nichtfunktionale Anforderungen

### 4.1 Funktionale Anforderungen (FA)

| ID | Bereich | Anforderung | Priorität | Zugehörige Story |
| :--- | :--- | :--- | :--- | :--- |
| **FA-01** | API-Endpunkt | Bereitstellung des synchronen Endpunkts `POST /api/v1/extract` (`multipart/form-data`) für PDF-Dateien bis 10 MB (max. 5 Seiten) sowie `GET /health`. | **Must** | US-01 |
| **FA-02** | Datei-Validierung | Lokale Prüfung des Datei-Headers auf gültige PDF-Magic-Bytes (`%PDF-`) sowie Prüfung auf Passwortschutz vor jeder KI-Interaktion. | **Must** | US-02, US-06 |
| **FA-03** | Text-Extraktion | Lokales Auslesen des PDF-Textlayers und der exakten Seitenzahl im Arbeitsspeicher. | **Must** | US-02 |
| **FA-04** | KI-Integration | Asynchrone Anbindung der externen Google Gemini API (`gemini-2.5-flash`) über HTTPS mit konfigurierbarem API-Key (`GEMINI_API_KEY`). | **Must** | US-01, US-03, US-04 |
| **FA-05** | Standard-Profil | Unterstützung des Parameters `profile=invoice` mit fest definiertem Ausgabeschema für Rechnungen. | **Must** | US-03 |
| **FA-06** | Custom Schema | Unterstützung des Parameters `schema` zur Übergabe eines benutzerdefinierten JSON-Schemas (Prüfung auf syntaktische Gültigkeit vor dem KI-Call). | **Must** | US-04 |
| **FA-07** | Output-Validierung | Serverseitige Deserialisierung und Schema-Validierung der KI-Antwort vor Rückgabe an den Client. | **Must** | US-05 |
| **FA-08** | Error-Handling | Einheitliches JSON-Fehlerschema (`status`, `error_code`, `message`) für alle Client- und Upstream-Fehler (`400`, `413`, `422`, `502`, `504`). | **Must** | US-06 |
| **FA-09** | In-Memory-Handling | Ausschließliche Verarbeitung der PDF-Bytes und extrahierten Texte im flüchtigen Arbeitsspeicher ohne Erstellung temporärer Dateien auf der Festplatte. | **Must** | US-01 |
| **FA-10** | Vision-Fallback | Automatischer Fallback auf multimodale PDF-/Bild-Übertragung bei PDFs ohne Textlayer. | **Should** | US-07 |
| **FA-11** | OpenAPI-Spec | Bereitstellung einer interaktiven Swagger-UI unter `/swagger-ui` und OpenAPI-JSON unter `/api-docs/openapi.json`. | **Should** | US-08 |
| **FA-12** | Async-Jobs | Endpunkte `POST /api/v1/jobs` und `GET /api/v1/jobs/{id}` für asynchrone Hintergrundverarbeitung. | **Should** | US-09 |
| **FA-13** | Grounding / Webhook | Ausgabe von Seitenzahlen pro Feld (`include_grounding=true`) und optionaler Webhook-Callback. | **Could** | US-10 |

### 4.2 Nichtfunktionale Anforderungen (NFA)

Alle nichtfunktionalen Anforderungen sind mit klaren, im Test überprüfbaren Zielwerten definiert:

| ID | Qualitätsmerkmal | Messbare Anforderung / Prüfmetrik | Priorität |
| :--- | :--- | :--- | :--- |
| **NFA-01** | **Pre-Processing-Latenz** | Die lokale Vorprüfung (Magic-Byte-Check, Validierung des Request-Schemas und lokale PDF-Textextraktion für 1–5 Seiten) darf **maximal 50 ms** dauern (exkl. Netzwerkzeit der externen KI-API). | **Must** |
| **NFA-02** | **Speichereffizienz (RAM)** | Der Serverprozess darf im Leerlauf (Idle) **maximal 50 MB RAM (RSS)** belegen und nach Abschluss von Requests keinen Speicher-Leak aufweisen. | **Must** |
| **NFA-03** | **100 % JSON-Konformität** | **100 % aller Antworten** der API (sowohl `2xx` als auch `4xx`/`5xx`) müssen valides `application/json` sein; `HTTP 200`-Antworten müssen zu 100 % dem angeforderten Ziel-Schema entsprechen. | **Must** |
| **NFA-04** | **Datenschutz (Zero Data Retention)** | **0 Bytes** an hochgeladenen PDF-Dokumenten oder extrahierten Nutzdaten dürfen nach Abschluss des HTTP-Requests auf dem Dateisystem oder in Logs verbleiben. | **Must** |
| **NFA-05** | **Fehlertoleranz & Timeouts** | Externe KI-Aufrufe müssen nach spätestens **15 Sekunden** mit einem kontrollierten Timeout (`HTTP 504`) abbrechen, ohne den Serverprozess oder andere parallele Requests zu blockieren. | **Must** |
| **NFA-06** | **Container-Leichtgewicht** | Das fertige Produktions-Container-Image der API soll **kleiner als 80 MB** sein, um schnelles Deployment in Cloud-/Container-Umgebungen zu ermöglichen. | **Should** |
| **NFA-07** | **Wartbarkeit & Testbarkeit** | Die externe KI-Schnittstelle muss so entkoppelt sein, dass automatisierte Integrationstests der gesamten Pipeline mittels Mock-Client in **unter 5 Sekunden** ohne Internetverbindung durchlaufen. | **Must** |

---

## 5. Priorisierung (MoSCoW-Übersicht)

Um den Fehler *„Zu viele Must-have-Anforderungen“* zu vermeiden, ist das MVP strikt auf den synchronen Kern-Durchstich für textbasierte PDFs begrenzt:

| Kategorie | Must-Have (MVP-Kern – verbindlich) | Should-Have (Ausbaustufe 1) | Could-Have (Ausblick) | Won't-Have (Ausgeschlossen) |
| :--- | :--- | :--- | :--- | :--- |
| **User Stories** | `US-01`, `US-02`, `US-03`, `US-04`, `US-05`, `US-06` (6 Stories) | `US-07`, `US-08`, `US-09` (3 Stories) | `US-10` (1 Story) | Eigene GUI, Login/Auth, DB-Speicherung |
| **Funktional** | `FA-01` bis `FA-09` (Synchroner Upload, lokaler PDF-Textparser, `invoice`-Profil, Custom Schema, Validierung, Error-Format) | `FA-10` (Vision-Fallback), `FA-11` (Swagger-UI), `FA-12` (Async-Jobs) | `FA-13` (Grounding & Webhooks) | DOCX/XLSX-Support, eigenes Modell-Training |
| **Nichtfunktional** | `NFA-01` bis `NFA-05`, `NFA-07` (< 50 ms lokales Parsing, < 50 MB RAM, 100 % JSON, Zero Disk, 15s Timeout, Mock-Tests) | `NFA-06` (Docker-Image < 80 MB) | Horizontales Auto-Scaling | Multi-Node-Cluster-Orchestrierung |

> 📌 Details zur genauen MVP-Abgrenzung sowie zu den **externen API-Abhängigkeiten und Kosten** befinden sich in **[docs/mvp.md](mvp.md)**.

---

## 6. Ableitung der Technologieentscheidung (Warum Rust?)

Gemäß gutem Requirements Engineering wurden zuerst die fachlichen und nichtfunktionalen Anforderungen (`NFA-01` bis `NFA-07`) definiert. Aus diesen Anforderungen leitet sich die Wahl von **Rust** (`axum`, `tokio`, `serde`) als Umsetzungstechnologie für Phase 2 ab:

1. **Erfüllung von `NFA-03` & `US-05` (Strikte JSON- & Schema-Garantie):** Das Rust-Crate-Ökosystem rund um `serde`, `serde_json` und `jsonschema` erzwingt Typsicherheit beim Deserialisieren und Serialisieren. Ungültige KI-Antworten können das Typsystem nicht unbemerkt passieren.
2. **Erfüllung von `NFA-01`, `NFA-02` & `NFA-06` (Ressourceneffizienz):** Ein kompilierter Rust-Webserver mit `axum` und `tokio` benötigt im Leerlauf nur wenige Megabyte RAM (< 20–30 MB), hat keine Garbage-Collection-Pausen und lässt sich als winziges Single-Binary-Container-Image (< 50 MB) ausliefern.
3. **Erfüllung von `NFA-04` (Zero Data Retention):** Durch Rusts Ownership- und RAII-Modell (automatisches `Drop` am Ende des Request-Scopes) werden alle PDF-Buffer im Arbeitsspeicher sofort nach Beendigung des HTTP-Requests deterministisch freigegeben.

---

## 7. Prüfung auf Widersprüche, Lücken und Unklarheiten

Im Rahmen der Anforderungsvalidierung wurden folgende potenzielle Widersprüche identifiziert und aufgelöst:

### Konflikt 1: Statische Typsicherheit (`US-03`) vs. dynamische Custom-Schemas zur Laufzeit (`US-04`)
- **Identifizierter Widerspruch:** `US-03` (`profile=invoice`) verlangt ein fest definiertes, kompilierbares Datenmodell, während `US-04` (`schema={...}`) beliebige, erst zur Laufzeit bekannte JSON-Strukturen vom Entwickler akzeptiert. Zudem entsteht eine Unklarheit, was passiert, wenn ein Client **beide** Parameter (`profile` und `schema`) gleichzeitig oder **keinen** von beiden sendet.
- **Auflösung:**
  1. **Exklusivitätsregel (XOR):** Ein Request an `POST /api/v1/extract` muss **genau einen** der beiden Parameter (`profile` **oder** `schema`) enthalten. Werden beide oder keiner übergeben, antwortet die API sofort mit `HTTP 400 Bad Request` (`error_code: "INVALID_PARAMETERS"`).
  2. **Duale Validierung:** Bei `profile=invoice` wird gegen ein festes `Invoice`-Struct validiert; bei `schema` wird die KI-Antwort als dynamisches `serde_json::Value` geparst und mittels JSON-Schema-Validator (`jsonschema`) gegen das übergebene Client-Schema geprüft.

### Konflikt 2: Lokale PDF-Textextraktion (`US-02`) vs. gescannte PDFs ohne Textlayer (`US-07`)
- **Identifizierter Widerspruch:** Wenn ein Entwickler ein gescanntes PDF (reines Bild im PDF-Container) hochlädt, liefert der lokale PDF-Textparser (`FA-03`) einen leeren String zurück. Würde dieser leere String an die KI gesendet, würde das Modell halluzinieren oder fehlschlagen.
- **Auflösung:** Im **MVP (`US-02`, Must-Have)** prüft der Server nach der Textextraktion die Zeichenanzahl. Liegt diese unter dem Schwellenwert (< 20 druckbare Zeichen), lehnt das MVP den Request deterministisch mit `HTTP 422` (`error_code: "EMPTY_TEXT_LAYER"`) ab, ohne die KI aufzurufen. Erst in der **Erweiterungsstufe (`US-07`, Should-Have)** wird dieser Pfad an den multimodalen Vision-Fallback weitergeleitet.

### Konflikt 3: CPU-synchrones PDF-Parsing (`NFA-01`) vs. nicht-blockierender Async-Server (`NFA-05`)
- **Identifizierter Widerspruch:** Das Auslesen und Dekomprimieren von PDF-Streams im Speicher ist eine synchrone, CPU-gebundene Operation. In einer asynchronen Server-Laufzeitumgebung würde direktes Parsing im Request-Handler die Event-Loop für andere Anfragen blockieren.
- **Auflösung:** Das lokale PDF-Parsing wird vom asynchronen Netzwerk-I/O entkoppelt (in Tokio über `spawn_blocking` auf einem dedizierten Thread-Pool ausgeführt), sodass Health-Checks und parallele Requests jederzeit blockierungsfrei bleiben.
