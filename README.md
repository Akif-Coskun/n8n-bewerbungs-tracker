# n8n-bewerbungs-tracker
KI-gestützter Bewerbungs-Tracker mit n8n, Gemini, Google Sheets und Telegram
# KI-Bewerbungs-Tracker mit n8n

Ein n8n-Workflow, der mein Bewerbungspostfach automatisch auswertet: Eine KI liest jede neue Mail, erkennt Zusagen, Einladungen zum Gespräch, Absagen und Eingangsbestätigungen und trägt den Stand jeder Bewerbung in ein Google Sheet ein. Bei Zusagen und Einladungen kommt sofort eine Nachricht aufs Handy.

Siehe Bild: `workflow.png`

## Das Problem

Bei vielen parallelen Bewerbungen geht schnell der Überblick verloren: Welche Firma hat geantwortet? Wo steht ein Gesprächstermin an? Bis wann muss ich auf ein Angebot reagieren? Wichtige Mails gehen zwischen Job-Alerts und Newslettern unter, und Fristen werden leicht übersehen.

## Die Lösung

Der Workflow prüft einmal pro Stunde das Bewerbungspostfach und verarbeitet jede neue Mail:

| Art der Mail | Was passiert |
|---|---|
| **Zusage** | Eintrag im Tracker + Telegram-Nachricht mit Position, Startdatum, Gehalt und Antwortfrist |
| **Einladung zum Gespräch** | Eintrag im Tracker + Telegram-Nachricht mit Termin und Ansprechpartner |
| **Absage** | Status im Tracker wird aktualisiert |
| **Eingangsbestätigung** | Status im Tracker wird aktualisiert |
| **Sonstiges** (Newsletter, Job-Alerts, Werbung) | wird ignoriert |

Im Google Sheet gibt es **pro Firma eine Zeile**, deren Status sich mit jeder neuen Mail aktualisiert, zum Beispiel von „Bewerbung eingegangen“ über „Einladung zum Gespräch“ zu „Zusage“.

Beispiel einer Telegram-Nachricht:

```
ZUSAGE von Müller!
Position: Junior Automation Engineer
Start: 01.12.2026
Gehalt: 48.000 €
Antwort bis: 2026-10-25
```

## Ablauf

```
Gmail → Konfiguration → Mail vorbereiten → KI-Klassifizierung → Antwort parsen → Bewerbungsmail?
                                                                                   ├─ nein → Ende
                                                                                   └─ ja → Google Sheet → Zusage/Einladung?
                                                                                                            ├─ ja → Telegram
                                                                                                            └─ nein → Ende
```

1. **Gmail Trigger:** prüft stündlich, ob neue Mails angekommen sind.
2. **Konfiguration:** alle Einstellungen (Name, Sheet-ID, Telegram-Chat-ID, KI-Modell) an einer zentralen Stelle.
3. **Mail vorbereiten:** reduziert die Mail auf Absender, Betreff, Datum und Text und kürzt lange Mails, um KI-Kosten zu sparen.
4. **KI-Klassifizierung:** Google Gemini ordnet die Mail einer Kategorie zu und extrahiert Firma, Position, Termin, Frist, Gehalt und Startdatum.
5. **Antwort parsen:** prüft die KI-Antwort, fängt fehlerhafte Antworten ab und baut den Text für Telegram.
6. **Bewerbungsmail?:** sortiert Newsletter und Job-Alerts aus.
7. **Im Tracker speichern:** legt pro Firma eine Zeile an oder aktualisiert sie („Append or Update“).
8. **Zusage oder Einladung?:** entscheidet, ob eine Benachrichtigung nötig ist.
9. **Telegram senden:** schickt die Nachricht aufs Handy.

## Besonderheiten

- **Freundliche Absagen werden erkannt.** Formulierungen wie „Wir waren beeindruckt, haben uns aber für eine andere Kandidatin entschieden“ werden als Absage eingeordnet, nicht als positive Nachricht.
- **Die KI wird nicht blind übernommen.** Ungültige Kategorien oder unlesbare Antworten werden abgefangen, statt den Workflow abstürzen zu lassen. Ist sich die KI unsicher, steht in der Nachricht „KI unsicher, bitte prüfen“.
- **Schutz vor manipulierten Mails.** Die KI wird angewiesen, den Mailinhalt nur als Daten zu behandeln und darin enthaltene Anweisungen zu ignorieren.
- **Fehlerüberwachung.** Ein separater Workflow meldet per Telegram, wenn der Tracker fehlschlägt, mit Workflow-Name, betroffenem Node, Fehlermeldung und Link zur Ausführung.
- **Anbieterunabhängige KI.** Die KI wird über eine OpenAI-kompatible Schnittstelle angesprochen. Ein Wechsel von Gemini zu OpenAI oder einem anderen Anbieter erfordert nur zwei Einstellungen in der Konfiguration.
- **Datenschutz.** Ein eigenes Postfach nur für Bewerbungen sorgt dafür, dass die KI keine privaten Mails zu sehen bekommt.

## Verwendete Tools

- **n8n:** Workflow-Automatisierung
- **Google Gemini API:** Klassifizierung und Datenextraktion
- **Gmail:** Eingang der Bewerbungsmails
- **Google Sheets:** Bewerbungsübersicht
- **Telegram Bot API:** Benachrichtigungen aufs Handy

## Dateien

| Datei | Inhalt |
|---|---|
| `bewerbungs-tracker.json` | Der Haupt-Workflow |
| `fehler-alarm.json` | Fehlerüberwachung per Telegram |
| `workflow.png` | Screenshot des Workflows |

## Einrichtung

1. **Workflows importieren:** In n8n „Import from File“ und beide JSON-Dateien laden.
2. **Gemini API-Key:** Auf aistudio.google.com einen Key erstellen. In n8n im Node „KI-Klassifizierung“ ein Credential vom Typ *Header Auth* anlegen: Name `Authorization`, Value `Bearer DEIN_KEY`.
3. **Google Sheet:** Neues Sheet mit einem Tab `Bewerbungen` anlegen. In Zeile 1 diese Spalten eintragen:
   `Firma | Position | Status | LetzteMail | Betreff | Absender | Ansprechpartner | Termin | Frist | Gehalt | Startdatum | Zusammenfassung | GmailLink`
4. **Telegram-Bot:** Bei @BotFather mit `/newbot` einen Bot erstellen, dem Bot eine Nachricht schicken und die Chat-ID über `https://api.telegram.org/bot<TOKEN>/getUpdates` auslesen. Den Token als Telegram-Credential in n8n hinterlegen.
5. **Konfiguration:** Im Node „Konfiguration“ die Platzhalter für Name, Sheet-ID und Telegram-Chat-ID ersetzen. Im Fehler-Alarm die Chat-ID im Telegram-Node eintragen.
6. **Gmail und Google Sheets** in den jeweiligen Nodes mit dem eigenen Google-Konto verbinden.
7. **Fehler-Alarm verknüpfen:** Im Tracker unter *Settings → Error Workflow* den Fehler-Alarm auswählen.
8. **Testen und aktivieren:** Eine Testmail ans Bewerbungspostfach schicken, „Execute workflow“ klicken und das Ergebnis prüfen. Danach den Workflow aktivieren.

## Mögliche Erweiterungen

- Gesprächstermine automatisch in Google Kalender eintragen
- Gmail-Labels je nach Kategorie, inklusive eines „Prüfen“-Labels für unsichere Einordnungen
- Erinnerung, wenn sich eine Firma längere Zeit nicht gemeldet hat
- Wöchentliche Zusammenfassung aller offenen Bewerbungen per Telegram
