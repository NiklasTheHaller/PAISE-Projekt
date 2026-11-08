# 🤖 Dokumentation des KI-Einsatzes & Kritische Reflexion (Phase 1)

**Projekt:** DocuPulse – Leichtgewichtige Document Processing API (PDF-zu-JSON)  
**Phase:** 1 – Produktspezifikation & Anforderungsanalyse  
**Eingesetzte KI-Anwendungen:**  
1. Exploration, Domänen-Schärfung & Generierung von User Stories / Akzeptanzkriterien  
2. Strukturierung funktionaler/nichtfunktionaler Anforderungen & MVP-Abgrenzung  
3. Qualitäts-Audit: Prüfung auf Widersprüche, Lücken und typische Requirements-Engineering-Fehler  

---

## 1. Übersicht & Methodik des KI-Einsatzes

Ausgangspunkt unseres Projekts war das bewusst offen formulierte Thema:
> *„Document Processing API: Dokumente hochladen, analysieren und strukturierte Daten extrahieren (ist mit Absicht nicht genau definiert).“*

Wir haben KI nicht als „blinden Textgenerator“ verwendet, sondern in einem **iterativen Mensch-KI-Prozess** (Prompt → KI-Entwurf → Kritische Prüfung durch das Team → Gezielte Korrektur). Dabei kam die KI in **drei unterschiedlichen Anwendungsbereichen** des Requirements Engineerings zum Einsatz:
1. **Anforderungs-Elicitation & User-Story-Generierung:** Ableitung konkreter Developer-Personas und testbarer User Stories (im Given-When-Then-Format) aus unserer groben Projektidee.
2. **Scoping & Priorisierung (MoSCoW):** Eingrenzung eines im Semester realistisch umsetzbaren MVPs („PDF rein → JSON raus“).
3. **Validierung & Widerspruchsprüfung:** Systematischer Abgleich gegen die Bewertungskriterien und „Typischen Fehler“ der Aufgabenstellung.

---

## 2. Ausgewählte Prompts, KI-Ergebnisse & Kritische Bewertung

### 💬 Schritt 1: Initiale Strukturierung der offenen Projektidee

#### Verwendeter Prompt (Auszug):
```text
Wir machen ein Projekt, in dem wir KI verwenden sollen.
Unser Thema -> Document Processing API: Dokumente hochladen, analysieren und strukturierte Daten extrahieren (ist mit Absicht nicht genau definiert).
Ziel: Aus einer groben Idee eine strukturierte und umsetzbare Produktspezifikation entwickeln.
```

#### Ergebnis der KI:
- Die KI erkannte, dass die Aufgabenstellung bewusst unterbestimmt war, und stellte gezielte Rückfragen zu Fachdomäne und Verarbeitungsmodus.
- Nach unserer Auswahl (*Flexible Developer API mit Standardprofil `invoice` + Custom JSON-Schema*) generierte die KI einen ersten kompletten Spezifikationsentwurf mit 11 User Stories, einer Python/FastAPI-Architektur, OCR-Pipelines und asynchroner Queue.

#### 🔍 Kritische Bewertung & Korrektur durch das Team:
- **Was gut war:** Die Strukturierung der Domäne (Kombination aus sofort nutzbarem `invoice`-Profil und flexiblem `schema`-Parameter) sowie die Gherkin-Akzeptanzkriterien (`Given/When/Then`) sparten enorm viel Zeit.
- **Was wir korrigieren mussten:** Der erste KI-Entwurf war für ein Semesterprojekt zu breit angelegt (unterstützte gleichzeitig Bilder, Scans, Webhooks und Datenbank-Konzepte) und legte ungefragt Python/FastAPI fest. Wir haben daraufhin aktiv gegengesteuert.

---

### 💬 Schritt 2: Fokussierung des MVPs & Technologische Ausrichtung auf Rust

#### Verwendeter Prompt (Wortlaut):
```text
Wir wollen zuerst einen MVP erstellen und definieren. Dafür wollen wir uns mal nur auf PDFs lesen können und als JSON ausgeben. Dafür wollen wir Rust verwenden.
- Zielgruppe: Wir denken Developer sind unsere Zielgruppe
- AI zur Anforderungsanalyse einsetzen: Bitte unterstütze uns hierbei
- Mindestens 8–10 User Stories erstellen
- Funktionale und nichtfunktionale Anforderungen definieren & priorisieren (Must/Should/Could)
- MVP-Scope festlegen: PDF lesen, JSON ausgeben
- Anforderungen auf Widersprüche und Unklarheiten prüfen
```

