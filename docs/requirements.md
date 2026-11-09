# 📋 Produktspezifikation & Anforderungsanalyse: DocuPulse

**Projekt:** DocuPulse – Leichtgewichtige Document Processing API (PDF-zu-JSON)  
**Phase:** 1 – Produktspezifikation & Anforderungsanalyse  
**Primäre Zielgruppe:** Software-Entwickler & API-Integratoren  
**MVP-Kern:** Textbasierte PDFs einlesen, über eine zweistufige Hybrid-Pipeline (**lokale semantische Informationsextraktion + externer KI-Fallback**) analysieren und als strikt validiertes JSON ausgeben  
**Stand:** November 2026  

---

## 1. Problemverständnis & Projektziel

### 1.1 Ausgangslage & Problemstellung
Software-Entwickler stehen in modernen Anwendungen regelmäßig vor der Aufgabe, unstrukturierte oder semi-strukturierte PDF-Dokumente (z. B. Rechnungen, Lieferscheine, Bescheinigungen, Berichte) in strukturierte JSON-Datensätze für Datenbanken, Microservices oder Web-Formulare umzuwandeln.

Bestehende Ansätze weisen gravierende Nachteile auf:
- **Reine RegEx-/OCR-Monolithen:** Sind oft starr programmiert oder erfordern schwere Laufzeitumgebungen (> 1 GB Container-Images mit Python/Tesseract-Abhängigkeiten).
- **Große Enterprise-Cloud-Suiten (AWS Textract, Google Document AI):** Haben komplexe SDKs, intransparente Preismodelle und liefern oft tausende Zeilen lange Geometrie-/Bounding-Box-Daten statt direkt nutzbarer Fachobjekte.
- **Reine LLM-Wrapper (Direkte Cloud-KI-Aufrufe für jedes Dokument):** Erzeugen bei jedem einzelnen Dokument hohe Netzwerk-Latenzen (1–4 Sekunden), laufende Token-Kosten und externe Cloud-Abhängigkeiten – selbst bei Standarddokumenten (wie typischen Rechnungen), deren semantische Kerninformationen eigentlich deterministisch und lokal in wenigen Millisekunden extrahierbar wären. Zudem liefern ungeschützte LLM-Aufrufe oft Markdown-Artefakte (` ```json ... ``` `) ohne harte Schema-Garantie.

### 1.2 Projektziel & Lösungsansatz
**DocuPulse** ist eine schlanke, entwicklerzentrierte REST-API nach dem Prinzip **„PDF rein → validiertes JSON raus“** mit einer **Local-First Hybrid-Architektur**:
1. Entwickler übergeben ein PDF-Dokument zusammen mit einem Standardprofil (`profile=invoice`) oder einem eigenen JSON-Schema (`schema={...}`).
2. **Lokale Vorprüfung & Textlayer-Extraktion:** DocuPulse prüft die Datei lokal im Arbeitsspeicher auf Gültigkeit und liest den Textlayer sowie Strukturmerkmale aus.
3. **Stufe 1 – Lokale semantische Informationsextraktion (In-Memory):** Eine leichtgewichtige lokale Extraktions-Engine analysiert den Text kontext- und musterorientiert (Schlüssel-Wert-Nachbarschaften, Synonymerkennung, Datums-/Währungs-/Betragsnormalisierung sowie Positions-/Tabellenheuristiken). Kann das angeforderte Schema lokal vollständig aufgelöst werden, liefert DocuPulse das Ergebnis **sofort lokal in < 50 ms und ohne externe API-Kosten** zurück (`extraction_source: "local"`).
4. **Stufe 2 – Externer KI-Fallback (Google Gemini Flash):** Nur wenn die lokale semantische Extraktion bei komplexen Custom-Schemas oder unkonventionellen Freitext-Layouts nicht alle Pflichtfelder sicher auflösen kann, schaltet DocuPulse nahtlos das angebundene Sprachmodell ein (`extraction_source: "cloud_ai"`), wobei lokal vorab erkannte Entitäten als strukturierter Kontext mitgegeben werden.
5. **Strikte End-to-End-Validierung:** Unabhängig davon, ob die Daten lokal oder über den KI-Fallback gewonnen wurden, validiert der Server das Ergebnis strikt gegen das Ziel-Schema und liefert garantiert wohlgeformtes JSON zurück.

