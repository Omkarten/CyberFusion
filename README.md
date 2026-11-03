# 🛡️ CyberFusion — AI Cybersecurity & Digital Forensics Platform

A unified platform for cybercrime investigators to manage cases, run AI-assisted analysis on digital evidence, gauge threat severity, and generate investigator-ready reports with recommended interventions — all while preserving a tamper-evident chain of custody.


---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Core Features](#2-core-features)
3. [System Architecture](#3-system-architecture)
4. [Recommended Tech Stack](#4-recommended-tech-stack)
5. [Data Model](#5-data-model)
6. [Analysis Modules — Implementation Plan](#6-analysis-modules--implementation-plan)
7. [Blockchain-Inspired Evidence Integrity](#7-blockchain-inspired-evidence-integrity)
8. [Risk Scoring & Report Generation](#8-risk-scoring--report-generation)
9. [Authentication & Role-Based Access Control](#9-authentication--role-based-access-control)
10. [API Surface (High-Level)](#10-api-surface-high-level)
11. [Deployment Strategy](#11-deployment-strategy)
12. [Project Folder Structure](#12-project-folder-structure)
13. [4-Week Roadmap (Team of 4)](#13-4-week-roadmap-team-of-4)
14. [Security & Privacy Considerations](#14-security--privacy-considerations)
15. [Testing Strategy](#15-testing-strategy)
16. [Open Decisions & Known Limitations](#16-open-decisions--known-limitations)
17. [Suggested Datasets & Pretrained Resources](#17-suggested-datasets--pretrained-resources)

---

## 1. Problem Statement

Cybercrime investigators today juggle disconnected tools: one for phishing checks, another for malware scanning, a spreadsheet for case tracking, email for chain-of-custody notes. Evidence can be altered (accidentally or intentionally) between collection and courtroom, and there's no single place that turns raw evidence into a risk-scored, report-ready finding.

**CyberFusion** aims to be that single place: upload evidence → run the right AI analysis module(s) → get a risk score and findings → attach everything to a case with a tamper-evident history → export a report.

---

## 2. Core Features

- **Auth & RBAC** — JWT-based login, roles such as `admin`, `investigator`, `analyst`, `viewer`
- **Dashboard** — active cases, recent analyses, risk summary, notifications
- **12 AI Analysis Modules** (detailed in [§6](#6-analysis-modules--implementation-plan))
- **Case Management** — create/track cases, assign evidence, investigation timeline, case status
- **Evidence Management** — upload, metadata extraction, SHA-256 hashing, full evidence history
- **Blockchain-Inspired Evidence Integrity** — append-only hash chain, timestamping, chain-of-custody log, tamper detection (see [§7](#7-blockchain-inspired-evidence-integrity) — **not** a real distributed ledger)
- **Risk & Findings Engine** — aggregates module outputs into a risk score with recommendations
- **Report Generation** — analysis report, case report, evidence summary, PDF export
- **Notifications** — analysis completed, high-risk finding, case updates
- **Audit Logs** — user/evidence/analysis/case activity trail
- **Admin Panel** — user & role management, module management, dataset/model management, system monitoring

---

## 3. System Architecture

CyberFusion is split into **three tiers** plus a **shared evidence/integrity layer**, because the AI workloads (mostly Python, GPU-friendly) and the web app (Node/React) have very different runtime needs:

```
                        ┌─────────────────────────┐
                        │        Frontend          │
                        │   React (Vite) + Tailwind │
                        │   Deployed on Vercel      │
                        └────────────┬─────────────┘
                                     │ HTTPS / REST (JWT)
                        ┌────────────▼─────────────┐
                        │      Core API Server      │
                        │  Node.js + Express         │
                        │  Auth, Cases, Evidence,     │
                        │  Reports, Notifications,    │
                        │  Audit Logs, Admin          │
                        │  Self-hosted (Docker)       │
                        └──────┬───────────┬─────────┘
                               │           │
                 ┌─────────────▼───┐   ┌───▼─────────────────┐
                 │   Primary DB     │   │   AI/ML Service      │
                 │ MongoDB / Postgres│   │ Python (FastAPI)     │
                 │ Cases, Evidence   │   │ 12 analysis modules   │
                 │ meta, Users,      │   │ pretrained models +   │
                 │ Findings, Audit   │   │ heuristics             │
                 └──────────────────┘   └──────┬────────────────┘
                                                │
                                  ┌─────────────▼────────────┐
                                  │   Evidence Object Storage  │
                                  │  (filesystem / MinIO / S3) │
                                  └─────────────┬─────────────┘
                                                │
                                  ┌─────────────▼────────────┐
                                  │ Hash-Chain Integrity Log   │
                                  │ (append-only, signed,       │
                                  │  stored in its own table)   │
                                  └─────────────────────────────┘
```

**Why split Node (core API) from Python (AI/ML service)?** Almost all mature pretrained models for forgery detection, face matching, malware analysis, and NLP classification live in the Python ecosystem (PyTorch/TensorFlow/scikit-learn/HuggingFace). Rather than forcing everything into one language, the Core API calls the AI/ML service over an internal REST API — this also lets the AI service scale, restart, or be swapped independently (important if your team splits work by module).

---

## 4. Recommended Tech Stack

| Layer | Recommendation | Why |
|---|---|---|
| Frontend | **React + Vite**, Tailwind CSS | Fast dev loop, matches prior MERN experience, deploys cleanly to Vercel |
| Core API | **Node.js + Express** | Reuses existing MERN familiarity; fast to scaffold CRUD, auth, RBAC |
| AI/ML Service | **Python + FastAPI** | Required for pretrained model ecosystem; FastAPI gives you auto-generated docs (`/docs`) which doubles as a team contract |
| Primary Database | **MongoDB** (or PostgreSQL if you prefer relational) | Case/evidence/finding documents are naturally nested and schema-flexible; Mongoose keeps consistency with your finance-tracker experience. PostgreSQL is a fine alternative if the team wants stronger relational integrity for audit logs |
| Evidence File Storage | **Local filesystem (Docker volume) or MinIO** (self-hosted S3-compatible) | Keeps raw evidence off the database; MinIO gives you S3 API compatibility for free, useful if you later migrate to real cloud storage |
| Auth | **JWT + bcrypt**, refresh tokens | Matches your existing pattern from the finance tracker |
| Containerization | **Docker + Docker Compose** | One `docker-compose.yml` brings up frontend, core API, AI service, DB, and MinIO together — ideal for your homelab and for grading/demo reproducibility |
| Frontend hosting | **Vercel** | Free, fast, trivial CI/CD for the React app |
| Backend/AI hosting | **Self-hosted (your Debian homelab) via Docker**, or **Render/Railway** as a cloud fallback | Vercel's serverless functions are a poor fit for long-running Python ML inference (cold starts, execution time limits, no persistent GPU/CPU-heavy process) — keep the Core API + AI service off Vercel |
| Report PDF export | **Puppeteer** (HTML→PDF) or **pdf-lib** | Puppeteer lets you template reports in HTML/CSS and render to PDF without a separate templating system |

---

## 5. Data Model

Core entities:

```
User          { id, name, email, passwordHash, role, createdAt }
Case          { id, title, description, status, createdBy, assignedTo[], createdAt, closedAt }
Evidence      { id, caseId, filename, fileType, sha256Hash, uploadedBy, uploadedAt, storagePath, metadata{} }
Analysis      { id, evidenceId, module, status, result{}, riskScore, runAt, modelVersion }
Finding       { id, analysisId, indicator, severity, description }
Report        { id, caseId, generatedBy, generatedAt, pdfPath, summary }
AuditLog      { id, actorId, action, targetType, targetId, timestamp, metadata{} }
IntegrityLog  { id, evidenceId, prevHash, entryHash, action, actorId, timestamp }  ← hash-chain (§7)
Notification  { id, userId, type, message, read, createdAt }
```

`Analysis.result` is intentionally a flexible object since each of the 12 modules returns a different shape (e.g., phishing returns a verdict + confidence; face matching returns bounding boxes + match scores).

---

## 6. Analysis Modules — Implementation Plan

For each module: **what it does**, a **pretrained-model/API approach** , and **implementation notes**. 

### 1. Fake Social Media Profile Detection — *Medium*
- **Approach:** Rule-based scoring (account age, follower/following ratio, post frequency, bio text patterns) combined with a lightweight pretrained text classifier (e.g., a HuggingFace bot-detection or spam-text model) on bio/post content, plus reverse image search on the profile picture.
- **Input:** Profile URL or exported JSON/screenshot + metadata.
- **Output:** Fake-probability score + flagged indicators (e.g., "stock photo detected", "account age < 7 days").

### 2. Phishing URL & Domain Detection — *Easy, build this first*
- **Approach:** Call existing threat-intel APIs — **Google Safe Browsing API** (free tier) and/or **VirusTotal API** (free tier, URL/domain scan) — rather than training a model. Optionally layer a lightweight lexical-features ML classifier (URL length, special chars, suspicious TLD) trained on a public phishing-URL dataset as a fallback when APIs are rate-limited.
- **Output:** Verdict (malicious/suspicious/clean), source engines that flagged it, confidence.

### 3. Image Forgery / Tampering Detection — *Hard*
- **Approach:** Error Level Analysis (ELA) — a well-documented, implementable-without-training algorithmic technique — as the primary signal, optionally combined with a pretrained CNN for splice/copy-move detection if your team finds a usable open checkpoint (models trained on the CASIA v2 dataset are common in open-source repos).
- **Output:** Tamper-probability heatmap + verdict.

### 4. Chat & Email Forensics — *Medium*
- **Approach:** Parse `.eml`/`.mbox`/exported chat JSON; check email headers for spoofing (SPF/DKIM/DMARC mismatches), run NLP (spaCy or a pretrained sentiment/toxicity model) over message content for threats/harassment/coercion language, build a timeline.
- **Output:** Header anomaly flags, flagged messages, timeline.

### 5. Malware / APK Static Analysis — *Medium (leans on existing tools)*
- **Approach:** Don't reinvent this — wrap **MobSF (Mobile Security Framework)**, an open-source static/dynamic analysis tool, for APKs, and use the **VirusTotal API** for general file hash/signature lookups. Your "module" is mostly an integration + results parser.
- **Output:** Permissions requested, known-malware signature matches, risk flags.

### 6. Fake News & Propaganda Detection — *Easy–Medium*
- **Approach:** Pretrained NLP classifier from HuggingFace (fake-news/misinformation classifiers trained on datasets like LIAR or Kaggle's Fake News dataset are commonly published). Add a simple source-credibility heuristic (domain reputation list).
- **Output:** Fake-probability score, flagged claims.

### 7. Credit Card Fraud Detection — *Easy, build this first*
- **Approach:** This is the most "textbook" module — use the well-known **Kaggle Credit Card Fraud Detection dataset** (anonymized, PCA-transformed transaction features) to either (a) use a published pretrained model/notebook as reference, or (b) train a small model yourselves (Random Forest / XGBoost / Isolation Forest) — this is lightweight enough to fit a 1-month timeline even without "no training" as a hard constraint.
- **Output:** Fraud probability per transaction, anomalous transaction list.

### 8. Face Matching & Person Identification — *Medium (leans on existing libraries)*
- **Approach:** Use the **`face_recognition`** Python library (dlib-based) or **DeepFace** — both ship pretrained face-embedding models, no training required. Compare embeddings via cosine/Euclidean distance against a reference set.
- **Output:** Match confidence, bounding boxes, closest matches.
- ⚠️ Flag for your team/advisor: face identification raises real privacy/consent questions even in an academic context — scope this to a controlled demo dataset, not real people without consent.

### 9. Network Intrusion Detection — *Hard*
- **Approach:** Pretrained/public-notebook models trained on **NSL-KDD** or **CICIDS2017** datasets are widely available as reference implementations; realistically for a 1-month project, treat this as feeding a **sample/historical** pcap or flow-log dataset through a pre-trained classifier rather than live network monitoring.
- **Output:** Flagged flows/sessions with attack-type classification.

### 10. Server / Firewall / Log Analysis — *Medium*
- **Approach:** Parse uploaded log files (syslog, firewall logs), apply rule-based anomaly detection (failed login bursts, port-scan patterns) plus an unsupervised anomaly detector (Isolation Forest) for "unusual but not rule-matched" events. (Full ELK/Graylog deployment is likely out of scope for 1 month — treat as a stretch goal.)
- **Output:** Flagged log lines, anomaly score over time.

### 11. Scam / Spam Message Detection — *Easy, build this first*
- **Approach:** Pretrained spam classifier (many published on HuggingFace, or train a quick Naive Bayes / logistic regression on the classic **SMS Spam Collection dataset** — small, fast, reliable for a demo).
- **Output:** Spam/scam probability, flagged keywords.

### 12. Cyber Triage & Investigation — *This is the aggregator, build it last*
- **Approach:** Not a standalone detector — this module pulls findings across all other modules attached to a case, computes an overall case risk score (see [§8](#8-risk-scoring--report-generation)), and can optionally use an LLM (e.g., the Claude API) to turn structured findings into a plain-language investigator summary and recommended next steps.
- **Output:** Case-level risk summary + recommendations.

**Given 12 modules and a 1-month timeline for 4 people, treat modules 2, 6, 7, 11 as your realistic MVP set** (all have free APIs or small, well-documented datasets and no heavy infra), modules 1, 4, 5, 8 as a stretch "phase 2" tier, and 3, 9, 10 as phase-3/demo-only (seeded with pre-canned sample data rather than live processing) unless the team has spare time. Module 12 should be built once at least 2–3 other modules exist to aggregate from.

---

## 7. Blockchain-Inspired Evidence Integrity

**Design:**
1. On evidence upload, compute `SHA-256(file bytes)` → `evidenceHash`.
2. Create an `IntegrityLog` entry: `{ evidenceId, action: "UPLOAD", entryHash: SHA-256(prevHash + evidenceHash + timestamp + actorId), prevHash, timestamp, actorId }`.
3. Every subsequent action on that evidence (viewed, re-hashed, analyzed, exported) appends a new `IntegrityLog` entry, each entry's hash depending on the previous entry's hash — this is what makes the chain "blockchain-inspired": altering any past entry breaks every hash after it.
4. **Verification:** recompute the chain from entry 1 forward; if any `entryHash` doesn't match, integrity is broken — surface this clearly in the UI ("⚠️ Evidence integrity check failed at step N").
5. Optionally sign each entry with a server-held private key (HMAC or RSA) so even direct database tampering is detectable without needing a separate ledger.


---

## 8. Risk Scoring & Report Generation

- Each `Analysis` result includes a per-module risk contribution (e.g., 0–100).
- The **Cyber Triage module** aggregates all analyses tied to a case into an overall case risk score 
- **Report Generation:** template an HTML report (case summary, evidence list with hashes, per-module findings, overall risk, recommendations) and render to PDF via Puppeteer. 
---

## 9. Authentication & Role-Based Access Control

| Role | Permissions |
|---|---|
| `admin` | Full access: user/role management, module/model management, system monitoring, audit logs |
| `investigator` | Create/manage cases, assign evidence, view all findings/reports for assigned cases |
| `analyst` | Upload evidence, run analysis modules, view findings — no case/user management |
| `viewer` | Read-only access to assigned cases and reports |

Implemented with JWT access tokens (short-lived) + refresh tokens, middleware-enforced role checks on each Express route, and log every privileged action to `AuditLog`.

---

## 10. API Surface (High-Level)

```
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/refresh

GET    /api/cases
POST   /api/cases
GET    /api/cases/:id
PATCH  /api/cases/:id

POST   /api/evidence/upload          (also triggers hashing + IntegrityLog entry)
GET    /api/evidence/:id
GET    /api/evidence/:id/integrity   (verify hash chain)

POST   /api/analysis/:module/run     (module = one of the 12; proxies to AI/ML service)
GET    /api/analysis/:id

GET    /api/reports/:caseId
POST   /api/reports/:caseId/generate (returns PDF)

GET    /api/notifications
GET    /api/audit-logs               (admin only)
```

The AI/ML service exposes its own internal API (`POST /ai/{module}/analyze`), called only by the Core API, never directly by the frontend.

---

## 11. Deployment Strategy

**Frontend on Vercel:**
- Deploy the React app to Vercel, pointing its API base URL at your homelab's public endpoint.

---

## 12. Project Folder Structure

```
cyberfusion/
├── frontend/                 # React + Vite app (→ Vercel)
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── context/
│   │   └── api/
│   └── package.json
├── core-api/                 # Node.js + Express
│   ├── src/
│   │   ├── routes/
│   │   ├── controllers/
│   │   ├── models/            # Mongoose/Prisma schemas
│   │   ├── middleware/        # auth, RBAC, audit logging
│   │   └── services/          # calls to ai-service, integrity chain logic
│   └── package.json
├── ai-service/                # Python + FastAPI
│   ├── app/
│   │   ├── modules/
│   │   │   ├── phishing/
│   │   │   ├── fraud/
│   │   │   ├── spam/
│   │   │   ├── fake_news/
│   │   │   ├── image_forgery/
│   │   │   ├── face_match/
│   │   │   ├── malware/
│   │   │   ├── ... (one folder per module)
│   │   ├── main.py
│   │   └── requirements.txt
├── docker-compose.yml
└── README.md
```

---

## 13. Security & Privacy Considerations

- **Evidence confidentiality:** encrypt evidence at rest (disk-level or MinIO server-side encryption) and in transit (HTTPS everywhere).
- **Access control:** enforce RBAC at the API layer, not just the UI.
- **PII handling:** evidence may contain real personal data (emails, chat logs, photos) — for an academic demo, use synthetic/sample data rather than real people's information wherever possible, especially for the Face Matching module.
- **Audit everything:** every read/write on `Case`, `Evidence`, and `Analysis` should produce an `AuditLog` entry — this is both a feature and a defensibility requirement for a forensics tool.
- **Secrets management:** API keys (VirusTotal, Google Safe Browsing, LLM provider) via environment variables / `.env`, never committed to git.

---

## 14. Testing Strategy

- **Unit tests:** Core API route/controller logic (Jest), AI service module logic (pytest).
- **Integration tests:** evidence upload → hash → analysis → finding pipeline end-to-end.
- **Hash-chain verification tests:** deliberately tamper with a stored entry in a test DB and confirm the verification endpoint detects it.
- **Manual QA checklist** before demo: one full walkthrough per role (admin, investigator, analyst, viewer).

---

## 15. Datasets & Pretrained Resources

| Module | Suggested resource |
|---|---|
| Phishing URLs | Google Safe Browsing API, VirusTotal API |
| Credit Card Fraud | Kaggle "Credit Card Fraud Detection" dataset |
| Spam/Scam | SMS Spam Collection Dataset (UCI) |
| Fake News | Kaggle "Fake News" dataset, LIAR dataset |
| Malware/APK | MobSF (open-source static analysis), VirusTotal API |
| Face Matching | `face_recognition` (dlib) or DeepFace Python libraries |
| Image Forgery | CASIA v2 dataset (for reference models), ELA algorithm (no dataset needed) |
| Network Intrusion | NSL-KDD, CICIDS2017 datasets |
| Fake Social Profiles | Kaggle "Fake and Real Profile Detection" style datasets |

