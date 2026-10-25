<p align="center">
  <a href="https://tonnenplan.app"><img src="logo.png" alt="TonnenPlan" width="325"></a>
</p>

# TonnenPlan – Abfuhrkalender als iCal, CSV, JSON und PDF

[TonnenPlan](https://tonnenplan.app) zeigt die Abfuhrtermine für Restmüll, Biomüll, Verpackungsmüll (Gelber Sack / Gelbe Tonne), Altpapier und mehr – für Deine Straße in mehreren Landkreisen in Niedersachsen. Mit einem kostenlosen Konto kannst Du Deine Termine exportieren: als Kalender-Abo, zur Weiterverarbeitung in Excel oder für Automatisierungen wie Home Assistant.

Dieses Repository beschreibt die Exportformate und enthält Beispieldateien sowie fertige Konfigurationen. Der Quellcode von TonnenPlan selbst ist nicht Teil dieses Repositorys.

## Unterstützte Regionen

| Landkreis / Region | Bundesland |
|---|---|
| Landkreis Goslar | Niedersachsen |
| Landkreis Hameln-Pyrmont | Niedersachsen |
| Region Hannover | Niedersachsen |
| Landkreis Hildesheim | Niedersachsen |
| Landkreis Holzminden | Niedersachsen |
| Landkreis Northeim | Niedersachsen |
| Landkreis Peine | Niedersachsen |

## Inhalt

- [Export-Links](#export-links)
- [Beispieldateien](#beispieldateien)
- [iCal (ICS)](#ical-ics)
- [JSON](#json)
- [CSV](#csv)
- [PDF](#pdf)
- [Abfallarten](#abfallarten)
- [Home Assistant](#home-assistant)
- [Kalender abonnieren](#kalender-abonnieren)

## Export-Links

1. Auf [tonnenplan.app](https://tonnenplan.app) ein kostenloses Konto anlegen und Deine Straße auswählen.
2. In den Konto-Einstellungen findest Du die Export-Links und Deinen **Export-Token** (32 Zeichen, Kleinbuchstaben und Ziffern).

Die Links haben diesen Aufbau:

```
https://tonnenplan.app/export/{format}/{token}
https://tonnenplan.app/export/{format}/{jahr}/{token}
```

| Platzhalter | Werte |
|---|---|
| `format` | `ics`, `json`, `csv`, `pdf` |
| `jahr` | optional; das aktuelle oder das nächste Jahr. Ohne Angabe gilt das aktuelle Jahr. |
| `token` | Dein Export-Token |

Wichtig zu wissen:

- **Der Token ist Deine Adresse.** Wer den Link kennt, sieht Deine Abfuhrtermine und damit Deine Straße. Teile ihn nicht öffentlich. In den Konto-Einstellungen kannst Du einen neuen Token erzeugen; alte Links funktionieren danach nicht mehr.
- **Nur kommende Termine.** iCal, JSON und CSV enthalten Termine ab heute; vergangene Termine fallen automatisch weg. Das PDF zeigt das ganze Jahr.
- **Jahreswechsel.** Ein Link ohne Jahr liefert immer das laufende Jahr. Sobald der Landkreis die Termine fürs nächste Jahr veröffentlicht hat, sind sie zusätzlich unter `/export/{format}/{nächstes Jahr}/{token}` abrufbar.
- Bei ungültigem Token oder Jahr antwortet der Server mit HTTP-Status `422`.

## Beispieldateien

Alle Beispiele stammen von einer echten Straße (Freden, Alfelder Straße, Landkreis Hildesheim) für 2026:

| Datei | Format |
|---|---|
| [`beispiele/TonnenPlan-Beispiel.ics`](beispiele/TonnenPlan-Beispiel.ics) | iCal |
| [`beispiele/TonnenPlan-Beispiel.json`](beispiele/TonnenPlan-Beispiel.json) | JSON |
| [`beispiele/TonnenPlan-Beispiel.csv`](beispiele/TonnenPlan-Beispiel.csv) | CSV |
| [`beispiele/TonnenPlan-Beispiel.pdf`](beispiele/TonnenPlan-Beispiel.pdf) | PDF |

## iCal (ICS)

Für Smartphone- und PC-Kalender. Am besten als **Abo** einbinden (siehe [Kalender abonnieren](#kalender-abonnieren)), dann kommen Terminänderungen des Landkreises automatisch an.

- Jede Abfuhr ist ein **ganztägiger Termin** (`DTSTART;VALUE=DATE`).
- `SUMMARY` ist der Name der Abfallart, z. B. `Restmüll`.
- `LOCATION` enthält Ortsteil und Straße, z. B. `Freden - Alfelder Straße`.
- `UID` ist pro Termin stabil, Kalender erkennen bereits bekannte Termine also wieder.
- Hast Du im Konto eine Erinnerungszeit eingestellt, enthält jeder Termin eine Erinnerung (`VALARM`) am Vorabend, z. B. `TRIGGER:-PT3H` = 21 Uhr am Tag vorher.

```
BEGIN:VEVENT
UID:99082c3644ac839d33048a48deeb1a14
DTSTART;VALUE=DATE:20260106
DTEND;VALUE=DATE:20260107
LOCATION:Freden - Alfelder Straße
SUMMARY:Verpackungsmüll
BEGIN:VALARM
ACTION:DISPLAY
TRIGGER:-PT3H
DESCRIPTION:Verpackungsmüll - Abholung am 06.01.2026
END:VALARM
END:VEVENT
```

## JSON

Für Entwickler und Automatisierungen. Ein Array von Terminen, sortiert nach Datum:

```json
[
    {
        "uid": "99082c3644ac839d33048a48deeb1a14",
        "date": "2026-01-06",
        "type": "plastic",
        "interval": null
    },
    {
        "uid": "98db56e5728658e7b58cc3365bcd5fc0",
        "date": "2026-01-07",
        "type": "residual",
        "interval": 28
    }
]
```

| Feld | Typ | Bedeutung |
|---|---|---|
| `uid` | string | Stabile ID des Termins (32 Hex-Zeichen), identisch mit der `UID` im iCal |
| `date` | string | Abfuhrdatum im Format `YYYY-MM-DD` |
| `type` | string | Abfallart, siehe [Abfallarten](#abfallarten) |
| `interval` | int \| null | Abfuhrrhythmus in Tagen, falls es in der Straße mehrere Rhythmen für dieselbe Abfallart gibt (z. B. `28` = 4-wöchig, `14` = 2-wöchig); sonst `null` |

Beispiel mit `curl` und `jq` – die nächste Abfuhr:

```sh
curl -s https://tonnenplan.app/export/json/DEIN_EXPORT_TOKEN | jq '.[0]'
```

Beispiel in Python:

```python
import json
import urllib.request

url = "https://tonnenplan.app/export/json/DEIN_EXPORT_TOKEN"
with urllib.request.urlopen(url) as response:
    termine = json.load(response)

for termin in termine[:5]:
    print(termin["date"], termin["type"])
```

## CSV

Zur Weiterverarbeitung in Excel, LibreOffice & Co. Dieselben Felder wie im JSON:

```
"uid";"date";"type";"interval"
"99082c3644ac839d33048a48deeb1a14";"2026-01-06";"plastic";""
"98db56e5728658e7b58cc3365bcd5fc0";"2026-01-07";"residual";"28"
```

- Trennzeichen Semikolon, alle Werte in Anführungszeichen
- Zeichensatz UTF-8, Zeilenende CRLF
- Leeres `interval` bedeutet: kein abweichender Rhythmus

## PDF

Ein Jahreskalender im A4-Querformat zum Ausdrucken, mit allen Abfuhrterminen des Jahres für Deine Straße. Siehe [`beispiele/TonnenPlan-Beispiel.pdf`](beispiele/TonnenPlan-Beispiel.pdf).

## Abfallarten

| `type` | Name im Kalender |
|---|---|
| `residual` | Restmüll |
| `organic` | Biomüll |
| `plastic` | Verpackungsmüll |
| `paper` | Altpapier |
| `glass` | Altglas |
| `clothing` | Altkleider |
| `bulky` | Sperrmüll |
| `harmful` | Schadstoffe |

Welche Arten vorkommen, hängt vom Landkreis ab.

## Home Assistant

Zwei fertige Konfigurationen liegen im Ordner [`home-assistant/`](home-assistant):

- [`waste_collection_schedule.yaml`](home-assistant/waste_collection_schedule.yaml) – über die beliebte Integration [Waste Collection Schedule](https://github.com/mampfes/hacs_waste_collection_schedule) (HACS) und deren iCal-Quelle. Empfohlen, wenn Du Karten, Icons und Countdown-Sensoren willst.
- [`rest_sensor.yaml`](home-assistant/rest_sensor.yaml) – ohne Zusatzintegration, nur mit der eingebauten RESTful-Integration und dem JSON-Export. Liefert Datum und Abfallart der nächsten Abfuhr.

In beiden Dateien `DEIN_EXPORT_TOKEN` durch Deinen Token ersetzen. Den Token am besten in `secrets.yaml` ablegen.

## Kalender abonnieren

Ein Abo hält den Kalender aktuell, ein einmaliger Import nicht. Verwende dafür den iCal-Link `https://tonnenplan.app/export/ics/DEIN_EXPORT_TOKEN`.

**Google Kalender** (am PC): Links neben „Weitere Kalender“ auf **+** → **Per URL** → Link einfügen → **Kalender hinzufügen**. Google aktualisiert Abos nach eigenem Zeitplan, das kann bis zu einem Tag dauern.

**Apple Kalender** (iPhone): Einstellungen → Kalender → Accounts → Account hinzufügen → Andere → **Kalenderabo hinzufügen** → Link einfügen.

**Outlook** (Web): Kalender → **Kalender hinzufügen** → **Aus dem Internet abonnieren** → Link einfügen.

**Android**: Über Google Kalender am PC abonnieren (siehe oben); der Kalender erscheint danach automatisch auf dem Smartphone.

## Hinweise

Die Abfuhrtermine stammen von den jeweiligen Landkreisen bzw. Entsorgungsbetrieben. TonnenPlan übernimmt keine Gewähr für Vollständigkeit und Richtigkeit; maßgeblich sind die Angaben des Landkreises.
