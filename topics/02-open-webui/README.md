# 02 · Open WebUI: Chat-Funktionalität

**Open WebUI** ist eine Chat-Oberfläche für LLMs. Sie läuft als Docker-Container und verbindet sich mit dem lokalen oMLX-Server aus [Topik 01](../01-omlx-local-llm/) über dessen OpenAI-kompatible API — ohne dass oMLX selbst im Docker-Container laufen muss.

<img width="1198" height="507" alt="image" src="https://github.com/user-attachments/assets/e36db9d5-a002-4d09-becb-bfbb7d09dd33" />


## Status & Änderungen

| | |
|---|---|
| **Letzte Prüfung** | 27.10.2026 |

| # | Thema | Status | Update |
|---|---|---|---|
| 1 | Image `ghcr.io/open-webui/open-webui:main`, Port 3000→8080, Volume `open-webui` | ✅ aktuell | Keine Änderung nötig |
| 2 | `host.docker.internal`-URL-Mapping | ✅ aktuell | Docker-spezifisch, bleibt so |

---

## 1. Voraussetzungen

1. **oMLX läuft** mit geladenem Modell (→ [Topik 01](../01-omlx-local-llm/))

```
curl -s http://127.0.0.1:8000/v1/models | jq
```

2. **Docker Desktop** installiert und betriebsbereit:

```
brew install --cask docker
```

Anschließend **Docker** einmal über Programme starten und warten, bis Docker betriebsbereit ist.

Prüfen:

```
docker version
```

> Warum Docker? Docker wird **nur** für die Weboberfläche verwendet, nicht für die MLX-Inference. Das Modell läuft direkt auf dem Mac in oMLX.

---

## 2. Schlüssel erzeugen (Optional)

```
WEBUI_SECRET_KEY="$(openssl rand -hex 32)"
echo "$WEBUI_SECRET_KEY"
```

Den Wert sicher speichern. Er sollte bei späteren Container-Neuanlagen gleich bleiben.

---

## 3. Container starten

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

Flag-Erklärung:

| Flag | Zweck |
|---|---|
| `-p 3000:8080` | Container-Port 8080 auf Host-Port 3000 |
| `--add-host=host.docker.internal:host-gateway` | macht `host.docker.internal` im Container auf den Mac auflösbar — **notwendig** für den Zugriff auf oMLX |
| `-v open-webui:/app/backend/data` | persistentes Volume: Benutzer, Chats, Einstellungen bleiben nach `docker rm` erhalten |
| `--restart always` | startet mit Docker/Boot |

---

## 4. oMLX mit Open WebUI verbinden (Konfiguration)

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

Danach sollte das Qwen-Modell aus [Topik 01](../01-omlx-local-llm/) im Modellmenü erscheinen und Chat-Interaktionen laufen über oMLX (inkl. MTP — das ist eine Eigenschaft von Modell/Server, nicht der Weboberfläche).

---

## 5. Start, Stopp, Logs, Updates

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

Danach den ursprünglichen `docker run`-Befehl (Abschnitt 3) mit demselben `WEBUI_SECRET_KEY` erneut ausführen. Die Daten bleiben wegen des Volumes `open-webui` erhalten.

---

## 6. Probleme und Lösungen

### Open WebUI erreicht oMLX nicht

In Open WebUI muss die URL lauten:

```
http://host.docker.internal:8000/v1
```

Nicht:

```
http://localhost:8000/v1
```

### Modell erscheint nicht im Modellmenü

```
docker logs --tail 100 open-webui
```

Verbindung aus dem Container testen:

```
docker exec open-webui \
  curl -s http://host.docker.internal:8000/v1/models
```

Erscheint hier die Modellliste, ist oMLX erreichbar — dann in Open WebUI Connection neu speichern und Modellmenü aktualisieren.

### Open WebUI ist langsam

- Kontextlänge in oMLX reduzieren (→ [Topik 01](../01-omlx-local-llm/), Abschnitt 4, Schritt 8)
- Open-WebUI-RAG und lokale Embedding-Modelle zunächst deaktivieren (speicherverbrauchend)
- `docker logs -f open-webui` auf oMLX-Fehler prüfen

---

## Nächste Schritte

- [Topik 03: OpenCode — Coding-Agent mit lokalem oMLX](../03-opencode-local-omlx/)

## Quellen

- Open WebUI: <https://github.com/open-webui/open-webui>
- Ursprungsdoku: <https://github.com/mart-ch/locale-ai-on-mac>
