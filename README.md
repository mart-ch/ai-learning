# AI Learning

Meine persönliche Dokumentation zu AI-Themen, die ich lerne. Jedes Thema hat einen eigenen Unterordner mit vollständiger Doku, beziehbaren Quellen und einem Datum zur letzten Prüfung.

> **Konvention:** Jeder Topic-Ordner enthält ein `README.md` mit
>
> - dem eigentlichen Lerninhalt,
> - einem Abschnitt **Status & Änderungen** (letztes Prüfdatum, was seit der Ursprungsdoku veraltet ist),
> - verlinkten Original-Quellen und einem Verweis auf das nächste Thema.

Die Topics bauen **logisch aufeinander auf**: oMLX (LLM-Server) → Open WebUI (Chat darauf) → OpenCode (Coding-Agent darauf) → LLM-Tuning (beide Configs abstimmen).

## Themen

| # | Thema | Ordner | Stand | Status |
|---|-------|--------|-------|--------|
| 01 | oMLX: Lokaler LLM-Server (Brew, Modell, MTP, Probleme) | [`topics/01-omlx-local-llm/`](topics/01-omlx-local-llm/) | Okt 2026 | ✅ dokumentiert |
| 02 | Open WebUI: Chat-Funktionalität (Docker, Konfiguration) | [`topics/02-open-webui/`](topics/02-open-webui/) | Okt 2026 | ✅ dokumentiert |
| 03 | OpenCode: Installation & Konfiguration mit lokalem oMLX | [`topics/03-opencode-local-omlx/`](topics/03-opencode-local-omlx/) | Okt 2026 | ✅ dokumentiert |
| 04 | LLM Tuning: OMLX Modellconfig + Opencode Config abstimmen | [`topics/04-llm-tuning/`](topics/04-llm-tuning/) | Okt 2026 | ✅ dokumentiert |
| 05 | _(folgt)_ | – | – | 🚧 in Arbeit |

## Themen im Detail

### 01 · oMLX: Lokaler LLM-Server auf dem Mac

**oMLX** ist die Inference-Schicht des lokalen Stacks: LLM-Server für Apple Silicon mit Continuous Batching, SSD-Caching und MTP (Multi-Token Prediction) für schnellere Generierung.

- Installation über Homebrew (**Tap `jundot/omlx` erforderlich** — nicht in homebrew-core), inkl. `--with-custom-kernel` für Qwen-Beschleunigung, oder native App
- Qwen-Modellwahl nach RAM (aktuelle Generation: Qwen 3.5/3.6/4 mit MTP)
- MTP-Test und -Verifikation über API + Logs
- Probleme & Lösungen: Port 8000, RAM-Auslastung, inaktives MTP

→ [Zur Doku](topics/01-omlx-local-llm/)

### 02 · Open WebUI: Chat-Funktionalität

Chat-Oberfläche als Docker-Container, angebunden an oMLX aus Thema 01 — ohne dass die Inference in Docker läuft.

- Docker-Setup: `docker run`-Flags, persistentes Volume, `WEBUI_SECRET_KEY`
- Konfiguration: OpenAI-API-Connection mit `host.docker.internal:8000/v1`
- Updates mit Datenverlust-Schutz, Troubleshooting (Erreichbarkeit, fehlendes Modell)

→ [Zur Doku](topics/02-open-webui/)

### 03 · OpenCode: Installation & Konfiguration mit lokalem oMLX

OpenCode **V2** als Coding-Agent, der das lokale Qwen-Modell aus Thema 01 nutzt — komplett ohne Cloud.

- V2-Installation (Homebrew bevorzugt: `anomalyco/tap/opencode-v2`; Alternativen: curl, npm) + V1→V2-Umstellung
- oMLX als Provider: native V2-Config, inkl. realistischem Beispiel (Qwen3.8-27B, ohne API-Key)
- Erster Test, `AGENTS.md`, Sicherheitskonfiguration
- ✅ Update Okt 2026: V1-Config-Syntax (`provider` + `npm`) durch V2-Syntax ersetzt

→ [Zur Doku](topics/03-opencode-local-omlx/)

### 04 · LLM Tuning: OMLX Modellconfig + Opencode Config

Vertiefung zu Thema 01 und 03 — die zwei Configs des lokalen Stacks aufeinander abstimmen, damit das System nicht beeinträchtigt wird:

- **OMLX Modellconfig** (Server): Memory Guard (Tiers, Watermarks), Memory-Mathematik (Gewichte + KV-Cache), Modell-Einstellungen, TTL, Admin API
- **Opencode Config** (Client): Config-Dateien, Modelle, Tuning, Agents, Permissions, Commands, `AGENTS.md`, MCP, Plugins, Compaction, Troubleshooting
- **Abstimmung:** Memory-Budget-Tabelle — `limit.context` (Client) ≤ `max_context_window` (Server) ≤ Memory-Guard-Limit; „Stable 64k" als angewendetes Beispiel

→ [Zur Doku](topics/04-llm-tuning/)

### 05 · folgt

_Nächste Themen-Auswahl in Arbeit._

## Quellen

- oMLX: <https://github.com/jundot/omlx>
- OpenCode V2 Doku: <https://opencode.ai/v2/docs/>
- Open WebUI: <https://github.com/open-webui/open-webui>
- Ursprungsdoku Topic 01–03: <https://github.com/mart-ch/locale-ai-on-mac>
