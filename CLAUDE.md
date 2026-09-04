# Club Facility Automation

Vier Steuerungssysteme plus Backend für eine Sportanlage in Hamburg. Produktivbetrieb —
Fehler haben physische Folgen (Wasser, Licht, Pumpen).

## Aufbau
- `firmware/`   ESP32 / ESP32-S3 / Arduino Opta, PlatformIO
- `services/`   Python 3.12, FastAPI, läuft in Docker auf einem Raspberry Pi 5
- `infra/`      Broker, Postgres, Keycloak, FreeRADIUS, Compose-Stack des Pi
- `shared/`     MQTT-Schemas — verbindlicher Vertrag zwischen Firmware und Backend
- `db/`         Schema und Migrationen

## Wichtige Regeln
- MQTT-Payloads NIE einseitig ändern. Erst `shared/mqtt-schemas/` anpassen, dann beide
  Seiten, dann Migration prüfen. Rückwärtskompatibilität ist Pflicht, solange nicht alle
  Geräte geflasht sind.
- Keine Zugangsdaten im Repository. Alles über `.env`, Vorlage in `.env.example`.
- Steuerungslogik bleibt lokal auf dem Gerät. Der Broker transportiert Zustand und
  Befehle, nicht die Logik.
- Sicherheitsrelevante Verriegelungen (Zonenüberschneidung, Trockenlaufschutz,
  Wiedereinschaltsperre) niemals ohne Rückfrage ändern.
- Datenbankänderungen ausschließlich als Migration in `db/migrations/`, nie direkt am
  Schema.

## Befehle
- Firmware bauen:   `cd firmware/<system> && pio run`
- Firmware flashen: `pio run -t upload`
- Backend-Tests:    `cd services/api && pytest`
- Stack starten:    `cd infra && docker compose up -d`

## Stil
- Python: ruff, type hints, keine ungefangenen Exceptions in Dauerläufer-Schleifen
- C++: keine dynamischen Allokationen in Loops, alle Timeouts explizit
- Commits: `bereich: was` (z. B. `poolcontrol: Fehlercode-Auswertung FC-202`)
