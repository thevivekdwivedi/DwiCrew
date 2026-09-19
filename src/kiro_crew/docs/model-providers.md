# Connecting other model providers (Gemini, Ollama, Claude, and more)

Kiro Crew's agent model always arrives through the ACP protocol, and `agent.provider`
stays `"acp"` no matter which model serves you — the choice you actually have is
*which ACP agent* Kiro Crew drives (`agent.acp_backend`), and two of the built-in
choices, **OpenCode** and **goose**, are themselves multi-provider agent CLIs. Point
one of them at Anthropic, OpenAI, Google Gemini, or a local Ollama model using that
harness's own provider configuration, and Kiro Crew drives it exactly as it drives
kiro-cli today.

This is a supported path, not the default one. kiro-cli remains the fully-supported
backend; OpenCode and goose are separately-maintained CLIs Kiro Crew adapts to over
ACP, so some session features (live subagent progress, mid-turn steering, and the
managed-MCP niceties kiro-cli/KAS offer) may not carry over one-for-one. See the
**ACP Backend** section of [configuration.md](configuration.md) for what each backend
is verified to support.

## 1. Install the harness

Pick the CLI that already supports the provider you want, then install it as an
ordinary command-line tool (Kiro Crew does not bundle either):

- **OpenCode**: `npm i -g opencode-ai`
- **goose**: `curl -fsSL https://raw.githubusercontent.com/block/goose/main/download_cli.sh | bash`

`kirocrew doctor` checks for whichever binary your `agent.acp_backend` names and
prints the install command above automatically if it is missing.

## 2. Configure the provider on the harness's own side

Kiro Crew does not manage provider credentials for OpenCode or goose — sign in (or
point at a local model) using that CLI directly, the same way you would if you ran
it outside Kiro Crew:

- **OpenCode**: run `opencode auth login` in a terminal to sign in to a hosted
  provider (Anthropic, OpenAI, Google, etc.); credentials land in
  `~/.local/share/opencode/auth.json`. For a locally served model (Ollama, LM
  Studio, ...), name it in that project's `opencode.json` instead — no sign-in step
  needed.
- **goose**: set `GOOSE_PROVIDER` / `GOOSE_MODEL` (and any provider API key goose
  expects, e.g. via its own env vars) in `~/.config/goose/config.yaml`, or store a
  secret in `~/.config/goose/secrets.yaml`. See goose's own documentation for the
  provider ids it supports and the exact variable names — Kiro Crew passes your
  environment through to the goose process unmodified, so anything that works when
  you run `goose` by hand works here too.

For Ollama specifically, either CLI just needs to reach your local Ollama server
(default `http://localhost:11434`) — no API key required.

## 3. Point Kiro Crew at the harness

```bash
kirocrew config set agent.acp_backend opencode   # or: goose
```

An unrecognized value falls back to the default backend (kiro-cli) with a warning
in the log, so a typo never leaves the gateway unable to start.

## 4. Pick a model

Leave `agent.model` at `"auto"` to defer to the harness's own default model, or set
it to a specific id once you see it in the model picker (Settings → Chat → Model,
or `GET /api/models`) — both harnesses advertise `provider/model` ids, for example
`ollama/llama3.2:3b`, `anthropic/claude-...`, or `openai/gpt-4o`, depending on what
you configured in step 2:

```bash
kirocrew config set agent.model auto
```

Kiro Crew never hardcodes or validates these ids against a static list; it only
offers whatever the harness itself advertises for the credentials/config you gave
it in step 2, so the picker is always a live reflection of what will actually work.

## 5. Restart and verify

Restart the gateway so it picks up the new backend, then run `kirocrew doctor` to
confirm the binary resolves. Kiro Crew still enforces its own tool-approval gate on
every call through either harness (it seeds OpenCode's `permission` setting and
goose's `GOOSE_MODE` to require asking before running a tool), so switching
providers does not weaken that guarantee.
