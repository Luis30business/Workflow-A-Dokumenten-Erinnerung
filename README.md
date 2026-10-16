# Dokumenten-Erinnerungen für Hypothekenvermittler (n8n)

Automatisierte Nachfass-Mails für fehlende Antragsunterlagen. Der Workflow geht
wöchentlich eine Fallliste durch, erkennt pro Fall, welche Dokumente fehlen,
schickt dem zuständigen Vermittler eine Erinnerung und protokolliert den Versand
zurück in die Liste.

> Nachbau eines Workflows, den ich für ein Erstgespräch mit einem
> Finanzdienstleister gebaut habe — hier mit frei erfundenen Testdaten und ohne
> jeden Kundenbezug.

---

## Problem

Ein Hypothekenvermittler sammelt für jeden Kreditantrag mehrere Pflichtdokumente
ein: Ausweis, Gehaltsnachweis, Kontoauszug, Steuerbescheid, Eigenkapitalnachweis.
Der Stand wird in einer Tabelle gepflegt — pro Fall eine Zeile, pro Dokument eine
Spalte mit „erhalten" oder „fehlt".

Das Nachfassen passiert manuell: jemand geht die Liste durch, schaut, bei welchem
Fall was fehlt, und schreibt eine Mail. Das kostet pro Woche Zeit, wird bei
Arbeitsspitzen übersprungen, und einzelne Fälle liegen wochenlang unbearbeitet,
weil niemand merkt, dass ein Dokument fehlt. Gleichzeitig darf nicht zu oft
erinnert werden — wer jede Woche dieselbe Mail bekommt, liest sie irgendwann nicht
mehr.

## Lösung

```
Schedule (Mo)
   └─ Zeilen aus Google Sheet lesen
        └─ Code-Node: welche Dokumente fehlen?
             ├─ nichts fehlt ─────────────────────► Ende (keine Mail)
             └─ etwas fehlt
                  ├─ noch nie erinnert ──────────► Reminder-Mail
                  └─ schon erinnert
                       ├─ > 13 Tage her ────────► Reminder-Mail
                       └─ kürzer ───────────────► Ende (keine Mail)

   Reminder-Mail
        └─ Versand erfolgreich?
             ├─ ja  → Datum + Status „Gesendet" ins Sheet
             └─ nein → Status „Fehlgeschlagen" ins Sheet (Datum bleibt unberührt)
```

1. **Zeitgesteuert**, jeden Montagmorgen. Jede Tabellenzeile wird ein eigener
   Datensatz und einzeln bewertet.
2. **Ein Code-Node ermittelt die fehlenden Dokumente** und hängt sie als Liste
   plus Anzahl an den Datensatz. Er entscheidet nichts — er rechnet nur.
3. **Drei Bedingungen entscheiden**, ob eine Mail rausgeht: fehlt überhaupt was,
   wurde schon erinnert, und wenn ja, ist es lange genug her.
4. **Die Mail nennt die fehlenden Dokumente konkret**, aufgezählt, mit Kunden- und
   Auftragsbezug.
5. **Der Versand wird zurückgeschrieben** — Datum und Status in die Zeile des
   Falls, gefunden über die eindeutige `Fall_ID`.


## Entscheidungen

Der interessantere Teil als die Nodes. Jede dieser Entscheidungen hätte man auch
anders treffen können.

**Warum ist eine Tabellenzeile ein Fall und nicht ein Vermittler?**
Weil eine Mail einem Fall entspricht. Die Anforderung war eine separate Mail pro
Kunde und Auftrag, nicht eine Sammelmail pro Vermittler. Damit gilt: ein
Datensatz rein, eine Entscheidung, eine Mail raus — ohne Gruppieren oder
Zusammenfassen. Bei einer Sammelmail pro Vermittler wäre zusätzlich ein
Aggregations-Schritt nötig, und die Zeilenstruktur der Tabelle müsste anders
aussehen. Die Datenstruktur folgt der Mail-Regel, nicht umgekehrt.

**Warum prüft der Code auf „nicht erhalten" statt auf „fehlt"?**
Weil die beiden Varianten unterschiedliche Fehlerfälle haben. Steht in einer
Zelle etwas Unerwartetes — leer, Tippfehler, anderer Status — dann gilt das
Dokument bei „nicht erhalten" als fehlend und es geht eine Erinnerung zu viel
raus. Bei „fehlt" würde die Zeile als vollständig durchgehen und ein fehlendes
Dokument fällt still unter den Tisch; der Antrag verzögert sich und niemand
merkt es. Beide Varianten können falsch liegen, aber eine davon ist harmlos und
die andere teuer.

**Warum keine Bestätigungsmail, wenn alles vollständig ist?**
Weil sie jede Woche erneut käme. „Für diesen Fall brauchen wir nichts mehr" ist
nach dem ersten Mal keine Information, sondern Rauschen — und Rauschen führt
dazu, dass auch die echten Erinnerungen überlesen werden. Vollständige Fälle
werden still übersprungen.

