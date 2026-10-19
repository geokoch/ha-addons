# Nährwert-Service

Ernährungstagebuch, Rezeptsammlung und Tagesbilanzen. Alle Nährwerte kommen aus dem
Bundeslebensmittelschlüssel (BLS 4.0) oder aus manuell erfassten Lebensmitteln mit Quelle.
Gerechnet wird ausschließlich im Dienst. Gesundheitsdaten bleiben in Home Assistant; der Dienst
schickt nichts in die Cloud.

## Einmalig: Zugriff auf das private Image

Das Image liegt privat auf GitHub (`ghcr.io/geokoch/naehrwert`):

1. Auf GitHub einen **Personal Access Token (classic)** mit nur dem Recht `read:packages` erzeugen.
2. In Home Assistant **Einstellungen → Apps → App-Store → ⋮ → Registries → Registry hinzufügen**:
   - Registry: `ghcr.io`
   - Benutzername: `geokoch`
   - Passwort: der Token

## Installation

1. App-Store → ⋮ → **Repositories** → `https://github.com/geokoch/ha-addons` hinzufügen.
2. **Nährwert-Service** installieren. Das Mosquitto-Add-on muss laufen.
3. In der Konfiguration einen **API-Token** setzen (mindestens 32 Zeichen).
4. Die BLS-Datei `BLS_4_0_2025_DE.zip` von <https://www.blsdb.de> („Download BLS-Daten“) in den
   Konfigurationsordner der App legen: `/addon_configs/<id>_naehrwert/` (Samba oder Studio Code
   Server). Sie wird beim nächsten Start einmalig importiert, das dauert etwa 10 s.
5. Starten. Im Protokoll erscheint „MQTT verbunden“. Unter Einstellungen → Geräte erscheint je
   Person ein Gerät „Ernährung &lt;Name&gt;“.

## Konfiguration

