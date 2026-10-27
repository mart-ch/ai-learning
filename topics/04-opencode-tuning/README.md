# 02 · OpenCode: Konfiguration und Tuning

OpenCode ist ein Open-Source-Coding-Agent (Terminal, Desktop, Web). Diese Doku beschreibt **OpenCode V2** — die 2026 veröffentlichte, Rust-kernbasierte Generation. Sie baut auf [Topik 03](../03-opencode-local-omlx/) auf (Installation + oMLX-Anbindung) und zeigt das komplette Konfigurations- und Tuning-Werkzeug. Alle Angaben nach offiziellem Stand: [opencode.ai/v2/docs](https://opencode.ai/v2/docs/).

## Status & Änderungen

| | |
|---|---|
| **Letzte Prüfung** | 27.10.2026 |
| **Quellen** | <https://opencode.ai/v2/docs/> (Config, Providers, Models, Agents, Permissions, Commands, Instructions, Compaction, Troubleshooting) |
| **Position** | Vertiefung zu [Topik 03](../03-opencode-local-omlx/); lokale oMLX-Verbindung selbst dort dokumentiert |

> ⚠️ **V1 vs. V2:** OpenCode 1 und 2 nutzen beide den `opencode`-Befehl, werden aber nicht mehr parallel installiert. Das V2-Curl-Installer ersetzt die V1-Binary; eine paketverwaltete V1-Installation vorher entfernen. V1-Config-Dateien bleiben großteils kompatibel, aber die nativen V2-Formen sind bevorzugt. V1-Doku: <https://opencode.ai/docs/> (veraltet).

---

## 1. Architektur: Client + Hintergrund-Service

OpenCode ist Client-Server. Die TUI verbindet sich mit einem **Hintergrund-Service**, der Sessions, Konfiguration, Plugins, Permissions und Tool-Ausführung besitzt.

```
opencode service status     # Status prüfen
opencode service restart    # nur wenn der Service klemmt (nicht für Config-Änderungen)
```

Config-Änderungen werden **ohne Neustart** wirksam: OpenCode beobachtet die Config-Dateien und lädt Agents, Commands, Skills, MCP-Server und Plugins nach dem Speichern neu. TUI-Neuladen erzwingen: `/reload`.

---

## 2. Config-Dateien und Priorität

Es gibt zwei getrennte Welten:

| Datei | Was |
|---|---|
| `opencode.json(c)` | Server- und Projekt-Config (Modelle, Provider, Agents, Permissions, Commands, Plugins, MCP, …) |
| `cli.json` | Nur Terminal-/TUI-Einstellungen (Theme, Keybinds, Tabs, Diffs, Alerts, Mini). Liegt in `~/.config/opencode/cli.json`. Viele Einstellungen lassen sich auch in der TUI ändern (`Ctrl+P` → **Open settings**). |

### Orte der Server-Config

| Scope | Pfad |
|---|---|
| Global | `~/.config/opencode/opencode.json(c)` |
| Projekt | `opencode.json(c)` oder `.opencode/opencode.json(c)` im Projekt (auch in Monorepo-Paketen) |

**Merge-Regel:** OpenCode sucht vom aktuellen Verzeichnis über alle Ancestors bis zur Dateisystemwurzel. Direkte `opencode.json(c)`-Dateien werden von der Wurzel zum aktuellen Verzeichnis gemergt, danach die `.opencode/opencode.json(c)`-Dateien. Daraus folgt: jede entdeckte `.opencode`-Config überschreibt jede entdeckte direkte Config; globale Config hat die niedrigste Priorität.

Immer das Schema mitgeben, dann validiert der Editor und liefert Autocomplete:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
}
```

JSONC erlaubt Kommentare und fehlende Trailing Commas.

### Typische Felder

`model`, `default_agent`, `permissions`, `agents`, `commands`, `plugins`, `providers`, `mcp`, `skills`, `instructions`, `references`, `formatter`, `lsp`.

---

## 3. Modelle auswählen

```
/models
```

listet Modelle aus dem [models.dev](https://models.dev)-Catalog, Provider-Integrationen und eurer eigenen Config. Wählbar sind nur enabled Modelle, deren Provider verfügbar ist.

Referenz-Syntax: `provider/model` (optional `#variant`):

