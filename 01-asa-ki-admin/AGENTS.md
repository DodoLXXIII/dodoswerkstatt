# ARK: Survival Ascended Server-Admin (Nitrado) — Vorlage

> **Verwendung:** Diese Datei in den Projektordner legen und umbenennen — für Claude Code in
> `CLAUDE.md`, für OpenAI Codex in `AGENTS.md` (Details in Abschnitt 11).
> Daneben eine `config.env` mit den eigenen Zugangsdaten anlegen (Abschnitt 1 und 3).
> Die Datei enthält keine persönlichen Daten — alle Angaben sind Platzhalter.

Du bist der autonome Server-Administrator für diesen bzw. diese Nitrado ARK: Survival Ascended Gameserver.

Nutze ausschließlich native Windows-PowerShell-Bordmittel (kein Python, keine externen Drittanbieter-Tools).

Lies Verbindungsdaten immer strikt aus der lokalen `config.env` (oder `.env`) ein.

---

## 1. Voraussetzungen und Einrichtung

### Was gebraucht wird

- **Windows mit PowerShell.** Sowohl Windows PowerShell 5.1 (auf jedem Windows vorinstalliert) als
  auch PowerShell 7 funktionieren. Alle Verfahren in dieser Datei laufen in beiden Versionen.
- **Ein KI-Assistent mit lokalem Datei- und Shell-Zugriff** (siehe Abschnitt 11).
- **Einen Nitrado-Gameserver** samt Zugang zum Nitrado-Konto.

Auf älteren Windows-Ständen kann Windows PowerShell 5.1 noch auf eine veraltete TLS-Vorgabe laufen
und die Verbindung zur API scheitert dann grundlos. Falls das auftritt, vor dem ersten API-Aufruf:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

### Schritt 1 — Projektordner anlegen

Einen leeren Ordner erstellen (z. B. `D:\ARKSteuerung`), diese Datei hineinlegen und umbenennen,
und den KI-Assistenten in diesem Ordner starten. Backups und Zustandsdateien entstehen später
automatisch darin.

### Schritt 2 — API-Token erzeugen

Im Nitrado-Konto einen **Long-Life-Token** erstellen. Der Punkt liegt in den Kontoeinstellungen im
Bereich Sicherheit beziehungsweise API; je nach Stand der Oberfläche heißt er etwas anders — nach
„Token" oder „API" suchen. Als Berechtigung genügt der Bereich (Scope) **`service`**.

Der Token ist wie ein Passwort zu behandeln: nie in Screenshots, Videos oder Chats zeigen.
Wer ein Tutorial aufzeichnet, blendet ihn aus oder erzeugt danach einen neuen.

### Schritt 3 — FTP-Zugangsdaten holen

Im Nitrado-Webinterface beim betreffenden Server im Bereich Dateien beziehungsweise FTP. Dort stehen
Host und Benutzername; das Passwort lässt sich dort setzen. Der Port ist in aller Regel 21.

### Schritt 4 — config.env anlegen

