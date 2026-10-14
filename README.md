# 🦀 DocuPulse (Rust Edition)
> **Ultra-performante, ressourcenschonende Document Processing API in Rust: PDFs einlesen, semantisch analysieren und als strikt typisiertes JSON ausgeben.**

[![Rust](https://img.shields.io/badge/Rust-1.75+-DEA584.svg?logo=rust&logoColor=white)](https://www.rust-lang.org/)
[![Axum](https://img.shields.io/badge/Axum-0.7+-black.svg)](https://github.com/tokio-rs/axum)
[![Tokio](https://img.shields.io/badge/Async-Tokio-000000.svg?logo=tokio&logoColor=white)](https://tokio.rs/)
[![Serde](https://img.shields.io/badge/Serde-JSON-orange.svg)](https://serde.rs/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 📌 Über das Projekt

Entwickler stehen regelmäßig vor dem Problem, Daten aus unstrukturierten oder semi-strukturierten PDF-Dateien (Rechnungen, Lieferscheine, Berichte) in saubere JSON-Daten für Microservices und Datenbanken zu überführen. Bestehende Python- oder OCR-Lösungen sind oft schwerfällig, speicherhungrig und erfordern gigantische Docker-Container.

**DocuPulse (Rust Edition)** ist eine entwicklerzentrierte, leichtgewichtige REST-API, geschrieben in **Rust**:
- **PDF rein, JSON raus:** Konzentriert auf den essenziellen MVP-Workflow.
- **Hohe Performance & Speichereffizienz:** < 50 MB RAM, Zero-Cost Abstractions und asynchrones I/O mit `tokio` und `axum`.
- **Typ- und Schemagarantie:** Kein ungültiges oder halbes JSON dank strikter Serialisierung via `serde_json`.
- **Duale Extraktion:** Sofort nutzbar mit Standard-Profil (`invoice`) oder dynamisch mit benutzerdefiniertem JSON-Schema.

---

## 🏛️ Systemarchitektur (Rust Backend)

```mermaid
flowchart TD
    Developer["Developer / Client<br>(curl, Frontend, Microservice)"]

    subgraph Rust_API["DocuPulse Engine (Rust / Axum)"]
        Router["Axum HTTP Router<br>POST /api/v1/extract"]
        Multipart["Multipart Stream Parser<br>(Tokio Async)"]
        Safety["Magic-Byte Check (%PDF-)<br>& In-Memory Buffer"]
        
        subgraph PDF_Engine["PDF Processing (spawn_blocking)"]
            NativeExtract["Native Text-Layer Parser<br>(lopdf / pdf-extract)"]
            VisionFallback["Scan / Vision Prepper<br>(Base64 Encoder)"]
        end

        subgraph AI_Client["AI Ingestion & Structuring"]
            ReqwestClient["Async Reqwest Client<br>(Gemini / Vision-LLM API)"]
            SerdeValidator["Serde JSON Validator<br>(Typed Structs / JSON Schema)"]
        end
    end

    Developer -->|Multipart POST PDF| Router
    Router --> Multipart
    Multipart --> Safety
    Safety --> NativeExtract
    NativeExtract -->|Text vorhanden| ReqwestClient
    NativeExtract -->|Reiner Scan| VisionFallback
    VisionFallback --> ReqwestClient

    ReqwestClient --> SerdeValidator
    SerdeValidator -->|Valides JSON (HTTP 200)| Developer
    SerdeValidator -.->|Schema-Error (HTTP 422)| Developer
```

---

## 🚀 Quickstart & Entwickler-Vorschau

### 1. Synchroner PDF-Upload (Standardprofil Rechnungen)
```bash
curl -X POST "http://localhost:8000/api/v1/extract" \
  -F "file=@rechnung.pdf" \
  -F "profile=invoice"
```

**Erwartete JSON-Antwort:**
```json
{
  "status": "success",
  "document": {
    "filename": "rechnung.pdf",
    "pages": 1
  },
  "data": {
    "invoice_number": "RE-2026-9812",
    "date": "2026-10-14",
    "vendor": "Acme Cloud Services GmbH",
    "currency": "EUR",
    "total_amount": 1190.00,
    "tax_amount": 190.00,
    "line_items": [
      {
        "description": "Server Hosting Q4",
        "quantity": 1,
        "unit_price": 1000.00,
        "total": 1000.00
      }
    ]
  },
  "processing_time_ms": 2840
}
```

### 2. Generische Extraktion mit individuellem JSON-Schema
```bash
curl -X POST "http://localhost:8000/api/v1/extract" \
  -F "file=@zertifikat.pdf" \
  -F 'schema={
        "type": "object",
        "properties": {
          "recipient": {"type": "string"},
          "course_name": {"type": "string"},
          "grade": {"type": "number"}
        },
        "required": ["recipient", "course_name"]
      }'
```

---

## 📂 Projektstruktur

```text
PAISE-Projekt/
│
├── README.md               # Projektüberblick & Schnellstart (Rust Edition)
│
├── docs/
│   ├── requirements.md     # Spezifikation: Rust-Stack, 10 User Stories, FA, NFA, MoSCoW
│   └── ai-usage.md         # Dokumentation aller Prompts, KI-Aktionen & Reflexion
│
└── [Phase 2: Cargo.toml, src/ (Axum Server), tests/]
```

---

## 🧭 Entwicklungs-Roadmap

- [x] **Phase 1: Produktspezifikation & Anforderungsanalyse (Rust & PDF Focus)**
  - [x] Problemstellung & Zielgruppe (Entwickler) geschärft
  - [x] MVP-Fokus: "PDFs einlesen, analysieren, JSON ausgeben"
  - [x] 10 entwicklerzentrierte User Stories mit Given-When-Then Akzeptanzkriterien
  - [x] Funktionale (FA-01..14) & Nichtfunktionale (NFA-01..08) Anforderungen für Rust
  - [x] MoSCoW-Priorisierung & Rust-spezifische Widerspruchsprüfung
  - [x] Deliverables: `README.md`, `docs/requirements.md`, `docs/ai-usage.md`
- [ ] **Phase 2: Rust-Architektur & Prototyp-Implementierung**
  - [ ] Projekt-Setup (`Cargo.toml` mit `axum`, `tokio`, `serde`, `reqwest`, `lopdf`)
  - [ ] Multipart PDF Upload & Header-Validierung
  - [ ] KI-Client Integration mit Prompt-Template & Structured Output
  - [ ] Serde Deserialisierung & Error Handling
- [ ] **Phase 3: Asynchrone Queue, Testing & Dockerfile**
  - [ ] Asynchrone Job-Queue für mehrseitige PDFs
  - [ ] Integrationstests (`cargo test`) mit echten Test-PDFs
  - [ ] Multi-Stage Dockerfile (Scratch / Distroless Image < 30 MB)