# Transpose

**One meaning, six vocabularies.**

Transpose takes a piece of text and re-renders it for a reader with a different
vocabulary — without changing what it says. Paste on the left, watch it come
back on the right with most of the content words swapped, each swapped word
carrying its original underneath in small type.

The point isn't a thesaurus. The point is that **a larger vocabulary is higher
resolution, not fancier packaging**. Moving up the scale lets a reader say a
more exact thing in the same space; moving down unpacks a compressed idea into
its plain parts. Same sentence, same meaning, six different amounts of
precision available to the person reading it.

It's a single HTML file. No build step, no dependencies, no server of its own.

---

## The scale

| Level | Name | Reader |
|---|---|---|
| 1 | Plain | ~age 8. Core everyday words. Where no simple word exists, it *unpacks* — "cephalalgia" becomes "a bad headache" |
| 2 | Everyday | ~age 13. Ordinary conversation, common Latin roots |
| 3 | General | Educated adult. Broadsheet register |
| 4 | Literary | Well-read. Exactness rather than length — "reticent" where you wrote "quiet" |
| 5 | Specialist | Trained in the field. Detects the domain, uses genuine terms of art |
| 6 | Research | Publishing in the field. Journal register, compresses clauses into single terms |

The page accent colour is driven by the level, so the whole interface shifts
temperature as you move up and down. That's deliberate — the level *is* the
content.

---

## Prerequisites

**Required**

1. **An OpenAI-compatible chat completions endpoint.** Anything that speaks
   `POST /v1/chat/completions` with SSE streaming:
   - vLLM
   - Ollama (`http://localhost:11434/v1`)
   - LM Studio (`http://localhost:1234/v1`)
   - llama.cpp `llama-server`
   - OpenAI / any hosted provider

2. **A model that follows format instructions reliably.** Transpose asks the
   model to emit inline `[[original|replacement]]` markup. Models below roughly
   7B tend to drift out of format mid-paragraph and the output stops aligning.
   Anything in the 30B+ class is comfortable. Reference config used during
   development: `intel/Qwen3.5-122B-A10B-int4-AutoRound` on vLLM 0.23.

3. **A modern browser.** Chrome, Edge, Firefox, or Safari. Needs `fetch` with
   streaming response bodies.

4. **A way to serve the file over HTTP.** Opening `transpose.html` directly
   from disk gives the page a `null` origin, which most inference servers will
   reject on CORS grounds. See below.

