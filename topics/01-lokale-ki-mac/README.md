# 01 · Lokale KI auf einem Mac installieren

Das Ziel ist ein vollständig lokaler Stack aus **oMLX als Inference-Server**, einem **Qwen-Modell mit MTP**, **Open WebUI für Chats** und **OpenCode als Coding-Agent**.

> **Wichtige Begriffsklärung:** **MTP** bedeutet *Multi-Token Prediction* und beschleunigt die Textgenerierung. **MCP** bedeutet *Model Context Protocol* und bindet Werkzeuge oder Datenquellen an. Für die Modellgeschwindigkeit braucht ihr MTP. MCP ist optional.

---

## Status & Änderungen

| | |
|---|---|
| **Letzte Prüfung** | 27.10.2026 |

### Was ist veraltet bzw. neu (Stand Okt 2026)?

| # | Thema in der Ursprungsdoku | Status | Update |
|---|---|---|---|
| 1 | **OpenCode als V1** (`brew install anomalyco/tap/opencode`, Config-Schlüssel `provider` (singular) mit `npm` + `options`) | 🔴 veraltet | OpenCode **V2** ist die aktuelle Version. Installation: `brew install anomalyco/tap/opencode-v2` oder `curl -fsSL https://opencode.ai/v2/install \| bash`. Config-Schlüssel ist jetzt `providers` (plural) mit `package` + `settings`. → Abschnitt 7 und 8 unten |
| 2 | **Qwen-Modellgröße** als „Qwen 4B / 9B / 27B" ohne Versionsgeneration | 🟡 unvollständig | Die RAM-Logik bleibt gültig, aber die Generation hat sich weiterentwickelt: oMLX v0.7.x (Okt 2026) bringt dedizierte Beschleunigungskerne für **Qwen 3.5 / 3.6 / 4**. Aktuelle Generation mit MTP-Support aus dem oMLX-Admin-UI wählen. → Abschnitt 4 |
| 3 | **oMLX ohne Versionsverweis** | 🟡 veraltet | Aktueller Stand: **v0.7.x** (Release v0.7.3.dev2, 25.10.2026). Optional: Build mit `--with-custom-kernel` für maximale Qwen-Geschwindigkeit. → Abschnitt 3 |
| 4 | **`/init` in OpenCode** erzeugt `AGENTS.md` | 🟡 unklar in V2 | Die V2-Doku definiert `AGENTS.md` als Projekt-Instructions-Datei (Entdeckung: Workspace + verschachtelt). In V2 die Datei bewusst anlegen oder vom Agenten anlegen lassen; `/init` ist eine V1-TUI-Verhaltensweise. → Abschnitt 9 |
| 5 | Homebrew-Tap oMLX, Port 8000, Admin-UI, Open WebUI-Docker-Setup, Sicherheitsabschnitt | ✅ aktuell | Keine Änderung nötig |

**Noch gültig:** `brew tap jundot/omlx`, `omlx start/stop/restart`, Admin unter `http://localhost:8000/admin`, Open WebUI als Docker-Container mit Volume `open-webui`, `host.docker.internal`-URL-Mapping.

---

## 1. Voraussetzungen prüfen

Empfohlen:

- Apple Silicon M3+ (Pro, Max)
- macOS 15 oder neuer (oMLX-Homebrew-Formel erfordert Sequoia; neuere macOS-Versionen funktionieren ebenfalls)
- mindestens 16 GB Unified Memory
- mindestens 30 bis 50 GB freier SSD-Speicher
- für ein 4B- oder 9B-Modell reichen meistens 16 bis 24 GB RAM
- für 27B-Modelle sollten mindestens 32 GB, besser 48 GB vorhanden sein

Im Terminal prüfen:

```
uname -m
sw_vers
system_profiler SPHardwareDataType | grep -E "Chip|Memory"
df -h /
```

Erwartet wird bei `uname -m`:

```
arm64
```

---

## 2. Grundwerkzeuge installieren

### 2.1 Homebrew

Falls Homebrew noch nicht installiert ist:

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Danach bei einem Apple-Silicon-Mac gegebenenfalls Homebrew in die Shell aufnehmen:

