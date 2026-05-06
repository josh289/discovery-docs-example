# Appendix D — Methodology

## Automation vs human review

| Phase | Automated | Human-reviewed |
|---|---|---|
| Build pipeline (`output/`) | Cascade-plan factory: 22 stations across 5 tracks (discovery, shared, backend, frontend, integration) orchestrated by the Pi SDK. Each station runs as an autonomous agent with deterministic quality gates. | Phase-3 clarification answers reviewed by domain experts; spec-readiness gates scored by LLM judge (≥70/100 threshold). |
| Audit pipeline (`audit-output/`) | 13 audit stations grouped into 7 workstreams (W1–W7), all agent-mode with deterministic gates. Station outputs are markdown files with embedded `[source:]` citations for mechanical traceability. | Final synthesis review (executive summary, SOW) cross-checked by human before delivery. |

## Tools

- **Pi SDK** — Agent runtime (`@mariozechner/pi-coding-agent`) that orchestrates each station. Each audit station is a Pi SDK session with scoped writable paths and a purpose-gate prompt. See `cascade-plan/pipeline/agent-runner.ts`.
- **Mermaid** — Diagramming format used by W2 (UML class diagrams in `domain-model/`) and W3 (C4 context/container/component diagrams in `c4/`). Rendered by any Mermaid-compatible viewer.
- **GLM / Claude** — LLM models invoked by stations for generation and analysis. Model selection driven by `.pi/config/factory.yaml` station config (model tiering: Opus for architecture, Sonnet for implementation).
- **TypeScript / tsx** — Runtime for the audit CLI (`audit/run-audit.ts`), quality gates (`audit/gates/*.gate.ts`), and the cascade-plan pipeline.
- **grep / bash** — Used for mechanical traceability: the citation index is built by grepping `[source:` markers across all audit artifacts.

## Extension → language map (used by file-inventory.md)

File extensions in the source codebase (`discovery/bugzilla/`) are mapped to languages as follows:

| Extension | Language |
|---|---|
| `.pm` | Perl (module) |
| `.cgi` | Perl (CGI script) |
| `.pl` | Perl (standalone script) |
| `.ts`, `.tsx` | TypeScript |
| `.js`, `.jsx` | JavaScript |
| `.css`, `.scss` | CSS |
| `.yaml`, `.yml` | YAML |
| `.json` | JSON |
| `.md` | Markdown |
| `.sql` | SQL |
| `.bpmn.yaml` | BPMN (YAML serialisation) |
| (other) | (unknown) |

The source codebase is monolingual Perl — all 61 key files inventoried in the cluster table use `.pm`, `.cgi`, or `.pl` extensions.

## Known limitations

- **LOC counts include comments and blank lines** — raw `wc -l` from the cluster inventory, not semantic LOC.
- **File-author column is uninformative** — the codebase was imported as a single bulk snapshot with no git history, so every file reports `(unversioned)` for author and zero churn.
- **Stability ratings are all `unknown`** — with zero git history and no test coverage signals, the stability heuristic falls back to test proximity and architectural analysis only.
- **Citations point to time-of-audit line numbers** — later edits to source files may shift line numbers, breaking exact traceability.
- **The cluster inventory lists 61 key files** — the full codebase is ~250 files at ~111 kLOC. The per-language breakdown in `file-inventory.md` covers only the inventoried subset.