---

## 2. Zielgruppe & Kern-Anwendungsfälle

### 2.1 Zielgruppen-Personas (Software-Entwickler)

| Persona | Rolle & Kontext | Bedürfnisse & Erwartungen an DocuPulse |
| :--- | :--- | :--- |
| **Backend-Basti** | Backend-/Microservice-Entwickler (FinTech / ERP) | Will Rechnungen und Belege ohne eigenen PDF-Parser zuverlässig in typsichere JSON-Objekte umwandeln; profitiert davon, dass Standardrechnungen durch die lokale semantische Extraktion in < 50 ms und ohne externe Token-Kosten verarbeitet werden. |
| **Frontend-Fabi** | Fullstack- / TypeScript-Entwickler | Baut ein Web-Dashboard, in dem Nutzer ein PDF hochladen, um Formularfelder automatisch vorzubefüllen; braucht schnelle synchrone Antworten und für individuelle Formulare ein flexibles JSON-Schema mit KI-Fallback. |
| **DevOps-Dana** | Platform- / Cloud-Engineer | Betreibt Container-Umgebungen; fordert einen zustandslosen Dienst ohne Datenbank-Abhängigkeit, mit minimalem RAM-Verbrauch (< 50 MB), lokaler Verarbeitung ohne schwere ML-Runtimes und datenschutzkonformer In-Memory-Verarbeitung. |

### 2.2 Kern-Anwendungsfälle (Use Cases)
- **UC-01 – Lokale & hybride Rechnungsextraktion (`profile=invoice`):** Ein Entwickler sendet eine PDF-Rechnung an `POST /api/v1/extract` mit `profile=invoice`. Die lokale semantische Extraktions-Engine erkennt Rechnungsnummer, Datum, Lieferant, Währung, Netto-/Steuer-/Bruttobetrag und Rechnungspositionen (`line_items`) direkt im Speicher; nur bei unvollständigen/ungewöhnlichen Layouts greift automatisch der KI-Fallback.
- **UC-02 – Schema-gesteuerte Individualextraktion (`schema`):** Ein Entwickler übergibt neben dem PDF ein eigenes JSON-Schema (z. B. für ein Teilnahmezertifikat oder Bestellformular). DocuPulse versucht zunächst ein lokales Schlüssel-Wert-Matching anhand der Schema-Properties und nutzt bei komplexen semantischen Anforderungen nahtlos das externe Sprachmodell.
- **UC-03 – Frühe Fehler- und Kostenabwehr (Fail-Fast):** Ein Client sendet eine beschädigte Datei, ein falsches Dateiformat oder ein ungültiges JSON-Schema. DocuPulse weist den Request lokal innerhalb weniger Millisekunden mit einer klaren JSON-Fehlermeldung ab.

---

## 3. User Stories & Akzeptanzkriterien

Alle 11 User Stories sind aus Sicht der Zielgruppe formuliert, nach **MoSCoW** priorisiert (7 Must-Have, 3 Should-Have, 1 Could-Have) und mit überprüfbaren **Given-When-Then**-Akzeptanzkriterien versehen.

### Must-Have User Stories (MVP-Scope)

#### US-01: Synchroner PDF-Upload und JSON-Extraktion (`Must-Have`)
- **Story:** Als **Entwickler** möchte ich ein PDF-Dokument per `POST /api/v1/extract` (`multipart/form-data`) hochladen können, um direkt in derselben HTTP-Antwort die extrahierten Daten als valides JSON zu erhalten.
- **Akzeptanzkriterien:**
  - **Given:** Ein valides, textbasiertes PDF-Dokument (1–5 Seiten, Dateigröße ≤ 10 MB) und ein gültiger Parameter `profile` oder `schema`.
  - **When:** Der Client einen `POST`-Request an `/api/v1/extract` sendet.
  - **Then:** Antwortet die API mit `HTTP 200 OK` und dem Header `Content-Type: application/json`.
  - **And:** Der JSON-Body enthält die Felder `status` (`"success"`), `extraction_source` (`"local"` oder `"cloud_ai"`), `document` (`filename`, `pages`), `data` (extrahierte Nutzdaten) sowie `processing_time_ms` (Integer > 0).

