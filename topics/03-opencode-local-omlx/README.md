# 03 · OpenCode: Installation & Konfiguration mit lokalem oMLX

**OpenCode** ist ein Open-Source-Coding-Agent für das Terminal. Dieses Thema installiert **OpenCode V2** und verbindet ihn mit dem lokalen oMLX-Server aus [Thema 01](../01-omlx-local-llm/) — damit der Coding-Agent das Qwen-Modell ohne Cloud nutzt. Allgemeinere OpenCode-V2-Konfiguration (Agents, Permissions, Commands, Tuning) und die Abstimmung mit der oMLX-Modellconfig folgen in [Thema 04](../04-llm-tuning/).

<img width="1413" height="608" alt="image" src="https://github.com/user-attachments/assets/a37ebb12-5265-47b5-8685-125c6f8579cf" />


## Inhaltsverzeichnis

- [Status & Änderungen](#status--änderungen)
- [1. Voraussetzungen](#1-voraussetzungen)
- [2. OpenCode installieren](#2-opencode-installieren)
- [3. oMLX als Provider konfigurieren](#3-omlx-als-provider-konfigurieren)
- [4. Erster Test in einem Projekt](#4-erster-test-in-einem-projekt)
- [5. Sicherheitskonfiguration](#5-sicherheitskonfiguration)
- [6. Start, Stopp und Updates](#6-start-stopp-und-updates)
- [7. Probleme und Lösungen](#7-probleme-und-lösungen)
  - [OpenCode zeigt das Modell nicht](#opencode-zeigt-das-modell-nicht)
  - [oMLX erreichbar, aber OpenCode schlägt fehl](#omlx-erreichbar-aber-opencode-schlägt-fehl)
- [Nächste Schritte](#nächste-schritte)
- [Quellen](#quellen)

## Status & Änderungen

| | |
|---|---|
| **Letzte Prüfung** | 27.10.2026 |

| # | Thema | Status | Update |
|---|---|---|---|
| 1 | **OpenCode als V1** (`brew install anomalyco/tap/opencode`, Config-Schlüssel `provider` (singular) mit `npm` + `options`) | 🔴 veraltet | OpenCode **V2** ist die aktuelle Version. Installation: `brew install anomalyco/tap/opencode-v2` oder `curl -fsSL https://opencode.ai/v2/install \| bash`. Config-Schlüssel ist jetzt `providers` (plural) mit `package` + `settings`. → Abschnitt 2 und 3 |
| 2 | **`/init` in OpenCode** erzeugt `AGENTS.md` | 🟡 unklar in V2 | Die V2-Doku definiert `AGENTS.md` als Projekt-Instructions-Datei (Workspace-Datei aktiv, verschachtelte Dateien beim Lesen geladen). In V2 die Datei bewusst anlegen oder vom Agenten anfertigen lassen. → Abschnitt 4 |
| 3 | Config-Pfad `~/.config/opencode/opencode.json` | ✅ aktuell | bleibt so; V2 unterstützt zusätzlich JSONC (`opencode.jsonc`) |

---

## 1. Voraussetzungen

**oMLX läuft** mit geladenem Modell (→ [Thema 01](../01-omlx-local-llm/)):

```
curl -s http://127.0.0.1:8000/v1/models | jq -r '.data[].id'
```

> OpenCode läuft **nativ auf dem Mac** (kein Docker), daher `127.0.0.1` statt `host.docker.internal` — das ist der Unterschied zur Open-WebUI-Verbindung in [Thema 02](../02-open-webui/).

---

## 2. OpenCode installieren

> 🔄 **Update Okt 2026:** OpenCode **V2** ist die aktuelle Version. Die Ursprungsdoku installierte V1 (`anomalyco/tap/opencode`). Das V2-Curl-Installer ersetzt die V1-Binary; falls eine paketverwaltete V1-Installation vorhanden ist, vorher entfernen.

**Empfohlen: Homebrew** — bequem installiert, upgradet mit `brew upgrade` und nutzt die vorhandene Paketverwaltung:

```
brew install anomalyco/tap/opencode-v2
```

Alternativen (offizielle Wege, je einer):

```
curl -fsSL https://opencode.ai/v2/install | bash
```

```
npm install -g @opencode/cli
```

(Auch verfügbar: `bun`, `pnpm`, `yarn`, `mise`, standalone-Binaries — Übersicht: <https://opencode.ai/v2/docs/>.)

Prüfen:

```
opencode --version
```

---

## 3. oMLX als Provider konfigurieren

Die globale Konfiguration liegt auf macOS unter:

```
~/.config/opencode/opencode.json
```

(Jeder JSON-Datei sollte das `$schema`-Feld mitgeben, damit der Editor Validierung und Autocomplete liefert. V2 unterstützt auch JSONC, d. h. Kommentare und fehlende Comma am Zeilenende sind erlaubt: `opencode.jsonc`.)

Verzeichnis anlegen:

```
mkdir -p ~/.config/opencode
```

Modell-ID erneut abfragen:

```
curl -s http://127.0.0.1:8000/v1/models | jq -r '.data[].id'
```

> 🔄 **Update Okt 2026 (V2-Syntax):** V1 nutzte `provider` (singular) mit `npm` und `options`. V2 verwendet `providers` (plural), `package` und `settings`. Die V1-Form wird vom Schema zwar noch akzeptiert, ist aber deprecated — die native V2-Form ist unten.

Dann `~/.config/opencode/opencode.json` anlegen. Beispiel: reale, getestete Konfiguration (Qwen3.8-27B mit MTP, ohne API-Key):

```
{
  "$schema": "https://opencode.ai/config.json",
  "model": "omlx/Qwen3.8-27B-oQ4e-mtp",
  "compaction": {
    "auto": true,
    "keep": { "tokens": 10000 },
    "buffer": 16000
  },
  "providers": {
    "omlx": {
      "name": "OMLX",
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": {
        "baseURL": "http://127.0.0.1:8000/v1"
      },
      "models": {
        "Qwen3.8-27B-oQ4e-mtp": {
          "name": "Qwen3.8-27B-oQ4e-mtp",
          "capabilities": {
            "tools": true,
            "input": ["text"],
            "output": ["text"]
          },
          "limit": {
            "context": 150000,
            "output": 16384
          }
        }
      }
    }
  },
  "mcp": {
    "servers": {
      "context-mode": {
        "type": "local",
        "command": ["npx", "-y", "context-mode@latest"],
        "codemode": false
      }
    }
  }
}
```

Wichtig:

- Die Modellreferenz folgt dem Muster `provider-id/model-id` (`"model"` = Default-Modell).
- Der Schlüssel im `models`-Map muss exakt der ID aus `/v1/models` entsprechen; `modelID` (optional) würde die ID überschreiben, die tatsächlich an den Server gesendet wird.
- Die URL verwendet `127.0.0.1`, weil OpenCode nativ auf dem Mac und nicht im Docker-Container läuft.
- `capabilities` und `limit` müssen realistisch sein — OpenCode prüft das nicht gegen den Server. Bei einem Coding-Agenten ist `tools: true` entscheidend (Tool-Use muss vom Modell unterstützt werden).
- `limit.context` passend zum Modell und RAM wählen (→ [Thema 01](../01-omlx-local-llm/), Abschnitt 4). Im Beispiel 150K — der Server muss das mit KV-Cache im Unified Memory stemmen können.
- `compaction`: Auto-Compaction für lange Sessions (Defaults: `auto: true`, `keep.tokens: 15000`, `buffer` = 15 % des Limits, min. 16k). `keep.tokens` kleiner halten = mehr Platz für neue Arbeit, weniger Detailerhalt.
- `mcp` ist optional (hier: MCP-Server `context-mode` per `npx`). Nicht vorhanden, nicht löschen — für reine oMLX-Nutzung einfach weglassen. Details: → [Thema 04](../04-llm-tuning/), Abschnitt 3.9.
- **API-Key:** Das Beispiel läuft ohne `apiKey`, weil der lokale oMLX-Server ohne Auth-Prüfung läuft. Ist im oMLX-Admin-UI eine API-Key-Prüfung aktiviert, `apiKey` in `settings` ergänzen — mit Umgebungsvariablen-Ersetzung statt Klartext: `"apiKey": "{env:OMLX_API_KEY}"` (Variable dann z. B. in `~/.zprofile` setzen). Den Key nie in committete Dateien schreiben.

JSON prüfen:

```
jq . ~/.config/opencode/opencode.json
```

---

## 4. Erster Test in einem Projekt

In ein Testprojekt wechseln:

```
mkdir -p ~/Developer/opencode-test
cd ~/Developer/opencode-test
git init
printf '# OpenCode-Test\n' > README.md
opencode
```

In OpenCode:

```
/models
```

Das lokale oMLX-Modell auswählen, falls es nicht bereits aktiv ist.

> 🔄 **Update Okt 2026:** OpenCode verwendet `AGENTS.md` als dauerhafte Projekt-Instructions-Datei (Build-Kommandos, Architektur, Konventionen, Prüfregeln). `AGENTS.md` im Workspace ist vom Start an aktiv; verschachtelte `AGENTS.md`-Dateien werden geladen, wenn der Agent in diese Bereiche liest. Die Datei bewusst anlegen oder vom Agenten anfertigen lassen (V1-`/init`-Verhalten). Projekt-Instructions-Dateien committen, damit jeder im Repo dieselbe Anleitung bekommt.

Erster ungefährlicher Test:

```
Erstelle einen Plan für ein kleines Python-CLI-Programm, das eine
Markdown-Datei einliest und Überschriften ausgibt. Ändere noch keine Dateien.
```

Anschließend:

```
Implementiere den Plan und erkläre jede erzeugte Datei kurz.
```

---

## 5. Sicherheitskonfiguration

- **oMLX nur lokal lauschen** (`127.0.0.1:8000`) — → [Thema 01](../01-omlx-local-llm/), Abschnitt 8
- Änderungen vor Ausführung kontrollieren
- Shell-Befehle bestätigen lassen (Permissions-Regeln, → [Thema 04](../04-llm-tuning/), Abschnitt 3.6)
- keine Produktions-Repositories verwenden
- keine Secrets oder `.env`-Dateien in Prompts aufnehmen
- keine automatischen Commits oder Pushes zulassen
- Projekt zunächst in einem separaten Testverzeichnis starten

Ein lokales Modell verhindert zwar die Übertragung an einen Cloud-Modellanbieter, aber OpenCode kann weiterhin Dateien verändern und Terminalbefehle ausführen.

---

## 6. Start, Stopp und Updates

```
brew update
brew upgrade opencode-v2
```

Alternativ curl-Installer (aktualisiert auf den neuesten Stand):

```
curl -fsSL https://opencode.ai/v2/install | bash
```

> Config-Änderungen wirken ohne Neustart (OpenCode beobachtet die Config-Dateien). Neuladen in der TUI erzwingen: `/reload`.

---

## 7. Probleme und Lösungen

### OpenCode zeigt das Modell nicht

```
jq . ~/.config/opencode/opencode.json
curl -s http://127.0.0.1:8000/v1/models | jq
```

Dann kontrollieren, ob die Modell-ID in der Konfiguration exakt mit der API-Ausgabe übereinstimmt. Zusätzlich:

```
opencode service status
```

und das Log prüfen:

```
~/.local/share/opencode/log/opencode.log
```

### oMLX erreichbar, aber OpenCode schlägt fehl

- `opencode service status` / `opencode service restart` (Service ist in V2 der Hintergrund-Prozess, der Sessions und Provider besitzt)
- `limit.context` in der Config nicht größer wählen als der Server tatsächlich liefert
- Log mit `OPENCODE_LOG_LEVEL=DEBUG` für eine Reproduktion (Details: → [Thema 04](../04-llm-tuning/), Abschnitt 3.12)

---

## Nächste Schritte

- [Thema 04: LLM Tuning — OMLX Modellconfig + Opencode Config abstimmen (Memory Guard, Memory-Budget, Agents, Permissions, Commands, AGENTS.md, MCP, Troubleshooting)](../04-llm-tuning/)

## Quellen

- OpenCode V2: <https://opencode.ai/v2/docs/>
- Migrate from V1: <https://opencode.ai/v2/docs/migrate-v1/>
- Ursprungsdoku: <https://github.com/mart-ch/locale-ai-on-mac>