```jsonc
{
  "model": "anthropic/claude-sonnet-5-5",          // Default-Modell (provider/model)
  "default_agent": "build",                        // Default-Agent
}
```

Der Root-Default trägt aktuell **keinen** `#variant` — Variants werden bei Agenten und Commands gewählt.

### Local Modelle (relevanter Teil fürs [Topik 01](../01-omlx-local-llm/))

OpenCode entdeckt **automatisch** Modelle von **Ollama**, **LM Studio** und **vLLM** an ihren Standard-Adressen:

```
ollama/gemma3:4b
lmstudio/google/gemma-4-26b-a4b
vllm/Qwen/Qwen3-Coder-30B-A3B-Instruct
```

Jeden anderen **OpenAI-kompatiblen Server** (z. B. **oMLX**) manuell konfigurieren:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "omlx/coder",
  "providers": {
    "omlx": {
      "name": "oMLX Local",
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": {
        "baseURL": "http://127.0.0.1:8000/v1"
      },
      "models": {
        "coder": {
          "modelID": "model-name-on-server",   // ID, die an den Server gesendet wird
          "capabilities": {
            "tools": true,
            "input": ["text"],
            "output": ["text"]
          },
          "limit": {
            "context": 32768,
            "output": 8192
          }
        }
      }
    }
  }
}
```

Regeln:

- `baseURL` in `settings` — das ist die OpenAI-kompatible Endpoint-URL (V1 nannte das `api`).
- `package` bestimmt das Runtime-Provider-Paket; für OpenAI-kompatible Endpunkte: `@opencode/ai/providers/openai-compatible`.
- `models`-Key = OpenCode-Modell-ID (steht in `provider/model`); `modelID` = ID, die tatsächlich an den Provider gesendet wird (wird weggelassen, gilt der Key).
- `capabilities` und `limit` sind **eure** Angaben — OpenCode validiert sie nicht gegen den Server. Für Coding-Agents: `tools: true` nur setzen, wenn der Server Tool-Use unterstützt.
- API-Key: in `settings` mit Umgebungsvariablen-Ersetzung, **nie im Klartext committen**: `"apiKey": "{env:LOCAL_API_KEY}"`.
- LM Studio mit anderem Port/Host: `settings.baseURL` setzen, Discovery bleibt aktiv; Discovery abschalten mit `"plugins": ["-opencode.provider.lmstudio"]`.

---

## 4. Tuning: Settings, Headers, Body, Limits, Variants

### 4.1 Einstellungen je Ebene

Provider- und Modell-Einträge dürfen Requests mit `settings`, `headers` und `body` anpassen:

```jsonc
{
  "providers": {
    "openai": {
      "settings": { "baseURL": "https://api.example.com/v1" },   // Provider-Ebene
      "headers": { "X-Team": "platform" },
      "models": {
        "gpt-6.1-sol": {
          "settings": { "reasoningEffort": "high" },             // Modell-Ebene
          "body": { "store": false }
        }
      }
    }
  }
}
```

**Priorität (merge):** Provider-Werte → Modell-Werte → ausgewählte Variant. Verschlachtelte `settings`/`body`-Objekte werden gemergt; spätere Arrays und Scalars ersetzen frühere. Header-Namen sind case-insensitiv. Optionen sind provider-spezifisch und dürfen von anderen Provider-Paketen ignoriert oder abgelehnt werden.

### 4.2 Limits

`limit` steuert Kontext- und Output-Budget je Modell:

```jsonc
"limit": { "context": 200000, "output": 32000 }
```

Für lokale Modelle (kleiner RAM) bewusst klein halten, siehe [Topik 01](../01-omlx-local-llm/), Abschnitt 4.

### 4.3 Variants

Variants sind benannte Option-Sets für ein Modell — typischerweise Reasoning-Effort oder Token-Budgets. Auswahl per `#variant`:

```
opencode run --model openai/gpt-6.1-sol#high "Review this migration plan"
```

Variant-Namen kommen aus dem Catalog-Metadaten des Modells; Namen wie `low`/`high`/`max` existieren nicht für jedes Modell, unbekannte Variants lösen eine Model-Resolution-Fehlermeldung. Eigene Variants definieren (oder Catalog-Variants mit gleicher ID ersetzen) im `variants`-Array:

```jsonc
{
  "providers": {
    "openai": {
      "models": {
        "gpt-6.1-sol": {
          "settings": { "reasoningEffort": "medium" },
          "variants": [
            { "id": "fast", "settings": { "reasoningEffort": "low" } },
            { "id": "deep", "settings": { "reasoningEffort": "high", "reasoningSummary": "auto" } }
          ]
        }
      }
    }
  }
}
```

Jede Variant enthält `settings`, `headers`, `body` und wird **nach** Provider- und Modell-Werten angewendet. Um Reasoning-Variants eines Modells zu ersetzen, die Reasoning-Controls in `capabilities.reasoning` (models.dev v2-Shape) beschreiben — OpenCode baut daraus die Variants.

### 4.4 Modell-Felder im Überblick

| Feld | Zweck |
|---|---|
| `modelID` | ID, die an den Provider gesendet wird |
| `name` | Anzeigename |
| `family` | Modell-Familie (Gruppierung) |
| `type` | `chat` (Default), `image`, `decision` — nur `chat` wird für Sessions angeboten |
| `releaseDate` | `YYYY-MM-DD` oder `YYYY-MM` |
| `package` | Runtime-Override je Modell |
| `settings` / `headers` / `body` | wie oben |
| `capabilities` | Tool-Support, Input-/Output-Mediatypen, Temperature, Reasoning |
| `compatibility` | API-Quirks je Modell (überschreibt Provider-Level) |
| `variants` | benannte Varianten |
| `cost` | Input/Output/Cache-Preis pro Mio. Tokens |
| `limit` | Kontext-, Input- und Output-Budget |

---

## 5. Agents

Eigene Agenten als Markdown-Datei unter `.opencode/agents/<name>.md`. Der Frontmatter-Body wird zum `system`-Prompt:

```markdown
---
description: Explains code without changing it
mode: subagent
---

Explain the relevant code with short examples. Do not edit files.
```

Wichtige Frontmatter-Felder (V2-nativ):

| Feld | Bedeutung |
|---|---|
| `description` | Beschreibung in Listen |
| `mode` | `primary` oder `subagent` |
| `model` | `provider/model#variant` |
| `permissions` | Permissions-Regeln (siehe unten) |

V1-Frontmatter ist deprecated und wird noch automatisch übersetzt — aber: **eine Datei entweder als V1 oder als V2 konvertieren, nicht mischen.** (Dateien mit `permissions`, `request`, `system` oder `disabled` werden als V2 gelesen; V1-Felder wie `name`, `tools`, `temperature` werden dann mit Log-Warnung ignoriert.)

Agents lassen sich auch in JSON konfigurieren:

```jsonc
{
  "agents": {
    "reviewer": {
      "permissions": [
        { "action": "*", "resource": "*", "effect": "deny" },
        { "action": "read", "resource": "src/**", "effect": "allow" }
      ]
    }
  }
}
```

Vorgesehene Agenten (Auszug): `build` (Standard-Coding, stellt Fragen), `plan` (Read-only-Planung, Edits nur unter `~/.opencode/plan`), `general` (Multi-Step, keine Fragen/Subagents), `explore` (Read-only-Suche), `title`/`summary`/`compaction` (interne Helfer, alle Aktionen verweigert).

---

## 6. Permissions

Permissions steuern, ob ein Agent eine Aktion auf einer Resource durchführen darf. **Geordnete Regel-Listen; die letzte passende Regel gewinnt** — breite Regeln zuerst, Ausnahmen danach.

```jsonc
{
  "permissions": [
    { "action": "shell", "resource": "*", "effect": "ask" },
    { "action": "shell", "resource": "git status*", "effect": "allow" },
    { "action": "shell", "resource": "git push*", "effect": "deny" }
  ]
}
```