```
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Prüfen:

```
brew --version
```

### 2.2 Hilfsprogramme

```
brew install git curl jq
```

Für Open WebUI wird außerdem Docker Desktop empfohlen:

```
brew install --cask docker
```

Anschließend **Docker** einmal über Programme starten und warten, bis Docker betriebsbereit ist.

Prüfen:

```
docker version
```

---

## 3. oMLX installieren

### Empfohlene Variante: Homebrew

Für dieses Setup wird die Installation über Homebrew empfohlen. Dadurch lässt sich oMLX bequem über das Terminal installieren, als Hintergrunddienst starten und später mit Homebrew aktualisieren.

Zunächst das oMLX-Repository zu Homebrew hinzufügen:

```
brew tap jundot/omlx https://github.com/jundot/omlx
```

Anschließend oMLX installieren:

```
brew install jundot/omlx/omlx
```

Den oMLX-Dienst starten:

```
omlx start
```

Das standardmäßige Modellverzeichnis ist:

```
~/.omlx/models
```

Das Admin-Interface ist anschließend erreichbar unter:

```
http://localhost:8000/admin
```

Eine einfache eingebaute Chat-Oberfläche gibt es unter:

```
http://localhost:8000/admin/chat
```

Status und API prüfen:

```
brew services info omlx
curl -s http://127.0.0.1:8000/v1/models | jq
```

Falls der Dienst nicht erreichbar ist, oMLX neu starten:

```
omlx restart
```

Die Protokolle können je nach Installation an einem der folgenden Orte liegen:

```
tail -f ~/.omlx/logs/server.log
```

Alternativ:

```
tail -f "$(brew --prefix)/var/log/omlx.log"
```

> 🔄 **Update Okt 2026:** Aktueller oMLX-Stand ist **v0.7.x** (Release v0.7.3.dev2 vom 25.10.2026). Die Homebrew-Formel bietet zusätzlich `--with-custom-kernel` (buildet native Kerne für Beschleunigung u. a. von Qwen 3.5/3.6/4). Für maximale Geschwindigkeit mit Qwen:
>
> ```
> brew tap jundot/omlx https://github.com/jundot/omlx
> brew install jundot/omlx/omlx --with-custom-kernel
> ```

### Alternative: native macOS-App

Falls eine grafische Installation bevorzugt wird, kann stattdessen die signierte oMLX-App (Menüleisten-App) verwendet werden:

1. Die aktuelle `.dmg` von der offiziellen oMLX-Projektseite herunterladen.
2. oMLX nach **Programme** ziehen.
3. oMLX starten.
4. Als Modellverzeichnis `~/.omlx/models` übernehmen.
5. Den lokalen Server über die App starten.

> Es sollte nur eine Installationsvariante aktiv verwendet werden. Läuft oMLX gleichzeitig über Homebrew und über die App, kann es zu einem Portkonflikt auf Port `8000` kommen.

---

## 4. Passendes Qwen-Modell auswählen

Das Modell muss ausdrücklich:

1. im **MLX-Format** vorliegen,
2. zu oMLX passen,
3. einen **MTP-Checkpoint beziehungsweise integrierte MTP-Heads** enthalten,
4. einschließlich KV-Cache in den Unified Memory passen.

> 🔄 **Update Okt 2026:** Die Qwen-Generation hat sich weiterentwickelt. oMLX v0.7.x bringt dedizierte Beschleunigungskerne für **Qwen 3.5 / 3.6 / 4**. Im oMLX-Admin-UI die **aktuelle Qwen-Generation** mit MTP (Lightning MTP) wählen. Die Größen-Orientierung unten bleibt als RAM-Logik gültig.

### Orientierung nach Arbeitsspeicher

| Unified Memory | Empfohlene Modellgröße |
| --- | --- |
| 16 GB | Qwen 4B, quantisiert |
| 18 bis 24 GB | Qwen 4B oder 9B, quantisiert |
| 32 bis 36 GB | Qwen 9B oder eventuell 27B mit aggressiver Quantisierung |
| 48 GB oder mehr | Qwen 27B, quantisiert |
| 64 GB oder mehr | Qwen 27B komfortabel, längerer Kontext |

### Empfehlung für ein normales M3 MacBook

- **16 oder 18 GB:** Qwen 4B, 4-Bit oder oQ4, mit MTP
- **24 GB:** Qwen 9B, 4-Bit oder oQ4, mit MTP
- **36 GB:** zunächst Qwen 9B; 27B nur mit niedriger Quantisierung und begrenztem Kontext
- **48 GB oder mehr:** Qwen 27B, 4-Bit oder oQ4, mit MTP

### Modell über oMLX herunterladen

1. `http://localhost:8000/admin` öffnen.
2. Zu **Models** beziehungsweise **Download Model** wechseln.
3. Nach einem aktuellen Qwen-Modell suchen.
4. Darauf achten, dass im Modellnamen oder in der Beschreibung `MLX` und `MTP` vorkommen.
5. Modell herunterladen.
6. Das Modell in den Einstellungen öffnen.
7. **MTP** beziehungsweise **Lightning MTP** aktivieren, sofern nicht automatisch erkannt.
8. Für den ersten Test die Kontextlänge auf `16384` oder `32768` begrenzen.

> Nicht irgendein Qwen-MLX-Modell auswählen. Ein normales MLX-Modell ohne MTP-Heads bekommt durch Aktivieren eines Schalters nicht nachträglich MTP-Unterstützung.

