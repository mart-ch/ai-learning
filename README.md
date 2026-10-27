# AI Learning

Meine persönliche Dokumentation zu AI-Themen, die ich lerne. Jedes Thema hat einen eigenen Unterordner mit vollständiger Doku, beziehbaren Quellen und einem Datum zur letzten Prüfung.

> **Konvention:** Jeder Topic-Ordner enthält ein `README.md` mit
>
> - dem eigentlichen Lerninhalt,
> - einem Abschnitt **Status & Änderungen** (letztes Prüfdatum, was seit der Ursprungsdoku veraltet ist),
> - verlinkten Original-Quellen und einem Verweis auf das nächste Topik.

Die Topics bauen **logisch aufeinander auf**: oMLX (LLM-Server) → Open WebUI (Chat darauf) → OpenCode (Coding-Agent darauf) → OpenCode-Vertiefung.

## Themen

| # | Thema | Ordner | Stand | Status |
|---|-------|--------|-------|--------|
| 01 | oMLX: Lokaler LLM-Server (Brew, Modell, MTP, Probleme) | [`topics/01-omlx-local-llm/`](topics/01-omlx-local-llm/) | Okt 2026 | ✅ dokumentiert |
| 02 | Open WebUI: Chat-Funktionalität (Docker, Konfiguration) | [`topics/02-open-webui/`](topics/02-open-webui/) | Okt 2026 | ✅ dokumentiert |
| 03 | OpenCode: Installation & Konfiguration mit lokalem oMLX | [`topics/03-opencode-local-omlx/`](topics/03-opencode-local-omlx/) | Okt 2026 | ✅ dokumentiert |
| 04 | OpenCode: Konfiguration und Tuning (V2) | [`topics/04-opencode-tuning/`](topics/04-opencode-tuning/) | Okt 2026 | ✅ dokumentiert |
| 05 | _(folgt)_ | – | – | 🚧 in Arbeit |

## Themen im Detail

### 01 · oMLX: Lokaler LLM-Server auf dem Mac

**oMLX** ist die Inference-Schicht des lokalen Stacks: LLM-Server für Apple Silicon mit Continuous Batching, SSD-Caching und MTP (Multi-Token Prediction) für schnellere Generierung.

- Installation über Homebrew (inkl. `--with-custom-kernel` für Qwen-Beschleunigung) oder native App
- Qwen-Modellwahl nach RAM (aktuelle Generation: Qwen 3.5/3.6/4 mit MTP)
- MTP-Test und -Verifikation über API + Logs
- Probleme & Lösungen: Port 8000, RAM-Auslastung, inaktives MTP

→ [Zur Doku](topics/01-omlx-local-llm/)

### 02 · Open WebUI: Chat-Funktionalität

Chat-Oberfläche als Docker-Container, angebunden an oMLX aus Topik 01 — ohne dass die Inference in Docker läuft.

- Docker-Setup: `docker run`-Flags, persistentes Volume, `WEBUI_SECRET_KEY`
- Konfiguration: OpenAI-API-Connection mit `host.docker.internal:8000/v1`
- Updates mit Datenverlust-Schutz, Troubleshooting (Erreichbarkeit, fehlendes Modell)

→ [Zur Doku](topics/02-open-webui/)

### 03 · OpenCode: Installation & Konfiguration mit lokalem oMLX

OpenCode **V2** als Coding-Agent, der das lokale Qwen-Modell aus Topik 01 nutzt — komplett ohne Cloud.

- V2-Installation (`anomalyco/tap/opencode-v2`, curl-Installer, npm) + V1→V2-Umstellung
- oMLX als Provider: native V2-Config (`providers`, `package`, `settings`, `models`)
- Erster Test, `AGENTS.md`, Sicherheitskonfiguration
- ✅ Update Okt 2026: V1-Config-Syntax (`provider` + `npm`) durch V2-Syntax ersetzt

→ [Zur Doku](topics/03-opencode-local-omlx/)

### 04 · OpenCode: Konfiguration und Tuning (V2)

Vertiefung zu Topik 03 — das komplette V2-Werkzeug:

- Config-Dateien (global/projekt, JSONC, Merge-Priorität, Hot-Reload)
- Modelle: lokale Discovery (Ollama, LM Studio, vLLM), `settings`/`variants`/`limits`, Compaction
- Agents, Permissions (`allow`/`ask`/`deny`), Commands, `AGENTS.md`, MCP-Server, Plugins
- Troubleshooting: Service, Logs, Standalone-Modus

→ [Zur Doku](topics/04-opencode-tuning/)

### 05 · folgt

_Nächste Themen-Auswahl in Arbeit._

## Quellen

- oMLX: <https://github.com/jundot/omlx>
- OpenCode V2 Doku: <https://opencode.ai/v2/docs/>
- Open WebUI: <https://github.com/open-webui/open-webui>
- Ursprungsdoku Topic 01–03: <https://github.com/mart-ch/locale-ai-on-mac>