| Feld | Bedeutung |
|---|---|
| `action` | Tool oder Permission-Action, Wildcards erlaubt |
| `resource` | Pfad, Befehl, Agent-ID oder anderer Match-Wert, Wildcards erlaubt |
| `effect` | `allow`, `ask` oder `deny` |

Gängige Actions: `shell`, `edit` (Edit/Write/Patch), `subagent`, `read`/`glob`/`grep`, `webfetch`, `websearch`, `skill`, `external_directory`.

Für `read`, `edit` und `external_directory` werden `~` und `$HOME` expandiert:

```jsonc
{ "action": "read", "resource": "~/notes/**", "effect": "allow" }
```

Shell-Resources bleiben Rohtext.

### Defaults

Jeder Agent startet mit dieser Basis-Policy (geordnet):

```jsonc
[
  { "action": "*", "resource": "*", "effect": "allow" },
  { "action": "read", "resource": "*.env", "effect": "ask" },
  { "action": "read", "resource": "*.env.*", "effect": "ask" },
  { "action": "read", "resource": "*.env.example", "effect": "allow" }
]
```

Achtung: **`external_directory` fragt per Default nicht** — Lesen und Editieren außerhalb des Workspace/Worktree läuft ohne Prompt. Einen `ask`- oder `deny`-Regel ergänzen, um diese Freiheit einzuschränken.

---

## 7. Commands

Commands sind wiederverwendbare Prompts. Projekt-Commands: `.opencode/commands/<name>.md`; global: `~/.config/opencode/commands/`. Nur `.md`-Dateien werden entdeckt; verschachtelte Pfade werden zu Namen mit `/`:

```
.opencode/commands/team/review.md  →  /team/review src/auth.ts
```

Der Markdown-Body ist die Prompt-Vorlage; `$ARGUMENTS` wird durch die Aufruf-Argumente ersetzt:

```markdown
---
description: Review code for correctness
agent: plan
model: anthropic/claude-sonnet-5-5#high
---

Review $ARGUMENTS. Report bugs first.
```

| Frontmatter-Feld | Verhalten |
|---|---|
| `description` | Text in Command-Listen |
| `agent` | Agent, der bei Ausführung gewählt wird |
| `model` | Model-Override, `provider/model` oder `provider/model#variant` |
| `subagent` | `true` = Hintergrund-Kind-Session, `false` = aktuelle Session |
| `subtask` | deprecated Alias für `subagent` |

In JSON-Config sind es dieselben Felder plus `template` (nur in JSON; nicht in Frontmatter — der Body liefert es dort).

---

## 8. Instructions: AGENTS.md

`AGENTS.md` im Workspace ist die dauerhafte Projekt-Instructions-Datei: Build-Kommandos, Architektur-Notizen, Code-Konventionen, Verifikations-Pflichten.

```markdown
# Project instructions

- Run `bun typecheck` after changing TypeScript.
- Keep database queries in `src/database`.
- Do not edit generated files directly.
```

**Discovery:** Die `AGENTS.md` im Workspace ist vom Start an aktiv. Verschachtelte `AGENTS.md`-Dateien werden geladen, wenn der Agent in diese Bereiche liest (nächtste zuerst, dedupliziert). Datei-Edits nach dem Laden werden nicht automatisch erkannt — für sofortige Wirkung neue Session starten. **Committen** der Instructions-Dateien, damit alle dieselbe Anleitung bekommen.

```
my-project/
├── AGENTS.md              ← am Anfang geladen
└── packages/
    └── web/
        ├── AGENTS.md      ← wenn dieser Bereich gelesen wird
        └── src/
            └── app.ts
```

---

## 9. MCP-Server

MCP-Server hängen in `opencode.json(c)` unter `mcp.servers`. Bevorzugt per CLI anlegen (erhält die übrige Config):

```
opencode mcp add <name> --global --url <remote-url>   # --global = userweit
opencode mcp list
```

