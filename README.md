<div align="center">

# BKT MSPDU PDU — Zabbix Template (SNMP)

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Zabbix 7.4](https://img.shields.io/badge/Zabbix-7.4-red.svg)](https://www.zabbix.com/)
[![SNMP](https://img.shields.io/badge/protocol-SNMP%20v2c-blue.svg)](#wymagania)
[![Outlets](https://img.shields.io/badge/outlets-24%20LLD-orange.svg)](#wykrywanie-gniazd-lld)

**Enterprise OID: `1.3.6.1.4.1.47394.7.1`**

### 🌐 Wybierz język / Choose language

**[🇵🇱 POLSKI](#-wersja-polska)**  ·  **[🇬🇧 ENGLISH](#-english-version)**

</div>

---
---

# 🇵🇱 Wersja polska

Szablon Zabbix 7.4 dla listew zasilających (PDU) **BKT MSPDU** — monitoring parametrów fazy,
24 gniazd przez Low-Level Discovery, czujników temperatury i wilgotności. Rozbudowane drzewo
zależności triggerów eliminuje lawiny alarmów przy zaniku zasilania.

## Spis treści

- [Funkcje](#funkcje)
- [Wymagania](#wymagania)
- [Mapa OID](#mapa-oid)
- [Instalacja](#instalacja)
- [Makra](#makra)
- [Metryki](#metryki)
- [Wykrywanie gniazd (LLD)](#wykrywanie-gniazd-lld)
- [Triggery](#triggery)
- [Drzewo zależności](#drzewo-zależności)
- [Przepisy konfiguracyjne](#przepisy-konfiguracyjne)
- [Rozwiązywanie problemów](#rozwiązywanie-problemów)
- [Walidacja pliku](#walidacja-pliku)
- [Najczęstsze pytania](#najczęstsze-pytania)
- [Plan rozwoju](#plan-rozwoju)
- [Współtworzenie](#współtworzenie)
- [Licencja](#licencja)

---

## Funkcje

- ✅ **LLD 24 gniazd** — automatyczne wykrywanie na podstawie nazw gniazd (`walk`)
- ✅ **Dependent items** — jeden `walk` SNMP zasila wszystkie metryki gniazd (minimalne obciążenie PDU)
- ✅ **Moc per gniazdo** — obliczana jako `I × U × PF`, zaokrąglana do pełnych watów
- ✅ **Wykrywanie braku obciążenia** — alarm, gdy gniazdo jest `ON`, ale nie pobiera mocy (awaria zasilacza)
- ✅ **Drzewo zależności triggerów** — brak SNMP wycisza wszystkie pozostałe alarmy
- ✅ **Histereza** — osobne `recovery_expression` eliminuje migotanie (flapping)
- ✅ **Wyciszanie kontekstowe** — per gniazdo, bez edycji triggerów
- ✅ **Detekcja restartu** — parsowanie tekstowego uptime → sekundy
- ✅ **Czujniki opcjonalne** — niepodłączony czujnik nie generuje fałszywych alarmów

---

## Wymagania

| Element | Wymaganie |
|---|---|
| Zabbix Server / Proxy | **7.4** lub nowszy (używa `walk[]` oraz UUID v4) |
| Protokół | SNMP v1 / v2c (v3 jeśli PDU wspiera) |
| Port | UDP/161 |
| Uprawnienia | odczyt (read-only community) |
| Interfejs hosta | **SNMP** (nie Agent!) |

> ⚠️ **Zabbix < 6.0 nie zadziała** — funkcja `walk[OID]` została wprowadzona w 6.0,
> a format eksportu `version: '7.4'` nie zaimportuje się do starszych wersji.

---

## Mapa OID

| Zakres OID | Zawartość |
|---|---|
| `...47394.7.1.1.x.0` | Informacje o urządzeniu (nazwa, typ, MAC, firmware, częstotliwość, uptime, buzzer) |
| `...47394.7.1.2.1.x.0` | Faza 1 — prąd, napięcie, moc, PF, energia |
| `...47394.7.1.4.x.0` | Czujniki — temperatura 1/2, wilgotność 1/2 |
| `...47394.7.1.5` | Nazwy gniazd *(źródło LLD)* |
| `...47394.7.1.7` | Stan przełącznika gniazd |
| `...47394.7.1.8.1` | Prąd gniazd |
| `...47394.7.1.9` | Współczynnik mocy gniazd |
| `...47394.7.1.10` | Energia gniazd |

**Weryfikacja przed importem:**

```bash
snmpwalk -v2c -c public 10.0.0.10 1.3.6.1.4.1.47394.7.1
```

---

## Instalacja

### 1. Import szablonu

```
Data collection → Templates → Import → wybierz plik .yaml → Import
```

Utworzy się grupa szablonów **`Templates/BKT_Mentel`** oraz szablon
**`BKT MSPDU PDU by SNMP`**.

### 2. Konfiguracja hosta

```
Data collection → Hosts → Create host
```

| Pole | Wartość |
|---|---|
| Host name | np. `pdu-rack-a01` |
| Templates | `BKT MSPDU PDU by SNMP` |
| Interfaces | **SNMP** → IP listwy, port `161` |
| SNMP version | SNMPv2 |
| SNMP community | `{$SNMP_COMMUNITY}` |

### 3. Ustaw community

W zakładce **Macros** hosta:

```
{$SNMP_COMMUNITY} = public
```

### 4. Weryfikacja

```
Monitoring → Latest data → filtr po hoście
```

Po ~1 minucie powinny pojawić się dane fazy, po ~1 godzinie (lub po
`Execute now` na itemie `MSPDU: RAW walk - outlet names`) — gniazda z LLD.

> 💡 **Przyspieszenie LLD:** `Latest data` → zaznacz `MSPDU: RAW walk - outlet names`
> → `Execute now`, następnie to samo dla reguły discovery.

---

## Makra

### Dostępność i restart

| Makro | Domyślnie | Opis |
|---|---|---|
| `{$MSPDU.NODATA}` | `10m` | Czas braku danych → alarm o nieosiągalności |
| `{$MSPDU.UPTIME.RESTART}` | `600` | Uptime poniżej tej wartości (s) po spadku licznika = restart |

### Napięcie

| Makro | Domyślnie | Opis |
|---|---|---|
| `{$MSPDU.VOLTAGE.MIN.WARN}` | `207` | Dolny próg ostrzegawczy (230 V −10%) |
| `{$MSPDU.VOLTAGE.MIN.CRIT}` | `195` | Dolny próg krytyczny |
| `{$MSPDU.VOLTAGE.MAX.WARN}` | `253` | Górny próg ostrzegawczy (230 V +10%) |
| `{$MSPDU.VOLTAGE.MAX.CRIT}` | `260` | Górny próg krytyczny |

### Częstotliwość i PF fazy

| Makro | Domyślnie | Opis |
|---|---|---|
| `{$MSPDU.FREQ.MIN}` | `49` | Minimalna częstotliwość (Hz) |
| `{$MSPDU.FREQ.MAX}` | `51` | Maksymalna częstotliwość (Hz) |
| `{$MSPDU.PF.MIN}` | `0.6` | Minimalny PF fazy |
| `{$MSPDU.PF.CURRENT.MIN}` | `1` | Poniżej tego prądu (A) PF jest ignorowany |

### Obciążenie fazy

| Makro | Domyślnie | Opis |
|---|---|---|
| `{$MSPDU.PHASE.CURRENT.MAX}` | `16` | Prąd znamionowy wejścia (A). **80% = WARNING, 100% = HIGH** |

### Gniazda *(obsługują kontekst)*

| Makro | Domyślnie | Opis |
|---|---|---|
| `{$MSPDU.OUTLET.STATE.ON}` | `ON` | Wartość oznaczająca gniazdo załączone (item normalizuje do WIELKICH liter) |
| `{$MSPDU.OUTLET.OFF.ALERT}` | `1` | `0` = nie alarmuj o wyłączonym gnieździe |
| `{$MSPDU.OUTLET.CURRENT.MAX}` | `16` | Maks. prąd gniazda (A). **`0` = triggery prądowe wyłączone** |
| `{$MSPDU.OUTLET.POWER.MAX}` | `3000` | Maks. moc gniazda (W). `0` = wyłączone |
| `{$MSPDU.OUTLET.POWER.MIN}` | `0` | Min. oczekiwana moc (W). **`0` = detekcja braku obciążenia WYŁĄCZONA** |
| `{$MSPDU.OUTLET.NOLOAD.PERIOD}` | `15m` | Okno oceny braku obciążenia |
| `{$MSPDU.OUTLET.PF.MIN}` | `0.6` | Minimalny PF gniazda |
| `{$MSPDU.OUTLET.PF.POWER.MIN}` | `50` | Poniżej tej mocy (W) PF gniazda jest ignorowany |

### Czujniki środowiskowe

| Makro | Domyślnie | Opis |
|---|---|---|
| `{$MSPDU.OUTLET.TEMP.MAX}` | `35` | Maks. temperatura (°C) |
| `{$MSPDU.OUTLET.TEMP.MIN}` | `5` | Min. temperatura (°C) |
| `{$MSPDU.OUTLET.HUM.MAX}` | `80` | Maks. wilgotność (%) |
| `{$MSPDU.HUM.MIN}` | `20` | Min. wilgotność (%) |
| `{$MSPDU.SENSOR1.ENABLED}` | `1` | Czy czujnik 1 jest podłączony (0/1) |
| `{$MSPDU.SENSOR2.ENABLED}` | `0` | Czy czujnik 2 jest podłączony (0/1) |

### Priorytet makr

```
Makro hosta  >  Makro szablonu  >  Makro globalne
```

**Nie edytuj makr w szablonie** — nadpisz je na poziomie hosta. Dzięki temu
re-import nowej wersji szablonu nie skasuje Twoich ustawień.

### Makra kontekstowe

Makra gniazd obsługują **kontekst numeryczny** = wartość `{#OUTLET_INDEX}`:

```
{$MSPDU.OUTLET.POWER.MIN:"3"}    = 30      ← gniazdo nr 3
{$MSPDU.OUTLET.OFF.ALERT:"17"}   = 0       ← gniazdo nr 17
```

> ⚠️ Kontekst musi być **numerem gniazda**, nie jego nazwą. Cudzysłowy obowiązkowe.

---

## Metryki

### Urządzenie

| Nazwa | Klucz | Interwał |
|---|---|---|
| Device name | `mspdu.device.name` | 15m |
| PDU type | `mspdu.device.type` | 15m |
| Number of output ports | `mspdu.device.output.number` | 1h |
| MAC address | `mspdu.device.mac` | 1h |
| Firmware version | `mspdu.device.firmware` | 1h |
| Frequency | `mspdu.device.frequency` | 5m |
| Device uptime | `mspdu.device.uptime` | 5m |
| Uptime (seconds) | `mspdu.device.uptime.sec` | *dependent* |
| Buzzer switch | `mspdu.device.buzzer` | 15m |

### Faza 1

| Nazwa | Klucz | Jedn. | Interwał |
|---|---|---|---|
| Phase 1 — Current | `mspdu.phase1.current` | A | 1m |
| Phase 1 — Voltage | `mspdu.phase1.voltage` | V | 1m |
| Phase 1 — Power | `mspdu.phase1.power` | W | 1m |
| Phase 1 — Power factor | `mspdu.phase1.pf` | — | 1m |
| Phase 1 — Energy | `mspdu.phase1.energy` | kWh | 5m |

> `mspdu.phase1.current` pełni rolę **wskaźnika dostępności** — to na nim
> oparty jest trigger `nodata()`, od którego zależą wszystkie pozostałe.

### Czujniki

| Nazwa | Klucz | Jedn. |
|---|---|---|
| Sensor 1 — Temperature | `mspdu.sensor.temp1` | °C |
| Sensor 2 — Temperature | `mspdu.sensor.temp2` | °C |
| Sensor 1 — Humidity | `mspdu.sensor.hum1` | % |
| Sensor 2 — Humidity | `mspdu.sensor.hum2` | % |

### RAW walk *(master items, `history: 0`)*

| Klucz | OID | Interwał |
|---|---|---|
| `mspdu.walk.outlet.names.raw` | `...7.1.5` | 1h |
| `mspdu.walk.outlet.switch.raw` | `...7.1.7` | 1m |
| `mspdu.walk.outlet.current.raw` | `...7.1.8.1` | 1m |
| `mspdu.walk.outlet.pf.raw` | `...7.1.9` | 1m |
| `mspdu.walk.outlet.energy.raw` | `...7.1.10` | 5m |

---

## Wykrywanie gniazd (LLD)

**Reguła:** `MSPDU: Outlet discovery` (`mspdu.outlet.discovery`)
**Master item:** `mspdu.walk.outlet.names.raw`

### Makra LLD

| Makro | Przykład |
|---|---|
| `{#OUTLET_INDEX}` | `1`, `2`, … `24` |
| `{#OUTLET_NAME}` | `SRV-DB-01`, `Switch-Core` |

### Filtrowanie

Skrypt JS pomija gniazda o nazwie `--` lub pustej — czyli **nieaktywne
sloty** listwy nie zaśmiecają monitoringu. Aby gniazdo pojawiło się w Zabbiksie,
nadaj mu nazwę w interfejsie WWW listwy.

### Prototypy itemów

| Nazwa | Klucz | Typ |
|---|---|---|
| Name | `mspdu.outlet.name[{#OUTLET_INDEX}]` | dependent |
| Switch state | `mspdu.outlet.switch[{#OUTLET_INDEX}]` | dependent |
| Current | `mspdu.outlet.current[{#OUTLET_INDEX}]` | dependent |
| Power factor | `mspdu.outlet.pf[{#OUTLET_INDEX}]` | dependent |
| **Power** | `mspdu.outlet.power.w[{#OUTLET_INDEX}]` | **calculated** |
| Energy | `mspdu.outlet.energy[{#OUTLET_INDEX}]` | dependent |

> 📐 **Moc gniazda jest wyliczana**, nie odczytywana z SNMP:
> `last(current) × last(phase1.voltage) × last(pf)`, zaokrąglona do 1 W.
> Jeśli listwa nie mierzy prądu per gniazdo — wynik zawsze wyniesie **0 W**.

---

## Triggery

### Dostępność

| Trigger | Waga |
|---|---|
| No SNMP data — PDU unreachable | 🔴 **HIGH** |
| PDU has been restarted | 🟡 WARNING |

### Zasilanie

| Trigger | Waga | Warunek |
|---|---|---|
| Voltage critically low | 🔴 HIGH | `< 195 V` przez 5m |
| Voltage critically high | 🔴 HIGH | `> 260 V` przez 3m |
| Voltage below normal | 🟡 WARNING | `< 207 V` |
| Voltage above normal | 🟡 WARNING | `> 253 V` |
| Frequency below normal | 🟡 WARNING | `< 49 Hz` **i `> 0`** |
| Frequency above normal | 🟡 WARNING | `> 51 Hz` |
| Phase 1 — low power factor | 🔵 INFO | PF `< 0.6` **przy prądzie `> 1 A`** |

### Obciążenie

| Trigger | Waga | Warunek |
|---|---|---|
| Phase 1 — OVERLOAD | 🔴 HIGH | `> 16 A` przez 3m |
| Phase 1 — current above 80% | 🟡 WARNING | `> 12.8 A` przez 5m |
| Outlet — current OVERLOAD | 🔴 HIGH | `> {$MSPDU.OUTLET.CURRENT.MAX}` |
| Outlet — current > 80% | 🟡 WARNING | — |
| Outlet — power above limit | 🟡 WARNING | `> {$MSPDU.OUTLET.POWER.MAX}` |

### Gniazda

| Trigger | Waga |
|---|---|
| Outlet — SWITCHED OFF | 🟠 AVERAGE |
| Outlet — no load while switched on | 🟡 WARNING |
| Outlet — switch state changed | 🔵 INFO |
| Outlet — energy counter has been reset | 🔵 INFO |
| Outlet — low PF | 🔵 INFO |

### Środowisko

| Trigger | Waga |
|---|---|
| Sensor 1/2 — temperature too high | 🟠 AVERAGE |
| Sensor 1/2 — humidity too high | 🟠 AVERAGE |
| Sensor 1/2 — temperature too low | 🟡 WARNING |
| Sensor 1/2 — humidity too low | 🟡 WARNING |

### Zmiany konfiguracji

| Trigger | Waga |
|---|---|
| Firmware version has changed | 🔵 INFO |
| Device name has changed | 🔵 INFO |

---

## Drzewo zależności

```
No SNMP data — PDU unreachable                      [HIGH]
│
├── Phase 1 — OVERLOAD                              [HIGH]
│   └── Phase 1 — current above 80%                 [WARNING]
│
├── Voltage critically low                          [HIGH]
│   └── Voltage below normal                        [WARNING]
│
├── Voltage critically high                         [HIGH]
│   └── Voltage above normal                        [WARNING]
│
├── Frequency below / above normal                  [WARNING]
├── Phase 1 — low power factor                      [INFO]
│
├── Outlet {N} — SWITCHED OFF                       [AVERAGE]
│   └── Outlet {N} — no load while switched on      [WARNING]
│
├── Outlet {N} — current OVERLOAD                   [HIGH]
│   └── Outlet {N} — current > 80% of limit         [WARNING]
│
├── Outlet {N} — power above limit                  [WARNING]
└── Outlet {N} — low PF                             [INFO]
```

**Efekt:** zanik zasilania całej listwy generuje **1 alarm**, a nie 100+.

---

## Przepisy konfiguracyjne

### 🔧 Listwa nie mierzy prądu per gniazdo (stale 0 A)

Na hoście:

```
{$MSPDU.OUTLET.CURRENT.MAX} = 0
{$MSPDU.OUTLET.POWER.MAX}   = 0
```

Wycisza wszystkie triggery prądowe i mocowe gniazd. Pozostaje monitoring
stanu przełącznika, fazy i czujników.

### 🔧 Wyciszenie konkretnego gniazda (celowo wyłączone)

```
{$MSPDU.OUTLET.OFF.ALERT:"7"} = 0
```

Gniazdo 7 może być wyłączone bez generowania alarmu. Pozostałe gniazda
działają normalnie.

### 🔧 Alarm o awarii zasilacza serwera

Serwer w gnieździe 3 pobiera normalnie ~120 W. Chcemy alarm, gdy spadnie poniżej 20 W
mimo stanu `ON`:

```
{$MSPDU.OUTLET.POWER.MIN:"3"}     = 20
{$MSPDU.OUTLET.NOLOAD.PERIOD:"3"} = 10m
```

> ⚠️ Działa tylko, jeśli listwa raportuje prąd per gniazdo. Przy stałym 0 A
> **nie włączaj** tego makra — wygenerujesz fałszywe alarmy.

### 🔧 Listwa 32 A

```
{$MSPDU.PHASE.CURRENT.MAX} = 32
```

Progi 80% (25.6 A → WARNING) i 100% (32 A → HIGH) przeliczą się automatycznie.

### 🔧 Drugi czujnik podłączony

```
{$MSPDU.SENSOR2.ENABLED} = 1
```

### 🔧 Serwerownia z restrykcyjną temperaturą

```
{$MSPDU.OUTLET.TEMP.MAX} = 27
{$MSPDU.OUTLET.TEMP.MIN} = 18
{$MSPDU.OUTLET.HUM.MAX}  = 60
{$MSPDU.HUM.MIN}         = 40
```

### 🔧 Zasilanie z agregatu (niestabilna częstotliwość)

```
{$MSPDU.FREQ.MIN} = 47
{$MSPDU.FREQ.MAX} = 53
```

---

## Rozwiązywanie problemów

<details>
<summary><b>LLD nie wykrywa gniazd</b></summary>

1. Sprawdź item `MSPDU: RAW walk - outlet names` → `Latest data` → czy zwraca dane
2. Zweryfikuj ręcznie:
   ```bash
   snmpwalk -v2c -c public 10.0.0.10 1.3.6.1.4.1.47394.7.1.5
   ```
3. Format musi odpowiadać: `.5.<index>.0 = STRING: "nazwa"`
4. Gniazda o nazwie `--` lub pustej są **celowo pomijane** — nadaj im nazwę w WWW listwy
5. Wymuś: `Execute now` na master itemie, potem na regule discovery
</details>

<details>
<summary><b>Moc gniazda zawsze 0 W</b></summary>

Moc jest **obliczana** (`I × U × PF`). Jeśli którykolwiek składnik = 0, wynik = 0.

```bash
snmpwalk -v2c -c public 10.0.0.10 1.3.6.1.4.1.47394.7.1.8.1
```

Jeśli wszystkie wartości to `0` lub `--` → sprzęt nie wspiera pomiaru per gniazdo.
Zastosuj przepis [„Listwa nie mierzy prądu per gniazdo"](#-listwa-nie-mierzy-prądu-per-gniazdo-stale-0-a).
</details>

<details>
<summary><b>Fałszywe alarmy „SWITCHED OFF"</b></summary>

Listwa może zwracać `1`/`0` zamiast `ON`/`OFF`. Sprawdź:

```bash
snmpget -v2c -c public 10.0.0.10 1.3.6.1.4.1.47394.7.1.7.2.0
```

Jeśli zwraca `1`, ustaw na hoście:

```
{$MSPDU.OUTLET.STATE.ON} = 1
```

Item normalizuje wartość do WIELKICH liter, więc `on`, `On`, `ON` → `ON`.
</details>

<details>
<summary><b>Alarmy o temperaturze przy braku czujnika</b></summary>

Niepodłączony czujnik zwraca `0°C`, co wyzwala trigger „temperature too low".

```
{$MSPDU.SENSOR2.ENABLED} = 0
```
</details>

<details>
<summary><b>Błąd importu: „Invalid parameter /uuid"</b></summary>

Zabbix 7.4 wymaga poprawnego **UUID v4**:
- 32 znaki hex, bez myślników
- 13. znak = `4`
- 17. znak ∈ `{8, 9, a, b}`

Uruchom [skrypt walidacyjny](#walidacja-pliku).
</details>

<details>
<summary><b>Alarm „low power factor" bez obciążenia</b></summary>

Nie powinien wystąpić — triggery PF są aktywne dopiero powyżej progu obciążenia.
Jeśli występuje, podnieś próg:

```
{$MSPDU.PF.CURRENT.MIN}      = 2      # faza
{$MSPDU.OUTLET.PF.POWER.MIN} = 100    # gniazdo
```
</details>

---

## Walidacja pliku

### Składnia YAML

```bash
#!/usr/bin/env bash
# check-yaml.sh
FILE="${1:-template_bkt_mspdu.yaml}"

python3 -c "
import sys, yaml
try:
    yaml.safe_load(open('$FILE', encoding='utf-8'))
    print('✅ YAML OK')
except yaml.YAMLError as e:
    print('❌ YAML ERROR:', e); sys.exit(1)
"
```

### Zgodność UUID v4

```bash
#!/usr/bin/env bash
# check-uuid.sh
FILE="${1:-template_bkt_mspdu.yaml}"
ERR=0

grep -oP '(?<=uuid: )[0-9a-f]{32}' "$FILE" | while read -r u; do
    [[ "${u:12:1}" != "4" ]] && { echo "❌ $u — 13. znak != 4"; ERR=1; }
    [[ ! "${u:16:1}" =~ [89ab] ]] && { echo "❌ $u — 17. znak != 8/9/a/b"; ERR=1; }
done

DUP=$(grep -oP '(?<=uuid: )[0-9a-f]{32}' "$FILE" | sort | uniq -d)
[[ -n "$DUP" ]] && { echo "❌ Duplikaty UUID:"; echo "$DUP"; ERR=1; }

[[ $ERR -eq 0 ]] && echo "✅ UUID OK"
```

### Uruchomienie

```bash
chmod +x check-yaml.sh check-uuid.sh
./check-yaml.sh template_bkt_mspdu.yaml
./check-uuid.sh template_bkt_mspdu.yaml
```

### Generowanie nowego UUID v4

```bash
python3 -c "import uuid; print(uuid.uuid4().hex)"
```

---

## Najczęstsze pytania

**Czy szablon obsługuje listwy 3-fazowe?**
Obecnie nie — zaimplementowana jest faza 1. Rozszerzenie wymaga dodania itemów
dla OID `...7.1.2.2.x.0` i `...7.1.2.3.x.0`. Zobacz [Plan rozwoju](#plan-rozwoju).

**Czy mogę sterować gniazdami z Zabbiksa?**
Nie. Szablon jest **read-only**. Sterowanie wymaga SNMP SET (`snmpset`) przez
skrypt zewnętrzny lub Zabbix Script item — świadomie pominięte ze względów bezpieczeństwa.

**Dlaczego techniczna nazwa szablonu jest po polsku?**
`template: 'SNMP Listwa MSPDU BKT'` występuje w każdym wyrażeniu triggera.
Widoczna nazwa (`name:`) jest po angielsku. Zmiana nazwy technicznej wymaga
podmiany w ~60 miejscach — planowana w wersji 2.0.

**Czy szablon zwiększa obciążenie listwy?**
Minimalnie. Zamiast ~150 zapytań SNMP wykonywanych jest **5 operacji walk**,
z których dependent items wyciągają wszystkie metryki.

**Jak zaktualizować szablon bez utraty ustawień?**
Ustawiaj makra **na poziomie hosta**, nigdy w szablonie. Przy re-imporcie
zaznacz `Update existing` — makra hosta pozostaną nietknięte.

**Czy działa z Zabbix 7.0?**
Format eksportu `version: '7.4'` nie zaimportuje się do 7.0. Możesz spróbować
zmienić wartość na `7.0` — funkcje (`walk[]`, dependent items, calculated) są dostępne.
Nie testowano.

---

## Plan rozwoju

- [ ] Wsparcie faz 2 i 3 (PDU 3-fazowe)
- [ ] Dashboard / widgety
- [ ] Value mapping dla stanu przełącznika
- [ ] Angielska nazwa techniczna szablonu (v2.0, breaking change)
- [ ] Wykresy prototypowe per gniazdo
- [ ] Wsparcie SNMPv3 w dokumentacji
- [ ] Zgłoszenie do `zabbix/community-templates`

---

## Współtworzenie

Pull requesty mile widziane. Przed zgłoszeniem:

1. Uruchom oba skrypty walidacyjne — muszą przejść
2. **Nowe UUID generuj**, nie kopiuj istniejących
3. Nazwy triggerów w `dependencies → name` muszą **dokładnie** odpowiadać nazwom triggerów
4. Testuj na realnym sprzęcie, w PR podaj model i wersję firmware
5. Zachowaj konwencję nazewnictwa `MSPDU: <obszar> - <metryka>`

Zgłaszając błąd, dołącz:

```bash
snmpwalk -v2c -c public <IP> 1.3.6.1.4.1.47394.7.1 > walk.txt
```

*(zanonimizuj nazwy gniazd, jeśli zawierają dane wrażliwe)*

---

## Licencja

Projekt udostępniony **bezpłatnie** na licencji [MIT](LICENSE).
Możesz go używać, kopiować, modyfikować i wdrażać — również **komercyjnie** —
pod warunkiem zachowania informacji o prawach autorskich i treści licencji.
Oprogramowanie dostarczane jest „**tak jak jest**", bez jakichkolwiek gwarancji.

```
MIT License · Copyright (c) 2026 Sebastian Mentel
```

### Zastrzeżenie

Szablon **nieoficjalny**, nie jest powiązany z producentem urządzenia
ani przez niego wspierany. Nazwy **BKT**, **MSPDU** oraz **Zabbix** są znakami
towarowymi ich właścicieli i zostały użyte wyłącznie w celach identyfikacyjnych.
Autor nie ponosi odpowiedzialności za skutki działania szablonu w środowisku
produkcyjnym, w tym za nietrafione alarmy lub ich brak.
**Przetestuj przed wdrożeniem produkcyjnym.**

### Podziękowania

Szablon powstał na bazie analizy rzeczywistego `snmpwalk` z urządzenia
BKT MSPDU (24 gniazda, 1 faza). Jeśli używasz go na innym modelu i działa —
[daj znać w Issues](../../issues), dopiszemy do listy zgodności.

<div align="center">

**[⬆ Powrót na górę](#bkt-mspdu-pdu--zabbix-template-snmp)**  ·  **[🇬🇧 Read in English](#-english-version)**

</div>

---
---

# 🇬🇧 English version

Zabbix 7.4 template for **BKT MSPDU** rack power distribution units — phase metrics,
24 outlets via Low-Level Discovery, temperature and humidity sensors. A complete trigger
dependency tree prevents alert storms during power loss.

## Table of contents

- [Features](#features)
- [Requirements](#requirements)
- [OID map](#oid-map)
- [Installation](#installation)
- [Macros](#macros)
- [Items](#items)
- [Discovery (LLD)](#discovery-lld)
- [Triggers](#triggers)
- [Dependency tree](#dependency-tree)
- [Configuration recipes](#configuration-recipes)
- [Troubleshooting](#troubleshooting)
- [Validation](#validation)
- [FAQ](#faq)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Features

- ✅ **24-outlet LLD** — automatic discovery from the SNMP outlet-name `walk`
- ✅ **Dependent items** — one SNMP walk feeds every outlet metric (minimal PDU load)
- ✅ **Per-outlet power** — calculated as `I × U × PF`, rounded to whole watts
- ✅ **No-load detection** — alerts when an outlet is `ON` but draws no power (PSU failure)
- ✅ **Trigger dependency tree** — SNMP loss silences every other alert
- ✅ **Hysteresis** — dedicated `recovery_expression` eliminates flapping
- ✅ **Context-macro muting** — per outlet, no trigger editing required
- ✅ **Reboot detection** — textual uptime parsed into seconds
- ✅ **Optional sensors** — a disconnected sensor produces no false alarms

---

## Requirements

| Item | Requirement |
|---|---|
| Zabbix Server / Proxy | **7.4** or newer (uses `walk[]` and UUID v4) |
| Protocol | SNMP v1 / v2c (v3 if the PDU supports it) |
| Port | UDP/161 |
| Privileges | read-only community |
| Host interface | **SNMP** (not Agent!) |

> ⚠️ **Zabbix < 6.0 will not work** — the `walk[OID]` function was introduced in 6.0,
> and the `version: '7.4'` export format cannot be imported into older releases.

---

## OID map

| OID range | Contents |
|---|---|
| `...47394.7.1.1.x.0` | Device info (name, type, MAC, firmware, frequency, uptime, buzzer) |
| `...47394.7.1.2.1.x.0` | Phase 1 — current, voltage, power, PF, energy |
| `...47394.7.1.4.x.0` | Sensors — temperature 1/2, humidity 1/2 |
| `...47394.7.1.5` | Outlet names *(LLD source)* |
| `...47394.7.1.7` | Outlet switch state |
| `...47394.7.1.8.1` | Outlet current |
| `...47394.7.1.9` | Outlet power factor |
| `...47394.7.1.10` | Outlet energy |

**Verify before importing:**

```bash
snmpwalk -v2c -c public 10.0.0.10 1.3.6.1.4.1.47394.7.1
```

---

## Installation

### 1. Import the template

```
Data collection → Templates → Import → select the .yaml file → Import
```

This creates the template group **`Templates/BKT_Mentel`** and the template
**`BKT MSPDU PDU by SNMP`**.

### 2. Configure the host

```
Data collection → Hosts → Create host
```

| Field | Value |
|---|---|
| Host name | e.g. `pdu-rack-a01` |
| Templates | `BKT MSPDU PDU by SNMP` |
| Interfaces | **SNMP** → PDU IP address, port `161` |
| SNMP version | SNMPv2 |
| SNMP community | `{$SNMP_COMMUNITY}` |

### 3. Set the community string

In the host's **Macros** tab:

```
{$SNMP_COMMUNITY} = public
```

### 4. Verify

```
Monitoring → Latest data → filter by host
```

Phase data should appear within ~1 minute; outlets appear after ~1 hour
(or immediately after `Execute now` on `MSPDU: RAW walk - outlet names`).

> 💡 **Speed up LLD:** `Latest data` → select `MSPDU: RAW walk - outlet names`
> → `Execute now`, then do the same for the discovery rule.

---

## Macros

### Availability and restart

| Macro | Default | Description |
|---|---|---|
| `{$MSPDU.NODATA}` | `10m` | No-data period before the unreachable alert fires |
| `{$MSPDU.UPTIME.RESTART}` | `600` | Uptime below this value (s) after a counter drop = restart |

### Voltage

| Macro | Default | Description |
|---|---|---|
| `{$MSPDU.VOLTAGE.MIN.WARN}` | `207` | Low warning threshold (230 V −10%) |
| `{$MSPDU.VOLTAGE.MIN.CRIT}` | `195` | Low critical threshold |
| `{$MSPDU.VOLTAGE.MAX.WARN}` | `253` | High warning threshold (230 V +10%) |
| `{$MSPDU.VOLTAGE.MAX.CRIT}` | `260` | High critical threshold |

### Frequency and phase PF

| Macro | Default | Description |
|---|---|---|
| `{$MSPDU.FREQ.MIN}` | `49` | Minimum frequency (Hz) |
| `{$MSPDU.FREQ.MAX}` | `51` | Maximum frequency (Hz) |
| `{$MSPDU.PF.MIN}` | `0.6` | Minimum phase power factor |
| `{$MSPDU.PF.CURRENT.MIN}` | `1` | Below this current (A) PF is ignored |

### Phase load

| Macro | Default | Description |
|---|---|---|
| `{$MSPDU.PHASE.CURRENT.MAX}` | `16` | Rated input current (A). **80% = WARNING, 100% = HIGH** |

### Outlets *(context-aware)*

| Macro | Default | Description |
|---|---|---|
| `{$MSPDU.OUTLET.STATE.ON}` | `ON` | Value meaning "outlet energised" (item normalises to UPPERCASE) |
| `{$MSPDU.OUTLET.OFF.ALERT}` | `1` | `0` = do not alert on a switched-off outlet |
| `{$MSPDU.OUTLET.CURRENT.MAX}` | `16` | Max outlet current (A). **`0` = current triggers disabled** |
| `{$MSPDU.OUTLET.POWER.MAX}` | `3000` | Max outlet power (W). `0` = disabled |
| `{$MSPDU.OUTLET.POWER.MIN}` | `0` | Min expected power (W). **`0` = no-load detection DISABLED** |
| `{$MSPDU.OUTLET.NOLOAD.PERIOD}` | `15m` | No-load evaluation window |
| `{$MSPDU.OUTLET.PF.MIN}` | `0.6` | Minimum outlet power factor |
| `{$MSPDU.OUTLET.PF.POWER.MIN}` | `50` | Below this power (W) outlet PF is ignored |

### Environmental sensors

| Macro | Default | Description |
|---|---|---|
| `{$MSPDU.OUTLET.TEMP.MAX}` | `35` | Max temperature (°C) |
| `{$MSPDU.OUTLET.TEMP.MIN}` | `5` | Min temperature (°C) |
| `{$MSPDU.OUTLET.HUM.MAX}` | `80` | Max humidity (%) |
| `{$MSPDU.HUM.MIN}` | `20` | Min humidity (%) |
| `{$MSPDU.SENSOR1.ENABLED}` | `1` | Sensor 1 connected (0/1) |
| `{$MSPDU.SENSOR2.ENABLED}` | `0` | Sensor 2 connected (0/1) |

### Macro priority

```
Host macro  >  Template macro  >  Global macro
```

**Never edit macros inside the template** — override them at host level.
That way re-importing a new template version will not wipe your settings.

### Context macros

Outlet macros accept a **numeric context** equal to `{#OUTLET_INDEX}`:

```
{$MSPDU.OUTLET.POWER.MIN:"3"}    = 30      ← outlet #3
{$MSPDU.OUTLET.OFF.ALERT:"17"}   = 0       ← outlet #17
```

> ⚠️ The context must be the **outlet number**, not its name. Quotes are mandatory.

---

## Items

### Device

| Name | Key | Interval |
|---|---|---|
| Device name | `mspdu.device.name` | 15m |
| PDU type | `mspdu.device.type` | 15m |
| Number of output ports | `mspdu.device.output.number` | 1h |
| MAC address | `mspdu.device.mac` | 1h |
| Firmware version | `mspdu.device.firmware` | 1h |
| Frequency | `mspdu.device.frequency` | 5m |
| Device uptime | `mspdu.device.uptime` | 5m |
| Uptime (seconds) | `mspdu.device.uptime.sec` | *dependent* |
| Buzzer switch | `mspdu.device.buzzer` | 15m |

### Phase 1

| Name | Key | Unit | Interval |
|---|---|---|---|
| Phase 1 — Current | `mspdu.phase1.current` | A | 1m |
| Phase 1 — Voltage | `mspdu.phase1.voltage` | V | 1m |
| Phase 1 — Power | `mspdu.phase1.power` | W | 1m |
| Phase 1 — Power factor | `mspdu.phase1.pf` | — | 1m |
| Phase 1 — Energy | `mspdu.phase1.energy` | kWh | 5m |

> `mspdu.phase1.current` acts as the **availability indicator** — the `nodata()`
> trigger is built on it, and every other trigger depends on that one.

### Sensors

| Name | Key | Unit |
|---|---|---|
| Sensor 1 — Temperature | `mspdu.sensor.temp1` | °C |
| Sensor 2 — Temperature | `mspdu.sensor.temp2` | °C |
| Sensor 1 — Humidity | `mspdu.sensor.hum1` | % |
| Sensor 2 — Humidity | `mspdu.sensor.hum2` | % |

### RAW walks *(master items, `history: 0`)*

| Key | OID | Interval |
|---|---|---|
| `mspdu.walk.outlet.names.raw` | `...7.1.5` | 1h |
| `mspdu.walk.outlet.switch.raw` | `...7.1.7` | 1m |
| `mspdu.walk.outlet.current.raw` | `...7.1.8.1` | 1m |
| `mspdu.walk.outlet.pf.raw` | `...7.1.9` | 1m |
| `mspdu.walk.outlet.energy.raw` | `...7.1.10` | 5m |

---

## Discovery (LLD)

**Rule:** `MSPDU: Outlet discovery` (`mspdu.outlet.discovery`)
**Master item:** `mspdu.walk.outlet.names.raw`

### LLD macros

| Macro | Example |
|---|---|
| `{#OUTLET_INDEX}` | `1`, `2`, … `24` |
| `{#OUTLET_NAME}` | `SRV-DB-01`, `Switch-Core` |

### Filtering

The JS preprocessing script skips outlets named `--` or empty, so **unused slots**
never clutter your monitoring. To make an outlet appear in Zabbix, give it a name
in the PDU's web interface.

### Item prototypes

| Name | Key | Type |
|---|---|---|
| Name | `mspdu.outlet.name[{#OUTLET_INDEX}]` | dependent |
| Switch state | `mspdu.outlet.switch[{#OUTLET_INDEX}]` | dependent |
| Current | `mspdu.outlet.current[{#OUTLET_INDEX}]` | dependent |
| Power factor | `mspdu.outlet.pf[{#OUTLET_INDEX}]` | dependent |
| **Power** | `mspdu.outlet.power.w[{#OUTLET_INDEX}]` | **calculated** |
| Energy | `mspdu.outlet.energy[{#OUTLET_INDEX}]` | dependent |

> 📐 **Outlet power is calculated**, not read from SNMP:
> `last(current) × last(phase1.voltage) × last(pf)`, rounded to 1 W.
> If the PDU does not meter current per outlet, the result is always **0 W**.

---

## Triggers

### Availability

| Trigger | Severity |
|---|---|
| No SNMP data — PDU unreachable | 🔴 **HIGH** |
| PDU has been restarted | 🟡 WARNING |

### Power quality

| Trigger | Severity | Condition |
|---|---|---|
| Voltage critically low | 🔴 HIGH | `< 195 V` for 5m |
| Voltage critically high | 🔴 HIGH | `> 260 V` for 3m |
| Voltage below normal | 🟡 WARNING | `< 207 V` |
| Voltage above normal | 🟡 WARNING | `> 253 V` |
| Frequency below normal | 🟡 WARNING | `< 49 Hz` **and `> 0`** |
| Frequency above normal | 🟡 WARNING | `> 51 Hz` |
| Phase 1 — low power factor | 🔵 INFO | PF `< 0.6` **while current `> 1 A`** |

### Load

| Trigger | Severity | Condition |
|---|---|---|
| Phase 1 — OVERLOAD | 🔴 HIGH | `> 16 A` for 3m |
| Phase 1 — current above 80% | 🟡 WARNING | `> 12.8 A` for 5m |
| Outlet — current OVERLOAD | 🔴 HIGH | `> {$MSPDU.OUTLET.CURRENT.MAX}` |
| Outlet — current > 80% | 🟡 WARNING | — |
| Outlet — power above limit | 🟡 WARNING | `> {$MSPDU.OUTLET.POWER.MAX}` |

### Outlets

| Trigger | Severity |
|---|---|
| Outlet — SWITCHED OFF | 🟠 AVERAGE |
| Outlet — no load while switched on | 🟡 WARNING |
| Outlet — switch state changed | 🔵 INFO |
| Outlet — energy counter has been reset | 🔵 INFO |
| Outlet — low PF | 🔵 INFO |

### Environment

| Trigger | Severity |
|---|---|
| Sensor 1/2 — temperature too high | 🟠 AVERAGE |
| Sensor 1/2 — humidity too high | 🟠 AVERAGE |
| Sensor 1/2 — temperature too low | 🟡 WARNING |
| Sensor 1/2 — humidity too low | 🟡 WARNING |

### Configuration changes

| Trigger | Severity |
|---|---|
| Firmware version has changed | 🔵 INFO |
| Device name has changed | 🔵 INFO |

---

## Dependency tree

```
No SNMP data — PDU unreachable                      [HIGH]
│
├── Phase 1 — OVERLOAD                              [HIGH]
│   └── Phase 1 — current above 80%                 [WARNING]
│
├── Voltage critically low                          [HIGH]
│   └── Voltage below normal                        [WARNING]
│
├── Voltage critically high                         [HIGH]
│   └── Voltage above normal                        [WARNING]
│
├── Frequency below / above normal                  [WARNING]
├── Phase 1 — low power factor                      [INFO]
│
├── Outlet {N} — SWITCHED OFF                       [AVERAGE]
│   └── Outlet {N} — no load while switched on      [WARNING]
│
├── Outlet {N} — current OVERLOAD                   [HIGH]
│   └── Outlet {N} — current > 80% of limit         [WARNING]
│
├── Outlet {N} — power above limit                  [WARNING]
└── Outlet {N} — low PF                             [INFO]
```

**Result:** a full PDU power loss generates **one** alert instead of 100+.

---

## Configuration recipes

### 🔧 PDU does not meter current per outlet (constant 0 A)

On the host:

```
{$MSPDU.OUTLET.CURRENT.MAX} = 0
{$MSPDU.OUTLET.POWER.MAX}   = 0
```

This silences all outlet current and power triggers. Switch-state, phase
and sensor monitoring remain active.

### 🔧 Mute a specific outlet (intentionally switched off)

```
{$MSPDU.OUTLET.OFF.ALERT:"7"} = 0
```

Outlet 7 may stay off without raising an alert. All other outlets behave normally.

### 🔧 Alert on a server PSU failure

The server in outlet 3 normally draws ~120 W. Alert when it falls below 20 W
while still reported as `ON`:

```
{$MSPDU.OUTLET.POWER.MIN:"3"}     = 20
{$MSPDU.OUTLET.NOLOAD.PERIOD:"3"} = 10m
```

> ⚠️ Works only if the PDU reports per-outlet current. With a constant 0 A
> **do not enable** this macro — you will generate false alarms.

### 🔧 32 A PDU

```
{$MSPDU.PHASE.CURRENT.MAX} = 32
```

The 80% (25.6 A → WARNING) and 100% (32 A → HIGH) thresholds recalculate automatically.

### 🔧 Second sensor connected

```
{$MSPDU.SENSOR2.ENABLED} = 1
```

### 🔧 Data centre with strict temperature limits

```
{$MSPDU.OUTLET.TEMP.MAX} = 27
{$MSPDU.OUTLET.TEMP.MIN} = 18
{$MSPDU.OUTLET.HUM.MAX}  = 60
{$MSPDU.HUM.MIN}         = 40
```

### 🔧 Generator-backed supply (unstable frequency)

```
{$MSPDU.FREQ.MIN} = 47
{$MSPDU.FREQ.MAX} = 53
```

---

## Troubleshooting

<details>
<summary><b>LLD does not discover any outlets</b></summary>

1. Check `MSPDU: RAW walk - outlet names` in `Latest data` — does it return data?
2. Verify manually:
   ```bash
   snmpwalk -v2c -c public 10.0.0.10 1.3.6.1.4.1.47394.7.1.5
   ```
3. The format must match: `.5.<index>.0 = STRING: "name"`
4. Outlets named `--` or empty are **intentionally skipped** — name them in the PDU web UI
5. Force a run: `Execute now` on the master item, then on the discovery rule
</details>

<details>
<summary><b>Outlet power always reads 0 W</b></summary>

Power is **calculated** (`I × U × PF`). If any factor is 0, the result is 0.

```bash
snmpwalk -v2c -c public 10.0.0.10 1.3.6.1.4.1.47394.7.1.8.1
```

If every value is `0` or `--`, the hardware does not support per-outlet metering.
Apply the [no per-outlet metering recipe](#-pdu-does-not-meter-current-per-outlet-constant-0-a).
</details>

<details>
<summary><b>False "SWITCHED OFF" alerts</b></summary>

The PDU may return `1`/`0` instead of `ON`/`OFF`. Check:

```bash
snmpget -v2c -c public 10.0.0.10 1.3.6.1.4.1.47394.7.1.7.2.0
```

If it returns `1`, set on the host:

```
{$MSPDU.OUTLET.STATE.ON} = 1
```

The item normalises the value to UPPERCASE, so `on`, `On`, `ON` all become `ON`.
</details>

<details>
<summary><b>Temperature alerts with no sensor attached</b></summary>

A disconnected sensor returns `0 °C`, which fires "temperature too low".

```
{$MSPDU.SENSOR2.ENABLED} = 0
```
</details>

<details>
<summary><b>Import error: "Invalid parameter /uuid"</b></summary>

Zabbix 7.4 requires a valid **UUID v4**:
- 32 hex characters, no dashes
- 13th character = `4`
- 17th character ∈ `{8, 9, a, b}`

Run the [validation script](#validation).
</details>

<details>
<summary><b>"Low power factor" alert with no load</b></summary>

This should not happen — PF triggers only activate above a load threshold.
If it does, raise the threshold:

```
{$MSPDU.PF.CURRENT.MIN}      = 2      # phase
{$MSPDU.OUTLET.PF.POWER.MIN} = 100    # outlet
```
</details>

---

## Validation

### YAML syntax

```bash
#!/usr/bin/env bash
# check-yaml.sh
FILE="${1:-template_bkt_mspdu.yaml}"

python3 -c "
import sys, yaml
try:
    yaml.safe_load(open('$FILE', encoding='utf-8'))
    print('✅ YAML OK')
except yaml.YAMLError as e:
    print('❌ YAML ERROR:', e); sys.exit(1)
"
```

### UUID v4 compliance

```bash
#!/usr/bin/env bash
# check-uuid.sh
FILE="${1:-template_bkt_mspdu.yaml}"
ERR=0

grep -oP '(?<=uuid: )[0-9a-f]{32}' "$FILE" | while read -r u; do
    [[ "${u:12:1}" != "4" ]] && { echo "❌ $u — 13th char != 4"; ERR=1; }
    [[ ! "${u:16:1}" =~ [89ab] ]] && { echo "❌ $u — 17th char != 8/9/a/b"; ERR=1; }
done

DUP=$(grep -oP '(?<=uuid: )[0-9a-f]{32}' "$FILE" | sort | uniq -d)
[[ -n "$DUP" ]] && { echo "❌ Duplicate UUIDs:"; echo "$DUP"; ERR=1; }

[[ $ERR -eq 0 ]] && echo "✅ UUID OK"
```

### Running the scripts

```bash
chmod +x check-yaml.sh check-uuid.sh
./check-yaml.sh template_bkt_mspdu.yaml
./check-uuid.sh template_bkt_mspdu.yaml
```

### Generating a new UUID v4

```bash
python3 -c "import uuid; print(uuid.uuid4().hex)"
```

---

## FAQ

**Does the template support three-phase PDUs?**
Not yet — only phase 1 is implemented. Extending it requires items for OIDs
`...7.1.2.2.x.0` and `...7.1.2.3.x.0`. See the [Roadmap](#roadmap).

**Can I switch outlets on/off from Zabbix?**
No. The template is **read-only**. Control requires SNMP SET (`snmpset`) via an
external script or a Zabbix Script item — deliberately omitted for safety reasons.

**Why is the template's technical name in Polish?**
`template: 'SNMP Listwa MSPDU BKT'` appears in every trigger expression.
The visible name (`name:`) is English. Renaming the technical identifier means
editing ~60 places — scheduled for version 2.0.

**Does the template load the PDU heavily?**
Barely. Instead of ~150 individual SNMP queries it performs **5 walk operations**,
from which dependent items extract every metric.

**How do I update the template without losing my settings?**
Always set macros **at host level**, never in the template. On re-import tick
`Update existing` — host macros stay untouched.

**Does it work with Zabbix 7.0?**
The `version: '7.4'` export format will not import into 7.0. You can try changing
the value to `7.0` — the underlying features (`walk[]`, dependent items, calculated
items) are available there. Untested.

---

## Roadmap

- [ ] Phase 2 and 3 support (three-phase PDUs)
- [ ] Dashboard / widgets
- [ ] Value mapping for switch state
- [ ] English technical template name (v2.0, breaking change)
- [ ] Per-outlet graph prototypes
- [ ] SNMPv3 documentation
- [ ] Submission to `zabbix/community-templates`

---

## Contributing

Pull requests are welcome. Before submitting:

1. Run both validation scripts — they must pass
2. **Generate new UUIDs**, never copy existing ones
3. Trigger names in `dependencies → name` must match trigger names **exactly**
4. Test on real hardware; state the model and firmware version in the PR
5. Follow the naming convention `MSPDU: <area> - <metric>`

When reporting a bug, attach:

```bash
snmpwalk -v2c -c public <IP> 1.3.6.1.4.1.47394.7.1 > walk.txt
```

*(anonymise outlet names if they contain sensitive data)*

---

## License

Released **free of charge** under the [MIT License](LICENSE).
Free for personal **and commercial** use. You may use, copy, modify and
redistribute it, provided the copyright notice and licence text are retained.
Provided "**as is**", without warranty of any kind.

```
MIT License · Copyright (c) 2026 Sebastian Mentel
```

### Disclaimer

**Unofficial** template — not affiliated with, endorsed by, or supported
by the hardware vendor. **BKT**, **MSPDU** and **Zabbix** are trademarks of their
respective owners, used here for identification purposes only. The author accepts
no liability for any consequences of using this template in production, including
false positives or missed alerts. **Test before production deployment.**

### Credits

Built from an analysis of a real `snmpwalk` capture of a BKT MSPDU device
(24 outlets, single phase). If you run it on a different model and it works —
[let us know in Issues](../../issues) and we will add it to the compatibility list.

<div align="center">

**[⬆ Back to top](#bkt-mspdu-pdu--zabbix-template-snmp)**  ·  **[🇵🇱 Czytaj po polsku](#-wersja-polska)**

</div>

---

<sub>
<b>Keywords:</b> zabbix template, zabbix 7.4, BKT MSPDU, PDU monitoring, SNMP PDU,
rack power strip monitoring, 1.3.6.1.4.1.47394, enterprise OID 47394,
outlet discovery, LLD, per-outlet power, power factor, energy monitoring,
data center monitoring, szablon zabbix listwa zasilająca, monitoring PDU SNMP,
temperature humidity sensor, snmpwalk 47394, zabbix pdu template free
</sub>
