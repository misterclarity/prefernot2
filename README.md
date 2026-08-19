# prefernot2

A single-file, dependency-free web chat UI for a local LLM served by
[llama.cpp](https://github.com/ggml-org/llama.cpp), wired to the OpenAI-compatible
`/v1/chat/completions` endpoint. The model is given a persona that would rather you
went and Googled it.

## Run it

1. Start llama.cpp's server:

   ```sh
   llama-server -m /path/to/model.gguf --host 0.0.0.0 --port 8080
   ```

2. Open `index.html` in a browser — double-clicking the file works, no build step,
   no server needed for the page itself.

The default endpoint is `http://100.119.213.123:8080/v1/chat/completions`. Change it
under **Settings** if your server lives elsewhere; the value is remembered in
`localStorage`.

## What it does

- **Streaming replies** over SSE (`stream: true`), with an automatic fallback to the
  plain `choices[0].message.content` response when a server ignores the flag.
- **Stop button / Esc** aborts an in-flight generation and keeps whatever streamed in.
- **Multi-turn context** — the whole conversation is resent each request, with the
  persona always as `messages[0]`.
- **Persistence** — conversation and settings survive a reload (toggleable).
- **Settings panel** — endpoint, model name, temperature, max tokens, streaming
  toggle, persistence toggle, and an editable system prompt with a "reset persona"
  button.
- **Connection status** — polls `GET /v1/models` every 30s and shows the loaded
  model's name.
- **Per-message actions** — copy any message, regenerate the last reply, clear the chat.
- **Timings** — time-to-first-token and total latency under each reply.
- Minimal markdown rendering (fenced code, inline code, bold, italic). Model and user
  text is HTML-escaped before anything is rendered.

## Notes

- `llama-server` sends permissive CORS headers by default, so opening the page from
  `file://` or another origin works. Behind a reverse proxy you may need to add
  `Access-Control-Allow-Origin` yourself — the UI says so when a request fails that way.
- Some servers reject an unknown `model` value. Either leave the field blank or set it
  to whatever `GET /v1/models` reports.
