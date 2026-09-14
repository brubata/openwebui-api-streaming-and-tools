# Tools *and* streaming over the Open WebUI API — without a browser

Working recipe for making Open WebUI actually execute tools and stream tokens when you
call it from your own client instead of the web frontend.

Derived from the Open WebUI 0.11.3 source and running in production on a voice assistant.

**Status: posted as-is, not maintained.** This is what worked for me. No warranty, no support,
no promise it still applies on a later release. Use it, fork it, correct it.

If you got here searching for any of these, you are in the right place: tool calling does not
work over the Open WebUI API, `tool_ids` has no effect, the backend returns the tool-call JSON
instead of executing it, `/api/chat/completions` returns `null`, the model announces "let me
check" and then stops, no token streaming without the browser.

Discussion thread this came from:
https://github.com/open-webui/open-webui/discussions/4663

---

## The problem

Open WebUI's tool loop and its token stream both live behind the frontend's websocket. Calling
`/api/chat/completions` from your own client gets you a model that announces "let me check the
weather" and then stops, or a body containing the single word `null`.

Tested on Open WebUI 0.11.3, native function calling, an OpenAI-compatible upstream model.

## 1. Make OWUI actually run your turn

OWUI only runs its real pipeline — the native tool loop included — when it can build an *event
emitter*, and it builds one only when your request carries a chat id and a message id.
Everything below goes **top level** in the POST body, not inside `params`:

| Field | Value | Why |
| --- | --- | --- |
| `chat_id` | the chat's id | needed for the event emitter |
| `id` | the **assistant** message id you generated | becomes `metadata["message_id"]` |
| `user_message` | the whole user message **object** | without it the assistant is stored with `parentId: None` and the model sees only the current sentence |
| `tool_ids` | `["your_tool", ...]` | popped at `middleware.py:2722` |
| `features` | `{"web_search": true}` | read at `:2646` / `:2669` |
| `stream` | `true` | `non_streaming_chat_response_handler` contains the string "tool" zero times — non-streaming can never call a tool |

Create the user and assistant messages in the chat record yourself first
(`POST /api/v1/chats/new`, then `POST /api/v1/chats/{id}`), so the ids you send already exist.

Source notes worth keeping: the entire body of `streaming_chat_response_handler` sits inside
`if event_emitter:` (`middleware.py:4252`), and `get_event_emitter_and_caller` needs only
chat_id + message_id — its own comment says it "works for backend-initiated calls (automations,
API)". `session_id` is only required for *builtin* tools, which check that the request came from
the UI.

## 2. The HTTP response is empty — that is not an error

With the fields above, the HTTP body comes back as one line: `null`. OWUI has taken the turn
over and will answer on its own websocket, writing the finished message to the database at the
end.

**The two failure states, so you can recognise them:** no `chat_id` → you get text back but no
tool ever runs. With `chat_id` → tools run but the body is empty. No payload gives you both; the
reply must be collected out of band.

The simple collector is polling: `GET /api/v1/chats/{chat_id}` every 0.5 s until
`history.messages[assistant_id].done` is true, then read the text. Read both shapes — OWUI's
final write stores `output` blocks (`type: "message"` → `content` list of `output_text` items)
and usually, but not always, a plain `content` string.

Polling works and needs no extra dependency. It can never give you partial text, because OWUI
writes the row once, at the end.

## 3. Streaming: join the socket.io channel

For token-by-token output, connect a socket.io client of your own.

```python
import socketio

sio = socketio.Client()
sio.connect("http://localhost:3000",
            socketio_path="/ws/socket.io",
            auth={"token": JWT},          # see section 4 - not an API key
            transports=["websocket"])
```

Every event arrives under **one** event name, `events`, shaped like:

```json
{"chat_id": "...", "message_id": "...", "data": {"type": "...", "data": {...}}}
```

`message_id` matches the assistant id you sent, so filtering to your own turn is exact. The
sequence of a real turn:

```
chat:active  →  chat:title  →  N × response:completion  →  chat:completion (done)  →  chat:outlet
```

The text lives in the inner payload of `response:completion`:

```python
@sio.on("events")
def on_event(msg):
    if msg.get("message_id") != my_assistant_id:
        return
    inner = msg.get("data") or {}
    payload = inner.get("data") or {}
    if inner.get("type") == "response:completion":
        if (payload.get("type") or "").endswith("output_text.delta"):
            emit(payload["delta"])                 # <- your text, word by word
    elif inner.get("type") == "chat:completion" and payload.get("done"):
        finish()
```

Two traps in that snippet:

- **Reasoning streams on the same channel**, as `response.reasoning_text.delta` with its own
  `item_id`, and it arrives first and in bulk. Match on `output_text.delta` only, or you will
  render the model's private thoughts.
- The delta shape follows your **upstream provider**, not OpenAI's chat format. Ours is the
  Responses API shape (`{"type": "response.output_text.delta", "delta": "..."}`), not
  `choices[0].delta.content`. Log one raw event before writing the parser.

Test `done` on `chat:completion` rather than the event type alone — a chunk of the same type
arrives just before the final one.

## 4. The token: a JWT, not an API key

The socket handler decodes a JWT and nothing else, so an `sk-...` API key authenticates your
REST calls but silently gives you a connection that receives nothing. Sign in at
`POST /api/v1/auths/signin` and use the `token` from the response.

If you host OWUI yourself and the password is lost, you can mint one: the signing secret is in
the container at `/app/backend/.webui_secret_key` (or the `WEBUI_SECRET_KEY` env var), the user
id is in `webui.db`, and the token is just `HS256({"id": user_id})` — about fifteen lines with
`hmac` and `base64`, no PyJWT needed. Verify it with `GET /api/v1/auths/`.

## 5. What this buys

Speech (or rendering) starts at the first sentence instead of after the last word. Measured on
our bridge: reasoning began at 21:35:02 and the first spoken word went out at 21:35:05, where
polling would have waited for the complete reply. On a multi-sentence answer that is several
seconds of dead air removed from every turn.

If you are feeding a TTS, buffer deltas and cut on sentence boundaries, merging fragments under
~16 characters so abbreviations like "Dr." don't become their own trip to the synthesizer. Keep
one audio output stream open for the whole reply rather than one per sentence — the seam is
audible otherwise.

## 6. Summary

1. Send `chat_id`, assistant `id`, the whole `user_message`, `tool_ids`, `features`,
   `stream: true`.
2. Expect an empty HTTP body. Poll the chat record, or —
3. Connect socket.io at `/ws/socket.io` with a **JWT**, listen to `events`, filter on
   `message_id`, read `output_text.delta`, stop on `chat:completion` + `done`.
4. Never speak the reasoning deltas.

Keep polling as the fallback. If the socket is down, a slow answer beats no answer.
