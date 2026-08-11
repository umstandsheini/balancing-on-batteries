# Nachtverbrauch richtig messen

Die Nachtreserve entscheidet, wann die Ladedeckel fallen. Ein zu niedriger Wert
bedeutet Netzbezug in der Nacht, ein zu hoher bedeutet unnötig volle Speicher.
Der Wert sollte deshalb gemessen sein, nicht geschätzt.

## Der naheliegende Weg führt in die Irre

Der erste Reflex ist, den Hausverbrauchssensor über die Nachtstunden zu
integrieren. Im Referenzsystem ergab das:

```
00:00      6 W
01:00      5 W
02:00      5 W
...
09:00   4107 W
```

**5 Watt Hauslast nachts** — offensichtlich falsch. Der Sensor stammte vom
PV-Wechselrichter, der nachts in den Standby geht und dann keine sinnvolle
Hauslast mehr meldet. Die Tageskurve folgte exakt dem Sonnenstand statt dem
Verbrauch.

Solche Sensoren heißen oft `load_power` oder ähnlich und wirken plausibel. Der
Test: Wenn der Nachtwert nahe null liegt, misst er nicht den Hausverbrauch.

## Der belastbare Weg

Gemessen wird, was die Nachtreserve tatsächlich decken muss:

```
Nachtbedarf = Entnahme aus den Speichern + Netzbezug
```

Beides über Langzeitstatistiken, die auch einen aggressiven Recorder-Purge
überleben:

1. **Speicherstand** (kWh, `measurement`) — Stundenwert um 21:00 minus Stundenwert
   um 06:00 des Folgetags
2. **Netzbezugszähler** (kWh, `total_increasing`) — Zählerdifferenz über denselben
   Zeitraum

Der Netzbezug muss dazu, sonst unterschätzt man Nächte, in denen die Speicher
leerliefen.

## Auswertung

Über 25 Nächte im Referenzsystem:

| Kennzahl | kWh |
|---|---|
| Minimum | 4,42 |
| Median | 7,18 |
| Mittel | 7,03 |
| 90 %-Wert | 9,07 |
| Maximum | 11,16 |

**Nicht den Mittelwert nehmen.** Er deckt nur die Hälfte der Nächte. Sinnvoll ist
der 90 %-Wert oder ein Kompromiss zwischen ihm und dem, was die Speicher im
Schutzband überhaupt hergeben.

Im Referenzsystem stehen bei 80 % Ladestand und 12 % Entladegrenze
`3 × (0,80 − 0,12) × 5,12 = 10,44 kWh` bereit. Das deckt rund 92 % der Nächte.
Gewählt wurden **10,0 kWh** — knapp darunter, damit der Deckel an normalen Tagen
hält und nur bei echtem Mangel fällt.

## Warum das ganze Nacht-Fenster zählt

Gemessen wurde 21:00 bis 06:00. Die Randstunden gehören dazu: Zwischen 19 und 21
Uhr wird oft gekocht, während die PV schon fast nichts mehr liefert. Wer erst ab
Mitternacht misst, unterschätzt den Bedarf deutlich.

## Wiederholen

Der Verbrauch ändert sich mit der Jahreszeit — Heizung, Beleuchtung, Warmwasser.
Zweimal jährlich nachmessen genügt, oder wenn größere Verbraucher dazukommen.

## Praktischer Hinweis

Kurzzeitdaten (`states`) verschwinden mit dem Purge, Langzeitstatistiken
(`statistics`) bleiben dauerhaft erhalten — sie werden von `purge_keep_days`
**nicht** gelöscht. Für diese Auswertung sind deshalb die Statistiken die richtige
Quelle, abrufbar über `recorder/statistics_during_period` in der Websocket-API.
