# Corvid Echo — Team Work Breakdown & Integration Contract

**Status:** Draft v0.1
**Team:** Anwesha (Messenger), Megha (Calling), Kamya (AI analysis + troubleshooting session), Aneesha (TBD)
**Companion to:** `HLD.md`, `LLD.md`, `USE_CASES.md`

---

## 1. Ownership Map

| Owner | Module | Status |
|---|---|---|
| Anwesha | Messenger (`messenger/`) — conversations, messages, attachments, read receipts | Designed (LLD §2–4.3) |
| Megha | Calling (`calling/`) — WebRTC signaling, call sessions | Designed (LLD §4.2) |
| Kamya | AI analysis of device reports + troubleshooting session with field technician | **Not yet designed** — see §5 |
| Aneesha | TBD | Pending |

Each module package depends only on `platform` and its own repository (LLD §1) — no cross-context repository access, ever. Cross-module calls go through a service interface. This is what makes the split below possible without merge conflicts.

---

## 2. Staging Strategy: Transport First, Persistence Later

Rather than waiting on Postgres schema + Flyway setup before anyone can write code, each owner builds against a **repository interface** from day one, backed first by an in-memory stub, then swapped for a real Postgres implementation once the transport/API behavior is proven. The controller and service layers never change between the two phases — only the implementation behind the interface does.

| Phase | Goal | Storage |
|---|---|---|
| **Phase 1 — Prove the transport** | Get send/receive and call initiate/receive working end-to-end against the real API shapes and the real-time (WebSocket) path | In-memory (`ConcurrentHashMap` or equivalent), no Postgres/Flyway needed |
| **Phase 2 — Add persistence** | Swap the stub repository for a JPA/Postgres-backed one; add the Flyway migration; nothing above the repository layer changes | Postgres (Cloud SQL), per LLD §2 schema |

Calling needs Phase 2 even less than Messenger does: per LLD §4.2, the live call (offer/answer/ICE exchange) never touches `call_sessions` while in progress — that table only exists to record history afterward. So Megha's entire 1:1 call flow can be built and demoed with zero database involvement; persistence is added purely to support a call-history view.

Messaging needs *some* storage to prove send → store → retrieve, but it doesn't need to be Postgres for Phase 1 — an in-memory list per conversation is enough to validate the API contract and the real-time push path.

---

## 3. Contracts to Freeze on Day 1

Freezing these before anyone writes code is what lets four people work in parallel without integration surprises later. None of these require the database to exist.

### 3.1 REST API shapes (already specified, LLD §3)
`POST /conversations`, `POST /conversations/{id}/messages`, `GET /conversations/{id}/messages` (cursor-based — keep the cursor contract real even when the data behind it is an in-memory list), `PUT /conversations/{id}/read`, `POST /conversations/{id}/calls`, `WS /v1/signaling`.

### 3.2 Real-time event envelope (new — needed for Phase 1)
Whether or not the Realtime Gateway unification (LLD §10) is approved yet, Anwesha and Megha should agree on one event shape now, so that if/when the two WS channels merge, it's a routing change, not a redesign:

```json
{
  "type": "message.new | message.read | call.incoming | call.offer | call.answer | call.ice | call.end",
  "conversationId": "uuid",
  "senderId": "uuid",
  "payload": { }
}
```

Each owner can run this over their *own* WS endpoint independently in Phase 1 (Anwesha: a temporary messaging WS; Megha: the existing `/v1/signaling`). Merging onto one Realtime Gateway later only requires agreement on this envelope, not a rewrite.

### 3.3 Repository interfaces (Phase 1 stub → Phase 2 real, no callers change)

```java
public interface MessageRepository {
    Message save(Message message);
    List<Message> findByConversation(UUID conversationId, Instant beforeCursor, int limit);
    void markRead(UUID conversationId, UUID userId, Instant lastReadAt);
}
// Phase 1: InMemoryMessageRepository (ConcurrentHashMap<UUID, List<Message>>)
// Phase 2: JpaMessageRepository (Spring Data JPA against `messages`, LLD §2)
```

Calling doesn't need a repository interface in Phase 1 at all — see §2. Add `CallSessionRepository` only when Phase 2 starts.

### 3.4 Stubbed cross-module dependencies
Per the dependency map already discussed: Anwesha mocks Notification (push-on-message) and Media (attachment upload) as fake/no-op service interfaces; Megha mocks Notification (push-on-incoming-call). Nobody waits on another module's real implementation to demo their own.

---

## 4. Per-Owner Breakdown

### Anwesha — Messenger
- **Phase 1:** `ConversationRepository` / `MessageRepository` interfaces + in-memory impls; `POST /conversations`, `POST /conversations/{id}/messages`, `GET /conversations/{id}/messages`, `PUT /conversations/{id}/read`; push `message.new` / `message.read` events per the §3.2 envelope over a temporary WS; mocked Notification + Media.
- **Phase 2:** `JpaConversationRepository` / `JpaMessageRepository` against Postgres; Flyway migration for `conversations`, `conversation_members`, `messages`, `message_attachments`; wire real Media (GCS) and Notification.

### Megha — Calling
- **Phase 1:** `/v1/signaling` WS — connect, SDP offer/answer relay, ICE candidate relay; `POST /conversations/{id}/calls` (callId can be a plain in-memory UUID, no DB row needed); an in-memory active-calls map for routing only (not persistence).
- **Phase 2:** `CallSessionRepository` / `CallParticipantRepository` + Postgres impl; persist start/end/duration for call history; Flyway migration for `call_sessions`, `call_participants`.

### Kamya — AI analysis + troubleshooting session
**No HLD/LLD design exists yet for this capability.** The current Vision module (HLD/LLD) is an async photo→OCR→report pipeline only — it has no concept of an interactive "troubleshooting session." Before Kamya's work can be staged into phases like the above, she and Anwesha need a joint decision on the integration point:
- Is the troubleshooting session a bot/system participant inside an existing Messenger conversation (in which case Kamya's Phase 1 rides Anwesha's §3.2 transport contract), or
- A separate, self-contained panel that only *escalates* into a real Messenger conversation on demand (in which case Kamya's Phase 1 is fully independent until that escalation point)?

This decision changes Kamya's dependency graph entirely, so it should happen before her phases are defined here.

### Aneesha — TBD
Placeholder. Fill in once her assignment is known.

---

## 5. Integration & Aggregation Plan

1. Each owner demos their own Phase 1 slice independently (own stub data, own WS endpoint) — no shared infrastructure needed yet.
2. Joint session: review the §3.2 event envelope against real usage from all slices; adjust once, together, before anyone builds Phase 2 against it.
3. Decide who owns the actual WS handler class if/when the Realtime Gateway unifies (likely Megha, since `/v1/signaling` already exists) — Anwesha's events become entries routed through it, not a code merge.
4. Phase 2 persistence swaps happen one owner at a time, each touching only their own repository implementation and Flyway migration file — no shared file edits.
5. Final end-to-end pass: walk through `USE_CASES.md` §4's actor workflow live across all four modules as the integration test script.

---

## 6. Open Decisions Before Work Starts

- Confirm the Realtime Gateway unification (LLD §10) — affects whether Megha's WS handler is the single shared one from day one, or two channels are built in parallel and merged later.
- Joint Kamya + Anwesha design session on the troubleshooting-session integration point (currently undocumented anywhere in HLD/LLD).
- Aneesha's assignment.
- Confirm single-instance assumption for Phase 1 in-memory state (no Redis/shared-state needed yet) — revisit only if Phase 1 needs to run on more than one instance.
