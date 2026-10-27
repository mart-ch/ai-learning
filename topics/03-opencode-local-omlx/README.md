# 03 · OpenCode: Installation & Konfiguration mit lokalem oMLX

**OpenCode** ist ein Open-Source-Coding-Agent für das Terminal. Dieser Topik installiert **OpenCode V2** und verbindet ihn mit dem lokalen oMLX-Server aus [Topik 01](../01-omlx-local-llm/) — damit der Coding-Agent das Qwen-Modell ohne Cloud nutzt. Allgemeinere OpenCode-V2-Konfiguration (Agents, Permissions, Commands, Tuning) folgt in [Topik 04](../04-opencode-tuning/).

<img width="1413" height="608" alt="image" src="https://github.com/user-attachments/assets/a37ebb12-5265-47b5-8685-125c6f8579cf" />


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

**oMLX läuft** mit geladenem Modell (→ [Topik 01](../01-omlx-local-llm/)):

```
curl -s http://127.0.0.1:8000/v1/models | jq -r '.data[].id'
```

> OpenCode läuft **nativ auf dem Mac** (kein Docker), daher `127.0.0.1` statt `host.docker.internal` — das ist der Unterschied zur Open-WebUI-Verbindung in [Topik 02](../02-open-webui/).

---

## 2. OpenCode installieren

> 🔄 **Update Okt 2026:** OpenCode **V2** ist die aktuelle Version. Die Ursprungsdoku installierte V1 (`anomalyco/tap/opencode`). Das V2-Curl-Installer ersetzt die V1-Binary; falls eine paketverwaltete V1-Installation vorhanden ist, vorher entfernen.

Offizielle Installationswege für V2 (Wahl eines):

```
curl -fsSL https://opencode.ai/v2/install | bash
```

```
brew install anomalyco/tap/opencode-v2
```

```
npm install -g @opencode/cli
```

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

Dann `~/.config/opencode/opencode.json` anlegen (native V2-Form):

```
{
  "$schema": "https://opencode.ai/config.json",
  "model": "omlx/HIER-DIE-MODELL-ID-EINTRAGEN",
  "providers": {
    "omlx": {
      "name": "oMLX Local",
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": {
        "baseURL": "http://127.0.0.1:8000/v1"
      },
      "models": {
        "HIER-DIE-MODELL-ID-EINTRAGEN": {
          "name": "Qwen lokal via oMLX",
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

Wichtig:

- Die Modellreferenz folgt dem Muster `provider-id/model-id`.
- Der Schlüssel im `models`-Map ist die Modell-ID, die OpenCode verwendet; `modelID` (optional) überschreibt die ID, die tatsächlich an den Server gesendet wird. Hier sollte der Schlüssel exakt der ID aus `/v1/models` entsprechen.
- Die URL verwendet hier `127.0.0.1`, weil OpenCode nativ auf dem Mac und nicht im Docker-Container läuft.
- `capabilities` und `limit` müssen realistisch sein — OpenCode prüft das nicht gegen den Server. Bei einem Coding-Agenten ist `tools: true` entscheidend (Tool-Use muss vom Modell unterstützt werden).
- `limit.context` passend zum Modell und RAM wählen (→ [Topik 01](../01-omlx-local-llm/), Abschnitt 4).
- Ein `apiKey` ist für einen rein lokalen oMLX-Server ohne Auth-Prüfung nicht nötig. Falls oMLX eine API-Key-Prüfung aktiviert hat, `apiKey` in `settings` ergänzen — mit Umgebungsvariablen-Ersetzung statt Klartext: `"apiKey": "{env:OMLX_API_KEY}"`.

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

- **oMLX nur lokal lauschen** (`127.0.0.1:8000`) — → [Topik 01](../01-omlx-local-llm/), Abschnitt 8
- Änderungen vor Ausführung kontrollieren
- Shell-Befehle bestätigen lassen (Permissions-Regeln, → [Topik 04](../04-opencode-tuning/), Abschnitt 6)
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
- Log mit `OPENCODE_LOG_LEVEL=DEBUG` für eine Reproduktion (Details: → [Topik 04](../04-opencode-tuning/), Abschnitt 12)

---

## Nächste Schritte

- [Topik 04: OpenCode — Konfiguration und Tuning (Agents, Permissions, Commands, AGENTS.md, MCP, Troubleshooting)](../04-opencode-tuning/)

## Quellen

- OpenCode V2: <https://opencode.ai/v2/docs/>
- Migrate from V1: <https://opencode.ai/v2/docs/migrate-v1/>
- Ursprungsdoku: <https://github.com/mart-ch/locale-ai-on-mac>