**Not required:** Python, Node, npm, a database, a build toolchain, an internet
connection (except for the Google Fonts link — the page falls back to system
serif/mono if it can't reach them).

---

## Install

### 1. Point it at your model

Open `transpose.html` and edit the `CONFIG` object — it's the first thing in
the `<script>` block:

```js
const CONFIG = {
  baseUrl: "http://your-host:8000/v1",
  model:   "your-model-name",
  apiKey:  ""            // leave blank if your server doesn't need one
};
```

You can also override all three at runtime with the **Endpoint** button in the
header. Those changes live in the tab only and aren't saved anywhere.

### 2. Deal with CORS

The browser will refuse to call your inference server from a different origin
unless one of you allows it. Pick whichever is less annoying.

**Option A — reverse proxy (recommended).** Serve the HTML and proxy the API
through the same nginx, and the whole CORS problem disappears because
everything is same-origin.

Check your existing routes first — if the inference endpoint is *already*
proxied through the same nginx behind some other path (a dashboard route, an
admin prefix), you don't need a new block at all. Just point `baseUrl` at that
existing path. A `location /gpu/ { proxy_pass http://127.0.0.1:9101/; }` block
already makes port 9101 reachable at `/gpu/…` on the main origin, so an API
living at `9101/vllm/v1` is reachable at `/gpu/vllm/v1` with zero config
changes.

If you do need a new block:

```nginx
location /public/ {
    alias /home/you/Public/;
    try_files $uri $uri/ =404;
}

location /vllm/ {
    proxy_pass http://127.0.0.1:9101/vllm/;   # your inference endpoint
    proxy_set_header Host $host;
    proxy_http_version 1.1;
    proxy_set_header Connection "";   # NOT "upgrade" — this is SSE, not websockets
    proxy_buffering off;              # required or the stream arrives in one lump
    proxy_cache off;
    gzip off;                         # gzip re-buffers SSE and stalls it
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;
}
```

Then set `baseUrl: "/vllm/v1"` — a relative path. Reload nginx and open the
page. Adjust the upstream port and path prefix to match your own server; if
vLLM is running bare on 8000, that line becomes
`proxy_pass http://127.0.0.1:8000/v1/;` and `baseUrl` becomes `"/v1"`.

> Three things people get wrong here. `proxy_buffering off` is the big one —
> with it on, nginx holds the whole response and output appears all at once at
> the end. `Connection ""` matters if you copied the block from a websocket
> proxy: SSE doesn't upgrade, and passing upgrade headers can confuse the
> stream. And if a longer prefix location already matches your API path, it
> wins — nginx picks the longest prefix match, not the first one written.

**Keeping the key out of the browser.** Once nginx is in the path, you can have
it attach the credential server-side instead of shipping it in a file anyone can
view-source:

```nginx
    proxy_set_header Authorization "Bearer YOUR_SECRET_KEY";
```

Add that inside the proxy location and leave `apiKey: ""` in the HTML. The
trade-off is that anyone who can reach the proxy can now use the model without
authenticating — fine on a private or campus-internal host, not fine on the
open internet.

**Option B — open up the inference server.**

vLLM:
```
--allowed-origins '["*"]' --allowed-methods '["*"]' --allowed-headers '["*"]'
```

Ollama: set the environment variable `OLLAMA_ORIGINS=*` before starting.

LM Studio: CORS is a toggle in the server settings panel.

Then serve the HTML from anywhere — even `python3 -m http.server 8080` in the
directory containing the file.

### 3. Open it

That's the whole install.

---

## Using it

- **Paste text** into the left pane and press **Transpose** (or `Ctrl`/`Cmd` +
  `Enter`).
- **Click a level** on the scale to change the reader, then Transpose again.
- **Diff** is the default view: source and output in aligned sentence rows with
  a numbered gutter, unchanged sentences dimmed, changed ones tinted and marked
  with a swap count. **Changes only** hides unchanged rows. Turn Diff off for
  continuous flowing prose in two panes instead.
- **Hover any swapped word** — its partner lights up in the other column.
- **Click a swapped word** — a small popover asks the model why that word fits
  this reader better. It's a separate, tiny request, so it's fast.
- **Gloss** toggles the small original-word text underneath each swap. Turn it
  off for a clean read of the output.
- **Field lens** — type a domain (`cardiology`, `machining`, `contract law`,
  `metallurgy`) and levels 5–6 will use that field's actual vocabulary rather
  than guessing from context.
- **Ladder** renders one sentence at all six levels stacked, top to bottom.
  This is the demo — it shows the whole range in a single screen.

The stats strip underneath reports word count, percentage of words swapped,
Flesch-Kincaid grade level before → after, and syllables per word. The grade
delta is usually the number that lands with an audience.

---

## How it works

The model is asked to return the rewrite with substitutions marked inline:

```
The [[quick|expeditious]] brown fox [[jumped over|vaulted]] the [[lazy|indolent]] dog.
```

Both panes are then rendered from that single string — left pane takes the
first half of each pair, right pane takes the second. That's why the alignment
and hover-pairing are exact rather than approximate, and it costs far fewer
tokens than returning JSON or returning both versions in full.

Some details worth knowing if you plan to modify it:

- **Substitutions can span multiple words on either side.** That's what makes
  levels 1 and 6 work — level 1 expands (`[[cephalalgia|a bad headache]]`),
  level 6 compresses (`[[her chest hurt on exertion|exertional angina]]`).
- **The diff aligns by sentence, not by scroll position.** Because the prompt
  fixes sentence count and order, each sentence pairs one-to-one across the two
  columns. Rendering them as rows of a shared grid means the columns are
  structurally aligned and there's only one scroller. Proportional scroll-sync
  of two free-flowing panes would drift badly here, since level 1 makes the
  output substantially longer and level 6 makes it shorter. The flow view still
  uses proportional sync, because it has no rows to align to.
- **The sentence splitter guards against the usual traps** — abbreviations
  (`Dr.`, `p.m.`, `et al.`), decimals (`3.14`, `12.5%`), and initials
  (`J. R. Tolkien`) don't start new rows. Add to the `ABBR` regex if your
  domain has its own.
- **Streaming is parsed incrementally.** An unclosed `[[` at the end of the
  buffer is held back rather than rendered as literal text, so you never see
  half-formed markup flicker on screen.
- **Input is split on blank lines** and processed one paragraph at a time. This
  keeps each response short enough to stay in format, and gives you visible
  progress on slow local hardware.
- **Thinking is disabled per request** via
  `chat_template_kwargs: {enable_thinking: false}`. On a reasoning model like
  Qwen3.5, thinking tokens are pure latency for a rewrite task — this is the
  difference between a few seconds and a minute. Harmlessly ignored by servers
  that don't support it.
- **`delta.content` may be `null`** on reasoning models mid-stream. The parser
  skips those frames rather than crashing.
- **Alignment drift check.** After each run, the app reconstructs your original
  from the left-hand side of every pair and compares it to what you pasted. If
  the model paraphrased something it should have echoed, an `alignment drift`
  badge appears. The output is still valid — it just means the left pane is a
  reconstruction rather than a literal copy.

### Tuning

| What | Where |
|---|---|
| Level definitions and their wording | `LEVELS` array — the `spec` field is the actual instruction sent to the model |
| Level colours | `LEVELS[].color` |
| Temperature (default 0.35) | `callModel()` |
| Output token ceiling | `run()` — scales with input length |
| The whole prompt | `systemPrompt()` |

The `spec` strings are the highest-leverage thing in the file. If output at a
given level isn't landing the way you want, rewrite that level's `spec` before
touching anything else.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| `Failed to fetch` / CORS error in console | Origin isn't allowed — see step 2. Different port counts as a different origin |
| Output appears all at once at the end | `proxy_buffering off` missing from nginx, or `gzip` still on for that location |
| Stream opens then stalls | Proxy is sending websocket upgrade headers — set `Connection ""` |
| `404` on the request | `baseUrl` needs the `/v1` suffix |
| `401` / `403` | API key missing or wrong |
| Literal `[[word\|word]]` visible in output | Model broke format. Try a larger model, or lower the temperature |
| Very few swaps | Text is already at that level. Try moving two or three levels away |
| Long pause before any text appears | Thinking wasn't suppressed, or the server is loading the model |
| `alignment drift` badge | The model paraphrased instead of echoing your source. Cosmetic; output is fine |
| Whole document takes minutes | Expected on local hardware. At ~27 tok/s a 2,000-word document is several minutes. Test on one section first |

---

## Notes

- The API key is in client-side JavaScript. On a local or campus network that's
  fine. Don't put a paid provider's key in this file and then host it publicly.
- Nothing is stored — no localStorage, no cookies, no telemetry. Reloading the
  page clears everything.
- Readability scores are computed in the browser with a syllable heuristic.
  They're directionally right and good for demonstration, not for anything that
  needs to be defensible.
- Responsive down to phone width; panes stack vertically. Keyboard focus is
  visible throughout and `prefers-reduced-motion` is respected.
