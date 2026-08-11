# Balancing on Batteries

Home-Assistant-Steuerung für mehrere parallel betriebene Marstek-Venus-Heimspeicher.

Regelt Lade- und Entladeleistung je Speicher über Modbus, mit drei Zielen in dieser
Reihenfolge:

1. **Netzbezug vermeiden** — schlägt alles andere
2. **Zellen schonen** — wenig Zeit in den Extrembereichen (< 20 %, > 80 %)
3. **Zyklen angleichen** — der am wenigsten genutzte Speicher arbeitet am meisten

Getestet mit drei Marstek Venus E (5,12 kWh / 2500 W), Generation 2 und 3 gemischt,
unter Home Assistant OS auf einem Raspberry Pi 4.

---

## Der Kern: konzentrieren beim Laden, verteilen beim Entladen

> **Update 06.08.2026:** Ursprünglich galt „konzentrieren statt drosseln" für
> beide Richtungen. Ein Nachtvergleich hat gezeigt, dass das beim Entladen ein
> Fehler war — Details unten und in
> [docs/kaskade.md](docs/kaskade.md#warum-entladen-nicht-mehr-konzentriert).

Die naheliegende Lösung für Zyklenausgleich ist, den viel genutzten Speicher zu
drosseln — etwa auf 500 W statt 2500 W. **Das ist ein Fehler.**

Wechselrichter haben einen weitgehend konstanten Eigenverbrauch. Bei 500 W von
2500 W fällt der prozentual vier- bis fünfmal so stark ins Gewicht wie bei
Volllast. Drei gedrosselte Speicher im Teillastbereich sind deutlich
verlustreicher als ein Speicher im guten Betriebspunkt.

Dazu kommt ein zweiter, gravierenderer Effekt: Ist der bevorzugte Speicher
gerade nicht ladefähig (etwa weil er seinen Ladedeckel erreicht hat), bleibt nur
ein gedrosselter übrig — und der Rest des Überschusses geht ins Netz.

**Deshalb beim Laden:** Es wird ausgerechnet, wie viele Speicher der aktuelle
Überschuss tatsächlich braucht. Genau so viele werden mit **voller Leistung**
freigegeben, alle anderen auf 0 gesetzt. Der Zyklenausgleich steuert, *welcher*
Speicher lädt — nicht, *wie stark* er gedrosselt wird.

```
Überschuss = Einspeisung + was die Speicher gerade schon aufnehmen
Anzahl     = aufrunden(Überschuss / 2500 W), maximal 3
Auswahl    = unter den tatsächlich fähigen Speichern, dreistufig sortiert
             (Untergrenze → Nachlade-Vorrang → Zyklen)
```

**Beim Entladen gilt das nicht mehr.** Konzentration auf einen einzigen Speicher
erzeugt nachts einen Einzelpunkt des Versagens: Läuft der ausgewählte Speicher
an seine geräteseitige Grenze, während andere noch voll sind, entsteht
Netzbezug — obwohl anlagenweit genug Energie vorhanden wäre. Gemessen:

| Nacht | Verfahren | Netzbezug |
|---|---|---|
| 04./05.08. | Entlade-Konzentration | 1,28 kWh in 2 h 50 min |
| 05./06.08. | freier Betrieb (keine Steuerung) | 0,14 kWh in 9 min |

Der Faktor neun kostet beim Verteilen nichts, weil Zyklen über den
**Ladedurchsatz** gezählt werden, nicht über das Entladen. Seither öffnet die
Kaskade beim Entladen **alle** Speicher oberhalb ihrer Untergrenze gleichzeitig
— auch das kostet nichts, denn ein geöffneter Speicher entlädt im
Anti-Feed-Modus ohnehin nur bei echtem Bedarf.

Das funktioniert, weil die Marstek-Speicher im Anti-Feed-Modus selbst regeln:
Die 2500 W sind eine **Obergrenze, kein Sollwert**. Ein freigegebener Speicher
nimmt oder liefert nur, was tatsächlich gebraucht wird.

---

## Die Prioritätskaskade

Alle 30 Sekunden, zusätzlich ein 5-Sekunden-Schnelltrigger für Netzbezug:

| Ebene | Regel |
|---|---|
| **0 — Netzbezug** | Bezug > 100 W → individuelle Untergrenzen fallen zugunsten der Notreserve. Bezug > 150 W für 5 s (eigener Schnelltrigger) → sofort alle Speicher oberhalb der Notreserve öffnen |
| **1 — Nachtreserve** | Ab 17 Uhr: reicht der Vorrat für die Nacht? Wenn nein, fallen die Ladedeckel |
| **2 — SoC-Band** | Laden bis zum individuellen Deckel, Entladen bis zur individuellen Untergrenze (bzw. Notreserve bei Netzbezug) |
| **3 — Nachlade-Vorrang** | Wer zuletzt am tiefsten stand (die Nacht getragen hat), wird beim Laden zuerst bedient |
| **4 — Zyklenausgleich** | Danach: wer die wenigsten Zyklen hat, lädt zuerst |

Entladen kennt ab Ebene 2 keine weitere Auswahl mehr — alle fähigen Speicher
öffnen gleichzeitig. Details in [docs/kaskade.md](docs/kaskade.md).

---

## Speicherindividuelle Bänder

Zyklen entstehen durch **Ladedurchsatz**, nicht durch den Ladestand. Wer aufholen
soll, braucht deshalb ein *breites* Band — tief entladen **und** voll geladen.
Wer geschont werden soll, ein schmales.

Beispielkonfiguration bei 264 / 138 / 125 Zyklen (Startwerte):

| Speicher | Zyklen | Deckel | Untergrenze | nutzbares Band |
|---|---|---|---|---|
| Speicher 1 | 264 | 65 % | 35 % | 1,5 kWh |
| Speicher 2 | 138 | 90 % | 25 % | 3,3 kWh |
| Speicher 3 | 125 | 100 % | 25 % | **3,8 kWh** |

Der am wenigsten genutzte Speicher leistet damit deutlich mehr als der meist
genutzte — pro Tag, automatisch.

> **Untergrenzen von 15/20 % auf 25 % angehoben (05.08.2026):** Die
> undokumentierte geräteseitige Entladegrenze der Gen-3-Speicher liegt bei rund
> 20 % (Literatur nennt 11 % — siehe [docs/register-map.md](docs/register-map.md)).
> Lag die konfigurierte Untergrenze darunter, erreichte die Kaskade ihr Ziel nie
> und der Speicher lief in die Geräteabschaltung, während andere noch Reserve
> hatten. 25 % gibt Abstand zur unbekannten Hardware-Grenze.

**Tempo des Ausgleichs:** Über eine Woche Betrieb gemessen: Der zyklenreichste
Speicher gewann 4 Zyklen, der zyklenärmste 7 — die Spanne schrumpfte von 139 auf
136. Bei diesem Tempo braucht ein vollständiger Angleich mehrere hundert Tage.
Wer schneller angleichen will, sollte den Deckel des meistgenutzten Speichers
stärker absenken, nicht nur seine Untergrenze anheben — Zyklen entstehen über
den Ladedurchsatz, ein engeres Ladeband bremst direkter.

**Zu den 100 %:** LiFePO4-BMS balancieren die Zellen nur im oberen Ladebereich.
Ein Speicher, der nie voll wird, driftet mit der Zeit auseinander. Regelmäßiges
Vollladen ist also nicht nur zulässig, sondern sinnvoll — solange er dort nicht
*verharrt*. Genau dafür gibt es die späte Volladung.

## Nachlade-Vorrang

Zusätzlich zum Zyklenausgleich gibt es seit 06.08.2026 eine vorgeschaltete
Priorität: Ein Speicher, der die Nacht getragen hat, steht morgens am tiefsten
— unabhängig davon, wie viele Zyklen er insgesamt hat. Er wird beim Laden
zuerst bedient, bis er eine konfigurierbare Schwelle (Standard 50 %) wieder
überschritten hat. Erst danach entscheidet wieder allein der Zyklenstand.

Ohne diese Stufe könnte ein Speicher mit vielen Zyklen morgens leer bleiben,
weil der Zyklenausgleich ihn zurückstellt — obwohl er als Erster wieder Reserve
für die kommende Nacht braucht.

---

## Späte Volladung

Ohne Zeitsteuerung wäre der Speicher an einem sonnigen Tag schon mittags voll und
stünde sechs bis acht Stunden bei 100 %. Deshalb:

- **Bis 15 Uhr** wird nur bis zu einem Tagesdeckel (Standard 80 %) geladen
- **Ab 15 Uhr** sind die vollen individuellen Deckel frei
- **Ausnahme:** Reicht die PV-Restprognose nicht mehr aus, um bis Sonnenuntergang
  voll zu werden, wird sofort freigegeben — an trüben Tagen wird nichts verschenkt

Damit sinkt die Zeit bei hohem Ladestand von sechs bis acht auf zwei bis drei
Stunden, direkt vor der abendlichen Entladung.

---

## Nachtreserve: gemessen, nicht geschätzt

Die Schwelle, ab der die Ladedeckel fallen, sollte auf echten Daten beruhen. Die
Methodik steht in [docs/messmethodik.md](docs/messmethodik.md) — kurz gefasst:
Nicht den Hausverbrauchssensor nehmen, sondern die **tatsächliche Entnahme aus den
Speichern** plus Netzbezug zwischen Abend und Morgen, über mehrere Wochen.

Im Referenzhaushalt (25 Nächte, 21:00–06:00):

| | kWh |
|---|---|
| Median | 7,18 |
| Mittel | 7,03 |
| 90 %-Wert | 9,07 |
| Maximum | 11,16 |

Die ursprüngliche Schätzung lag bei 8,0 kWh — nah am Median, aber zu niedrig für
die oberen 10 % der Nächte.

---

## Betriebserfahrung: drei reale Fehler

Drei Fehler traten erst im laufenden Betrieb zutage, nicht beim Entwurf. Alle
drei haben dieselbe Grundform: Ein Speicher fällt unbemerkt aus der Kaskade
heraus, obwohl er zur Verfügung stünde. Ausführlich dokumentiert in
[docs/kaskade.md](docs/kaskade.md), hier die Kurzfassung:

| Fehler | Symptom | Fix |
|---|---|---|
| Entlade-Konzentration | 1,28 kWh Netzbezug in einer Nacht, während ein zweiter Speicher mit 81 % danebenstand | Entladen öffnet seither immer alle fähigen Speicher |
| `unavailable` → `float(0)` | Ein kurzer Modbus-Aussetzer sperrte einen Speicher fürs Entladen | Gültigkeit separat prüfen (`is_number`); bei Entladen im Zweifel offen |
| Netzbezugsschwelle 50 W | Speicher liefen nachts routinemäßig bis 11–15 % leer, Morgen-Ausbruch bis 0,43 kWh in einer Stunde | Schwelle für das Aufgeben der Untergrenzen auf 100 W angehoben |

Gemeinsame Lehre: **Sicherheitslogik, die "im Zweifel sperren" wählt, kann
selbst zur Ursache von Netzbezug werden.** Bei einer Flotte mit Redundanz ist
"im Zweifel liefern" oft die risikoärmere Richtung — ein fälschlich geöffneter
Speicher entlädt im Anti-Feed-Modus nur bei echtem Bedarf, ein fälschlich
gesperrter fehlt genau dann, wenn er gebraucht wird.

---

## Voraussetzungen

- Home Assistant mit `packages`-Konfiguration
- Marstek Venus E per Modbus TCP erreichbar (direkt oder über einen Proxy)
- Ein Netzzähler-Sensor mit Vorzeichen: **positiv = Bezug, negativ = Einspeisung**
- Zyklen- und SoC-Sensoren je Speicher
- Optional: PV-Restprognose für die späte Volladung

## Installation

1. Dateien aus `packages/` nach `config/integrations/` (oder das eigene
   Package-Verzeichnis) kopieren
2. Entity-IDs anpassen — siehe [CUSTOMIZE.md](CUSTOMIZE.md)
3. Home Assistant neu starten
4. Regler im Dashboard auf die eigenen Werte setzen

## Enthalten

| Datei | Zweck |
|---|---|
| `packages/battery_fleet_manager.yaml` | die Steuerung |
| `packages/fleet_monitoring.yaml` | Kennzahlen zur Wirksamkeitsprüfung |
| `packages/recorder.yaml` | Recorder-Filter (Beispiel) |
| `docs/register-map.md` | verifizierte Marstek-Modbus-Register |
| `docs/kaskade.md` | die Prioritätsebenen im Detail |
| `docs/messmethodik.md` | Nachtverbrauch korrekt messen |

## Stand

Produktiv im Einsatz seit August 2026. Die Kennzahlen aus `fleet_monitoring.yaml`
dienen der laufenden Überprüfung: Netzbezugsdauer, Zeit in den Extrembereichen je
Speicher, Angleichung der Zyklen.

Die 100-W-Schwelle (Betriebserfahrung oben) hat sich in der ersten Nacht bewährt
(0,17 kWh gleichmäßig über 11 Stunden, kein Morgenausbruch), gilt aber noch nicht
als abschließend bestätigt — weitere Nächte stehen aus. Der Registerscan nach
einer Gen-3-Entladegrenze (siehe [docs/register-map.md](docs/register-map.md))
läuft im Hintergrund weiter; 169 Register im Bereich 30000–48000 sind bislang
bestätigt, ohne einen plausiblen Kandidaten zu finden.

## Lizenz

MIT
