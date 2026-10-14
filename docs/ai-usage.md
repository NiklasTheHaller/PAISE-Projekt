# 🤖 Dokumentation des KI-Einsatzes & Reflexion: Phase 1

**Projekt:** DocuPulse (Rust Edition)  
**Phase:** 1 – Produktspezifikation & Anforderungsanalyse  
**Technologie-Fokus:** Rust (Axum, Tokio, Serde)  
**Zweck:** Transparente Protokollierung der tatsächlichen Interaktion zwischen dem Projektteam (Mensch) und dem KI-Assistenten sowie Vorbereitung der menschlichen Reflexion.

---

## 1. Übersicht & Ziel des KI-Einsatzes

In Phase 1 wurde die KI als **interaktiver Requirements Engineer, Sparring-Partner und Dokumentations-Assistent** eingesetzt. Ausgangspunkt war eine bewusst offene Aufgabenstellung aus dem Modul:

> *„Document Processing API: Dokumente hochladen, analysieren und strukturierte Daten extrahieren (ist mit Absicht nicht genau definiert).“*

Ziel des KI-Einsatzes war es, durch iterativen Dialog aus dieser vagen Vorgabe eine fokussierte, realistische und technisch anspruchsvolle Produktspezifikation zu entwickeln.

---

## 2. Chronologisches Interaktionsprotokoll (Prompts & KI-Aktionen)

### 💬 Interaktionsschritt 1: Initialer Projektauftrag

#### Prompt des Nutzers (Wortlaut):
```text
Mir machen ein projekt indem wir KI verwenden sollen. Das folgende ist wichtig:

Unser Thema -> Document Processing API: Dokumente hochladen, analysieren und strukturierte Daten extrahieren

Ist mit absicht nicht genau definiert.

Hier ist phase 1:

Ziel
Aus einer groben Idee eine strukturierte und umsetzbare Produktspezifikation entwickeln.

TODOs
- Projektidee und Problemstellung definieren
- Zielgruppe und wichtigste Anwendungsfälle identifizieren
- AI zur Anforderungsanalyse einsetzen
- Mindestens 8–10 User Stories erstellen
- Funktionale und nichtfunktionale Anforderungen definieren
- Anforderungen priorisieren (Must/Should/Could)
- MVP-Scope festlegen
- Anforderungen auf Widersprüche und Unklarheiten prüfen

Deliverables
- README.md – Projektüberblick
- docs/requirements.md – Anforderungen und User Stories
- docs/ai-usage.md – AI-Einsatz und Reflexion
```

#### Was die KI damit gemacht hat:
1. **Workspace- und Codebase-Inspektion:**
   - Analyse des Repositories: Nur eine leere `README.md` (`# PAISE-Projekt`) war vorhanden.
2. **Strukturierung des Vorgehens:**
   - Identifikation der offenen Parameter (Fachdomäne und Verarbeitungsmodus).
3. **Interaktive Rückfrage an den Nutzer:**
   - Formulierung konkreter Optionen zur Schärfung des Projektkerns via Multiple-Choice.

---

### 💬 Interaktionsschritt 2: Initiale Domänen- & Modus-Wahl

#### Rückfrage der KI an den Nutzer:
- Fachlicher Schwerpunkt (Rechnungen, Schema-API, Verträge, HR)?
- Verarbeitungsmodus (Synchron, Asynchron, Hybrid)?

#### Entscheidung des Nutzers:
```text
A1: (Recommended) Flexible Developer API: Generische Extraktion via benutzerdefiniertem JSON-Schema mit vorkonfiguriertem Profil für Rechnungen & Belege (Invoices/Receipts).
A2: (Recommended) Synchron & Asynchron: Schnelle synchrone Analyse für kleine Dokumente plus asynchrone Job-Queue (Polling/Status-Check) für mehrseitige PDFs.
```

#### Was die KI damit gemacht hat:
- Erste Ausarbeitung einer Produktspezifikation (zunächst auf Python/FastAPI-Basis), Erstellung von 11 User Stories, MoSCoW-Matrix und erster Widerspruchsprüfung.

---

### 💬 Interaktionsschritt 3: Präzisierung des Dokumentationszwecks

#### Prompt des Nutzers (Wortlaut):
```text
ai-usage.md soll dokumentieren wie WIR KI verwendet haben. Bedeutet, dort dokumentierst du welche prompts wir verwendet haben, und was du damit gemacht hast. Späte reflektieren die menschen die an dem projekt gearbeitet haben wie das war
```

#### Was die KI damit gemacht hat:
- Umstellung von `docs/ai-usage.md` auf ein reales, chronologisches Interaktionsprotokoll mit Original-Prompts und vorbereitetem Reflexionsteil für das Team.

---

### 💬 Interaktionsschritt 4: Technologie-Entscheidung für Rust & MVP-Fokussierung

#### Prompt des Nutzers (Wortlaut):
```text
Es wir wollen einmal zuerst einen MVP erstellen und definieren. Dafür wollen wir uns mal nur auf PDFs lesen können und als json ausgeben. Dafür wollen wir rust verwenden. 

Projektidee und Problemstellung definieren
- wie wir erklärt haben

Zielgruppe und wichtigste Anwendungsfälle identifizieren
- Wir denken developer sind unser zielgruppe

AI zur Anforderungsanalyse einsetzen
- Das bist du. Bitte danke unterstütze uns hierbei.

Mindestens 8–10 User Stories erstellen
- plane uns unsere 8-10 user stories. Wie können dann ausbauen.

Funktionale und nichtfunktionale Anforderungen definieren
- erstelle einen plan hierfür

Anforderungen priorisieren (Must/Should/Could)
- priorisiere unserse anforderungen

MVP-Scope festlegen
- PDF lesen, json ausgeben.

Anforderungen auf Widersprüche und Unklarheiten prüfen
```

