# Anpassung an die eigene Anlage

Die Steuerung ist auf drei Speicher ausgelegt und referenziert feste Entity-IDs.
Diese Datei listet alles auf, was angepasst werden muss.

## 1. Modbus-Hubs

In `battery_fleet_manager.yaml` erscheinen die Hub-Namen in jedem
`modbus.write_register`-Aufruf sowie in den Auswahllisten:

| Platzhalter | Bedeutung |
|---|---|
| `MarstekVenus` | Hub des ersten Speichers |
| `Marstek2` | Hub des zweiten Speichers |
| `Marstek3` | Hub des dritten Speichers |

Diese Namen müssen den `name:`-Feldern der eigenen `modbus:`-Konfiguration
entsprechen.

## 2. Sensoren je Speicher

| Verwendung | Referenz-Entity | erwartete Einheit |
|---|---|---|
| Ladestand | `sensor.my_battery_battery_soc` | % |
| | `sensor.marstek2_battery_soc` | |
| | `sensor.marstek3_battery_soc` | |
| AC-Leistung | `sensor.my_battery_ac_power` | W, **negativ = lädt** |
| | `sensor.marstek2_ac_power` | |
| | `sensor.marstek3_ac_power` | |
| Ladezyklen | `sensor.my_battery_charge_cycles` | Zahl |
| | `sensor.marstek2_charge_cycles` | |
| | `sensor.marstek3_charge_cycles` | |

**Wichtig zum Vorzeichen der AC-Leistung:** Die Kaskade rechnet mit
*negativ = laden, positiv = entladen*. Meldet die eigene Integration es
andersherum, müssen die Vorzeichen in `lade_jetzt` und `entlade_jetzt` getauscht
werden.

**Zyklen:** Sind keine Zyklensensoren vorhanden, lassen sie sich aus der
kumulierten Ladeenergie berechnen:

```yaml
- name: "Speicher1 Charge Cycles"
  state: "{{ (states('sensor.speicher1_total_charging_energy')|float / 5.12)|round(0) }}"
```

(5.12 = nutzbare Kapazität in kWh, an das eigene Gerät anpassen.)

## 3. Anlagenweite Sensoren

| Verwendung | Referenz-Entity | Anforderung |
|---|---|---|
| Netzzähler | `sensor.leistung_stromzaehler` | W, **positiv = Bezug, negativ = Einspeisung** |
| Speicherinhalt | `sensor.batterysumme_verbleibendekwh` | kWh, Summe über alle Speicher |
| PV-Restprognose | `sensor.pv_rest_heute_noch` | kWh bis Sonnenuntergang |

Das Vorzeichen des Netzzählers ist entscheidend — bei umgekehrter Konvention
kehrt sich die gesamte Logik um. Vor der Inbetriebnahme prüfen.

Fehlt die PV-Restprognose, greift die späte Volladung nicht sicher. Dann entweder
`pv_knapp` fest auf `false` setzen (rein zeitgesteuert) oder eine Prognose
ergänzen, etwa über Forecast.Solar oder Open-Meteo.

## 4. Kapazität

Der Wert `5.12` (kWh je Speicher) erscheint in `fehlt_kwh` und in den
Zyklensensoren. Bei anderen Geräten überall anpassen.

## 5. Anzahl der Speicher

Die Kaskade ist auf **genau drei** ausgelegt. Für eine andere Anzahl müssen
angepasst werden:

- die Variablen `soc1..3`, `cyc1..3`, `cap1..3`, `flr1..3`, `ac1..3`
- die Listen in `lade_hubs` und `entlade_hubs`
- die `for_each`-Schleife am Ende der Automation
- die Obergrenze `3` in `n_lade` und `n_entlade`

Bei mehr als vier Speichern wird die YAML-Variante unübersichtlich — dann lohnt
der Umstieg auf eine eigene Integration in Python.

## 6. Startwerte der Regler

Die `input_number`-Helfer haben bewusst kein `initial:`, damit Werte einen
Neustart überleben. Nach der Erstinstallation sind sie deshalb auf ihrem Minimum
und müssen einmalig gesetzt werden:

| Regler | Vorschlag |
|---|---|
| Ladedeckel je Speicher | nach Zyklenstand staffeln, z. B. 65 / 90 / 100 % |
| Untergrenze je Speicher | 35 / 25 / 25 % — Abstand zur unbekannten Geräte-Abschaltgrenze lassen (siehe [register-map.md](docs/register-map.md)) |
| Absolute Notreserve | 12 % |
| Nachlade-Vorrang unter | 50 % (0 = aus) |
| Geschätzter Nachtverbrauch | **messen** — siehe [messmethodik.md](docs/messmethodik.md) |
| Volladung ab Stunde | 15 |
| Tagesdeckel | 80 % |
| Netzbezugsschwelle (Untergrenzen aufgeben) | 100 W — nicht niedriger, siehe [kaskade.md](docs/kaskade.md#die-50-w-falle) |

## 7. Vor dem Scharfschalten prüfen

1. **Vorzeichen des Netzzählers** — bei Einspeisung negativ?
2. **Vorzeichen der AC-Leistung** — beim Laden negativ?
3. **Registerwirkung** — 44002 testweise auf 0 setzen, prüfen ob das Laden stoppt
4. **Notreserve** — greift sie, bevor der Speicher hardwareseitig abschaltet?

Punkt 3 ist der wichtigste: Wirken 44002/44003 auf dem eigenen Gerät nicht im
Anti-Feed-Modus, funktioniert die gesamte Steuerung nicht.