Remote-Server nutzen **standardmäßig OAuth**. Meldet `mcp list`, dass ein Server Auth braucht: in der TUI `/mcps` öffnen, Server wählen, sign in. OAuth-Credentials liegen **außerhalb** der OpenCode-Config. Header-basierte Credentials nur, wenn OAuth nicht verfügbar ist oder ausdrücklich verlangt wird — dann mit Umgebungsvariablen-Ersetzung, z. B. `"headers": { "Authorization": "{env:MCP_API_KEY}" }`, nie Secrets in die Config.

---

## 10. Plugins

Plugins unter `plugins` in der Config: Strings (Paketnamen) oder Objekte mit `package` (Pfad oder Name) und `options`:

```jsonc
{
  "plugins": [
    "opencode-example-plugin",
    { "package": "./plugin/local", "options": { "enabled": true } }
  ]
}
```

V2 entdeckt lokale Plugins aus `.opencode/plugin/` und `.opencode/plugins/` — für V2-Dateien `.opencode/plugins/` verwenden. Lokale Config-Pfade müssen Verzeichnisse sein (`./plugin/local/index.ts`). **V1-Plugin-Implementierungen laufen in V2 nicht** — Portierung nach [Plugin-Migration-Guide](https://opencode.ai/v2/docs/build/plugins/migrate-v1).

---

## 11. Compaction (lange Sessions)

Wenn eine Session an die Kontextgrenze des Modells stößt, ersetzt OpenCode ältere Konversation durch eine Zusammenfassung, damit die Session weiterlaufen kann. Relevant bei lokalen Modellen mit kleiner `limit.context`: Sessions kompakt halten, bei Bedarf neu starten.

---

## 12. Troubleshooting

1. **Service prüfen:** `opencode service status`, API: `opencode api get /api/info`
2. **Isolieren:** `opencode --standalone` läuft die TUI mit privatem Server — trennt Shared-Service-Probleme von Projekt-Problemen
3. **Logs:** `~/.local/share/opencode/log/opencode.log`; `role=cli` für Client-Start, `role=server` für Sessions/Provider/Plugins/Permissions/Tools
4. **DEBUG:** eine Reproduktion mit `OPENCODE_LOG_LEVEL=DEBUG`
5. **Nicht anfassen:** Database, Service-Registration, Service-Config während der Diagnose nicht löschen/editieren; Backup vor externer Inspektion
6. Vor dem Teilen von Diagnosen: API-Keys, Auth-Header, Prompts, Datei-Inhalte redigieren

Free-Modelle im Catalog ausblenden:

```
opencode service set env OPENCODE_DISABLE_FREE_MODELS 1
```

---

## 13. Checkliste für ein neues Projekt

```
1. opencode.jsonc anlegen (Schema + model + provider für lokale/oMLX)
2. .opencode/agents/ für Projekt-spezifische Agenten (z. B. read-only Reviewer)
3. permissions: shell ask, git status/branch allow, git push deny, external_directory ask
4. .opencode/commands/ für wiederkehrende Prompts (z. B. /review)
5. AGENTS.md: Build-Kommandos, Konventionen, Prüfregeln — committen
6. /models wählen, /models-Test mit lokalem Modell, opencode service status prüfen
```

## Quellen

- Config: <https://opencode.ai/v2/docs/config/>
- Providers: <https://opencode.ai/v2/docs/providers/>
- Models (inkl. Local): <https://opencode.ai/v2/docs/models/>
- Agents: <https://opencode.ai/v2/docs/agents/>
- Permissions: <https://opencode.ai/v2/docs/permissions/>
- Commands: <https://opencode.ai/v2/docs/commands/>
- Instructions: <https://opencode.ai/v2/docs/instructions/>
- MCP: <https://opencode.ai/v2/docs/mcp-servers/>
- Compaction: <https://opencode.ai/v2/docs/compaction/>
- Migrate from V1: <https://opencode.ai/v2/docs/migrate-v1/>
- Troubleshooting: <https://opencode.ai/v2/docs/troubleshooting/>
- CLI-Settings: <https://opencode.ai/v2/docs/cli/config/>
