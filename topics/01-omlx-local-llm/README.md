# 01 · oMLX: Lokaler LLM-Server auf dem Mac

**oMLX** ist ein LLM-Inference-Server für Apple Silicon (Continuous Batching, SSD-Caching, macOS-Menüleiste). Er ist die **einzige Inference-Schicht** des lokalen Stacks: danach bauen [Open WebUI (Thema 02)](../02-open-webui/) für Chats und [OpenCode (Thema 03)](../03-opencode-local-omlx/) für Coding auf dieselbe OpenAI-kompatible API auf.

<img width="1320" height="931" alt="image" src="https://github.com/user-attachments/assets/c3316931-aa5e-4542-8076-3e08f8134beb" />


> **Wichtige Begriffsklärung:** **MTP** bedeutet *Multi-Token Prediction* und beschleunigt die Textgenerierung. **MCP** bedeutet *Model Context Protocol* und bindet Werkzeuge oder Datenquellen an. Für die Modellgeschwindigkeit braucht ihr MTP. MCP ist optional.

## Inhaltsverzeichnis

- [Status & Änderungen](#status--änderungen)
- [1. Voraussetzungen prüfen](#1-voraussetzungen-prüfen)
- [2. Grundwerkzeuge installieren](#2-grundwerkzeuge-installieren)
  - [2.1 Homebrew](#21-homebrew)
  - [2.2 Hilfsprogramme](#22-hilfsprogramme)
- [3. oMLX installieren](#3-omlx-installieren)
  - [Empfohlene Variante: Homebrew](#empfohlene-variante-homebrew)
  - [Alternative: native macOS-App](#alternative-native-macos-app)
- [4. Passendes Qwen-Modell (LLM) auswählen](#4-passendes-qwen-modell-llm-auswählen)
  - [Orientierung nach Arbeitsspeicher](#orientierung-nach-arbeitsspeicher)
  - [Empfehlung für ein normales M3 MacBook](#empfehlung-für-ein-normales-m3-macbook)
  - [Modell über oMLX herunterladen](#modell-über-omlx-herunterladen)
- [5. oMLX und MTP testen](#5-omlx-und-mtp-testen)
- [6. Start, Stopp und Updates](#6-start-stopp-und-updates)
- [7. Probleme und Lösungen](#7-probleme-und-lösungen)
  - [`Connection refused` auf Port 8000](#auf-port-8000)
  - [Mac wird langsam oder Anwendungen schließen sich](#mac-wird-langsam-oder-anwendungen-schließen-sich)
  - [MTP scheint nicht aktiv zu sein](#mtp-scheint-nicht-aktiv-zu-sein)
  - [Port 8000 belegt (App + Homebrew parallel)](#port-8000-belegt-app--homebrew-parallel)
- [8. Sicherheit: nur lokal lauschen](#8-sicherheit-nur-lokal-lauschen)
- [9. Funktionstest](#9-funktionstest)
- [Nächste Schritte](#nächste-schritte)
- [Quellen](#quellen)

## Status & Änderungen

| | |
|---|---|
| **Letzte Prüfung** | 27.10.2026 |

| # | Thema | Status | Update |
|---|---|---|---|
| 1 | **oMLX in offiziellen Homebrew?** | 🔍 geprüft 27.10.2026 | oMLX ist **nicht** in homebrew-core. `formulae.brew.sh/api/formula/omlx.json` → 404, Formula-Datei in `Homebrew/homebrew-core` → 404. Das Bare-`omlx`-Eintrag in `brew formulae` ist die installierte **Tap-Formula**, keine offizielle Core-Formula. → Tap ist erforderlich (Abschnitt 3) |
| 2 | **oMLX ohne Versionsverweis** | 🟡 veraltet | Aktueller Stand: **v0.7.x** (Release v0.7.3.dev2, 25.10.2026). Optional: Build mit `--with-custom-kernel` für maximale Qwen-Geschwindigkeit. → Abschnitt 3 |
| 3 | **Qwen-Modellgröße** als „Qwen 4B / 9B / 27B" ohne Generation | 🟡 unvollständig | Die RAM-Logik bleibt gültig, die Generation hat sich weiterentwickelt: oMLX v0.7.x bringt dedizierte Kerne für **Qwen 3.5 / 3.6 / 4**. Aktuelle Generation mit MTP aus dem Admin-UI wählen. → Abschnitt 4 |
| 4 | Port 8000, Admin-UI, Log-Pfade | ✅ aktuell | Keine Änderung nötig |

**Gültig:** `brew tap jundot/omlx`, `omlx start/stop/restart`, Admin unter `http://localhost:8000/admin`, Chat-UI unter `http://localhost:8000/admin/chat`.

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

> Docker wird für oMLX **nicht** benötigt — erst für Open WebUI in [Thema 02](../02-open-webui/).

---

## 3. oMLX installieren

### Empfohlene Variante: Homebrew

Für dieses Setup wird die Installation über Homebrew empfohlen. Dadurch lässt sich oMLX bequem über das Terminal installieren, als Hintergrunddienst starten und später mit Homebrew aktualisieren.

> **Wichtig — Tap ist erforderlich:** oMLX liegt **nicht** im offiziellen homebrew-core (prüft: `formulae.brew.sh/api/formula/omlx.json` → 404). Ein Bare-`omlx`-Eintrag in `brew formulae` ist nur die bereits installierte Tap-Formula. Vor der Installation daher zuerst den Tap hinzufügen:

Zunächst das oMLX-Repository zu Homebrew hinzufügen:

```
brew tap jundot/omlx https://github.com/jundot/omlx
```

Anschließend oMLX installieren:

```
brew install jundot/omlx/omlx
```

(Alternative Schreibweise nach Tap-Installation: `brew install omlx`.)

> 🔄 **Update Okt 2026:** Aktueller oMLX-Stand ist **v0.7.x** (Release v0.7.3.dev2 vom 25.10.2026). Die Homebrew-Formel bietet zusätzlich `--with-custom-kernel` (buildet native Kerne für Beschleunigung u. a. von Qwen 3.5/3.6/4). Für maximale Geschwindigkeit mit Qwen:
>
> ```
> brew tap jundot/omlx https://github.com/jundot/omlx
> brew install jundot/omlx/omlx --with-custom-kernel
> ```

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

### Alternative: native macOS-App

Falls eine grafische Installation bevorzugt wird, kann stattdessen die signierte oMLX-App (Menüleisten-App) verwendet werden:

1. Die aktuelle `.dmg` von der offiziellen oMLX-Projektseite herunterladen.
2. oMLX nach **Programme** ziehen.
3. oMLX starten.
4. Als Modellverzeichnis `~/.omlx/models` übernehmen.
5. Den lokalen Server über die App starten.

> Es sollte nur eine Installationsvariante aktiv verwendet werden. Läuft oMLX gleichzeitig über Homebrew und über die App, kann es zu einem Portkonflikt auf Port `8000` kommen.

---

## 4. Passendes Qwen-Modell (LLM) auswählen

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

## 6. Start, Stopp und Updates

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

---

## 7. Probleme und Lösungen

### `Connection refused` auf Port 8000

```
omlx start
curl -v http://127.0.0.1:8000/v1/models
```

Außerdem prüfen:

```
lsof -nP -iTCP:8000 -sTCP:LISTEN
```

### Mac wird langsam oder Anwendungen schließen sich

- kleineres Modell verwenden
- Kontext auf 8K oder 16K reduzieren
- parallele Anfragen auf 1 setzen
- Browser-Tabs und speicherintensive Anwendungen schließen

### MTP scheint nicht aktiv zu sein

- Modellname und Modellbeschreibung auf `MTP` prüfen
- model-specific settings in oMLX kontrollieren
- Logs nach `MTP` durchsuchen
- MTP nur mit einem dafür vorgesehenen Checkpoint aktivieren
- oMLX auf die aktuelle stabile Version aktualisieren

### Port 8000 belegt (App + Homebrew parallel)

Nur eine Installationsvariante aktiv halten (Abschnitt 3, Alternative). Dann:

```
brew services info omlx
lsof -nP -iTCP:8000 -sTCP:LISTEN
```

---

## 8. Sicherheit: nur lokal lauschen

oMLX sollte auf Folgendem gebunden bleiben:

```
127.0.0.1:8000
```

Nicht ohne Authentifizierung auf `0.0.0.0` oder im Firmennetz freigeben.

---

## 9. Funktionstest

```
# Server erreichbar
curl -s http://127.0.0.1:8000/v1/models | jq
```

Die fertige Architektur sieht dann so aus:

```
Open WebUI (Thema 02)         OpenCode (Thema 03)
        |                            |
        | OpenAI-kompatible API      |
        v                            v
oMLX auf macOS, Port 8000
        |
        v
Qwen MLX-Modell mit MTP
```

**Warum diese Struktur sinnvoll ist:**

- oMLX ist die einzige Inference-Schicht.
- Open WebUI und OpenCode verwenden dasselbe Modell.
- Das Modell wird nicht doppelt geladen.
- Chats und Coding laufen über einen einheitlichen lokalen API-Endpunkt.
- MTP bleibt eine Eigenschaft von Modell und oMLX und muss in den Clients nicht konfiguriert werden.

---

## Nächste Schritte

- [Thema 02: Open WebUI — Chat-Funktionalität](../02-open-webui/)
- [Thema 03: OpenCode — Coding-Agent mit lokalem oMLX](../03-opencode-local-omlx/)

## Quellen

- oMLX: <https://github.com/jundot/omlx>
- Ursprungsdoku: <https://github.com/mart-ch/locale-ai-on-mac>
