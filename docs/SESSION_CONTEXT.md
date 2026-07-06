# Corvid Echo — Session Context Transfer Document

> **Purpose:** Hand this document to a new Claude session (or another laptop) to resume exactly where we left off. Paste the contents into the chat and say "continue from this context."

---

## 1. Project Overview

**Name:** Corvid Echo
**Type:** Field technician support system — a mobile app + server platform that lets field technicians submit device status reports (photos), receive AI-analysed diagnostics, communicate with support agents via messaging and calls.

**Stack:**
- **Backend:** Spring Boot (Java), modular monolith, PostgreSQL (Cloud SQL), Google Pub/Sub for async events, Cloud Run (GCP)
- **Mobile:** Android (Kotlin, Jetpack Compose)
- **AI/Vision:** Vertex AI for OCR/device reading from photos
- **Auth:** Google Identity Platform → JWT (access 15min TTL, refresh 14d)
- **Real-time:** WebSocket (`/v1/signaling`), FCM push for offline

**GitHub repo:** `AmitDeb/curious-next`, branch `main`
**Local workspace:** `C:\Users\amitr\Documents\PROJECTS\`
**Daily auto-push:** A scheduled task runs every day to push the PROJECTS folder to GitHub automatically ("Automated daily sync" commits). Do not be alarmed by these commits — they are unrelated to design work.

---

## 2. Team & Module Ownership

| Person | Module | Design status |
|---|---|---|
| **Anwesha** | `messenger/` — conversations, messages, attachments, read receipts | Fully designed (see LLD §2–4.3) |
| **Megha** | `calling/` — WebRTC signaling, call sessions | Designed (LLD §4.2) |
| **Kamya** | AI analysis of device reports + **troubleshooting session** with field technician | ⚠️ Partially designed — async photo pipeline exists (Vision module), but the interactive "troubleshooting session" is NOT yet designed anywhere in HLD/LLD. Needs a joint design session with Anwesha before her phases can be planned. |
| **Aneesha** | TBD | Not yet assigned |

---

## 3. Files Created & Committed to GitHub

All files live in `docs/` inside the repo. Latest commit on `main`: `96eff22`.

| File | Description |
|---|---|
| `docs/HLD.md` | High-Level Design — architecture, modules, infrastructure, SLOs |
| `docs/LLD.md` | Low-Level Design — DB schema, REST API table, sequence diagrams, security, resilience, observability, testing strategy, open items |
| `docs/USE_CASES.md` | Use cases in "As a / I want to / So that" format + actor workflow Mermaid diagram |
| `docs/TEAM_WORK_BREAKDOWN.md` | Team staging plan — Phase 1 (transport-first, in-memory) vs Phase 2 (Postgres), per-owner breakdown, shared WS event envelope contract |
| `docs/Messenger_Design_OnePager.docx` | One-page Word document summarising Messenger design for mentor review |
| `docs/SESSION_CONTEXT.md` | This file |

---

## 4. Key Architecture Decisions Already Made

### 4.1 Module / Package Structure
```
com.corvid.echo
├── identity/        # users, roles, auth
├── device/          # device registry, status reports
├── messenger/       # conversations, messages, attachments
├── calling/         # WebRTC signaling, call sessions
├── media/           # GCS/Drive adapter
├── vision/          # async OCR worker (separate deployable)
├── notification/    # async push worker (separate deployable)
└── platform/        # cross-cutting: security, logging, tracing, config
```
**Rule:** each bounded context depends only on `platform` + its own repository. Cross-context calls go through a package-local **service interface** only — no cross-context repository access, ever.

### 4.2 Role Naming
Renamed `END_USER` → `FIELD_TECHNICIAN` throughout all docs. Three roles: `FIELD_TECHNICIAN`, `SUPPORT_AGENT`, `ADMIN`.

### 4.3 Messenger Data Model (LLD §2)
- `conversations` (type: DIRECT | GROUP)
- `conversation_members` (with `last_read_at` — entire read-receipt mechanism in one column)
- `messages` (soft edit/delete via `edited_at`/`deleted_at`)
- `message_attachments` (junction to `media_objects`)

### 4.4 Messenger Sequence Flow (LLD §4.3)
```
Sender → POST /conversations/{id}/messages
       → DB: check membership, insert message
       → Pub/Sub: publish "message.sent"
       → 201 Created

Pub/Sub → Realtime Gateway (WS) → push to connected recipient
       OR → Notification Worker → FCM push (if offline)

Recipient → GET /conversations/{id}/messages (on reconnect)
          → PUT /conversations/{id}/read {lastReadAt}
          → DB: update last_read_at
          → Pub/Sub: publish "message.read"
          → Realtime Gateway → push read-receipt to sender
```
The key design: the `POST /messages` write path never blocks on delivery. Fan-out is event-driven via Pub/Sub.

### 4.5 Realtime Gateway (Open Item — LLD §10)
Recommendation (not yet confirmed): generalise the existing `/v1/signaling` WebSocket (used for call signaling) into a shared **Realtime Gateway** that multiplexes both call signaling AND message events over one per-device WebSocket connection, using a shared event envelope:
```json
{ "type": "message.new|message.read|call.incoming|call.offer|call.answer|call.ice|call.end",
  "conversationId": "uuid", "senderId": "uuid", "payload": {} }
