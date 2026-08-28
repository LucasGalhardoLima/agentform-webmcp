# agentform-webmcp

Two-sided forms: the creator builds a form with their agent; the published form
exposes **WebMCP** tools so the respondent's agent can read, fill and submit it.

Built for the [OpenAI WebMCP Challenge](https://webmcp.devpost.com/) — submissions
close **3 Sep 2026, 1:00 pm PT (17h BRT)**.

## Status

`SPIKE-01` is complete. See **[`spike/RELATORIO.md`](spike/RELATORIO.md)** for the
full findings and the go/no-go verdict.

## Repo layout

```
spike/
  index.html      throwaway probe page — registers tools, logs every call
  RELATORIO.md    spike report: what each environment actually does
  screenshots/
```

## Running the spike page

WebMCP needs a **secure context**. `localhost` counts, so no tunnel is needed for
local work:

```sh
cd spike && python3 -m http.server 8899
# open http://localhost:8899/index.html
```

### Chrome

- Chrome **149+**. On Chrome 151 the API was already present with no flag toggling
  needed; if `document.modelContext` is missing, enable
  `chrome://flags/#enable-webmcp-testing` and relaunch.
- Tools are inspectable in **DevTools → Application → WebMCP**.
- To drive them with a model, install the
  [Model Context Tool Inspector](https://chromewebstore.google.com/detail/gbpdfapgefenggkahomfgkhfehlcenpd)
  extension (it is separate from Gemini in Chrome).

### ChatGPT

Site tools work in the **ChatGPT desktop app's built-in browser**, ChatGPT Work and
Codex. Not on mobile, not in Enterprise/Edu workspaces. The page must be served over
public HTTPS — the in-app browser cannot reach `localhost`. Users can turn the
feature off under *Settings → Browser → Permissions → Enable site tools*.

The page also works with no WebMCP at all: the banner says so, and the
*Local invocation* buttons call the same `execute` handlers directly.

## License

MIT — see [LICENSE](LICENSE).