**Warum 13 Tage und nicht 14?**
Die Regel lautet „maximal alle zwei Wochen". Der Workflow läuft aber nur montags.
Liegt das letzte Versanddatum auf einem Dienstag, sind am übernächsten Montag
erst 13 Tage vergangen — mit einer 14-Tage-Schwelle fällt der Fall durch und
wartet eine weitere Woche. Aus „alle zwei Wochen" würden faktisch drei. Die
Schwelle muss unter dem Lauf-Intervall liegen, nicht genau darauf.

**Warum hat der Versandstatus eine eigene Spalte?**
Ein erster Entwurf schrieb „Senden fehlgeschlagen" in die Datumsspalte. Damit war
die Spalte kein Datumsfeld mehr, der Datumsvergleich konnte nicht mehr rechnen,
und betroffene Fälle wären nie wieder erinnert worden — ausgerechnet die, bei
denen die Mail nicht angekommen ist. Ein Feld, eine Bedeutung: `Erinnerung_gesendet`
enthält ausschließlich ein Datum, der Status steht daneben.

**Warum wird der Gmail-Versand nachträglich geprüft?**
„Der Node ist durchgelaufen" heißt nicht „die Mail ist raus". Die Antwort der
Gmail-API enthält bei Erfolg das Label `SENT` — darauf wird geprüft, und erst
dann wird ein Versanddatum geschrieben. Andernfalls stünde in der Tabelle ein
Datum für eine Mail, die nie ankam, und der Fall wäre für zwei Wochen stillgelegt.

---

## Fehlerfälle

| Fall | Verhalten |
|---|---|
| Leere oder unerwartete Statuszelle | Dokument gilt als fehlend (siehe Entscheidungen) |
| Ungültige Mailadresse | Workflow läuft weiter, Status „Fehlgeschlagen", Datum bleibt unberührt → nächster Lauf versucht es erneut |
| Dauerhaft fehlgeschlagener Versand | Zwei-Wochen-Regel greift weiter, weil das alte Datum stehen bleibt |
| Dokument nachgereicht, Erinnerung war raus | Fall fällt aus der Logik, weil nichts mehr fehlt — keine weitere Mail |
| Sheet nicht erreichbar | Lauf schlägt fehl, es wird nichts verschickt und nichts geschrieben |

---

## Verwendete Technik

- **n8n** (self-hosted auf eigenem Server, Docker)
- **Google Sheets** als Datenquelle und Protokoll, via OAuth
- **Gmail API** für den Versand
- **JavaScript** im Code-Node für die Dokumentenprüfung
- **Expressions** für Mailtext, Datumsvergleich und Datumsformatierung

---

## Nachbauen

Die Spreadsheet-ID in der Workflow-Datei ist bewusst ein Platzhalter
(`YOUR_SPREADSHEET_ID`) — jeder Nachbau zeigt auf sein eigenes Sheet.

1. `workflow-a-testdaten.csv` in Google Sheets importieren
   (*Datei → Importieren*, nicht per Copy/Paste).
   Die Spalte `Sende Status` ergänzen, falls sie fehlt.
2. In n8n Credentials für **Google Sheets** und **Gmail** anlegen.
3. `workflow-a-dokumenten-erinnerungen.json` importieren.
4. In den drei Google-Sheets-Nodes das eigene Spreadsheet und Tabellenblatt
   auswählen.
5. Die Mailadressen im Sheet auf eine eigene Adresse ändern. Mit Gmail lässt sich
   dafür ein Plus-Suffix nutzen (`name+broker1@gmail.com`) — alles landet im
   eigenen Postfach, der Empfänger bleibt aber unterscheidbar.
6. Workflow **deaktiviert** lassen und über *Test workflow* starten. So feuert der
   Zeitplan nicht versehentlich.

### Testfälle

Die Testdaten decken bewusst alle Zweige ab:

| Fall | Situation | Erwartung |
|---|---|---|
| F-1001, F-1005 | alle Dokumente da | keine Mail |
| F-1003, F-1007 | genau eines fehlt | Mail, Aufzählung im Singular |
| F-1002, F-1004 | mehrere fehlen | Mail mit allen fehlenden Dokumenten |
| F-1006 | alle fünf fehlen | Mail mit fünf Positionen |
| F-1003 + F-1004 | derselbe Vermittler, zwei Fälle | zwei getrennte Mails |
| F-1008 | Erinnerung vor wenigen Tagen raus | keine Mail (Zwei-Wochen-Regel) |

Für den Zwei-Wochen-Zweig in einer Zeile ein Datum von vor mehr als 13 Tagen
eintragen — sonst wird dieser Pfad von den Standard-Testdaten nie erreicht.

---

## Dateien

```
README.md
workflow-a-dokumenten-erinnerungen.json   gesäuberter n8n-Export
workflow-a-testdaten.csv                  erfundene Testdaten
docs/workflow.png                         Screenshot des Workflows
```