---

## 5. oMLX und MTP testen

Zuerst nachsehen, welche Modell-ID der Server meldet:

```
curl -s http://127.0.0.1:8000/v1/models | jq -r '.data[].id'
```

Eine der ausgegebenen IDs als Variable setzen:

```
MODEL_ID="HIER-DIE-MODELL-ID-EINTRAGEN"
```

Testanfrage:

```
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"${MODEL_ID}\",
    \"messages\": [
      {
        \"role\": \"user\",
        \"content\": \"Antworte mit genau einem kurzen deutschen Satz: Läuft das Modell lokal?\"
      }
    ],
    \"temperature\": 0.2,
    \"max_tokens\": 100
  }" | jq
```

Während der Anfrage im oMLX-Dashboard oder Log prüfen, ob MTP aktiv ist.

Logs bei einer Homebrew-Installation:

```
tail -f ~/.omlx/logs/server.log
```

Oder:

```
tail -f "$(brew --prefix)/var/log/omlx.log"
```

Nach Begriffen wie `MTP`, `native MTP`, `Lightning MTP`, `accepted tokens` oder `draft acceptance` suchen:

```
grep -iE "mtp|lightning|accepted|draft" ~/.omlx/logs/server.log | tail -50
```

---

## 6. Open WebUI installieren

Open WebUI läuft bequem als Docker-Container. Ein persistentes Volume speichert Benutzer, Chats und Einstellungen.

### 6.1 Schlüssel erzeugen (Optional)

Im Terminal:

```
WEBUI_SECRET_KEY="$(openssl rand -hex 32)"
echo "$WEBUI_SECRET_KEY"
```

Den Wert sicher speichern. Er sollte bei späteren Container-Neuanlagen gleich bleiben.

### 6.2 Container starten

```
docker run -d \
  -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  -e WEBUI_SECRET_KEY="$WEBUI_SECRET_KEY" \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Die Oberfläche öffnen:

```
http://localhost:3000
```

Der erste angelegte Benutzer wird üblicherweise Administrator.

### 6.3 oMLX mit Open WebUI verbinden

In Open WebUI:

1. **Admin Panel** öffnen.
2. **Settings** auswählen.
3. **Connections** öffnen.
4. **OpenAI API** auswählen.
5. Eine neue Verbindung anlegen.

Eintragen:

```
URL:
http://host.docker.internal:8000/v1

API key:
local-omlx
```

Der API-Key kann bei einem ausschließlich lokalen oMLX-Server ein beliebiger nichtleerer Wert sein, sofern in oMLX keine API-Key-Prüfung aktiviert wurde.

> Innerhalb eines Docker-Containers zeigt `localhost` auf den Container selbst. Deshalb muss hier `host.docker.internal` statt `localhost` verwendet werden.

Danach sollte das Qwen-Modell im Modellmenü erscheinen.

Falls es nicht erscheint:

```
docker logs --tail 100 open-webui
```

Verbindung aus dem Container testen:

```
docker exec open-webui \
  curl -s http://host.docker.internal:8000/v1/models
