# Marstek Venus — Modbus-Register

Empirisch verifiziert an drei Geräten (2× Gen 3 „Venus E v3", 1× Gen 2 „Venus E v12")
im August 2026. Die für diese Steuerung entscheidenden Register verhalten sich auf
**beiden Generationen identisch**.

Alle Angaben: Holding Register, Slave/Unit ID 1.

---

## Die beiden zentralen Register

| Adresse | Bedeutung | Einheit | Gen 2 | Gen 3 |
|---|---|---|---|---|
| **44002** | Max Charge Power | W | ✅ | ✅ |
| **44003** | Max Discharge Power | W | ✅ | ✅ |

### Warum diese beiden entscheidend sind

Der naheliegende Weg, einen Marstek zu steuern, führt über die RS485-Fremdsteuerung
(Register 42000 / 42010 / 42020 / 42021): Man übernimmt die Kontrolle und gibt feste
Leistungssollwerte vor.

**Das hat sich als schlechter Weg erwiesen.** Ein fester Sollwert kann der
Wirklichkeit immer nur hinterherlaufen — steigt die Hauslast zwischen zwei
Regelzyklen an, entsteht Netzbezug. Im Referenzsystem führte das zu einem
Netzbezug-Ereignis über mehrere Stunden mit Spitzen bis 4,3 kW.

Die Register 44002/44003 wirken dagegen als **Obergrenze**, während der Speicher
im eigenen Anti-Feed-Modus bleibt und selbst gegen den Netzzähler regelt — deutlich
schneller, als es eine Automation je könnte.

### Der wichtige, schlecht dokumentierte Punkt

> **44002 und 44003 wirken auch im Anti-Feed-Modus**, nicht nur während einer
> aktiven RS485-Fremdsteuerung.

Das wurde durch Live-Tests bestätigt: Bei gesetztem `Max Charge Power = 0` nimmt
ein Speicher im Anti-Feed-Modus auch bei vorhandenem Überschuss keine Leistung mehr
auf. Bei `= 1000` begrenzt er sich auf etwa 1000 W.

Damit lässt sich der Speicher steuern, **ohne** ihm die Regelung wegzunehmen — man
setzt nur die Leitplanken.

---

## Weitere verifizierte Register

| Adresse | Bedeutung | Skalierung | Anmerkung |
|---|---|---|---|
| 32100 | Battery Voltage | 0,01 | |
| 32101 | Battery Current | 0,01 | int16, Vorzeichen |
| 32102 | Battery Power | 1 | int32 |
| 32104 | Battery SOC | 1 | % |
| **32105** | **Nominal Capacity** | 1 | Wh — bestätigt: 5120 = 5,12 kWh, entspricht exakt den Gerätedaten |
| 32202 | AC Power | 1 | int32; **negativ = lädt, positiv = entlädt** |
| 33000 | Total Charging Energy | 0,01 | kWh, uint32 |
| 33002 | Total Discharging Energy | 0,01 | kWh, uint32 |
| 35000 | Internal Temperature | 0,1 | °C |
| 35001 / 35002 | MOS1 / MOS2 Temperature | 0,1 | °C |
| 35010 / 35011 | Max / Min Cell Temperature | 0,1 | °C |
| 35100 | Inverter State | 1 | |
| 36103 | Netzspannung | 0,1 | V — bestätigt: 2348 = 234,8 V |
| 37016 | AC-Spannung | 0,1 | V — bestätigt: 2333 = 233,3 V |
| 42000 | RS485 Control Mode | — | 21930 (0x55AA) = an, 21947 (0x55BB) = aus |
| 42010 | Force Charge/Discharge | — | 0 = Stop, 1 = Laden, 2 = Entladen |
| 42020 / 42021 | Forcible Charge / Discharge Power | 1 | W |
| 43000 | User Work Mode | — | 0 = Manual, 1 = Anti-Feed, 2 = Trade |

### Vermutete Register (aus dem vollständigen Scan, unbestätigt)

Diese Deutungen stammen aus Wertlage und Nachbarschaft, nicht aus Gegentests —
mit entsprechender Vorsicht zu lesen:

| Adresse | vermuteter Zweck | Beobachtung |
|---|---|---|
| 30401–30403 | **Geräte-IP-Adresse** | Byte-Interpretation ergibt `192.168.x.x` — passte auf einem Testgerät exakt zur bekannten IP |
| 34021, 34023, 34025–34028, 34030, 34033 | Einzelzellspannungen | 8 Werte im Fenster 3280–3402 mV (plausibel für LiFePO4); Spreizung 122 mV zwischen den Extremwerten — für einen ausbalancierten Pack hoch, aber Momentaufnahme unter Last |
| 41503 | Text/Kennung | Bytepaar ergibt ASCII „LA" — vermutlich Teil einer Seriennummer oder Modellbezeichnung, die sich über mehrere Register erstreckt |
| 30305, 30307, 30353–30355 | Text/Kennung | ähnliches Muster wie 41503 |
| 43101, 43102, 43107 | Ladezeitplan-Parameter | Werte 300 / 2300 / 2316 im sonst leeren Block 43100–43127 — Marstek-Apps bieten bis zu zehn Zeitfenster mit Start/Ende/Leistung an; Lage und Wertgröße passen zu einer Leistungsangabe in W |

