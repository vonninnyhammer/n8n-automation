# n8n-automation

A hardened, self-hosted **n8n** stack: n8n plus its task sandbox (API + runner +
Docker-in-Docker sandbox), a separate code runner, and SearXNG — wired so that
**only n8n publishes a port**. Every helper service is reachable by Compose
service name and must never be exposed publicly; the sandbox runner in particular
runs privileged Docker-in-Docker and should stay on the internal network.

## Layout

- `compose.yml` — the stack definition (based on n8n's `get-n8n.sh` compose).
- `.env.example` — required secrets; copy to `.env` and fill in.
- `searxng-settings.yml` — internal-only search backend settings.
- `docs/` — field notes from running this stack.

## Field notes

- [Hardening an n8n → local-LLM step](docs/ollama-fanout-hardening.md) —
  diagnosing a step that "timed out" at exactly 300 s, when the real cause was a
  queue: an unbounded parallel fan-out onto a single-threaded model server, plus
  an oversized generation cap.
