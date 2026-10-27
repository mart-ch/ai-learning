# AI Learning

Meine persönliche Dokumentation zu AI-Themen, die ich lerne. Jedes Thema hat einen eigenen Unterordner mit vollständiger Doku, beziehbaren Quellen und einem Datum zur letzten Prüfung.

> **Konvention:** Jeder Topic-Ordner enthält ein `README.md` mit
>
> - dem eigentlichen Lerninhalt,
> - einem Abschnitt **Status & Änderungen** (letztes Prüfdatum, was seit der Ursprungsdoku veraltet ist),
> - verlinkten Original-Quellen.

## Themen

| # | Thema | Ordner | Stand | Status |
|---|-------|--------|-------|--------|
| 01 | Lokale KI auf einem Mac installieren | [`topics/01-lokale-ki-mac/`](topics/01-lokale-ki-mac/) | Okt 2026 | ✅ dokumentiert |
| 02 | OpenCode: Konfiguration und Tuning | [`topics/02-opencode-konfiguration/`](topics/02-opencode-konfiguration/) | Okt 2026 | ✅ dokumentiert |
| 03 | _(folgt)_ | – | – | 🚧 in Arbeit |

## Themen im Detail

### 01 · Lokale KI auf einem Mac installieren

Vollständiger lokaler Stack auf Apple Silicon: **oMLX** als Inference-Server, **Qwen-Modell mit MTP** (Multi-Token Prediction), **Open WebUI** für Chats und **OpenCode** als Coding-Agent.

- Architektur: eine Inference-Schicht, zwei Clients, keine Cloud
- Modellwahl nach RAM, MTP-Test, Docker-Setup für Open WebUI
- ✅ Update Okt 2026: OpenCode-V1-→V2-Änderungen, aktuelle oMLX-Version, aktuelle Qwen-Generation

→ [Zur Doku](topics/01-lokale-ki-mac/)

### 02 · OpenCode: Konfiguration und Tuning

OpenCode **V2** (Rust-kernbasiert, 2026) von Grund auf: Config-Dateien, lokale Modelle (u. a. oMLX), Tuning über `settings`/`variants`/`limits`, Agenten, Permissions, Commands, `AGENTS.md`, MCP-Server und Troubleshooting.

- Konfigurations-Priorität: global → Projekt → `.opencode/`
- Lokale OpenAI-kompatible Server konfigurieren (praktisches oMLX-Beispiel)
- Permissions-Regeln (`allow`/`ask`/`deny`), Custom Commands, AGENTS.md

→ [Zur Doku](topics/02-opencode-konfiguration/)

### 03 · folgt

_Nächste Themen-Auswahl in Arbeit._

## Quellen

- oMLX: <https://github.com/jundot/omlx>
- OpenCode V2 Doku: <https://opencode.ai/v2/docs/>
- Open WebUI: <https://github.com/open-webui/open-webui>
- Ursprungsdoku Topik 01: <https://github.com/mart-ch/locale-ai-on-mac>
