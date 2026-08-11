# Die Prioritätskaskade

Jeden Regelzyklus (30 s) wird die gesamte Kaskade neu ausgewertet. Es gibt keine
Zustandsspeicherung zwischen den Durchläufen — jede Entscheidung ergibt sich neu
aus den aktuellen Messwerten. Das macht das Verhalten nachvollziehbar und
unempfindlich gegen verpasste Zyklen.

**Laden und Entladen werden seit dem 06.08.2026 unterschiedlich behandelt.**
Der Grund steht in [Warum Entladen nicht mehr konzentriert](#warum-entladen-nicht-mehr-konzentriert)
weiter unten — kurz: Konzentration spart Wirkungsgrad beim Laden, erzeugt beim
Entladen aber einen Einzelpunkt des Versagens.

---

## Schritt 1: Den echten Bedarf ermitteln

Der Netzzähler allein genügt nicht. Steht er auf null, weil die Speicher gerade
den kompletten Überschuss aufnehmen, wäre die naive Schlussfolgerung „kein
Überschuss vorhanden" — und man würde die Speicher schließen, woraufhin plötzlich
eingespeist würde.

Deshalb wird gegengerechnet, was die Speicher gerade schon abdecken:

```
Überschuss = max(0, −Netzleistung) + was die Speicher gerade aufnehmen
```

Diese Größe ist unabhängig davon, wie die Speicher gerade eingestellt sind.
(Für das Entladen wird kein äquivalenter „Bedarf" mehr gebraucht — siehe Schritt 3.)

## Schritt 2: Anzahl bestimmen (nur Laden)

```
Anzahl = aufrunden(Überschuss / 2500 W), begrenzt auf 0…3
```

Bis 06.08.2026 gab es eine analoge Rechnung fürs Entladen samt Sicherheitsfaktor.
Sie ist entfallen — Entladen öffnet jetzt immer alle fähigen Speicher, siehe unten.

## Schritt 3: Auswählen

### Laden — Auswahl unter den fähigen Speichern

Fähig ist ein Speicher, wenn sein Ladestand **lesbar** ist und unter dem eigenen
Deckel liegt. Aus den fähigen Speichern werden die ersten *n* genommen (n aus
Schritt 2), sortiert nach dreistufigem Schlüssel:

```
Schlüssel = SoC − 2000   wenn unter der eigenen Untergrenze     (dringend: zurück ins Band)
          = SoC − 1000   wenn unter der Nachlade-Vorrangsgrenze (hat die Nacht getragen)
          = Zyklen       sonst                                  (wenigste zuerst)
```

Die drei Stufen sortieren garantiert in dieser Reihenfolge, weil Zyklenstände
(typisch 100–300) immer über den `SoC − 1000`- und `SoC − 2000`-Werten liegen.
Innerhalb jeder Stufe entscheidet der niedrigste Wert zuerst.

**Warum eine Nachlade-Vorrangsstufe (neu, 06.08.2026):** Der Speicher, der die
Nacht getragen hat, steht morgens am tiefsten — unabhängig vom Zyklenstand. Ohne
diese Stufe hätte reiner Zyklenausgleich ihn unter Umständen zurückgestellt,
obwohl er als Erster wieder Reserve braucht. Schwellwert konfigurierbar
(`fleet_nachlade_grenze`, Standard 50 %, 0 = aus).

### Entladen — alle fähigen Speicher, ohne Auswahl

```
Fähig = Ladestand nicht lesbar  ODER  SoC über der eigenen Untergrenze
        (bei echtem Netzbezug: über der absoluten Notreserve statt der
         individuellen Untergrenze)
```

Alle fähigen Speicher werden geöffnet — keine Sortierung, keine Begrenzung auf
eine Anzahl. Das kostet nichts: Im Anti-Feed-Modus entlädt ein geöffneter
Speicher nur, wenn tatsächlich Bedarf besteht. 2500 W ist eine Obergrenze, kein
Sollwert.

**Bei unlesbarem Ladestand gilt: offen.** Siehe
[Der unavailable-Sensor-Fehler](#der-unavailable-sensor-fehler).

## Schritt 4: Schreiben

Ausgewählte Lade-Kandidaten und alle fähigen Entlade-Kandidaten bekommen 2500 W
auf ihr jeweiliges Register, alle anderen 0.

---

## Die Ebenen im Einzelnen

### Ebene 0 — Netzbezug

**Seit 10.08.2026 mit angepasster Schwelle (100 W statt 50 W) und geänderter
Bedeutung.** Weil Entladen ohnehin immer alle fähigen Speicher öffnet, steuert
dieser Schalter nur noch **eines**: ob die individuellen Untergrenzen zugunsten
der absoluten Notreserve aufgegeben werden.

```
netzbezug = Netzleistung > 100 W
fähig     = SoC > (Notreserve wenn netzbezug sonst individuelle Untergrenze)
```

Bei 50 W sprach der Schalter auf das normale Rauschen des Netzzählers an — die
Speicher liefen dadurch nachts routinemäßig bis auf 11–15 % leer, obwohl ihre
Untergrenze bei 25 % lag. Am Morgen war dann nichts mehr da, und der Netzbezug
verlagerte sich nur, statt zu sinken. Details unter
[Die 50-W-Falle](#die-50-w-falle).

Zusätzlich zum 30-Sekunden-Zyklus läuft ein eigener Trigger, der nach 5 Sekunden
anhaltendem Bezug **über 150 W** sofort auslöst und alle Speicher oberhalb der
Notreserve öffnet — unabhängig vom Hauptzyklus.

### Ebene 1 — Nachtreserve

Ab einer konfigurierbaren Stunde (Standard 17 Uhr) wird geprüft:

```
Defizit = max(0, geschätzter Nachtverbrauch − gespeicherte Energie)
```

Bei Defizit fallen die Ladedeckel — alle Speicher dürfen bis 100 %.

**Warum erst ab 17 Uhr:** Früher am Tag sind die Speicher naturgemäß noch nicht
am Deckel. Eine Prüfung um 14 Uhr erzeugt ein Scheindefizit und hebt den Schutz
unnötig auf.

Auch mit gefallenem Deckel bleibt die Ladepriorität erhalten: Untergrenze vor
Nachlade-Vorrang vor Zyklenausgleich.

### Ebene 2 — SoC-Band

Jeder Speicher hat einen eigenen Deckel und eine eigene Untergrenze. Zusätzlich
gilt anlagenweit eine absolute Notreserve, die nur bei echtem Netzbezug (Ebene 0)
unterschritten werden darf.

Die späte Volladung setzt vor der konfigurierten Stunde einen Tagesdeckel über
alle Speicher — außer die PV-Restprognose reicht nicht mehr aus, um bis
Sonnenuntergang voll zu werden.

### Ebene 3 — Nachlade-Vorrang und Zyklenausgleich

Innerhalb des erlaubten Ladebandes entscheidet zuerst der Nachlade-Vorrang (wer
zuletzt am tiefsten stand), dann die Zyklenzahl. Da Zyklen über die
**Ladeenergie** gezählt werden, holt ein Speicher nur auf, wenn er tief entladen
*und* voll geladen wird — deshalb die speicherindividuellen Bänder statt eines
globalen.

---

## Warum Entladen nicht mehr konzentriert

Bis zum 06.08.2026 folgte das Entladen demselben Prinzip wie das Laden: möglichst
wenige Speicher, dafür mit voller Leistung, ausgewählt nach Zyklenstand mit einer
„sanften Übergabe" an den nächsten Speicher, sobald der aktive sich seiner
Untergrenze näherte.

**Ein direkter Vergleich zweier Nächte widerlegte das:**

| Nacht | Verfahren | Netzbezug | Dauer |
|---|---|---|---|
| 04./05.08. | Entlade-Konzentration | 1,28 kWh | 2 h 50 min |
| 05./06.08. | freier Anti-Feed-Betrieb (keine Steuerung) | 0,14 kWh | 9 min |

Der Faktor neun ist kein Messrauschen. Die Ursache: Konzentration erzeugt einen
**Einzelpunkt des Versagens**. Der ausgewählte Speicher lief in der
Problemnacht an seine geräteseitige Entladegrenze (beim Gen 3 rund 20 %,
undokumentiert — siehe [register-map.md](register-map.md)), während ein zweiter
Speicher mit 81 % Ladestand daneben gesperrt blieb. Bis zum nächsten Zyklus
entstand Netzbezug, obwohl anlagenweit reichlich Energie vorhanden war.

**Für den Zyklenausgleich kostet das Verteilen nichts**, weil Zyklen über den
*Ladedurchsatz* gezählt werden (`geladene kWh / Kapazität`), nicht über das
Entladen. Ein Speicher, der beim Entladen mitmacht, aber seltener geladen wird,
sammelt trotzdem weniger Zyklen als einer, der oft und viel geladen wird.

**Für das Laden gilt die Konzentration weiterhin.** Dort ist sie unverändert
richtig: Wechselrichter haben einen weitgehend konstanten Eigenverbrauch, und ein
gedrosselter Speicher im Teillastbereich verliert überproportional Wirkungsgrad.
Beim Laden gibt es außerdem keinen Einzelpunkt-Effekt in dieselbe Richtung — ein
zu voller Speicher ist unkritisch, ein zu leerer beim Entladen schon.

---

## Der unavailable-Sensor-Fehler

`states(sensor) | float(0)` ist ein verbreitetes Muster in Home-Assistant-
Templates — und eine Falle. Ist der Sensor `unavailable` (etwa durch einen
kurzen Modbus-Aussetzer), liefert der Ausdruck **0**, nicht „unbekannt".

In der ursprünglichen Entladebedingung `SoC > Untergrenze` wurde daraus
`0 > 25` — falsch. Ein Speicher, den Home Assistant für ein paar Sekunden nicht
lesen konnte, wurde dadurch **vom Entladen ausgesperrt**, obwohl er einwandfrei
lief. Das ist dieselbe Fehlerform wie die Entlade-Konzentration oben: Ein Speicher
fällt aus der Kaskade heraus, während er tatsächlich zur Verfügung stünde.

Entdeckt wurde der Fehler, als ein Registerscan (siehe
[register-map.md](register-map.md#methodik-des-vollständigen-scans)) einen
Speicher zeitweise unerreichbar machte und dessen Entladegrenze bei 0 hängen
blieb.

**Die Korrektur unterscheidet zwei Richtungen bewusst:**

```
LADEN:    fähig nur bei GÜLTIGEM Ladestand   (blind laden wäre riskant)
ENTLADEN: bei UNGÜLTIGEM Ladestand OFFEN     (im Zweifel liefern statt sperren)
```

Das ist keine symmetrische Lösung, sondern eine bewusste Asymmetrie: Ein
fälschlich geöffneter Speicher entlädt im Anti-Feed-Modus nur bei echtem Bedarf
— das Risiko ist gering. Ein fälschlich gesperrter Speicher fehlt dagegen genau
dann, wenn er gebraucht wird.

In Jinja prüft `| is_number` die Gültigkeit, bevor `| float(0)` überhaupt
aufgerufen wird:

```jinja
g2:   "{{ states('sensor.marstek2_battery_soc') | is_number }}"
soc2: "{{ states('sensor.marstek2_battery_soc') | float(0) }}"
...
'ok': (not g2) or soc2 > untergrenze
```

---

## Die 50-W-Falle

Ebene 0 (Netzbezug) hatte ursprünglich eine Schwelle von 50 W — dieselbe Schwelle,
die auch den schnellen Schutztrigger auslöste. Das führte zu einem Nebeneffekt,
der erst nach mehreren Tagen auffiel: Der Netzzähler pendelt im Normalbetrieb
durch die Anti-Feed-Regelung der Speicher ständig knapp um die Null. Jedes
Überschwingen über 50 W gab die individuellen Untergrenzen zugunsten der
Notreserve (12 %) frei.

**Gemessene Folge über mehrere Nächte:** Speicher fielen routinemäßig auf 11–15 %,
obwohl ihre konfigurierte Untergrenze 25 % war. Am Morgen — wenn der Verbrauch
wirklich anzieht — war dann nichts mehr übrig, und ausgerechnet die Stunde
zwischen 5 und 6 Uhr trug an einem Tag 0,43 kWh bei, mehr als die fünf Stunden
davor zusammen. Das Tiefziehen verhinderte den Netzbezug nicht, es verschob ihn
nur und vergrößerte ihn dabei.

**Behoben durch Trennung der beiden Funktionen** (siehe Ebene 0 oben): Der
schnelle Schutztrigger blieb bei 150 W (er öffnet nur, verändert keine
Untergrenzen). Die Schwelle für das Aufgeben der Untergrenzen wurde auf 100 W
angehoben. Erste Messnacht danach: 0,17 kWh gleichmäßig verteilt über elf
Stunden, kein Morgenausbruch. Weitere Nächte stehen zur Bestätigung noch aus.

---

## Was bewusst nicht enthalten ist

**Keine Leistungssollwerte.** Es werden ausschließlich Obergrenzen gesetzt. Die
Speicher regeln im Anti-Feed-Modus selbst — schneller, als es eine Automation
könnte. Ein Versuch mit festen Sollwerten führte im Referenzsystem zu stundenlangem
Netzbezug, weil ein 45 Sekunden alter Sollwert der Wirklichkeit nicht folgen kann.

**Keine Drosselung beim Laden.** Nicht benötigte Speicher werden auf 0 gesetzt,
nicht auf einen kleinen Wert. Teillast kostet Wirkungsgrad, und ein gedrosselter
Speicher kann einen Überschuss nicht aufnehmen, den er eigentlich aufnehmen
könnte.

**Keine Auswahl beim Entladen.** Siehe oben — das war bis 06.08.2026 anders und
hat sich als Fehler erwiesen.

**Keine Zustandsspeicherung.** Jeder Zyklus rechnet neu. Ein verpasster Durchlauf
hat keine Folgen.