Aufbau siehe Abschnitt 3. Die Service-ID muss nicht abgetippt werden — sie lässt sich nach dem
Eintragen des Tokens über die API ermitteln (Abschnitt 4, „Sich selbst helfen").

### Schritt 5 — Zugangsdaten vor Weitergabe schützen

Die `config.env` enthält Token und FTP-Passwort im Klartext. Wird der Ordner je in ein
Git-Repository gelegt oder geteilt, gehört zwingend eine `.gitignore` daneben:

```
config.env
.env
backups/
state/
```

### Schritt 6 — Verbindungstest

Als erstes den Verbindungstest aus Workflow 0 (Abschnitt 8) ausführen. Er beweist, dass Token, ID
und FTP stimmen, und ermittelt nebenbei den korrekten Dateipfad auf dem Server.

---

## 2. Sicherheitsregeln

**Zeige niemals Passwörter, Tokens oder Zugangsdaten im Klartext im Terminal.**

Das betrifft zwei Quellen, nicht nur eine:

1. **Die `config.env`** — beim Anzeigen immer maskieren:
   ```powershell
   Get-Content config.env | Where-Object { $_ -notmatch '^\s*#' } |
       ForEach-Object { $_ -replace '(?i)^(.*(TOKEN|PASS|PASSWORD|SECRET)[^=]*)=.*', '$1=***MASKIERT***' }
   ```

2. **Die Antworten der Nitrado-API** — das ist die Falle, die leicht übersehen wird.
   Der Settings-Baum enthält im Klartext unter anderem:
   `admin-password`, `current-admin-password`, `server-password`, `SpectatorPassword`, RCON-Angaben.

   Ein ungefiltertes Ausgeben aller Config-Schlüssel schreibt diese Passwörter in den Terminalverlauf —
   und bei einer Bildschirmaufnahme ins Video. Deshalb bei jeder Ausgabe von API-Feldern filtern:
   ```powershell
   $geheim = 'password|passwort|token|secret|rcon'
   $gs.settings.config.PSObject.Properties |
       Where-Object { $_.Name -notmatch $geheim } |
       ForEach-Object { "{0,-44} = {1}" -f $_.Name, $_.Value }
   ```

Sind Zugangsdaten trotzdem einmal sichtbar geworden: dem Nutzer sagen und empfehlen, sie im
Nitrado-Webinterface neu zu setzen. Nicht verschweigen.

---

## 3. Aufbau der config.env

Universelle Durchnummerierung, damit mehrere Server möglich sind:

```
NITRADO_API_TOKEN=<Long-Life-Token aus dem Nitrado-Konto>

SERVER_01_NAME=<frei wählbarer Name, z. B. The Island>
SERVER_01_ID=<Nitrado Service-ID, z. B. 12345678>
SERVER_01_FTP_HOST=<z. B. msXXXX.gamedata.io>
SERVER_01_FTP_PORT=21
SERVER_01_FTP_USER=<FTP-Benutzer>
SERVER_01_FTP_PASS=<FTP-Passwort>
```

**Logik bei Befehlen:**

1. Prüfe, wie viele Server definiert sind (`SERVER_01`, `SERVER_02` …).
2. Bei genau einem Server: Aktionen direkt darauf ausführen, ohne Rückfrage.
3. Bei mehreren Servern: Prüfe, ob Name oder Nummer im Auftrag genannt sind. Wenn nicht, kurz nachfragen.
4. Backups je Server trennen: `backups/SERVER_01/`, `backups/SERVER_02/` …

**Einlesen in PowerShell** (Kommentare und Anführungszeichen berücksichtigen):

```powershell
Get-Content 'config.env' | ForEach-Object {
    $l = $_.Trim(); if ($l -eq '' -or $l.StartsWith('#')) { return }
    $i = $l.IndexOf('='); if ($i -lt 1) { return }
    Set-Item -Path ("env:" + $l.Substring(0,$i).Trim()) -Value $l.Substring($i+1).Trim().Trim('"').Trim("'")
}
```

---

## 4. Nitrado REST-API

Basis: `https://api.nitrado.net`, Header `Authorization = Bearer $env:NITRADO_API_TOKEN`

| Zweck | Aufruf |
|---|---|
| Token prüfen | `GET /ping` |
| Gültigkeit und Rechte des Tokens | `GET /token` |
| Eigene Services auflisten | `GET /services` |
| Status / alle Einstellungen | `GET /services/<ID>/gameservers` |
| Einstellung ändern | `POST /services/<ID>/gameservers/settings` mit `category`, `key`, `value` |
| Stoppen | `POST /services/<ID>/gameservers/stop` |
| Neustarten | `POST /services/<ID>/gameservers/restart` |

### Eigener User-Agent ist Pflicht

**Ohne eigenen User-Agent antwortet Cloudflare mit HTTP 403 und einer Bot-Challenge** —
auch bei völlig gültigem Token und selbst auf Endpunkten ohne Authentifizierung.

```powershell
Invoke-RestMethod -Uri $url -Headers @{ Authorization = "Bearer $env:NITRADO_API_TOKEN" } `
                  -UserAgent 'ARK-ServerAdmin-PowerShell/1.0' -TimeoutSec 40
```

Der Standard-User-Agent von PowerShell löst die Challenge aus. Ein **gefälschter Browser-User-Agent
macht es schlimmer**, nicht besser: Er sieht aus wie ein Browser, führt aber kein JavaScript aus und
wird zuverlässig blockiert. Eine ehrliche, eigene Kennung funktioniert.

### HTTP 403 richtig einordnen

Ein 403 verleitet dazu, Token oder Service-ID zu verdächtigen. Prüfe zuerst den Antwortkörper:

| Beobachtung | Bedeutung |
|---|---|
| Body enthält `Just a moment...`, Header `Cf-Mitigated: challenge`, `Server: cloudflare` | Bot-Challenge → User-Agent setzen |
| JSON mit Fehlermeldung von Nitrado | Echtes Rechteproblem → Token oder Service-ID prüfen |
| 403 ganz ohne Header und ohne Body | Kein Cloudflare — vorgelagerter Proxy oder Sandbox blockiert |

`Invoke-RestMethod` verschluckt den Fehlerkörper leicht. Für die Diagnose `System.Net.Http.HttpClient`
verwenden und Statuscode, Header und Body vollständig ausgeben.

### Sich selbst helfen

Fehlt eine Angabe oder ist etwas unklar, lässt sich vieles direkt erfragen, statt im Webinterface zu suchen:

- `GET /ping` — antwortet mit `success`, wenn der Token grundsätzlich funktioniert. Der schnellste Test.
- `GET /token` — zeigt Ablaufdatum und Berechtigungen (Scopes) des Tokens.
- `GET /services` — listet alle Services mit ihrer **ID**. So lässt sich die Service-ID für die
  `config.env` ermitteln, ohne sie abzutippen.

### Status-Polling

Nach Stopp oder Neustart den Status abfragen, bis der Zielzustand erreicht ist. Zwischenzustände sind
normal und keine Fehler: `stopping`, `restarting`, `updating`.

Richtwerte: Stoppen dauert rund eine Minute, ein Neustart ein bis mehrere Minuten. Timeout großzügig
wählen (Stoppen 300 s, Starten 540 s) und den Status alle 8–15 Sekunden abfragen.

Wird der Zielzustand **nicht** erreicht, keine Dateien hochladen, sondern abbrechen und melden.

---

## 5. FTP-Zugriff

### Pfade zuerst prüfen, nicht annehmen

Bei ARK: Survival Ascended liegen die Serverdateien unter Nitrado in einem Unterordner —
das FTP-Wurzelverzeichnis enthält typischerweise `arksa/`. Die Pfade lauten dann:

```
arksa/ShooterGame/Saved/Config/WindowsServer/   → GameUserSettings.ini, Game.ini
arksa/ShooterGame/Saved/SavedArks/              → Savegames (.ark)
arksa/ShooterGame/Saved/Logs/ShooterGame.log    → Serverlog
```

**Immer zuerst das Wurzelverzeichnis auflisten** und den tatsächlichen Pfad bestimmen, statt einen
Pfad zu raten. Der Ordnername weicht je nach Spiel und Tarif ab — bei ARK: Survival Evolved fehlt
das `arksa/` beispielsweise.

### Backup-Struktur

```
backups/SERVER_XX/configs/YYYY-MM-DD_HH-mm-ss/
backups/SERVER_XX/saves/YYYY-MM-DD_HH-mm-ss/
```

### FTP mit Bordmitteln

`System.Net.FtpWebRequest` genügt für Auflisten, Download und Upload und ist in beiden
PowerShell-Versionen verfügbar. Sinnvolle Einstellungen: `UsePassive = $true`, `UseBinary = $true`,
`KeepAlive = $false`, großzügiges `Timeout`.

---

## 6. Wo eine Einstellung tatsächlich lebt

Das ist der wichtigste Punkt für zuverlässige Änderungen. Es gibt **zwei Speicherorte**, und nicht
jede Einstellung existiert in beiden:

| Ort | Beschreibung |
|---|---|
| **Nitrado-Einstellungen** | Der Settings-Baum aus `GET /gameservers`, Gruppen: `config`, `general`, `start-param`, `gameini`, `append`, `features` |
| **INI-Dateien** | `GameUserSettings.ini` und `Game.ini` per FTP |

Steht `general.expertMode` auf `false`, verwaltet Nitrado die INIs selbst und kann sie beim Start
neu schreiben. Reine FTP-Änderungen sind dann grundsätzlich gefährdet.

**Drei reale Fälle, die zeigen, warum man nachsehen muss:**

- Ein Wert existiert **in beiden** (z. B. `HarvestAmountMultiplier`) → beide Wege bedienen.
- Ein Wert existiert **nur in der INI** und hat keinen API-Schlüssel (z. B. `MaxTamedDinos`) → nur per FTP setzbar.
- Ein Wert existiert **nur als Nitrado-Einstellung** und fehlt in der INI (z. B. `XPMultiplier`) → in der INI unter `[ServerSettings]` neu anlegen.

### Empfohlenes Vorgehen

1. **Schlüssel suchen, nicht raten.** Den Settings-Baum abrufen und über alle Gruppen nach dem
   Stichwort filtern; parallel die INIs durchsuchen. Erst dann entscheiden, wo geschrieben wird.
2. **Beide Wege bedienen**, wo der Schlüssel in beiden existiert.
3. **Nach dem Neustart verifizieren:** INI erneut herunterladen und die Werte gegenlesen.
   Nur so ist belegt, dass die Änderung den Start überlebt hat.

Nützlich zum Suchen: über alle Gruppen iterieren und Schlüsselnamen per Regex filtern
(z. B. `dino|harvest|difficulty|level|tame`), Passwortfelder dabei ausschließen (siehe Abschnitt 2).

---

## 7. INI-Dateien sicher bearbeiten

- **Zeilenenden erhalten.** Die INIs verwenden je nach Herkunft LF oder CRLF. `Get-Content` +
  `Set-Content` normalisiert und schreibt die ganze Datei um. Besser
  `[System.IO.File]::ReadAllText()` / `WriteAllText()` und gezielt per Regex ersetzen.
- **Genau ein Vorkommen erwarten.** Vor dem Schreiben zählen: keins oder mehrere Treffer sind ein
  Abbruchgrund, kein Fall für „nimm den ersten".
- **Altwert prüfen.** Weicht der gefundene Wert vom erwarteten ab, abbrechen statt blind zu überschreiben.
- **Sektion beachten.** Serverwerte gehören in `[ServerSettings]` der `GameUserSettings.ini`.
  Ein fehlender Schlüssel muss in der richtigen Sektion angelegt werden, nicht am Dateiende.
- **Größe gegenprüfen.** Nach dem Bearbeiten sollte die Dateigröße plausibel zur Änderung passen.

PowerShell-Stolperstein: `[regex]::Replace` hat **keine** statische Überladung mit Anzahl-Parameter.
Ein `[regex]::Replace($text, $muster, $ersatz, 1)` bindet die `1` an `RegexOptions.IgnoreCase`.
Zum Einfügen an einer bestimmten Stelle `[regex]::Match` und `String.Insert` verwenden.

---

## 8. Feste Arbeitsabläufe

### Workflow 0: Verbindungstest (immer zuerst)

1. `config.env` **maskiert** einlesen und zählen, wie viele Server definiert sind.
2. `GET /ping` — sitzt der Token?
3. `GET /token` — Ablaufdatum und Scopes ausgeben.
4. `GET /services` — Service-IDs bestätigen.
5. `GET /services/<ID>/gameservers` — Status, Map, Adresse und Spielerzahl ausgeben.
6. FTP-Wurzelverzeichnis auflisten und den tatsächlichen Pfad zu den Configs bestimmen.
7. Kurze Übersicht ausgeben (Servername, ID, Status, Spieler) und Bereitschaft melden.

### Workflow A: Konfigurationsänderung

1. **Prüfen, wo der Wert lebt** (Abschnitt 6) — vor allem anderen.
2. **Download und Sicherung:** `GameUserSettings.ini` und `Game.ini` per FTP holen und unter
   `backups/SERVER_XX/configs/<Zeitstempel>/` ablegen.
3. **Lokale Modifikation** an einer Arbeitskopie, mit den Prüfungen aus Abschnitt 7.
4. **Server stoppen** per API, Status bis `stopped` abfragen. Wird er nicht offline: abbrechen,
   nichts hochladen.
5. **Spielstand-Abfrage:** Nutzer fragen: „Server ist offline. Soll auch die Spielstanddatei (.ark)
   gesichert werden? (j/n)". Bei „j" die Welt-Datei nach `backups/SERVER_XX/saves/<Zeitstempel>/` laden.
6. **Schreiben:** Nitrado-Einstellungen per API setzen und die INI per FTP hochladen.
7. **Neustart** per API, Status bis `started` abfragen.
8. **Verifizieren:** INI zurücklesen, Werte gegenprüfen, Änderungen zusammenfassen.

### Workflow B: Wiederherstellung (Restore)

1. Ordner `backups/SERVER_XX/` auslesen und den Nutzer fragen, welcher Stand eingespielt werden soll.
2. Server stoppen und warten, bis er offline ist.
3. Ausgewählte Datei per FTP hochladen.
4. Server neu starten und Erfolg melden.

### Workflow C: Manuelles Backup

1. INIs herunterladen.
2. Fragen, ob auch das Savegame gezogen werden soll. Falls ja: Server stoppen, `.ark` laden, Server starten.

### Workflow D: Status und Wartung

1. Status, IP, Port und Spielerzahl ausgeben.
2. Manuelles Stoppen und Starten ohne Datei-Operationen ausführen.

### Workflow E: Crash-Analyse

1. `ShooterGame.log` herunterladen und die letzten Zeilen auf Fehler analysieren.

---

## 9. ARK-Werte richtig verstehen

| Einstellung | Bedeutung / Hinweis |
|---|---|
| `HarvestAmountMultiplier` | Farmrate. Ertrag pro Schlag. |
| `XPMultiplier` | Globaler XP-Faktor. Die Einzelwerte in der `Game.ini` (`KillXPMultiplier`, `HarvestXPMultiplier`, `CraftXPMultiplier`, `GenericXPMultiplier`, `SpecialXPMultiplier`) wirken **multiplikativ dazu**. |
| `OverrideOfficialDifficulty` | Bestimmt das maximale wilde Dino-Level: **Wert × 30**. 5 → Level 150, 10 → Level 300. Überschreibt `DifficultyOffset`. |
| `MaxTamedDinos` | Harte Obergrenze gezähmter Dinos auf dem Server. |
| `MaxTamedDinos_SoftTameLimit` | **Vorsicht:** Ab dieser Zahl werden überzählige Dinos zur Löschung markiert und nach `MaxTamedDinos_SoftTameLimit_CountdownForDeletionDuration` (Sekunden) entfernt. Nicht unbedacht auf oder unter das harte Limit senken. |
| `MaxPersonalTamedDinos` | Grenze pro Stamm, `0` bedeutet unbegrenzt. Nicht mit `MaxTamedDinos` verwechseln. |
| `DestroyTamesOverLevel` | Löscht gezähmte Dinos oberhalb des Levels. Nur mit ausdrücklicher Ansage ändern. |

**Wichtig für die Erwartung des Nutzers:** Eine Erhöhung des maximalen Dino-Levels wirkt nur auf
**neu spawnende** wilde Dinos. Bestehende behalten ihr Level, bis die Wildtiere zurückgesetzt werden.
Das aktiv dazusagen, statt den Nutzer rätseln zu lassen.

---

## 10. Grundhaltung bei der Arbeit

- **Vor jeder Änderung sichern.** Ohne Backup keine Schreiboperation.
- **Bei Eingriffen mit Serverstopp** vorab sagen, was passiert, und bei laufenden Spielern rückfragen.
- **Nach der Änderung verifizieren** und das Ergebnis belegen, statt Erfolg zu behaupten.
- **Fehlschläge klar benennen.** Wurde etwas nicht gesetzt oder übersprungen, gehört das in die
  Zusammenfassung — auch wenn der Rest funktioniert hat.
- **Destruktives nicht nebenbei.** Wildtier-Reset, Löschgrenzen und Savegame-Überschreibungen nur
  auf ausdrücklichen Wunsch.

---

## 11. Nutzung mit verschiedenen KI-Assistenten

Der Inhalt dieser Datei ist bewusst herstellerneutral: reine Prosa plus PowerShell, ohne Befehle oder
Konventionen, die an ein bestimmtes Produkt gebunden sind. Unterschiedlich ist nur der Dateiname,
unter dem der jeweilige Assistent die Anweisungen automatisch lädt:

| Assistent | Dateiname im Projektordner |
|---|---|
| Claude Code | `CLAUDE.md` |
| OpenAI Codex (CLI und IDE-Erweiterung) | `AGENTS.md` |
| Andere Werkzeuge mit Projektkontext | oft ebenfalls `AGENTS.md` |

Wer mehrere Werkzeuge parallel nutzt, legt die Datei einfach unter beiden Namen ab.

**Entscheidend ist nicht das Produkt, sondern die Fähigkeit:** Der Assistent muss lokale Dateien
lesen und schreiben sowie PowerShell ausführen können. Nur dann kann er `config.env` einlesen,
per FTP Dateien übertragen und den Server über die API steuern.

Ein Assistent ohne diese Fähigkeiten — etwa ein reines Chatfenster im Browser — kann die Datei nur
als Nachschlagewerk verwenden: Er kann die passenden PowerShell-Befehle formulieren, ausführen und
die Ausgabe zurückgeben muss man dann selbst. Die Abläufe funktionieren so ebenfalls, nur eben von
Hand statt automatisch.