#### Ergebnis der KI:
- Komplette Überarbeitung der User Stories auf die Zielgruppe **Software-Entwickler** (Backend-, Frontend-, DevOps-Personas).
- Reduktion des Eingabeformats auf reine PDF-Dateien (`application/pdf`) und strikte JSON-Ausgabe.
- Identifikation wichtiger technischer Konflikte (z. B. *Statische Typisierung vs. dynamisches JSON-Schema zur Laufzeit* sowie *Synchrone PDF-Parser vs. asynchrone Server-Laufzeit*).

#### 🔍 Kritische Bewertung & Korrektur durch das Team:
- **Identifiziertes Problem 1 (Vermischung von Tech-Stack und Anforderungen):** Die KI war nach unserem Hinweis auf Rust „übereifrig“ und schrieb spezifische Rust-Bibliotheken (`axum`, `tokio`, `lopdf`) direkt in die fachlichen Anforderungen und definierte sogar `NFA-01: Tech-Stack = Rust`. Laut Aufgabenstellung (*„5. Typische Fehler vermeiden: Technologiewahl, bevor die Anforderungen verstanden wurden“*) ist genau das ein klassischer Fehler.
- **Identifiziertes Problem 2 (Zu viele Must-Haves):** Die KI stufte im zweiten Entwurf **8 von 10 User Stories** und fast alle funktionalen Anforderungen als `Must-Have` ein – inklusive eines komplexen Vision-/OCR-Fallbacks für gescannte PDFs.

---

### 💬 Schritt 3: KI-gestütztes Qualitäts-Audit & Widerspruchsprüfung gegen die Aufgabenstellung

#### Verwendeter Prompt:
```text
take a look at the instructions as well as what has already been done, is there something we are missing and needs to be done? is there something that can be improved before handing it in?
[+ Vollständige Aufgabenstellung & Bewertungskriterien]
```

#### Ergebnis der KI (Audit-Befunde):
Die KI führte einen systematischen Soll-Ist-Abgleich mit der Aufgabenstellung durch und deckte folgende Lücken und Schwachstellen auf:
1. **Fehlendes Deliverable `docs/mvp.md`:** Im ersten Prompt fehlte `docs/mvp.md` in der Aufzählung, weshalb der MVP-Scope nur als Unterkapitel in `docs/requirements.md` stand.
2. **Fehlende Kosten- & Abhängigkeitsanalyse:** Die Rahmenbedingung *„Externe APIs und Services sind erlaubt. Abhängigkeiten und Kosten müssen berücksichtigt werden“* war noch nicht explizit dokumentiert.
3. **Kritik an der eigenen Vorarbeit (Must-Have-Inflation & Tech-Bias):** Die KI erkannte beim Abgleich mit Abschnitt 5 (*„Typische Fehler vermeiden“*), dass zu viele Stories als `Must-Have` markiert waren und Rust-Crates von den eigentlichen Anforderungen getrennt werden müssen.

#### 🔍 Unsere Entscheidung & finale Korrektur:
Auf Rückfrage der KI haben wir folgende verbindliche Architekturentscheidungen getroffen und in allen Dokumenten umgesetzt:
- **MVP-Abgrenzung (`docs/mvp.md`):** Wir behalten sowohl das Standardprofil (`profile=invoice`) als auch das benutzerdefinierte JSON-Schema (`schema`) im MVP (`Must-Have`), beschränken das MVP aber strikt auf **textbasierte PDFs**. Reine Bild-Scans ohne Textlayer werden im MVP lokal erkannt und vor dem KI-Aufruf mit `HTTP 422 (EMPTY_TEXT_LAYER)` abgewiesen; der multimodale Vision-Fallback wandert als `Should-Have` (`US-07`) in die nächste Ausbaustufe.
- **Saubere MoSCoW-Verteilung:** Reduktion von 8 auf **6 Must-Have User Stories** (`US-01` bis `US-06`), ergänzt durch **3 Should-Have** (`US-07` bis `US-09`) und **1 Could-Have** (`US-10`).
- **Externe API & Kostenschutz:** Festlegung auf die **Google Gemini API (`gemini-2.5-flash`)** im kostenlosen Free-Tier für das Semesterprojekt, kombiniert mit technischen Kostenschutz-Regeln (lokaler Magic-Byte- und Textlayer-Check *vor* jedem API-Aufruf, 10-MB-/5-Seiten-Limit, Mock-Trait für kostenlose Offline-Tests).
- **Trennung von Anforderung und Technologie:** In `docs/requirements.md` sind Anforderungen nun messbar und verhaltensbasiert formuliert (z. B. `< 50 ms` lokales Pre-Processing, `< 50 MB` RAM im Leerlauf, `100 %` JSON-Konformität), während die Wahl von **Rust** in Kapitel 6 logisch aus diesen Qualitätsanforderungen abgeleitet wird.

---

## 3. Zusammenfassung: Wo wir KI-Vorschläge korrigiert haben