#### US-02: Lokale PDF-Validierung & Textlayer-Extraktion (`Must-Have`)
- **Story:** Als **API-Betreiber** möchte ich, dass hochgeladene Dateien lokal auf gültige PDF-Struktur und vorhandenen Textlayer geprüft werden, bevor nachgelagerte Analyseschritte oder externe KI-Aufrufe erfolgen, um unnötige API-Kosten und Latenzen zu vermeiden.
- **Akzeptanzkriterien:**
  - **Given:** Ein eingehender Datei-Upload an `/api/v1/extract`.
  - **When:** Der Server die Datei entgegennimmt.
  - **Then:** Prüft der Server lokal die PDF-Magic-Bytes (`%PDF-`), ermittelt die exakte Seitenzahl und extrahiert den enthaltenen Textlayer im Arbeitsspeicher.
  - **And:** Enthält ein PDF im MVP keinen auslesbaren Textlayer (z. B. reiner Bild-Scan mit < 20 Zeichen Text), bricht der Server sofort mit `HTTP 422` und `error_code: "EMPTY_TEXT_LAYER"` ab.

#### US-03: Lokale semantische Informationsextraktion & Hybrid-Fallback (`Must-Have`)
- **Story:** Als **Backend-Entwickler** möchte ich, dass die API strukturierte Informationen (wie Schlüssel-Wert-Paare, Beträge, Datumsangaben, Währungen und Tabellenpositionen) primär lokal im Arbeitsspeicher semantisch extrahiert und nur bei unvollständigen Ergebnissen oder komplexen Schemas auf die externe KI zurückgreift, um minimale Latenzen (< 50 ms) und 0 € API-Kosten bei Standarddokumenten zu erreichen.
- **Akzeptanzkriterien:**
  - **Given:** Ein valides, textbasiertes PDF-Dokument mit extrahiertem Textlayer.
  - **When:** Die semantische Extraktions-Pipeline gestartet wird.
  - **Then:** Führt der Server zuerst die lokale semantische Informationsextraktion (Kontext-/Label-Matching, Datums-/Betragsnormalisierung, Positionsheuristik) im Arbeitsspeicher aus.
  - **And:** Werden alle Pflichtfelder des angeforderten Ziel-Schemas (`profile=invoice` oder `schema`) bereits lokal valide erkannt, antwortet die API direkt ohne externen Netzwerkaufruf mit `"extraction_source": "local"`.
  - **And:** Können lokal nicht alle Pflichtfelder sicher aufgelöst werden, delegiert der Server die Anfrage automatisch (inklusive lokal vorab erkannter Entitäten) an die externe KI und kennzeichnet die Antwort mit `"extraction_source": "cloud_ai"`.

#### US-04: Vordefiniertes Extraktionsprofil für Rechnungen (`profile=invoice`) (`Must-Have`)
- **Story:** Als **Backend-Entwickler** möchte ich beim Upload lediglich `profile=invoice` angeben können, um ohne eigenes Schema sofort strukturierte Rechnungsdaten zu erhalten.
- **Akzeptanzkriterien:**
  - **Given:** Ein textbasiertes Rechnungs-PDF und der Form-Parameter `profile=invoice`.
  - **When:** Die Extraktion (lokal oder hybrid) erfolgreich durchgeführt wird.
  - **Then:** Enthält das Objekt `data` exakt die typisierten Felder `invoice_number` (String), `date` (String im ISO-8601-Format `YYYY-MM-DD`), `vendor` (String), `currency` (3-stelliger ISO-Code, z. B. `"EUR"`), `total_amount` (Number), `tax_amount` (Number oder `null`) und `line_items` (Array aus Objekten mit `description`, `quantity`, `unit_price`, `total`).