```

---

## 7. OpenCode installieren

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

## 8. OpenCode mit oMLX verbinden

Die globale Konfiguration liegt auf macOS unter:

```
~/.config/opencode/opencode.json
```

(Jeder JSON-Datei sollte das `$schema`-Feld mitgeben, damit der Editor Validierung und Autocomplete liefert. V2 unterstützt auch JSONC, d. h. Kommentare und ohne Kommas am Zeilenende sind erlaubt: `opencode.jsonc`.)

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
- `capabilities` und `limit` müssen realistisch sein — OpenCode prüft das nicht gegen den Server. Bei einem Coding-Agenten ist `tools: true` entscheidend (Tool-Use / Function Calling muss vom Modell unterstützt werden).
- Ein `apiKey` ist für einen rein lokalen oMLX-Server ohne Auth-Prüfung nicht nötig. Falls oMLX eine API-Key-Prüfung aktiviert hat, `apiKey` in `settings` ergänzen — mit Umgebungsvariablen-Ersetzung statt Klartext: `"apiKey": "{env:OMLX_API_KEY}"`.
- JSON prüfen:

```
jq . ~/.config/opencode/opencode.json
```

---

## 9. OpenCode ausprobieren

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

> 🔄 **Update Okt 2026:** OpenCode verwendet `AGENTS.md` als dauerhafte Projekt-Instructions-Datei (Build-Kommandos, Architektur, Konventionen, Prüfregeln). `AGENTS.md` im Workspace liegt bei Projektstart aktiv; verschachtelte `AGENTS.md`-Dateien werden geladen, wenn der Agent in diese Bereiche liest. Die Datei bewusst anlegen oder vom Agenten anfertigen lassen (V1-`/init`-Verhalten). Projekt-Instructions-Dateien committen, damit jeder im Repo dieselbe Anleitung bekommt.

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

## 10. Empfohlene Sicherheitskonfiguration

### Nur lokal lauschen

oMLX sollte auf Folgendem gebunden bleiben:

```
127.0.0.1:8000
```

Nicht ohne Authentifizierung auf `0.0.0.0` oder im Firmennetz freigeben.

### OpenCode-Rechte vorsichtig verwenden

Für die ersten Tests:

- Änderungen vor Ausführung kontrollieren
- Shell-Befehle bestätigen lassen (Permissions, siehe [Topik 02](../02-opencode-konfiguration/))
- keine Produktions-Repositories verwenden
- keine Secrets oder `.env`-Dateien in Prompts aufnehmen
- keine automatischen Commits oder Pushes zulassen
- Projekt zunächst in einem separaten Testverzeichnis starten

Ein lokales Modell verhindert zwar die Übertragung an einen Cloud-Modellanbieter, aber OpenCode kann weiterhin Dateien verändern und Terminalbefehle ausführen.

---

## 11. Start, Stopp und Updates

### oMLX

```
omlx start
omlx stop
omlx restart
```

Update bei Homebrew:

```
brew update
brew upgrade omlx
```

### Open WebUI

Starten:

```
docker start open-webui
```

Stoppen:

```
docker stop open-webui
```

Logs:

```
docker logs -f open-webui
```

Update:

```
docker pull ghcr.io/open-webui/open-webui:main
docker rm -f open-webui
```

Danach den ursprünglichen `docker run`-Befehl mit demselben `WEBUI_SECRET_KEY` erneut ausführen. Die Daten bleiben wegen des Volumes `open-webui` erhalten.

### OpenCode

```
brew update
brew upgrade opencode-v2
```

Alternativ curl-Installer (aktualisiert auf den neuesten Stand):

```
curl -fsSL https://opencode.ai/v2/install | bash
```

---

## 12. Häufige Fehler

### `Connection refused` auf Port 8000

```
omlx start
curl -v http://127.0.0.1:8000/v1/models
```

Außerdem prüfen:

```
lsof -nP -iTCP:8000 -sTCP:LISTEN
```

### Open WebUI erreicht oMLX nicht

In Open WebUI muss die URL lauten:

```
http://host.docker.internal:8000/v1
```

Nicht:

```
http://localhost:8000/v1
```

### OpenCode zeigt kein Modell

```
jq . ~/.config/opencode/opencode.json
curl -s http://127.0.0.1:8000/v1/models | jq
```

Dann kontrollieren, ob die Modell-ID in der Konfiguration exakt mit der API-Ausgabe übereinstimmt. In V2 zusätzlich prüfen:

```
opencode service status
```

und das Log:

```
~/.local/share/opencode/log/opencode.log
```

### Mac wird langsam oder Anwendungen schließen sich

- kleineres Modell verwenden
- Kontext auf 8K oder 16K reduzieren
- parallele Anfragen auf 1 setzen
- Open-WebUI-RAG und lokale Embedding-Modelle zunächst deaktivieren
- Browser-Tabs und speicherintensive Anwendungen schließen

### MTP scheint nicht aktiv zu sein

- Modellname und Modellbeschreibung auf `MTP` prüfen
- model-specific settings in oMLX kontrollieren
- Logs nach `MTP` durchsuchen
- MTP nur mit einem dafür vorgesehenen Checkpoint aktivieren
- oMLX auf die aktuelle stabile Version aktualisieren

---

## Abschließender Funktionstest

Diese drei Tests sollten funktionieren:

```
# 1. Server erreichbar
curl -s http://127.0.0.1:8000/v1/models | jq

# 2. Open WebUI läuft
open http://localhost:3000

# 3. OpenCode im Projekt starten
cd ~/Developer/opencode-test
opencode
```

Die fertige Architektur sieht dann so aus:

```
Open WebUI im Docker-Container
        |
        | OpenAI-kompatible API
        v
oMLX auf macOS, Port 8000
        |
        v
Qwen MLX-Modell mit MTP

OpenCode auf macOS
        |
        | OpenAI-kompatible API
        v
oMLX auf macOS, Port 8000
```

## Warum diese Struktur sinnvoll ist

- oMLX ist die einzige Inference-Schicht.
- Open WebUI und OpenCode verwenden dasselbe Modell.
- Das Modell wird nicht doppelt geladen.
- Chats und Coding laufen über einen einheitlichen lokalen API-Endpunkt.
- Docker wird nur für die Weboberfläche verwendet, nicht für die MLX-Inference.
- MTP bleibt eine Eigenschaft von Modell und oMLX und muss in Open WebUI oder OpenCode nicht separat konfiguriert werden.