#### Was die KI damit gemacht hat:
1. **Technologische Neuausrichtung auf Rust:**
   - Komplette Neufassung der Spezifikationen auf das **Rust-Ökosystem** (`axum`, `tokio`, `serde`, `serde_json`, `reqwest`, `lopdf`/`pdf-extract`).
   - Herausarbeiten der spezifischen Vorteile von Rust (Memory Safety, < 50 MB RAM, kein GC-Stutter, Single-Binary-Container).
2. **Schärfung des MVP-Scopes:**
   - **Klarer MVP-Fokus:** Reine PDF-Dateien einlesen (`application/pdf`) und als strikt typisiertes JSON (`application/json`) ausgeben. Keine Ablenkung durch Bildformate oder Datenbanken im ersten Schritt.
3. **Ausrichtung auf die Zielgruppe Software-Entwickler:**
   - Definition von Developer-Personas (Backend-Basti, Frontend-Fabi, DevOps-Dana).
   - Fokus auf einfache Curl-Nutzung, saubere HTTP-Statuscodes und OpenAPI/Swagger-Generierung.
4. **Erstellung von 10 entwicklerzentrierten User Stories (US-01 bis US-10):**
   - Vollständige Gherkin-Akzeptanzkriterien für PDF-Upload, nativen Text-Parser in Rust, Serde-Validierung, Error-Handling für defekte PDFs und flüchtigen Speicher.
5. **Rust-spezifische Widerspruchs- und Machbarkeitsprüfung:**
   - *Blocking I/O in Tokio:* Lösung durch Auslagerung CPU-intensiven PDF-Parsings in `tokio::task::spawn_blocking`.
   - *Statische Typen vs. dynamisches Schema:* Lösung durch duale Serde-Architektur (stark typisiertes `Invoice`-Struct vs. `serde_json::Value` mit `jsonschema`-Validierung für Custom Schemas).
   - *Scans vs. Text-PDFs:* Intelligenter Fallback von nativem Text-Parsing auf Base64-Multimodal-Inferenz.
6. **Aktualisierung aller Deliverables:**
   - `README.md` (Rust-Architekturdiagramm, Curl-Vorschau, Badges).
   - `docs/requirements.md` (Vollständige Rust-Produktspezifikation).
   - `docs/ai-usage.md` (Aktualisiertes Protokoll).

---

## 3. Rollenverteilung & Zusammenarbeitsmodell

| Dimension | Rolle des menschlichen Teams | Rolle des KI-Assistenten |
| :--- | :--- | :--- |
| **Technologie-Vorgabe** | Setzt die Leitplanke: **Rust** statt Python; Fokus auf **PDF-zu-JSON**. | Wählt passende Rust-Crates (`axum`, `tokio`, `serde`) und entwirft die Rust-spezifische Architektur. |
| **Scope-Fokussierung** | Begrenzt den MVP-Scope bewusst auf das Wesentliche ("PDF lesen, JSON ausgeben"). | Definiert klare Grenzen (In-Scope vs. Out-of-Scope) und verhindert Feature-Creep. |
| **Zielgruppen-Definition** | Definiert **Developer** als primäre Zielgruppe. | Formuliert zielgruppengerechte Personas, User Stories und CLI-/API-Beispiele. |
| **Qualitätssicherung** | Hinterfragt und korrigiert die Richtung im Dialog. | Führt Widerspruchsprüfungen für das Rust-Ökosystem durch und garantiert Konsistenz. |

---

## 4. Spätere Reflexion durch das Projektteam (Menschliche Perspektive)

> *Hinweis: Dieser Abschnitt wird im weiteren Projektverlauf von den beteiligten Studierenden/Entwicklern ausgefüllt, um die menschliche Erfahrung mit dem KI-Einsatz in Phase 1 zu reflektieren.*

### 4.1 Leitfragen für die Team-Reflexion:
1. **Effizienz & Zeitersparnis:**
   - *Wie viel Zeit hat die KI bei der Ausarbeitung der Rust-spezifischen Anforderungen und User Stories im Gherkin-Format eingespart?*
   - *(Hier Notizen des Teams eintragen)*
2. **Technologischer Richtungswechsel (Python zu Rust):**
   - *Wie gut hat die KI den Schwenk auf Rust und die damit verbundenen Besonderheiten (Tokio Blocking Tasks, Serde Typisierung) erfasst?*
   - *(Hier Notizen des Teams eintragen)*
3. **Menschliche Steuerung:**
   - *Welche Entscheidungen des Teams waren entscheidend (z. B. MVP-Fokus auf PDF-only, Rust als Core-Sprache), um das Projekt realistisch zu halten?*
   - *(Hier Notizen des Teams eintragen)*
4. **Learnings für Phase 2 (Implementierung):**
   - *Wie soll die KI in Phase 2 bei der Erstellung des `Cargo.toml`, der Axum-Handler und der Tests eingesetzt werden?*
   - *(Hier Notizen des Teams eintragen)*

---

### Vorläufiges Zwischenfazit des Projektteams (Stand Phase 1):
- Durch die klare Vorgabe, **Rust** zu verwenden und den Scope radikal auf **„PDF rein -> JSON raus“** für **Entwickler** zu fokussieren, hat das Projekt ein messerscharfes Profil erhalten.
- Die KI half entscheidend dabei, die theoretischen Stärken von Rust (Typsicherheit, Memory Safety, Async I/O) in konkrete funktionale und nichtfunktionale Anforderungen sowie Akzeptanzkriterien zu übersetzen.
- Phase 1 ist damit mit einer umsetzbaren, widerspruchsfreien und realistischen Spezifikation abgeschlossen.
