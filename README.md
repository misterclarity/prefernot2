# prefernot2

A single-file, dependency-free web chat UI for a local LLM served by
[llama.cpp](https://github.com/ggml-org/llama.cpp), wired to the OpenAI-compatible
`/v1/chat/completions` endpoint. The model is given a persona that would rather you
went and Googled it.

## Run it

Open `index.html` in a browser — double-clicking the file works, no build step, no
server needed for the page itself.

Two servers are configured out of the box, picked from the **Server** dropdown in
Settings:

| Server | Note |
| --- | --- |
| `https://linuxllm.ling-escalator.ts.net/llm-api/v1` | Tailscale HTTPS, works from any page |
| `http://100.126.69.66:8080/v1` | plain HTTP — see below |

Whichever is selected is the **API base URL**; `/chat/completions` and `/models` are
derived from it, so pasting either a bare origin or a full `…/chat/completions` URL
works. Type a URL that is not in the list and save, and it joins the list; **forget this
one** removes it. The two built-ins always come back. Switching servers resets the model
to Auto, since a model pinned on one server means nothing on another. The header shows
which server is live, and everything is remembered in `localStorage`.

> **HTTP servers and HTTPS pages.** A page served over HTTPS — including
> `https://…github.io` — cannot call a plain `http://` API: the browser blocks it before
> the request is sent, and no code can work around it. The `100.126.69.66:8080` entry
> therefore only works when the page itself is opened over `file://` or `http://`. Run
> the connection test and it says exactly this rather than leaving you guessing. To use
> that server from an HTTPS page, put it behind TLS (a Tailscale `serve` proxy does
> this) and add the HTTPS URL as a new entry.

## You don't have to know the model name

That host serves several models, so the app works out which one to ask for:

1. `GET {base}/models` — if it advertises exactly one model, that is the answer.
2. If it advertises several, `GET {base}/running` (llama-swap) says which is actually
   loaded, and that one wins.
3. Otherwise the first advertised model is used.
4. If nothing can be resolved, the `model` field is omitted entirely — single-model
   servers accept that, and no name beats a wrong name.

If a request is rejected with a model error (the server swapped models under you), the
app re-detects and retries once, preferring what the server now advertises over a name
pinned in settings. The Settings dropdown lists every advertised model if you want to
pin one anyway; **Auto** is the default and shows which model it resolved to.

## Reasoning models

Qwen3, DeepSeek-R1 and similar models think before they answer, and llama.cpp returns
that thinking in a separate `reasoning_content` field (or, with some chat templates,
inlined as `<think>…</think>` in the content). Both shapes are handled:

- Thinking streams live into a collapsible block above the reply, so the bubble is
  never blank while the model works. It auto-collapses the moment the real answer
  starts, and can be turned off in Settings.
- Thinking is display-only. It is stored with the conversation but never sent back to
  the server, and an assistant turn that produced no answer is left out of the next
  request entirely.
- An empty `content` is treated as valid — `"" || fallback` quietly discards a real
  answer, so every read is an explicit type check.
- If the model spends its whole token budget thinking and never reaches an answer
  (`finish_reason: "length"`), the app says exactly that and offers a button that
  raises Max tokens and retries. The default is 2048 for this reason.
- **Ask the server to skip thinking** sends `chat_template_kwargs: {enable_thinking:
  false}`, which Qwen-style templates honour. It is off by default, since servers that
  don't know the field may reject it.

## When it won't connect

**Settings → Test connection** (or click the status chip in the header) runs the whole
path and logs each step with timings:

1. Parses the base URL and prints the endpoints it derived, warning if there is no `/v1`.
2. Reports this page's own origin — `file://` pages send `Origin: null`, which some
   servers reject.
3. Fails fast on an HTTPS page calling an HTTP API, which browsers block outright.
4. `GET /models`, with per-status advice for 404 / 401 / 403 / 502.
5. On a network-level failure, re-probes with `mode: 'no-cors'` to tell
   *host unreachable* (DNS, TLS, Tailscale logged out, refused connection) apart from
   *reachable but CORS-blocked* — `fetch()` alone cannot distinguish them.
6. A real `POST /chat/completions`, checking the response is OpenAI-shaped. It asks for
   64 tokens, not 1 — a one-token probe on a reasoning model only ever proves the model
   can start thinking — and reports the reply, the thinking and `finish_reason`,
   warning when thinking consumed the whole budget.
7. A streaming probe that catches a reverse proxy buffering the SSE stream.

Failing steps print a `→` line with the fix.

**Copy log** puts the whole thing on the clipboard — including the parts the panel
abbreviates. Long server responses are shortened on screen (with a note saying how many
characters were held back) but stored in full, so a 3KB stack trace from the model
server survives the copy. The copied text carries a header with the timestamp, the base
URL tested, this page's origin and the browser, and the button confirms how many lines
went across. If the browser refuses clipboard access — which happens on non-secure
origins and when the page isn't focused — it says **Copy blocked** and drops the whole
log into a pre-selected textarea instead of failing quietly.

## What else it does

- **Streaming replies** over SSE (`stream: true`), with an automatic fallback to the
  plain `choices[0].message.content` response when a server ignores the flag.
- **Stop button / Esc** aborts an in-flight generation and keeps whatever streamed in.
- **Multi-turn context** — the whole conversation is resent each request, with the
  persona always as `messages[0]`.
- **Persistence** — conversation and settings survive a reload (toggleable).
- **Settings panel** — base URL, model dropdown, temperature, max tokens, streaming
  toggle, persistence toggle, and an editable system prompt with a "reset persona"
  button.
- **Connection status** — polls `GET {base}/models` every 30s and shows the model in
  use; click it to run the connection test.
- **Per-message actions** — copy any message, regenerate the last reply, clear the chat.
- **Timings** — time-to-first-token and total latency under each reply.
- Minimal markdown rendering (fenced code, inline code, bold, italic). Model and user
  text is HTML-escaped before anything is rendered.

## Notes

- `llama-server` sends permissive CORS headers by default, so opening the page from
  `file://` or another origin works. Behind a reverse proxy — which a Tailscale
  `serve` setup is — you may need to add `Access-Control-Allow-Origin` yourself. The
  connection test tells you when that is the problem, and for which origin.
- A `*.ts.net` address only resolves while this machine is logged into the same
  tailnet.