---

## Generationsunterschiede

| Adresse | Bedeutung | Gen 2 | Gen 3 |
|---|---|---|---|
| 42011 | Charge to SOC | ✅ | nicht geprüft |
| 44000 | Charging cutoff capacity | ✅ (Skalierung 0,1) | ❌ „Illegal data address" |
| 44001 | Discharging cutoff capacity | ✅ (Skalierung 0,1) | ❌ „Illegal data address" |
| 30200 / 30202 | EMS / VMS Version | ❌ liefert 0 | ✅ |

**Praktische Folge:** Eine geräteseitige Entladegrenze lässt sich nur auf Gen 2
setzen. Für generationsübergreifende Steuerung muss die Untergrenze in der
Automation abgebildet werden — über 44003 = 0, sobald der gewünschte SoC erreicht
ist. Genau das macht dieses Projekt.

Auf Gen 2 ist 44001 zusätzlich nützlich als **hardwareseitiges Sicherheitsnetz**:
Es gilt auch dann, wenn Home Assistant ausfällt.

### Gibt es auf Gen 3 wirklich kein Entladegrenzen-Register?

Diese Frage wurde im August 2026 mit einem vollständigen Registerscan verfolgt
(Methodik unten). Ergebnis:

- **Kein SunSpec.** Die genormte Kennung `"SunS"` an den SunSpec-Basisadressen
  (0, 40000, 50000) antwortet nicht — Marstek implementiert keinen SunSpec-Layer
  mit den dort definierten genormten SoC-Grenzen (Model 803/804).
- **Kein anderer Hersteller mit identischer Registerkarte gefunden.** Weder BYD
  noch andere geprüfte Hersteller teilen dieselben Adressen — die Blockstruktur
  (30xxx/32xxx/34xxx/4xxxx) ist bei chinesischen Speicher-OEMs verbreitet, aber
  nicht standardisiert.
- **169 Register über einen Teilscan (30000–48000) bestätigt**, darunter kein
  einziges mit einem plausiblen Prozentwert (10–30 oder 100–300) außerhalb der
  bereits bekannten Bereiche.
- Ein zusammenhängender, vollständig undokumentierter Block bei **41601–41630**
  (13 Register, alle Werte 0) ist der aussichtsreichste verbleibende Kandidat,
  ließ sich aber bisher keiner Funktion zuordnen.

**Zwischenstand: Es spricht viel dafür, dass es keine geräteseitige
Entladegrenze auf Gen 3 gibt** — aber der Scan ist mit ~13–25 % Trefferquote pro
Durchlauf (siehe unten) noch nicht vollständig genug, um das sicher
auszuschließen.

---

## Methodik des vollständigen Scans

Für die obige Frage wurde ein eigener Modbus-TCP-Client gebaut (reine
Lesefunktionen 0x03/0x04, keine Schreibfunktion), der den Adressraum
systematisch abklopft. Zwei Eigenschaften der Geräte erzwangen ein
ungewöhnliches Vorgehen:

**Der Gen 3 wird nach einer einzigen ungültigen Anfrage kurzzeitig blind.**
Nach einer Anfrage an eine nicht existierende Adresse antwortet das Gerät für
250–700 ms auch auf **gültige** Register mit Exception 4 — inklusive Register,
die im selben Testlauf zuvor erfolgreich gelesen wurden. Ein einfacher
sequenzieller Sweep erzeugt dadurch massenhaft falsche Negative: Ein bekanntes,
vorhandenes Register (etwa 32104) kann in einem Durchlauf als „nicht vorhanden"
erscheinen, weil die Anfrage direkt in eine solche Blindphase fiel.

**Gemessene Trefferquote pro Durchlauf unter realistischen Bedingungen: 13–25 %**
(zehn bekannte Register, eingemischt in einen Strom überwiegend ungültiger
Adressen, mit denselben Timing-Parametern wie der eigentliche Scan). Bei
150 ms Pause nach einem Fehlschlag lag die Quote bei 13,3 %; bei 700 ms bei 25 %
— die Gesamtdauer blieb dabei nahezu gleich, weil die längere Pause pro
Durchlauf mehr Zeit kostet, als sie an zusätzlichen Durchläufen einspart. Für
95 % Vollständigkeit über den gesamten 16-Bit-Adressraum wären dadurch rund
21 Durchläufe nötig — bei den gemessenen 4,5 Stunden pro Durchlauf und Gerät
etwa vier Tage Dauerlast.

