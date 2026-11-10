# UniFi-Türklingel: Gegensprechen in Home Assistant

Live-Gegensprechen mit einer **UniFi-Protect-Klingel** direkt aus dem **Home-Assistant-Dashboard**, auch in der HA-App auf einem Wandtablet. Die Protect-App wird nicht gebraucht.

- Verzögerung: **unter 1 Sekunde**
- Mit normalem **go2rtc** (kein Spezial-Build) und der **WebRTC-Camera**-Karte
- Getestet mit **UniFi G6 Entry** (Firmware 5.5.45), Home Assistant 2026.11, iPadOS 18.7

> 🇬🇧 English version: [README.md](README.md)

## So funktioniert es

```
iPad-/Browser-Mikrofon ──WebRTC──▶ go2rtc ──stdin (PCMA 8 kHz)──▶ ffmpeg ──Opus 24 kHz / RTP / UDP──▶ Klingel:7004
Bild + Ton der Klingel ◀──WebRTC── go2rtc ◀──RTSPS── UniFi Protect (Port 7441)
```

UniFi-Kameras mit Lautsprecher nehmen Sprachaudio als einfachen RTP/UDP-Stream auf **Port 7004 der Kamera selbst** an. Das erwartete Format steht in Protect in den `talkbackSettings` jeder Kamera. Die HA-Integration für UniFi Protect nutzt diesen Weg schon für Sprachausgaben (TTS). Hier wird derselbe Weg mit dem **Live-Mikrofon** über den `exec`-Rückkanal von go2rtc genutzt.

## Voraussetzungen

| Was | Hinweis |
|---|---|
| Home Assistant mit **HTTPS** | Browser und Apps geben das Mikrofon nur bei sicherer Verbindung frei |
| **go2rtc**-Add-on (AlexxIT) | getestet mit 1.9.15 |
| **WebRTC Camera** (AlexxIT) | über *Einstellungen → Marketplace* (ab HA 2026.11), früher HACS |
| UniFi-Klingel mit Lautsprecher | RTSP für einen Kanal aktiviert |

## Einrichtung

### 1. Talkback-Daten der Klingel auslesen

Im Browser bei der UniFi-Konsole anmelden und aufrufen:

```
https://<KONSOLEN-IP>/proxy/protect/api/cameras
```

Bei deiner Klingel notieren:

- `host` → IP der Klingel
- `talkbackSettings` → z.B. `{"typeFmt":"opus","typeIn":"serverudp-rtp","bindPort":7004,"samplingRate":24000}`
- `channels[].rtspAlias` eines Kanals mit aktiviertem RTSP (Protect → Gerät → Einstellungen → Erweitert)

Optionaler Schnelltest: In HA `tts.speak` auf die Entität `media_player.*_lautsprecher` der Klingel ausführen. Kommt die Ansage an der Tür an, funktioniert der Weg.

### 2. go2rtc-Stream mit Rückkanal

In `/config/go2rtc.yaml` eintragen (siehe [`examples/go2rtc.yaml`](examples/go2rtc.yaml)) und das go2rtc-Add-on neu starten. Codec und Abtastrate an deine `talkbackSettings` anpassen.

Test: Die go2rtc-Oberfläche **über die Seitenleiste von Home Assistant** öffnen (damit läuft sie über deine HTTPS-Adresse; unter `http://<HA-IP>:1984` blockiert der Browser das Mikrofon). An die Adresse `webrtc.html?src=doorbell_talk&media=video+audio+microphone` anhängen und sprechen. Du hörst dich über das Mikrofon der Klingel selbst zurück.

### 3. Helfer und Sicherheits-Automation

- Schalter-Helfer `input_boolean.doorbell_talk` anlegen
- [`examples/automation.yaml`](examples/automation.yaml) importieren: Mikrofon geht nach 3 Minuten und beim Schließen des Klingel-Popups aus

### 4. Dashboard-Karte

Siehe [`examples/card.yaml`](examples/card.yaml). Die Kamerakarte ist zweimal drin: ohne Mikrofon, wenn der Schalter aus ist, und mit Mikrofon, wenn er an ist.

## iOS-App: Mikrofon geht nicht

In der HA-App auf iPhone/iPad kommt eventuell **gar keine Mikrofon-Abfrage**, und in den iOS-Einstellungen unter „Home Assistant“ fehlt der Eintrag „Mikrofon“. Zwei Dinge waren nötig:

1. **HTTPS in der App:** *App-Einstellungen → Companion App → Server*: **Interne und externe URL auf `https://deine-domain`** setzen. Mit `http://` im Heimnetz fehlt der Mikrofonzugriff komplett.
2. **Siri am iPad einschalten** (*Einstellungen → Siri*). In unserem Test hing die Mikrofon-Anfrage ohne Abfrage und ohne Fehlermeldung, solange Siri aus war (das Diktat war schon an, das allein reichte nicht). Nach dem Einschalten von Siri ging es sofort.

## ⚠️ Datenschutz

Die Mikrofon-Erlaubnis gilt **pro Domain**. Jede offene Seite mit einer Karte mit `media: …microphone` sendet sofort, z.B. ein vergessener Browser-Tab am PC.

- Mikrofon-Karte nur hinter einem Schalter anzeigen (wie im Beispiel)
- Auto-Aus-Automation behalten
- Beim Testen andere Dashboards schließen

## Bekannte Einschränkungen

- Nach dem Ausschalten läuft das Audio noch ca. 2 Sekunden nach, bis die WebRTC-Verbindung beendet ist.

## Weitere Modelle

| Modell | Firmware | talkbackSettings | Geht |
|---|---|---|---|
| G6 Entry | 5.5.45 | opus / serverudp-rtp / 24000 Hz / Port 7004 | ✅ |

Mit diesem Aufbau noch nicht getestet:

- **G4 Doorbell Pro / Doorbell Lite**: Laut [go2rtc PR #2340](https://github.com/AlexxIT/go2rtc/pull/2340) verlangt Protect dort Opus mit etwa 22 kHz, und der PR öffnet vorher eine Talkback-Sitzung über Protect. Ob diese Modelle Audio auch direkt auf Port 7004 annehmen (wie die G6 Entry), ist offen.

Prüfe immer deine eigenen `talkbackSettings`. **Ergänze gern dein Modell per Issue oder Pull Request!**

## Danke

- [go2rtc](https://github.com/AlexxIT/go2rtc) und [WebRTC Camera](https://github.com/AlexxIT/WebRTC) von AlexxIT
- [uiprotect](https://github.com/uilibs/uiprotect)
- Siehe auch: [go2rtc PR #2340](https://github.com/AlexxIT/go2rtc/pull/2340), [go2rtc Issue #1760](https://github.com/AlexxIT/go2rtc/issues/1760)

## Lizenz

MIT