| Bereich | Ursprünglicher KI-Vorschlag | Menschliche Prüfung & Korrektur | Warum war die Korrektur wichtig? |
| :--- | :--- | :--- | :--- |
| **Must-Have-Priorisierung** | 8 von 10 User Stories als `Must-Have` (inkl. Vision-Fallback für Scans und DevOps-RAM-Story). | Reduziert auf **6 echte Must-Have-Stories**; Vision-Fallback auf `Should-Have` verschoben; RAM-Metrik rein als `NFA-02` geführt. | Vermeidet den Fehler *„Zu viele Must-have-Anforderungen“* und hält das MVP im Semester sicher machbar. |
| **Technologiewahl vs. Anforderungen** | Spezifische Rust-Crates (`lopdf`, `axum`) standen direkt in den User Stories; `Rust` war als `NFA-01` definiert. | Anforderungen fachlich & messbar formuliert; Rust als abgeleitete Architekturentscheidung in eigenem Abschnitt begründet. | Vermeidet den Fehler *„Technologiewahl, bevor die Anforderungen verstanden wurden“*. |
| **Widerspruch bei Request-Parametern** | `profile=invoice` und `schema={...}` standen unverbunden nebeneinander. | Explizite **XOR-Regel** ergänzt: Ein Request muss genau einen der beiden Parameter enthalten, sonst `HTTP 400`. | Schließt eine echte Spezifikationslücke für die spätere Implementierung und Tests. |
| **Externe API-Kosten & Fehlerpfade** | KI-API wurde als „Blackbox“ angenommen, die immer antwortet. | Lokalen Pre-Check (Magic-Bytes, Schema-Check, Textlayer-Check) vor den API-Call geschaltet + 15s Timeout & Kostenanalyse in `docs/mvp.md` ergänzt. | Verhindert unnötigen Token-Verbrauch bei ungültigen Dateien und erfüllt die Rahmenbedingungen. |

---

## 4. Kritische Reflexion des Projektteams (Phase 1)

### 4.1 Was hat durch den KI-Einsatz besonders gut funktioniert?
- **Große Zeitersparnis bei Strukturierung & Gherkin-Kriterien:** Aus einer sehr offenen Aufgabenstellung (*„Dokumente hochladen, analysieren und strukturierte Daten extrahieren“*) konnten wir im Dialog schnell ein präzises Produktkonzept entwickeln. Besonders beim Ausformulieren konsistenter, überprüfbarer **Given-When-Then-Akzeptanzkriterien** und konkreter HTTP-Fehlerfälle (`400`, `413`, `422`, `502`, `504`) hat die KI die Dokumentationsarbeit erheblich beschleunigt.
- **Aufdecken technischer Randfälle (Edge Cases):** Bei der Widerspruchsprüfung half die KI sehr effektiv dabei, frühzeitig auf Probleme hinzuweisen, die sonst erst beim Programmieren in Phase 2 aufgefallen wären – insbesondere die Frage, wie man statische Typisierung (für `profile=invoice`) mit dynamischer Laufzeit-Validierung (für `schema`) kombiniert und was bei PDFs ohne Textlayer passiert.

### 4.2 Wo lagen die Grenzen und Risiken der KI?
- **Tendenz zu Feature-Creep & Must-Have-Inflation:** Ohne klare menschliche Eingrenzung neigt die KI dazu, ein „perfektes Enterprise-Produkt“ zu spezifizieren und fast jede gute Idee als `Must-Have` einzustufen. Hätten wir den ersten Entwurf ungeprüft übernommen, wäre der Scope für ein einzelnes Semester zu groß geworden.
- **Vermischung von „Was“ (Anforderung) und „Wie“ (Implementierung):** Sobald wir erwähnten, dass wir das Projekt in Rust umsetzen möchten, formulierte die KI selbst fachliche Anforderungen plötzlich als Bibliotheks-Vorgaben. Erst durch unser gezieltes Review gegen die Aufgabenstellung haben wir Anforderung und Technologieentscheidung wieder sauber getrennt.

### 4.3 Fazit & Ausblick auf Phase 2
Die Verantwortung für den Scope und die Qualitätsmaßstäbe lag zu jedem Zeitpunkt beim Team: Die KI lieferte schnelle Entwürfe und strukturierte Analysen, während wir die strategischen Leitplanken (Fokus auf Entwickler, Beschränkung auf PDF-zu-JSON, realistisches MVP mit 6 Must-Haves, Kostenkontrolle über Gemini Flash) gesetzt und geschärft haben.  
Für **Phase 2 (Implementierung)** bildet diese Spezifikation ein klares Fundament, auf dem wir die KI gezielt für Rust-Boilerplate (`axum`-Routing, `serde`-Modelle) und automatisierte Integrationstests einsetzen können.