**Konsequenzen für das Vorgehen:**

- **Mehrfachdurchläufe statt Einzelprüfung mit Gegenprobe.** Eine Gegenprobe je
  Adresse (zweite Lesung bei Erfolg, Kontrollregister bei Misserfolg) ist
  korrekt, aber mit rund 14,5 s pro Adresse zu langsam für den vollen
  Adressraum. Schnelle Durchläufe in zufälliger Reihenfolge, mehrfach
  wiederholt, sind für große Bereiche effizienter — ein real vorhandenes
  Register muss dann mehrfach hintereinander verpasst werden, um dauerhaft zu
  fehlen.
- **Bereichseinschränkung.** Alle gefundenen Register lagen zwischen 30000 und
  48000; unterhalb von 30000 lieferte ein voller Durchlauf (32.768 Adressen)
  keinen einzigen Treffer. Eine Beschränkung auf 30000–48000 reduziert die
  nötige Zeit erheblich, ohne (bisher) etwas zu verpassen.
- **Aufteilung auf beide Gen-3-Geräte.** Da beide baugleich sind, geteilte
  Trefferliste: Was das eine Gerät findet, muss das andere nicht erneut prüfen.
- **Nur nachts scannen.** Während ein Gerät gescannt wird, kann die Kaskade
  (siehe [kaskade.md](kaskade.md)) ihre Schreibvorgänge nicht zuverlässig
  zustellen. Beim Entladen ist das inzwischen abgesichert (unlesbarer Ladestand
  → Speicher bleibt offen), beim Laden nicht — ein während des Scans
  gesperrter Ladekanal bliebe gesperrt. Nachts wird ohnehin nicht geladen.
- **Grenzen vor dem Scan öffnen.** Schreibversuche der Kaskade während eines
  Scanfensters können scheitern, ohne dass das auffällt (die Meldung
  „erfolgreich aufgerufen" bestätigt nur die Entgegennahme durch Home
  Assistant, nicht die Zustellung ans Gerät). Ein Gerät mit offenen Grenzen vor
  Scanbeginn bleibt bei einem gescheiterten Schreibvorgang im sicheren Zustand.

---

## Bekannte Fallstricke

**Nach einem Ausfall der Steuerung bleibt die letzte Freigabe aktiv.** Fällt Home
Assistant aus, behalten die Speicher ihren zuletzt geschriebenen Registerwert und
regeln munter weiter. Im Referenzsystem lud ein Speicher dadurch über seinen
Deckel hinaus von 75 % auf 90 %.

Gegenmittel, je nach Anspruch:
- Gen 2: 44000/44001 als geräteseitige Grenze setzen
- Beim geordneten Herunterfahren konservative Werte schreiben (hilft nicht beim
  harten Absturz)
- Konservative Standardwerte akzeptieren

**Der Sensorwert hinkt nach.** Wird ein Register geschrieben, zeigt der
zugehörige Modbus-Sensor je nach `scan_interval` noch den alten Wert. Beim Prüfen
also entweder das Polling abwarten oder direkt die AC-Leistung betrachten.

**Modbus-Ausnahme 4 bei paralleler Nutzung.** Greifen mehrere Systeme gleichzeitig
auf denselben Speicher zu (etwa evcc und Home Assistant), treten sporadisch
`exception '4' (server device failure)` auf. Das ist derselbe Mechanismus wie die
Blindphasen beim Scan oben, nur ausgelöst durch Fremdzugriff statt durch eigene
ungültige Adressen. Unkritisch für die Kaskade, solange Leseausfälle korrekt
behandelt werden — siehe den `unavailable`-Sensor-Fehler in
[kaskade.md](kaskade.md#der-unavailable-sensor-fehler). **Nicht** unkritisch ist
das naive `float(0)`-Muster ohne Gültigkeitsprüfung: Es macht aus einem
Leseausfall unbemerkt eine Null.

---

## Quellen

Register aus der Community-Dokumentation zur Marstek-Venus-Reihe, verifiziert durch
eigene Lese- und Schreibtests an laufenden Geräten. Wo Community-Angaben von den
Messungen abwichen, gilt die Messung. Der vollständige Registerscan (169 bestätigte
Adressen im Bereich 30000–48000, Stand August 2026) ist eine eigene Erhebung ohne
bekannte externe Parallele — im photovoltaikforum.com-Thread zum Thema wird
ausdrücklich vermerkt, dass die dort gelisteten Adressen „nicht praktisch getestet"
wurden.