| Option | Bedeutung |
|---|---|
| `api_token` | Bearer-Token für REST-API und MCP (Pflicht, ≥ 32 Zeichen) |
| `log_level` | `trace` … `error`, Standard `info` |
| `heap_mb` | Maximaler Java-Heap, Standard 160 MB |
| `anthropic_api_key` | Schlüssel aus der [Anthropic-Console](https://console.anthropic.com/) für „Gericht fotografieren“; leer = Fotoerkennung aus |
| `photo_model` | Claude-Modell für die Fotoerkennung, Standard `claude-sonnet-5-5` |

Im Konfigurationsordner liegen nach dem ersten Start:

- `seed/persons.yaml`: Personen und Tagesziele. Neue Personen werden beim Start angelegt,
  bestehende nicht überschrieben (danach Ziele per API ändern).
- `seed/recipes.yaml`: Rezepte. Platzhalter werden ergänzt, sobald Zutaten eingetragen sind.

## Zugriff

- Port **8099** (änderbar unter „Netzwerk“): REST-API `/api/...`, Swagger-UI `/swagger-ui.html`,
  MCP `/mcp`. Jede Anfrage braucht `Authorization: Bearer <api_token>`; nur
  `/actuator/health` und die Swagger-Doku sind offen.
- Den Port **nicht** ungeschützt ins Internet freigeben.
- Die Seitenleiste „Nährwert“ steht allen angemeldeten HA-Benutzern offen, nicht nur Admins
  (ab 0.8.2) – nötig für Konten wie das des Wandpanels.

## Oberfläche (Seitenleiste „Nährwert“)

Über Ingress, ohne Token – die Anmeldung macht Home Assistant:

- **Heute:** kcal-Rest, nächste Mahlzeit, Mahlzeiten; frühere Tage korrigieren (Menge ändern,
  löschen). Die Makro-Karte zeigt zugeklappt fünf kleine Ringe (Eiweiß, Kohlenhydrate, Fett,
  Ballaststoffe, Salz) mit Prozent vom Maximum und Statussymbol; der Pfeil rechts klappt die Balken
  mit Restmengen auf (Balkenende = Maximum, getönte Zone ab Minimum). Der Zustand wird je Gerät
  gemerkt
- **Erfassen:** Suche über Lebensmittel und Rezepte, Favoriten, zuletzt gegessen; Rückgängig.
  Mit `anthropic_api_key` zusätzlich **Kamera-Knopf** in der Suche: Gericht fotografieren →
  Vorschlag prüfen → als Mahlzeit speichern (siehe „Gericht fotografieren“). Nicht gefunden →
  **„Neues Lebensmittel“** (siehe unten)
- **Plan:** abends den nächsten Tag zusammenstellen (gleiche Suche wie Erfassen) und vorab gegen die
  Ziele prüfen: kcal, Nährstoffe, Salz-Ampel. „Plan von heute übernehmen“ für ähnliche Tage. Auf
  **Heute** stehen die geplanten Posten unter „Noch geplant“; „Gegessen“ trägt sie mit der
  aktuellen Uhrzeit ein (Menge vorher änderbar). Geplantes zählt nie in Tageswerte, Woche oder
  Berichte, solange es nicht gegessen ist
- **Rezepte:** ansehen, anlegen, bearbeiten, als Favorit markieren, als PDF, mit Foto
- **Woche:** Heatmap der Tage, Mittelwerte, Tage im Ziel, Körperwerte
- **Berichte:** alle PDFs zum Öffnen oder Herunterladen
- **Assist-Knopf** (Sprechblase oben rechts und rund unten rechts auf Heute und Plan, im Wandpanel
  „Assist fragen“): öffnet den Assist-Dialog
  von Home Assistant mit dem Assistenten „Ernährung …“ – nur in der Seitenleiste/im Panel
- **Wandpanel:** Ansicht fürs Küchen-Tablet (große Zahlen, echtes Schwarz, Favoriten mit einem
  Tipp) – unter Berichte → „Wandpanel-Modus auf diesem Gerät“ einschalten

## Gericht fotografieren

Auf **Heute** und **Plan** sitzt ein runder Kamera-Knopf links neben „Erfassen“ bzw. „Planen“, in
**Erfassen** rechts in der Suche. Ein Tipp öffnet eine eigene Kamera-Vorschau mit Auslöser – auch
in der HA Companion App für Android, die bei normalen Datei-Feldern nur die Galerie anbietet. Beim
ersten Mal fragt das Handy nach der Kamera-Berechtigung. „Galerie“ unten links wählt ein vorhandenes
Foto; ohne Kamera-Zugriff (abgelehnt, PC ohne Kamera) bleibt nur dieser Weg. Das Foto wird im Browser auf etwa 1600 px verkleinert und an Claude geschickt
(Anthropic-API, `photo_model`). Claude erkennt die Komponenten und schätzt die Mengen; die
**Nährwerte kommen immer aus dem BLS bzw. den eigenen Rezepten**, nie vom Modell. Ein zweiter, kurzer
Aufruf ohne Bild (Claude Haiku 5.5) wählt aus den BLS-Suchtreffern den passenden Eintrag (die
BLS-Namen sind eigenwillig, z. B. „Broccoli“). Heißt ein Treffer genau wie die Komponente, entfällt
er. Das Protokoll nennt je Foto die Dauer beider Schritte („Fotoerkennung: Foto … ms, Zuordnung …
ms“).

- Komponenten mit sicherem Treffer sind vorausgewählt; unsichere Mengen tragen „geschätzt“.
- Pro Komponente: anderen Treffer wählen, „Anderes Lebensmittel suchen“, Menge ±10 g, abhaken.
- Speichern legt alle gewählten Komponenten als Einträge einer Mahlzeit an; „Rückgängig“ im Toast
  nimmt alle zurück.
- Das Foto wird nicht gespeichert – weder im Add-on noch in Home Assistant. Anthropic verarbeitet es
  nach seinen API-Bedingungen.
- Kosten: etwa zwei bis drei Cent je Foto mit Sonnet 5.5 ($2 / $10 je Mio. Tokens, Stand
  2026-10-06; Bild ≈ 2.500 Tokens plus Antwort und Auswahl). Dauer je nach Gericht 10–20 s.
- Lehnt die Sicherheitsprüfung von Anthropic ein Foto fälschlich ab, beantwortet automatisch ein
  Ersatzmodell die Anfrage (Server-Fallback, Beta).

## Neues Lebensmittel

Fehlt ein Lebensmittel im BLS (Markenprodukte, Fertiggerichte), legt man es in **Erfassen** an:
„Neues Lebensmittel anlegen“ unter den Favoriten bzw. „„…“ neu anlegen“ in den Suchergebnissen
(der Suchbegriff wird zum Namen).

- **Werte vom Etikett:** Felder in der Reihenfolge der EU-Nährwerttabelle – Energie (kJ oder kcal,
  das andere wird umgerechnet), Fett, davon gesättigte Fettsäuren, Kohlenhydrate, davon Zucker,
  Ballaststoffe, Eiweiß, Salz; Komma als Dezimalzeichen. Umschalter **je 100 g / je 100 ml**, bei ml
  optional die Dichte (Milch, Säfte ≈ 1,03). Ballaststoffe dürfen fehlen und zählen dann in Bilanzen
  als „Wert fehlt“.
- **Etikett fotografieren** (mit `anthropic_api_key`): Claude liest die Nährwerttabelle ab und füllt
  die Felder (hervorgehoben, bitte mit dem Etikett vergleichen). Umgerechnet wird auf dem Server:
  kJ → kcal, Natrium → Salz (× 2,5), Werte nur je Portion → je 100 g. Das Foto wird als
  Zutaten-Foto des neuen Lebensmittels gespeichert.
- **Prüfung beim Tippen:** „davon“-Werte nicht größer als der Oberwert und höchstens 100 g Nährstoffe
  je 100 g (blockieren das Speichern); Energie passt nicht zu den Makros oder auffällig viel Salz
  (nur Hinweis).
- **Doppelte vermeiden:** Beim Namen erscheinen ähnliche vorhandene Lebensmittel zum Antippen.
- **Gekauft bei:** optionaler Händler, mit Vorschlägen aus den bisher genutzten.
- **Quelle** (vorbelegt „Etikett, MM/JJJJ“), Favorit. „Anlegen und Menge wählen“ speichert und
  öffnet direkt die Mengenauswahl.

## Fotos zu Lebensmitteln und Rezepten

Eigene Lebensmittel haben zwei Fotos: **Packung** und **Zutaten & Nährwerte**; Rezepte ein
**Hauptfoto**. Aufnehmen, ansehen und ersetzen im Mengen-Sheet bzw. in der Rezeptansicht;
Vorschaubilder erscheinen in der Suche und der Rezeptliste. Der Browser verkleinert die Fotos
(Bild 1200 px, Vorschau 320 px). Sie liegen als JPEG unter `/data/fotos` der App – nicht in Home
Assistant und nicht bei Anthropic (nur das Etikett-Foto zum Ablesen geht einmalig an Claude).

## Personen und Benutzer

Beim ersten Öffnen über die Seitenleiste fragt die App „Wer bist du?“ und merkt sich die Antwort
je Home-Assistant-Benutzer – danach öffnet sie auf jedem Gerät direkt die eigenen Daten.

- **Person:** sieht nur ihre eigenen Daten (kein Umschalter).
- **Haushalts-Gerät** (Küchen-Tablet): alle Personen.
- **Admins:** in der Option **Admins der App** (`admin_users`) die HA-Benutzer eintragen,
  z. B. `Georg`. Sie sehen alle Personen; die eigene Person wählen sie unter *Berichte → Konto*.
- Falsch zugeordnet? *Berichte → Konto → Zuordnung ändern*.
- **Ziele ändern:** *Berichte → Konto → Meine Ziele*. Admins legen unter *Personen verwalten*
  Personen an (alle Ziele Pflicht, keine geschätzten Vorgaben) und löschen Personen ohne Daten.
- Die Rechte gelten auch in der API: Wer über die Seitenleiste angemeldet ist, bekommt fremde
  Daten nicht (HTTP 403). Der API-Token (Port 8099) hat weiterhin Vollzugriff.

## PDF-Reports

Am einfachsten über **Berichte** in der Seitenleiste „Nährwert“. „Herunterladen“ speichert das
PDF, falls die App es nicht anzeigt.

Direkt über Port 8099 mit Token:

- `/api/reports/day/<JJJJ-MM-TT>.pdf` – Tagesbilanz, danach jede Mahlzeit auf eigener Seite
- `/api/reports/recipe/<Rezept-ID>.pdf` – Rezept
- `/api/reports/collection.pdf` – Rezeptsammlung mit Nährstoffziel
- `/api/reports/week/<JJJJ-Www>.pdf` – Wochenbilanz, z. B. `2026-W42`

Bei mehreren Personen `?personId=<id>` anhängen.

Rezept-PDFs und die Rezeptsammlung werden nach dem Start und nach jeder Änderung an Rezepten,
Favoriten oder Nährstoffzielen im Hintergrund vorab erzeugt und unter `/data/pdf-cache`
gespeichert (nicht unter `/config/www`, das wäre ohne Anmeldung erreichbar). Solange sich nichts
geändert hat, kommt das PDF sofort aus dem Cache. Tages- und Wochenberichte entstehen beim
Aufruf. Der Cache ist vom HA-Backup ausgenommen und füllt sich nach einer Wiederherstellung
von selbst.

## Chat mit Assist (MCP)

Die App bietet MCP-Tools (Lebensmittel suchen, Gegessenes eintragen, Tagesbilanz …) für einen
Konversations-Agenten in Home Assistant:

1. Integration **Anthropic** (o. ä.) mit eigenem API-Schlüssel einrichten, Modell z. B. Claude Haiku.
2. Integration **Model Context Protocol** hinzufügen, URL:
   `http://<hostname>:8081/mcp` – den Hostnamen zeigt die App-Info (z. B. `dd4fef93-naehrwert`).
   Dieser interne Port braucht keinen Token und ist von außen nicht erreichbar.
3. Im Sprachassistenten den Agenten auswählen und die MCP-Tools für ihn freigeben.

Von außen (Port 8099) ist MCP unter `/mcp` nur mit `Authorization: Bearer <api_token>` erreichbar.

Für den Essensplan gibt es eigene Tools: Plan ansehen, Posten planen (Standard morgen), geplanten
Posten als gegessen buchen und aus dem Plan entfernen, z. B. „Plan mir für morgen 300 g
Linsensuppe zum Mittagessen“.

Der Sensor **Geplant** (`sensor.<person>_geplant`) zeigt die nächste noch geplante Mahlzeit von
heute, z. B. „Mittagessen: Linsensuppe 300 g“ – für Wandpanel oder Erinnerungen.

## Körperwerte (Withings)

Gewicht, Blutdruck und Puls werden nicht in der App erfasst, sondern von der Waage in Home Assistant
übernommen. Dafür eine Automation anlegen, die bei neuer Messung auf
`naehrwert/<person>/cmd` den Befehl `{"action":"log_body", "measuredAt": …, "weightKg": …,
"bpSys": …, "bpDia": …, "pulse": …}` sendet. Die Werte fließen in die Wochenbilanz ein
(Gewichtstrend als Wochenmittel gegen die Vorwoche).

## Sicherung

Datenbank und Fotos (`/data/fotos`) liegen unter `/data` der App und sind in den
Home-Assistant-Backups enthalten. Für ein
konsistentes Backup hält Home Assistant die App kurz an (`backup: cold`). Zusätzlich erzeugt
`POST /api/admin/backup` jederzeit eine Kopie per SQLite `VACUUM INTO` (die neuesten 14 werden
behalten) und kopiert neue Fotos nach `backup/fotos` (ersetzte bleiben dort erhalten). Diese
Kopien sind vom HA-Backup ausgenommen.

## Daten

Nährwerte: Max Rubner-Institut (2025): Bundeslebensmittelschlüssel (BLS), Version 4.0, CC BY 4.0.