#### US-05: Dynamische Extraktion mit benutzerdefiniertem JSON-Schema (`schema`) (`Must-Have`)
- **Story:** Als **Fullstack-Entwickler** möchte ich im Request ein eigenes JSON-Schema übergeben können, damit die API genau die für meinen Anwendungsfall benötigten Felder und Datentypen aus dem PDF extrahiert.
- **Akzeptanzkriterien:**
  - **Given:** Ein textbasiertes PDF und ein valides JSON-Schema (Draft 7 / 2020-12) im Form-Feld `schema`.
  - **When:** Die API das Dokument über die lokale Schlüssel-Wert-Analyse bzw. den KI-Fallback analysiert.
  - **Then:** Entspricht das zurückgegebene `data`-Objekt vollständig den im Schema definierten `properties`, Datentypen (`string`, `number`, `boolean`, `array`, `object`) und `required`-Feldern.
  - **And:** Ist das vom Client übergebene `schema` selbst kein valides JSON-Schema, antwortet die API sofort mit `HTTP 400 Bad Request` und `error_code: "INVALID_SCHEMA"`.

#### US-06: Strikte JSON- und Schema-Garantie ohne Markdown-Artefakte (`Must-Have`)
- **Story:** Als **API-Konsument** möchte ich mich darauf verlassen können, dass jede `HTTP 200`-Antwort – egal ob lokal oder per KI-Fallback extrahiert – zu 100 % syntaktisch und strukturell valides JSON ist und niemals ungeparste LLM-Markdown-Blöcke (` ```json `) enthält.
- **Akzeptanzkriterien:**
  - **Given:** Ein Extraktionsergebnis aus der lokalen Engine oder dem externen KI-Modell.
  - **When:** Der Server die Antwort vor dem Zurücksenden an den Client prüft.
  - **Then:** Wird die Antwort serverseitig deserialisiert und gegen die Zielstruktur (`Invoice`-Typ bzw. übergebenes `schema`) validiert.
  - **And:** Falls nach lokaler Analyse und KI-Fallback das Ziel-Schema weiterhin verletzt wird oder die Antwort nicht als JSON geparst werden kann, liefert der Server niemals kaputtes JSON mit `HTTP 200`, sondern antwortet kontrolliert mit `HTTP 422 Unprocessable Entity` (`error_code: "SCHEMA_VALIDATION_FAILED"`).

#### US-07: Strukturierte Fehlerdiagnose bei ungültigen Eingaben (`Must-Have`)
- **Story:** Als **Entwickler** möchte ich bei fehlerhaften Requests (defektes PDF, Passwortschutz, Datei zu groß, fehlende Parameter) ein einheitliches JSON-Fehlerobjekt mit sprechendem Fehlercode erhalten, um Fehler im Client-Code sauber behandeln zu können.
- **Akzeptanzkriterien:**
  - **Given:** Ein Request mit einer Nicht-PDF-Datei (z. B. `.png` oder `.txt`), einem passwortgeschützten/korrupten PDF oder einer Datei > 10 MB.
  - **When:** Der Request von der API verarbeitet wird.
  - **Then:** Bricht die Verarbeitung sofort lokal ab und liefert `HTTP 400` (ungültiges PDF / Parameter) bzw. `HTTP 413` (Datei > 10 MB).
  - **And:** Der Response-Body folgt immer dem Schema `{"status": "error", "error_code": "<CODE>", "message": "<Detailbeschreibung>"}`.

---

### Should-Have & Could-Have User Stories (Erweiterungen nach dem MVP)

#### US-08: Multimodaler Vision-Fallback für gescannte PDFs (`Should-Have`)
- **Story:** Als **Entwickler** möchte ich auch gescannte PDFs ohne Textlayer verarbeiten können, indem die API bei leerem Textlayer automatisch auf eine visuelle/multimodale KI-Analyse umschaltet.
- **Akzeptanzkriterien:**
  - **Given:** Ein valides PDF, dessen lokaler Textlayer leer ist (reiner Bild-Scan).
  - **When:** Der Request mit aktiviertem Vision-Fallback verarbeitet wird.
  - **Then:** Übergibt der Server das Dokument als Base64-kodiertes PDF-Payload an die multimodale KI-Schnittstelle und liefert das extrahierte JSON mit dem zusätzlichen Metadaten-Feld `"extraction_mode": "vision"` zurück.

#### US-09: Interaktive OpenAPI- / Swagger-Dokumentation (`Should-Have`)
- **Story:** Als **Entwickler** möchte ich unter `GET /swagger-ui` eine interaktive OpenAPI-Dokumentation im Browser öffnen können, um Endpunkte, Schemas und Fehlercodes ohne zusätzliche Tools zu testen.
- **Akzeptanzkriterien:**
  - **Given:** Ein laufender DocuPulse-Server.
  - **When:** Ein Entwickler `/swagger-ui` im Browser oder `/api-docs/openapi.json` per HTTP aufruft.
  - **Then:** Wird eine valide OpenAPI-3.x-Spezifikation inklusive aller Request-/Response-Modelle ausgeliefert.

#### US-10: Asynchrone Job-Queue für große, mehrseitige PDFs (`Should-Have`)
- **Story:** Als **Backend-Entwickler** möchte ich größere PDF-Dokumente (> 5 Seiten) über `POST /api/v1/jobs` asynchron einreichen und das Ergebnis über `GET /api/v1/jobs/{job_id}` abrufen können, um HTTP-Timeouts zu vermeiden.
- **Akzeptanzkriterien:**
  - **Given:** Ein PDF-Dokument mit bis zu 25 Seiten.
  - **When:** Das Dokument an `POST /api/v1/jobs` gesendet wird.
  - **Then:** Antwortet die API innerhalb von < 100 ms mit `HTTP 202 Accepted` und einer eindeutigen `job_id` (UUID v4), über die der Status (`queued`, `processing`, `completed`, `failed`) abgefragt werden kann.

#### US-11: Quellen-Grounding & Webhook-Benachrichtigung (`Could-Have`)
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
| **FA-02** | Datei-Validierung | Lokale Prüfung des Datei-Headers auf gültige PDF-Magic-Bytes (`%PDF-`) sowie Prüfung auf Passwortschutz vor jeder weiteren Analyse. | **Must** | US-02, US-07 |
| **FA-03** | Text-Extraktion | Lokales Auslesen des PDF-Textlayers und der exakten Seitenzahl im Arbeitsspeicher. | **Must** | US-02 |
| **FA-04** | Lokale semantische Extraktion | Lokale kontext- und heuristikbasierte Informationsextraktion im Arbeitsspeicher (Schlüssel-Wert-Nachbarschaftsanalyse, Datums-/Währungs-/Betragsnormalisierung sowie Positions-/Tabellenerkennung ohne externen Netzwerkaufruf). | **Must** | US-03, US-04, US-05 |
| **FA-05** | Hybrid-Router & KI-Fallback | Automatische Prüfung der lokalen Extraktionsergebnisse auf Vollständigkeit gegenüber den Pflichtfeldern des Ziel-Schemas; nur bei unvollständiger lokaler Auflösung erfolgt ein asynchroner Fallback-Aufruf an die externe Google Gemini API (`gemini-2.5-flash`, `GEMINI_API_KEY`). | **Must** | US-01, US-03, US-04, US-05 |
| **FA-06** | Standard-Profil | Unterstützung des Parameters `profile=invoice` mit fest definiertem Ausgabeschema für Rechnungen (primär lokal extrahiert, optionaler KI-Fallback). | **Must** | US-03, US-04 |
| **FA-07** | Custom Schema | Unterstützung des Parameters `schema` zur Übergabe eines benutzerdefinierten JSON-Schemas (Prüfung auf syntaktische Gültigkeit vor der Extraktion; lokales Property-Matching mit KI-Fallback). | **Must** | US-03, US-05 |
| **FA-08** | Output-Validierung | Serverseitige Deserialisierung und Schema-Validierung des Ergebnisses (sowohl aus lokaler Extraktion als auch aus dem KI-Fallback) sowie Ausweisung von `extraction_source` (`"local"` \| `"cloud_ai"`). | **Must** | US-03, US-06 |
| **FA-09** | Error-Handling | Einheitliches JSON-Fehlerschema (`status`, `error_code`, `message`) für alle Client- und Upstream-Fehler (`400`, `413`, `422`, `502`, `504`). | **Must** | US-07 |
| **FA-10** | In-Memory-Handling | Ausschließliche Verarbeitung der PDF-Bytes und extrahierten Texte im flüchtigen Arbeitsspeicher ohne Erstellung temporärer Dateien auf der Festplatte. | **Must** | US-01 |
| **FA-11** | Vision-Fallback | Automatischer Fallback auf multimodale PDF-/Bild-Übertragung bei PDFs ohne Textlayer. | **Should** | US-08 |
| **FA-12** | OpenAPI-Spec | Bereitstellung einer interaktiven Swagger-UI unter `/swagger-ui` und OpenAPI-JSON unter `/api-docs/openapi.json`. | **Should** | US-09 |
| **FA-13** | Async-Jobs | Endpunkte `POST /api/v1/jobs` und `GET /api/v1/jobs/{id}` für asynchrone Hintergrundverarbeitung. | **Should** | US-10 |
| **FA-14** | Grounding / Webhook | Ausgabe von Seitenzahlen pro Feld (`include_grounding=true`) und optionaler Webhook-Callback. | **Could** | US-11 |

### 4.2 Nichtfunktionale Anforderungen (NFA)

Alle nichtfunktionalen Anforderungen sind mit klaren, im Test überprüfbaren Zielwerten definiert:

| ID | Qualitätsmerkmal | Messbare Anforderung / Prüfmetrik | Priorität |
| :--- | :--- | :--- | :--- |
| **NFA-01** | **Lokale Pipeline-Latenz** | Die gesamte lokale Verarbeitung (Magic-Byte-Check, PDF-Textextraktion **und lokale semantische Informationsextraktion** für 1–5 Seiten) darf **maximal 50 ms** dauern. Wird das Dokument vollständig lokal gelöst (`extraction_source: "local"`), beträgt die Gesamt-Antwortzeit somit **≤ 50 ms**. | **Must** |
| **NFA-02** | **Speichereffizienz (RAM)** | Der Serverprozess (inklusive der lokalen semantischen Extraktions-Engine) darf im Leerlauf (Idle) **maximal 50 MB RAM (RSS)** belegen und nach Abschluss von Requests keinen Speicher-Leak aufweisen. | **Must** |
| **NFA-03** | **100 % JSON-Konformität** | **100 % aller Antworten** der API (sowohl `2xx` als auch `4xx`/`5xx`) müssen valides `application/json` sein; `HTTP 200`-Antworten müssen zu 100 % dem angeforderten Ziel-Schema entsprechen. | **Must** |
| **NFA-04** | **Datenschutz (Zero Data Retention)** | **0 Bytes** an hochgeladenen PDF-Dokumenten oder extrahierten Nutzdaten dürfen nach Abschluss des HTTP-Requests auf dem Dateisystem oder in Logs verbleiben; bei erfolgreicher lokaler Extraktion verlassen **0 Bytes** den lokalen Server. | **Must** |
| **NFA-05** | **Fehlertoleranz & Timeouts** | Externe KI-Fallback-Aufrufe müssen nach spätestens **15 Sekunden** mit einem kontrollierten Timeout (`HTTP 504`) abbrechen, ohne den Serverprozess oder andere parallele Requests zu blockieren. | **Must** |
| **NFA-06** | **Container-Leichtgewicht** | Das fertige Produktions-Container-Image der API soll (trotz integrierter lokaler semantischer Extraktion) **kleiner als 80 MB** sein, um schnelles Deployment in Cloud-/Container-Umgebungen zu ermöglichen. | **Should** |
| **NFA-07** | **Offline-Fähigkeit & Testbarkeit** | Die lokale semantische Extraktion muss komplett **offline ohne API-Key** lauffähig sein; die optionale externe KI-Schnittstelle muss so entkoppelt sein, dass automatisierte Integrationstests der gesamten Hybrid-Pipeline in **unter 5 Sekunden** ohne Internetverbindung durchlaufen. | **Must** |

---

## 5. Priorisierung (MoSCoW-Übersicht)

Um den Fehler *„Zu viele Must-have-Anforderungen“* zu vermeiden, ist das MVP strikt auf den synchronen Kern-Durchstich für textbasierte PDFs mit der zweistufigen Hybrid-Extraktion begrenzt:

| Kategorie | Must-Have (MVP-Kern – verbindlich) | Should-Have (Ausbaustufe 1) | Could-Have (Ausblick) | Won't-Have (Ausgeschlossen) |
| :--- | :--- | :--- | :--- | :--- |
| **User Stories** | `US-01` bis `US-07` (7 Stories: Upload, PDF-Check, Lokale semantische Extraktion & Hybrid-Fallback, `invoice`, Custom Schema, Validierung, Error-Format) | `US-08`, `US-09`, `US-10` (3 Stories) | `US-11` (1 Story) | Eigene GUI, Login/Auth, DB-Speicherung |
| **Funktional** | `FA-01` bis `FA-10` (Synchroner Upload, lokaler PDF-Textparser, lokale semantische Extraktion, Hybrid-Router & KI-Fallback, `invoice`-Profil, Custom Schema, Validierung, Error-Format) | `FA-11` (Vision-Fallback), `FA-12` (Swagger-UI), `FA-13` (Async-Jobs) | `FA-14` (Grounding & Webhooks) | DOCX/XLSX-Support, eigenes Modell-Training |
| **Nichtfunktional** | `NFA-01` bis `NFA-05`, `NFA-07` (< 50 ms lokale Extraktion, < 50 MB RAM, 100 % JSON, Zero Disk, 15s Timeout, Offline-/Mock-Tests) | `NFA-06` (Docker-Image < 80 MB) | Horizontales Auto-Scaling | Multi-Node-Cluster-Orchestrierung |

> 📌 Details zur genauen MVP-Abgrenzung sowie zur **Hybrid-Architektur, externen API-Abhängigkeiten und Kosten** befinden sich in **[docs/mvp.md](mvp.md)**.

---

## 6. Ableitung der Technologieentscheidung (Warum Rust?)

Gemäß gutem Requirements Engineering wurden zuerst die fachlichen und nichtfunktionalen Anforderungen (`NFA-01` bis `NFA-07`) definiert. Aus diesen Anforderungen leitet sich die Wahl von **Rust** (`axum`, `tokio`, `serde`, `regex`) als Umsetzungstechnologie für Phase 2 ab:

1. **Erfüllung von `NFA-01`, `NFA-02` & `FA-04` (Ultraschnelle lokale semantische Extraktion ohne ML-Bloat):** Durch eine in Rust implementierte kontextuelle und heuristische Extraktions-Engine (DFA-basierte `regex`-Automaten, Token-Nachbarschaftsanalyse und deterministische Normalisierer) läuft die gesamte lokale semantische Analyse in wenigen Millisekunden und benötigt im Gegensatz zu Python-/NLP-Containern keine hunderte Megabyte großen Laufzeitumgebungen.
2. **Erfüllung von `NFA-03` & `US-06` (Strikte JSON- & Schema-Garantie):** Das Rust-Crate-Ökosystem rund um `serde`, `serde_json` und `jsonschema` erzwingt Typsicherheit beim Deserialisieren und Serialisieren – sowohl für die Ergebnisse der lokalen Extraktion als auch für den KI-Fallback.
3. **Erfüllung von `NFA-02` & `NFA-06` (Ressourceneffizienz):** Ein kompilierter Rust-Webserver mit `axum` und `tokio` benötigt im Leerlauf nur wenige Megabyte RAM (< 20–30 MB), hat keine Garbage-Collection-Pausen und lässt sich als winziges Single-Binary-Container-Image (< 50 MB) ausliefern.
4. **Erfüllung von `NFA-04` (Zero Data Retention):** Durch Rusts Ownership- und RAII-Modell (automatisches `Drop` am Ende des Request-Scopes) werden alle PDF-Buffer im Arbeitsspeicher sofort nach Beendigung des HTTP-Requests deterministisch freigegeben.

---

## 7. Prüfung auf Widersprüche, Lücken und Unklarheiten

Im Rahmen der Anforderungsvalidierung wurden folgende potenzielle Widersprüche identifiziert und aufgelöst:

### Konflikt 1: Lokale semantische Extraktion (`FA-04`) vs. Externe KI-Analyse (`FA-05`) – Wann greift welche Engine?
- **Identifizierter Widerspruch:** Wenn das System sowohl über eine eigene lokale semantische Extraktions-Engine als auch über eine externe KI-Anbindung verfügt, ist ohne klare Regel unklar, welches Subsystem Vorrang hat, wann externe API-Kosten entstehen und wie verhindert wird, dass beide Pfade widersprüchliche Ergebnisse liefern.
- **Auflösung (Zweistufige Local-First-Regel):**
  1. **Stufe 1 (Immer lokal zuerst):** Für jedes valide, textbasierte PDF führt der Server zuerst die lokale semantische Extraktion (`FA-04`) aus und prüft das Ergebnis direkt gegen das Ziel-Schema (`Invoice`-Struct bzw. `required`-Felder des Custom-Schemas).
  2. **Kurzschluss bei lokalem Erfolg (Short-Circuit):** Sind alle Pflichtfelder valide und typsicher befüllt, gibt der Server das Ergebnis **sofort** mit `"extraction_source": "local"` zurück – ohne externen KI-Aufruf (0 Token-Kosten, < 50 ms Latenz, 0 Bytes externer Datentransfer).
  3. **Stufe 2 (Gezielter KI-Fallback):** Nur wenn nach der lokalen Extraktion Pflichtfelder fehlen oder die Schema-Validierung noch nicht vollständig erfüllt ist (z. B. bei unkonventionellen Layouts oder freien Text-Zusammenfassungen im Custom-Schema), ruft der Hybrid-Router (`FA-05`) die externe Gemini API auf, übergibt die lokal erkannten Felder als Vorab-Kontext und liefert das final validierte Ergebnis mit `"extraction_source": "cloud_ai"` zurück.

### Konflikt 2: Statische Typsicherheit (`US-04`) vs. dynamische Custom-Schemas zur Laufzeit (`US-05`)
- **Identifizierter Widerspruch:** `US-04` (`profile=invoice`) verlangt ein fest definiertes, kompilierbares Datenmodell, während `US-05` (`schema={...}`) beliebige, erst zur Laufzeit bekannte JSON-Strukturen vom Entwickler akzeptiert. Zudem entsteht eine Unklarheit, was passiert, wenn ein Client **beide** Parameter (`profile` und `schema`) gleichzeitig oder **keinen** von beiden sendet.
- **Auflösung:**
  1. **Exklusivitätsregel (XOR):** Ein Request an `POST /api/v1/extract` muss **genau einen** der beiden Parameter (`profile` **oder** `schema`) enthalten. Werden beide oder keiner übergeben, antwortet die API sofort mit `HTTP 400 Bad Request` (`error_code: "INVALID_PARAMETERS"`).
  2. **Duale Validierung:** Bei `profile=invoice` wird gegen ein festes `Invoice`-Struct validiert; bei `schema` wird das Ergebnis als dynamisches `serde_json::Value` aufgebaut und mittels JSON-Schema-Validator (`jsonschema`) gegen das übergebene Client-Schema geprüft.

### Konflikt 3: Lokale PDF-Textextraktion (`US-02`) vs. gescannte PDFs ohne Textlayer (`US-08`)
- **Identifizierter Widerspruch:** Wenn ein Entwickler ein gescanntes PDF (reines Bild im PDF-Container) hochlädt, liefert der lokale PDF-Textparser (`FA-03`) einen leeren String zurück. Weder die lokale semantische Extraktion noch ein textbasierter KI-Prompt können daraus sinnvolle Daten gewinnen.
- **Auflösung:** Im **MVP (`US-02`, Must-Have)** prüft der Server nach der Textextraktion die Zeichenanzahl. Liegt diese unter dem Schwellenwert (< 20 druckbare Zeichen), lehnt das MVP den Request deterministisch mit `HTTP 422` (`error_code: "EMPTY_TEXT_LAYER"`) ab. Erst in der **Erweiterungsstufe (`US-08`, Should-Have)** wird dieser Pfad an den multimodalen Vision-Fallback weitergeleitet.

### Konflikt 4: CPU-synchrones PDF-Parsing & lokale Extraktion (`NFA-01`) vs. nicht-blockierender Async-Server (`NFA-05`)
- **Identifizierter Widerspruch:** Das Auslesen von PDF-Streams sowie die lokale heuristische Textanalyse sind synchrone, CPU-gebundene Operationen. In einer asynchronen Server-Laufzeitumgebung würde die direkte Ausführung im Request-Handler die Event-Loop für andere Anfragen blockieren.
- **Auflösung:** Das lokale PDF-Parsing und die lokale semantische Extraktion werden vom asynchronen Netzwerk-I/O entkoppelt (in Tokio über `spawn_blocking` auf einem dedizierten Thread-Pool ausgeführt), sodass Health-Checks und parallele Requests jederzeit blockierungsfrei bleiben.