```
If not approved, Messenger runs its own separate WS temporarily. Decision needed before Phase 2.

### 4.6 New REST Endpoint Added (LLD §3)
`PUT /v1/conversations/{id}/read` — marks conversation read up to a timestamp, updates `conversation_members.last_read_at`.

---

## 5. Staging Plan (from TEAM_WORK_BREAKDOWN.md)

### Phase 1 — Transport First, DB Deferred
Build against a **repository interface**, implemented with in-memory stubs (`ConcurrentHashMap`). No Postgres/Flyway needed to start. Prove the API contract and real-time WS behavior first.

**Anwesha (Messenger) Phase 1:** `ConversationRepository`/`MessageRepository` interfaces + in-memory impls; `POST/GET /conversations`, `POST/GET /conversations/{id}/messages` (cursor-based), `PUT /conversations/{id}/read`; push `message.new`/`message.read` WS events; mock Notification + Media service interfaces.

**Megha (Calling) Phase 1:** `/v1/signaling` WS (SDP offer/answer + ICE relay); `POST /conversations/{id}/calls` (callId = UUID in memory, no DB); in-memory active-calls map for routing. **No DB needed at all for the live call** — `call_sessions` table is only for history.

### Phase 2 — Add Persistence
Swap in-memory repo for `JpaMessageRepository` / `JpaConversationRepository` backed by Postgres (Cloud SQL). Add Flyway migration. Nothing in the controller or service layer changes — only the implementation behind the interface is swapped.

---

## 6. Open Items & Decisions Needed

From LLD §10 and TEAM_WORK_BREAKDOWN.md:

1. **Realtime Gateway confirmation** — shared WS (Megha owns the handler, Anwesha's events route through it) vs separate channels. Affects how Phase 1 WS work is structured.
2. **Kamya + Anwesha joint design session** — define the "troubleshooting session" capability and whether it integrates into Messenger (bot-in-conversation) or escalates from a separate panel into a new conversation.
3. **Aneesha's assignment** — not yet decided.
4. **GCS vs Google Drive** for attachment storage — Drive quota/throughput is a poor fit for chat volume; GCS recommended but not confirmed.
5. **Vision/OCR prototyping spike** — model choice and confidence thresholds not locked.
6. **Design principles doc** — may change module structure or naming.
7. **SLO/DR numbers** (HLD §6/§7) — proposed defaults, not signed off.
8. **Single-instance assumption** for Phase 1 in-memory state — revisit if multi-instance needed (would require Redis for shared state).

---

## 7. Learning Levels Discussed (for Anwesha's own study)

A 5-level progression for building a messaging app, each level producing something runnable:

| Level | Theme | What you build |
|---|---|---|
| 1 | Messages in memory | Spring Boot controller + ArrayList; Android polls GET every 2s |
| 2 | Persistence + history | JPA Message entity, H2/Postgres, cursor-based pagination |
| 3 | Users + conversations | `users`/`conversations` tables, sender_id FK, Android login + conv list |
| 4 | Real-time WebSocket | Spring WebSocketHandler pushes `message.new` JSON; Android OkHttp WS |
| 5 | Read receipts + offline | `last_read_at` col, `PUT /read`, `message.read` event, FCM fallback — this is LLD §4.3 complete |

---

## 8. Environment Notes (for the new laptop's Claude session)

- **Java:** OpenJDK 11 available in sandbox.
- **Android emulator:** NOT available in the Claude sandbox (no KVM/hardware virt). For Android tests use **Robolectric** (JVM-based, runs in sandbox) for unit/integration tests; generate Espresso UI tests but run those on your own machine with Android Studio.
- **Git strategy for this repo:** The sandbox bash mount can be flaky (different commands sometimes see stale file content). Reliable techniques:
  - Use `Write` tool (not bash heredoc) to create/overwrite files.
  - Use `git hash-object -w <file>` + `git update-index --cacheinfo <mode>,<hash>,<path>` when `git add` fails to detect changes.
  - Use pure git plumbing (`git ls-tree`, `git mktree`, `git commit-tree`, `git update-ref`) for merge commits to avoid index corruption from `git read-tree -m`.
  - Use `git reset --mixed HEAD` (not `--hard`) after plumbing operations to avoid touching locked ExoPlanet files.
- **ExoPlanet/ folder:** Exists in the repo — it's a separate, unrelated hackathon project (BAH2026). Do not delete or modify files there. Some `.pptx` files may have `~$...` Office lock files (open in PowerPoint on the user's machine) — ignore them.

---

## 9. Suggested Next Steps

1. **Anwesha:** Start coding Level 1 (Spring Boot in-memory server + Android REST client) to learn the patterns, then migrate toward Phase 1 of the actual Corvid Echo Messenger implementation.
2. **Team:** Schedule the Kamya + Anwesha design session to define the troubleshooting-session integration point before Kamya's phases can be planned.
3. **Megha:** Can begin Phase 1 of Calling independently — the entire live call flow needs no DB and no dependency on Anwesha's work.
4. **All:** Confirm the Realtime Gateway decision (LLD §10 open item) — this is the one architectural choice that affects both Anwesha's and Megha's WS implementations.
5. **Aneesha:** Once role is assigned, add her entry to TEAM_WORK_BREAKDOWN.md.

---

*Generated from design session — Corvid Echo v0.1 | Repo: AmitDeb/curious-next | Branch: main | Latest commit: 96eff22*
