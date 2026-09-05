---
name: Drive a Wizehire Scout agent conversation
description: >-
  Hold a multi-turn conversation with the Wizehire Scout recruiting agent over Server-Sent Events,
  including handling the human-approval pause and resuming an approved turn.
api: openapi/wizehire-scout-service-openapi.yml
base_url: https://scout.wizehire.com
operations:
  - agentChat
generated: '2026-09-04'
method: generated
source: openapi/wizehire-scout-service-openapi.yml
---

# Drive a Wizehire Scout agent conversation

`POST /v1/agent/chat` — operationId `agentChat`. Bearer auth required.

This endpoint runs Wizehire's own recruiting agent. You are not the agent here; you are its client.
It can invoke tools, so a turn can have side effects beyond the reply text.

## The request

Body is `AgentChatRequest`:

| Field | Type | Notes |
|---|---|---|
| `message` | string \| null | The user's natural-language message. Optional **only** on a resumed turn. |
| `session_id` | string \| null | You mint this UUID, one per chat, and rotate it on reset. It also doubles as the LangGraph `thread_id`, which is what lets a paused turn resume. |
| `context` | `AgentContext` \| null | Page context: `job_key`, `apply_id`, `user_id`, `page`, `candidate_name`, `job_title`, `registry_set`, `quick_replies`. |
| `conversation_history` | array\<`HistoryMessage`\> \| null | Prior turns, replayed by you on every request. |
| `resume` | `ResumePayload` \| null | Set only to continue a paused turn. |

**Keep the same `session_id` for the whole conversation.** Rotate it only when the user starts fresh.

`conversation_history` entries carry `role` (`user` \| `assistant` \| `tool`), optional `content`,
optional `tool_calls` (each `{id, name, args}`), and optional `tool_call_id` on a `tool` message. The
older `{role, content}` shape still validates — the extra fields are optional.

## The response is a stream

The contract's prose says this returns **Server-Sent Events** with text deltas and tool-use
notifications, even though the declared 200 content type is `application/json` with an empty schema.
That mismatch is real: **the event shapes are not machine-readable anywhere.** Parse the stream
defensively, and do not assume an event name or field you have not seen.

## Handling the approval pause — this is the important part

A turn can **pause for human review** before the agent acts. When it does, do not start a new turn.
Instead:

1. Show the user what the agent wants to do and ask for approval.
2. On approval, POST to the **same endpoint** with the **same `session_id`** and a `resume` payload.
   `message` may be omitted — a resumed turn picks up from checkpointer state and has no new user input.
3. The v1 contract is blunt about semantics: **any resume means "approved, continue."** `ResumePayload.value`
   is reserved for future decision shapes and is passed verbatim into LangGraph's `Command(resume=...)`.
   There is **no way to express a rejection or a modification** in v1.

**Never auto-resume.** The pause exists so a person decides. If the user has not approved, do not send a
`resume`; abandon the turn instead. Wizehire operates under NYC Local Law 144 as an automated employment
decision tool — the approval gate is a compliance control, not a UI nicety.

## Error handling

- **422** — `HTTPValidationError`. Read `detail[].loc`. Deterministic; do not retry unchanged.
- Everything else is undeclared. **Do not retry a failed turn blind** — there is no idempotency key, and
  the agent may already have invoked a tool before the connection dropped. Report the failure and let the
  user decide.
