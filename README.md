# Zabbix Template: SNMP Listwa MSPDU BKT

[![Zabbix](https://img.shields.io/badge/Zabbix-7.4-red)](https://www.zabbix.com/documentation/7.4/)
[![Protocol](https://img.shields.io/badge/protocol-SNMPv2c%2Fv3-blue)]()
[![Enterprise OID](https://img.shields.io/badge/OID-1.3.6.1.4.1.47394.7.1-lightgrey)]()

> 🇵🇱 [Wersja polska](#-wersja-polska) · 🇬🇧 [English version](#-english-version)

---

# 🇵🇱 Wersja polska

## Spis treści

1. [Opis](#1-opis)
2. [Wymagania](#2-wymagania)
3. [Instalacja](#3-instalacja)
4. [Gdzie i jak zmieniać wartości](#4-gdzie-i-jak-zmieniać-wartości)
5. [Rozpiska makr](#5-rozpiska-makr)
6. [Rozpiska triggerów](#6-rozpiska-triggerów)
7. [Mapa zależności](#7-mapa-zależności)
8. [Przepisy na typowe sytuacje](#8-przepisy-na-typowe-sytuacje)
9. [Pułapki](#9-pułapki)
10. [Rozwiązywanie problemów](#10-rozwiązywanie-problemów)
11. [Walidacja pliku przed importem](#11-walidacja-pliku-przed-importem)

---

## 1. Opis

Szablon monitoruje listwy zasilające (PDU) **MSPDU** producenta BKT po SNMP.

**Zakres monitoringu:**

| Obszar | Metryki |
|---|---|
| Urządzenie | nazwa, typ, MAC, firmware, liczba portów, uptime (tekst + sekundy), buzzer |
| Faza 1 | prąd, napięcie, moc, współczynnik mocy (PF), energia, częstotliwość |
| Czujniki | temperatura ×2, wilgotność ×2 |
| Gniazda (LLD, do 24 szt.) | nazwa, stan przełącznika, prąd, PF, energia, moc obliczana (I × U × PF) |

**Zasada działania:** pojedyncze zapytania `walk[...]` pobierają całe tabele SNMP, a JavaScript w preprocessingu rozdziela je na pozycje zależne (*dependent items*). Dzięki temu 24 gniazda × 5 metryk = 120 wartości kosztuje **5 zapytań SNMP**, a nie 120.

---

## 2. Wymagania

- Zabbix Server / Proxy **7.4 lub nowszy** (format eksportu `7.4`)
- Włączony SNMP na listwie (v2c lub v3)
- Sieciowa dostępność UDP/161 z serwera/proxy Zabbix
- Interfejs SNMP skonfigurowany na hoście w Zabbiksie

---

## 3. Instalacja

1. **Import szablonu**
   `Data collection → Templates → Import` → wskaż plik YAML → *Import*

2. **Utworzenie hosta**
   - Dodaj host z interfejsem typu **SNMP**
   - Ustaw `{$SNMP_COMMUNITY}` (v2c) lub poświadczenia v3
   - Podłącz szablon **SNMP Listwa MSPDU BKT**

3. **Weryfikacja odczytu** (z serwera Zabbix):
   ```bash
   snmpwalk -v2c -c public <IP_LISTWY> 1.3.6.1.4.1.47394.7.1.1
   snmpwalk -v2c -c public <IP_LISTWY> 1.3.6.1.4.1.47394.7.1.5   # nazwy gniazd
   snmpwalk -v2c -c public <IP_LISTWY> 1.3.6.1.4.1.47394.7.1.7   # stany ON/OFF
   ```

4. **Sprawdzenie wartości stanu gniazda**
   Jeśli OID `...7.<nr>.0` zwraca `1`/`0` zamiast `ON`/`OFF`, ustaw na hoście:
   `{$MSPDU.OUTLET.STATE.ON} = 1`

5. **Dostrojenie makr** — patrz [sekcja 8](#8-przepisy-na-typowe-sytuacje)

---

## 4. Gdzie i jak zmieniać wartości

Zabbix rozstrzyga makro wg priorytetu (od najwyższego):

| Priorytet | Miejsce | Zastosowanie |
|---|---|---|
| 1 | **Host** → *Macros* | Konkretna listwa, np. bezpiecznik 32 A |
| 2 | **Szablon** → *Macros* | Zmiana dla wszystkich listew z tym szablonem |
| 3 | **Administration → Macros** | Standard całej organizacji |

> ⚠️ **Nie edytuj wyrażeń triggerów.** Zmieniaj wyłącznie makra — edycja wyrażenia w szablonie zostanie nadpisana przy ponownym imporcie YAML.

### Makra kontekstowe (per gniazdo)

Sześć makr obsługuje **kontekst**, czyli inną wartość dla pojedynczego gniazda:

```
{$MSPDU.OUTLET.POWER.MIN:"3"} = 30
                          ↑
                    numer gniazda = {#OUTLET_INDEX}
```

> ⚠️ **Kontekstem jest NUMER gniazda, nie jego nazwa.**
> `{$MSPDU.OUTLET.POWER.MIN:"Prox1"}` **nie zadziała**.

---

## 5. Rozpiska makr

### 5.1 Dostępność i urządzenie

| Makro | Domyślnie | Jedn. | Co robi | Kiedy zmieniać |
|---|---|---|---|---|
| `{$MSPDU.NODATA}` | `10m` | czas | Po tym czasie bez danych SNMP → alarm HIGH „listwa nieosiągalna". Trigger **nadrzędny** — wycisza prawie wszystkie pozostałe | `5m` przy krytycznych szafach; `20m` przy niestabilnym WAN |
| `{$MSPDU.UPTIME.RESTART}` | `600` | s | Spadek licznika uptime **i** nowa wartość poniżej progu → alarm „Listwa zrestartowana" | `1800` przy rzadszym odpytywaniu niż co 5 min |

### 5.2 Napięcie (faza 1)

| Makro | Domyślnie | Waga | Histereza (recovery) |
|---|---|---|---|
| `{$MSPDU.VOLTAGE.MIN.CRIT}` | `195` V | HIGH | brak — natychmiastowy OK |
| `{$MSPDU.VOLTAGE.MIN.WARN}` | `207` V | WARNING (230 V −10 %) | zamknięcie przy `> próg+3` V |
| `{$MSPDU.VOLTAGE.MAX.WARN}` | `253` V | WARNING (230 V +10 %) | zamknięcie przy `< próg−3` V |
| `{$MSPDU.VOLTAGE.MAX.CRIT}` | `260` V | HIGH | brak |

> Zachowaj relację `MIN.CRIT < MIN.WARN < MAX.WARN < MAX.CRIT`, inaczej zależności triggerów stracą sens.

### 5.3 Częstotliwość

| Makro | Domyślnie | Uwagi |
|---|---|---|
| `{$MSPDU.FREQ.MIN}` | `49` Hz | Trigger wymaga odczytu `>0`, by nie alarmować przy braku pomiaru. Histereza ±0,2 Hz |
| `{$MSPDU.FREQ.MAX}` | `51` Hz | Typowo rośnie przy pracy z agregatu |

Praca z generatora i szum alarmów → rozszerz do `48` / `52`.

### 5.4 Prąd fazy (wejście listwy)

| Makro | Domyślnie | Co robi |
|---|---|---|
| `{$MSPDU.PHASE.CURRENT.MAX}` | `16` A | **Jedno makro steruje dwoma progami:** 80 % → WARNING, 100 % → HIGH |

Listwa 32 A → `32` daje WARNING przy 25,6 A i HIGH przy 32 A.

### 5.5 Współczynnik mocy (faza)

| Makro | Domyślnie | Co robi |
|---|---|---|
| `{$MSPDU.PF.MIN}` | `0.6` | Próg alarmu INFO o niskim PF |
| `{$MSPDU.PF.CURRENT.MIN}` | `1` A | **Blokada fałszywek** — poniżej tego prądu PF jest ignorowany (bez obciążenia PF = 0 jest normalne) |

### 5.6 Gniazda — stan załączenia

| Makro | Domyślnie | Kontekst | Co robi |
|---|---|---|---|
| `{$MSPDU.OUTLET.STATE.ON}` | `ON` | nie | Wartość oznaczająca gniazdo załączone. Item normalizuje odczyt do WIELKICH liter (`on`, `On`, `ON` → `ON`). Gdy listwa zwraca `1` — zmień na `1` |
| `{$MSPDU.OUTLET.OFF.ALERT}` | `1` | **tak** | `1` = alarmuj o wyłączonym gnieździe, `0` = cisza |

**Najczęstsza modyfikacja — puste gniazda:**
```
{$MSPDU.OUTLET.OFF.ALERT:"7"}  = 0
{$MSPDU.OUTLET.OFF.ALERT:"8"}  = 0
{$MSPDU.OUTLET.OFF.ALERT:"24"} = 0
```

### 5.7 Gniazda — prąd i moc

| Makro | Domyślnie | Kontekst | Co robi |
|---|---|---|---|
| `{$MSPDU.OUTLET.CURRENT.MAX}` | `16` A | **tak** | Limit prądu: 80 % → WARNING, 100 % → HIGH. **`0` = oba triggery wyłączone** |
| `{$MSPDU.OUTLET.POWER.MAX}` | `3000` W | **tak** | Limit mocy → WARNING. `0` = wyłączony |
| `{$MSPDU.OUTLET.POWER.MIN}` | `0` W | **tak** | Wykrywanie awarii zasilacza: moc poniżej progu mimo stanu ON. **`0` = wyłączony (domyślnie)** |
| `{$MSPDU.OUTLET.NOLOAD.PERIOD}` | `15m` | **tak** | Jak długo moc musi być poniżej progu przed alarmem |

> 🔧 **Listwa bez pomiaru prądu per gniazdo** (stale `0 A` lub `--`) — ustaw na hoście:
> ```
> {$MSPDU.OUTLET.CURRENT.MAX} = 0
> {$MSPDU.OUTLET.POWER.MAX}   = 0
> ```
> Pozostaw `POWER.MIN = 0`, bo wyliczona moc (I × U × PF) też będzie zerowa.

### 5.8 Gniazda — współczynnik mocy

| Makro | Domyślnie | Co robi |
|---|---|---|
| `{$MSPDU.OUTLET.PF.MIN}` | `0.6` | Próg INFO |
| `{$MSPDU.OUTLET.PF.POWER.MIN}` | `50` W | Poniżej tej mocy PF gniazda jest ignorowany |

### 5.9 Czujniki środowiskowe

| Makro | Domyślnie | Co robi |
|---|---|---|
| `{$MSPDU.SENSOR1.ENABLED}` | `1` | **Wyłącznik bezpieczeństwa.** `0` = sonda niepodłączona → brak alarmów |
| `{$MSPDU.SENSOR2.ENABLED}` | `0` | Domyślnie **wyłączony** — włącz (`1`) po fizycznym podłączeniu |
| `{$MSPDU.OUTLET.TEMP.MAX}` | `35` °C | AVERAGE, histereza −2 °C |
| `{$MSPDU.OUTLET.TEMP.MIN}` | `5` °C | WARNING, histereza +2 °C |
| `{$MSPDU.OUTLET.HUM.MAX}` | `80` % | AVERAGE, histereza −5 % |
| `{$MSPDU.HUM.MIN}` | `20` % | WARNING, histereza +5 % |

> Bez `SENSOR*.ENABLED = 0` odłączona sonda zwraca `0` i generuje stały alarm „temperatura za niska" oraz „wilgotność za niska".

---

## 6. Rozpiska triggerów

### 6.1 Poziom listwy

| Trigger | Waga | Warunek (uproszczony) | Sterowany przez | Zależny od |
|---|---|---|---|---|
| Brak danych SNMP – listwa nieosiągalna | **HIGH** | `nodata(prąd fazy)` | `NODATA` | — (korzeń) |
| Faza 1 – PRZECIĄŻENIE | **HIGH** | `min(3m) > MAX` | `PHASE.CURRENT.MAX` | Brak danych |
| Faza 1 – prąd > 80 % nominału | WARNING | `min(5m) > MAX×0.8` | `PHASE.CURRENT.MAX` | Przeciążenie, Brak danych |
| Napięcie krytycznie niskie | **HIGH** | `max(5m) < MIN.CRIT` | `VOLTAGE.MIN.CRIT` | Brak danych |
| Napięcie poniżej normy | WARNING | `avg(5m) < MIN.WARN` | `VOLTAGE.MIN.WARN` | Napięcie kryt. niskie |
| Napięcie krytycznie wysokie | **HIGH** | `min(3m) > MAX.CRIT` | `VOLTAGE.MAX.CRIT` | Brak danych |
| Napięcie powyżej normy | WARNING | `avg(5m) > MAX.WARN` | `VOLTAGE.MAX.WARN` | Napięcie kryt. wysokie |
| Częstotliwość poniżej normy | WARNING | `min(10m) < FREQ.MIN` i `>0` | `FREQ.MIN` | Brak danych |
| Częstotliwość powyżej normy | WARNING | `max(10m) > FREQ.MAX` | `FREQ.MAX` | Brak danych |
| Faza 1 – niski PF | INFO | `avg(15m) < PF.MIN` **i** prąd `> PF.CURRENT.MIN` | `PF.MIN`, `PF.CURRENT.MIN` | Brak danych |
| Listwa została zrestartowana | WARNING | spadek uptime i `< UPTIME.RESTART` | `UPTIME.RESTART` | — |
| Zmieniono wersję oprogramowania | INFO | zmiana wartości | — | — |
| Zmieniono nazwę urządzenia | INFO | zmiana wartości | — | — |

### 6.2 Czujniki (×2 komplety)

| Trigger | Waga | Sterowany przez |
|---|---|---|
| Czujnik N – temperatura za wysoka | AVERAGE | `OUTLET.TEMP.MAX` + `SENSORN.ENABLED` |
| Czujnik N – temperatura za niska | WARNING | `OUTLET.TEMP.MIN` + `SENSORN.ENABLED` |
| Czujnik N – wilgotność za wysoka | AVERAGE | `OUTLET.HUM.MAX` + `SENSORN.ENABLED` |
| Czujnik N – wilgotność za niska | WARNING | `HUM.MIN` + `SENSORN.ENABLED` |

### 6.3 Prototypy triggerów (per gniazdo, ×24)

| Trigger | Waga | Warunek | Sterowany przez | Zależny od |
|---|---|---|---|---|
| Gniazdo – PRZECIĄŻENIE prądowe | **HIGH** | `CURRENT.MAX>0` i `min(3m)>CURRENT.MAX` | `OUTLET.CURRENT.MAX` | Brak danych |
| Gniazdo – prąd > 80 % limitu | WARNING | `CURRENT.MAX>0` i `min(5m)>80 %` | `OUTLET.CURRENT.MAX` | Przeciążenie gniazda, Brak danych |
| Gniazdo – moc powyżej limitu | WARNING | `POWER.MAX>0` i `min(5m)>POWER.MAX` | `OUTLET.POWER.MAX` | Brak danych |
| Gniazdo – brak obciążenia mimo załączenia | WARNING | `POWER.MIN>0` i `max(okno)<POWER.MIN` i stan `= ON` | `OUTLET.POWER.MIN`, `NOLOAD.PERIOD` | Gniazdo WYŁĄCZONE, Brak danych |
| Gniazdo – WYŁĄCZONE | AVERAGE | stan `≠ ON` i `OFF.ALERT=1` | `OUTLET.OFF.ALERT`, `OUTLET.STATE.ON` | Brak danych |
| Gniazdo – zmiana stanu przełącznika | INFO | wartość ≠ poprzednia | — (manual close) | — |
| Gniazdo – niski PF | INFO | `avg(15m)<PF.MIN` i moc `> PF.POWER.MIN` | `OUTLET.PF.MIN`, `OUTLET.PF.POWER.MIN` | Brak danych |
| Gniazdo – licznik energii wyzerowany | INFO | `change() < 0` | — (manual close) | — |

---

## 7. Mapa zależności

```
Brak danych SNMP (HIGH)
 ├── Faza 1 – PRZECIĄŻENIE
 │    └── Faza 1 – prąd > 80 %
 ├── Napięcie krytycznie niskie
 │    └── Napięcie poniżej normy
 ├── Napięcie krytycznie wysokie
 │    └── Napięcie powyżej normy
 ├── Częstotliwość poniżej / powyżej normy
 ├── Faza 1 – niski PF
 ├── Gniazdo – PRZECIĄŻENIE prądowe
 │    └── Gniazdo – prąd > 80 % limitu
 ├── Gniazdo – moc powyżej limitu
 ├── Gniazdo – niski PF
 └── Gniazdo – WYŁĄCZONE
      └── Gniazdo – brak obciążenia mimo załączenia
```

**Efekt:** zanik zasilania listwy = **1 alarm**, nie 50.

---

## 8. Przepisy na typowe sytuacje

### A. Listwa bez pomiaru prądu per gniazdo
```
{$MSPDU.OUTLET.CURRENT.MAX} = 0
{$MSPDU.OUTLET.POWER.MAX}   = 0
```

### B. Gniazda puste / zapasowe (nie alarmuj o OFF)
```
{$MSPDU.OUTLET.OFF.ALERT:"13"} = 0
{$MSPDU.OUTLET.OFF.ALERT:"14"} = 0
```

### C. Wykrywanie awarii zasilacza serwera (gniazdo 3, ~120 W)
```
{$MSPDU.OUTLET.POWER.MIN:"3"}     = 40
{$MSPDU.OUTLET.NOLOAD.PERIOD:"3"} = 10m
```
Próg ustaw na **30–40 % typowego poboru** — z zapasem na idle.

### D. Listwa 32 A zamiast 16 A
```
{$MSPDU.PHASE.CURRENT.MAX} = 32
```

### E. Druga sonda podłączona
```
{$MSPDU.SENSOR2.ENABLED} = 1
```

### F. Restrykcyjny reżim cieplny
```
{$MSPDU.OUTLET.TEMP.MAX} = 27
{$MSPDU.OUTLET.HUM.MAX}  = 60
```

### G. Listwa zwraca `1`/`0` zamiast `ON`/`OFF`
```
{$MSPDU.OUTLET.STATE.ON} = 1
```

### H. Przykładowa konfiguracja hosta `row1rack6pduA`
```
{$MSPDU.PHASE.CURRENT.MAX}     = 16
{$MSPDU.OUTLET.CURRENT.MAX}    = 0     # brak pomiaru per gniazdo
{$MSPDU.OUTLET.POWER.MAX}      = 0
{$MSPDU.SENSOR2.ENABLED}       = 0
{$MSPDU.OUTLET.OFF.ALERT:"9"}  = 0     # gniazdo puste
{$MSPDU.OUTLET.OFF.ALERT:"10"} = 0
{$MSPDU.OUTLET.OFF.ALERT:"11"} = 0
```

---

## 9. Pułapki

1. **Wyłączenie triggera makrem `= 0`** działa tylko tam, gdzie wyrażenie zawiera warunek `MAKRO > 0` — czyli dla `OUTLET.CURRENT.MAX`, `OUTLET.POWER.MAX`, `OUTLET.POWER.MIN`. Pozostałych progów nie da się „wyzerować"; aby je wyłączyć, zdezaktywuj trigger w GUI.
2. **Makra tekstowe porównywane jako string.** `{$MSPDU.OUTLET.STATE.ON}` z wartością `ON ` (ze spacją) nie zadziała — item robi `trim()`, ale makro już nie.
3. **Histereza działa w obie strony.** Zawężenie progów napięcia do np. 228–232 V przy marginesach ±3 V sprawi, że trigger prawie nigdy się nie zamknie. Przy ciasnych progach zwiększ marginesy w recovery expression.
4. **Moc gniazda jest wyliczana** (`I × U × PF`, zaokrąglona do W), nie czytana z SNMP. Bez pomiaru prądu per gniazdo wynosi zawsze 0.
5. **Zmiana makra działa natychmiast** — bez restartu serwera i bez ponownego LLD. Wyjątek: kontekst dla gniazda jeszcze niewykrytego.
6. **Nie edytuj wyrażeń w szablonie** — przy re-imporcie YAML zmiany przepadną. Zmiany logiki wprowadzaj w pliku YAML i wersjonuj w repo.
7. **Zależności triggerów w YAML.** Jeśli trigger docelowy ma `recovery_mode: RECOVERY_EXPRESSION`, blok `dependencies` musi zawierać **wszystkie trzy pola** (`name`, `expression`, `recovery_expression`) skopiowane 1:1 — inaczej import zgłosi „trigger ... which does not exist".

---

## 10. Rozwiązywanie problemów

| Objaw | Przyczyna | Rozwiązanie |
|---|---|---|
| Import: „trigger ... which does not exist" | Brak `recovery_expression` w bloku `dependencies` | Uzupełnij wszystkie trzy pola zależności (patrz pułapka 7) |
| Import: „invalid UUID" | UUID nie spełnia UUIDv4 (13. znak ≠ `4`, 17. znak ∉ `8,9,a,b`) | Popraw znaki lub wygeneruj nowy UUID |
| LLD nie wykrywa gniazd | Walk zwraca pusto lub nazwy to `--` | Sprawdź `snmpwalk` OID `...7.1.5`; gniazda o nazwie `--` są pomijane celowo |
| Lawina alarmów „gniazdo wyłączone" | Puste gniazda lub inna wartość stanu ON | `{$MSPDU.OUTLET.OFF.ALERT:"N"} = 0` lub popraw `{$MSPDU.OUTLET.STATE.ON}` |
| Ciągły alarm temperatury/wilgotności | Sonda niepodłączona, odczyt 0 | `{$MSPDU.SENSORn.ENABLED} = 0` |
| Moc gniazda zawsze 0 W | Listwa nie mierzy prądu per gniazdo | Ustaw `OUTLET.CURRENT.MAX = 0` i `OUTLET.POWER.MAX = 0` |
| Ciągły alarm o niskim PF | Gniazdo/faza bez obciążenia | Podnieś `PF.CURRENT.MIN` lub `OUTLET.PF.POWER.MIN` |

---

## 11. Walidacja pliku przed importem

```bash
# 1. Poprawność składni YAML
python3 -c "import yaml,sys; yaml.safe_load(open('mspdu.yaml')); print('YAML OK')"

# 2. UUID niezgodne z UUIDv4 (13. znak musi być '4', 17. z zestawu 8/9/a/b)
grep -oE 'uuid: [0-9a-f]{32}' mspdu.yaml \
  | grep -vE 'uuid: [0-9a-f]{12}4[0-9a-f]{3}[89ab][0-9a-f]{15}'
# oczekiwany wynik: brak linii

# 3. Duplikaty UUID
grep -oE 'uuid: [0-9a-f]{32}' mspdu.yaml | sort | uniq -d
# oczekiwany wynik: brak linii

# 4. Liczba UUID (kontrola kompletności)
grep -cE 'uuid: [0-9a-f]{32}' mspdu.yaml
```

> 💡 Po udanym imporcie wyeksportuj szablon z GUI — Zabbix wygeneruje kanoniczną postać zależności, którą warto zapisać w repo jako wzorzec.

---
---

# 🇬🇧 English version

## Table of contents

1. [Overview](#1-overview)
2. [Requirements](#2-requirements)
3. [Installation](#3-installation)
4. [Where and how to change values](#4-where-and-how-to-change-values)
5. [Macro reference](#5-macro-reference)
6. [Trigger reference](#6-trigger-reference)
7. [Dependency map](#7-dependency-map)
8. [Recipes for common scenarios](#8-recipes-for-common-scenarios)
9. [Pitfalls](#9-pitfalls)
10. [Troubleshooting](#10-troubleshooting)
11. [Validating the file before import](#11-validating-the-file-before-import)

---

## 1. Overview

This template monitors **MSPDU** power distribution units by BKT over SNMP.

**Monitoring scope:**

| Area | Metrics |
|---|---|
| Device | name, type, MAC, firmware, port count, uptime (text + seconds), buzzer |
| Phase 1 | current, voltage, power, power factor (PF), energy, frequency |
| Sensors | temperature ×2, humidity ×2 |
| Outlets (LLD, up to 24) | name, switch state, current, PF, energy, calculated power (I × U × PF) |

**How it works:** single `walk[...]` requests fetch entire SNMP tables, and JavaScript preprocessing splits them into dependent items. As a result, 24 outlets × 5 metrics = 120 values cost **5 SNMP requests**, not 120.

---

## 2. Requirements

- Zabbix Server / Proxy **7.4 or newer** (export format `7.4`)
- SNMP enabled on the PDU (v2c or v3)
- Network reachability on UDP/161 from the Zabbix server/proxy
- SNMP interface configured on the host in Zabbix

---

## 3. Installation

1. **Import the template**
   `Data collection → Templates → Import` → select the YAML file → *Import*

2. **Create the host**
   - Add a host with an **SNMP** interface
   - Set `{$SNMP_COMMUNITY}` (v2c) or v3 credentials
   - Link the **SNMP Listwa MSPDU BKT** template

3. **Verify the readings** (from the Zabbix server):
   ```bash
   snmpwalk -v2c -c public <PDU_IP> 1.3.6.1.4.1.47394.7.1.1
   snmpwalk -v2c -c public <PDU_IP> 1.3.6.1.4.1.47394.7.1.5   # outlet names
   snmpwalk -v2c -c public <PDU_IP> 1.3.6.1.4.1.47394.7.1.7   # ON/OFF states
   ```

4. **Check the outlet state value**
   If OID `...7.<n>.0` returns `1`/`0` instead of `ON`/`OFF`, set on the host:
   `{$MSPDU.OUTLET.STATE.ON} = 1`

5. **Tune the macros** — see [section 8](#8-recipes-for-common-scenarios)

---

## 4. Where and how to change values

Zabbix resolves macros by priority (highest first):

| Priority | Location | Use case |
|---|---|---|
| 1 | **Host** → *Macros* | A specific PDU, e.g. a 32 A breaker |
| 2 | **Template** → *Macros* | All PDUs linked to this template |
| 3 | **Administration → Macros** | Organisation-wide standard |

> ⚠️ **Do not edit trigger expressions.** Change macros only — expression edits in the template are overwritten on the next YAML import.

### Context macros (per outlet)

Six macros support **context**, i.e. a different value for a single outlet:

```
{$MSPDU.OUTLET.POWER.MIN:"3"} = 30
                          ↑
                 outlet number = {#OUTLET_INDEX}
```

> ⚠️ **The context is the outlet NUMBER, not its name.**
> `{$MSPDU.OUTLET.POWER.MIN:"Prox1"}` **will not work**.

---

## 5. Macro reference

### 5.1 Availability and device

| Macro | Default | Unit | What it does | When to change |
|---|---|---|---|---|
| `{$MSPDU.NODATA}` | `10m` | time | After this period without SNMP data → HIGH alert "PDU unreachable". This is the **root** trigger — it suppresses almost all others | `5m` for critical racks; `20m` on unstable WAN |
| `{$MSPDU.UPTIME.RESTART}` | `600` | s | Uptime counter drops **and** the new value is below this threshold → "PDU restarted" alert | `1800` if polling less often than every 5 min |

### 5.2 Voltage (phase 1)

| Macro | Default | Severity | Hysteresis (recovery) |
|---|---|---|---|
| `{$MSPDU.VOLTAGE.MIN.CRIT}` | `195` V | HIGH | none — immediate OK |
| `{$MSPDU.VOLTAGE.MIN.WARN}` | `207` V | WARNING (230 V −10 %) | closes at `> threshold+3` V |
| `{$MSPDU.VOLTAGE.MAX.WARN}` | `253` V | WARNING (230 V +10 %) | closes at `< threshold−3` V |
| `{$MSPDU.VOLTAGE.MAX.CRIT}` | `260` V | HIGH | none |

> Keep the relation `MIN.CRIT < MIN.WARN < MAX.WARN < MAX.CRIT`, otherwise trigger dependencies stop making sense.

### 5.3 Frequency

| Macro | Default | Notes |
|---|---|---|
| `{$MSPDU.FREQ.MIN}` | `49` Hz | The trigger also requires a reading `>0` so it does not fire when measurement is missing. Hysteresis ±0.2 Hz |
| `{$MSPDU.FREQ.MAX}` | `51` Hz | Typically rises on generator power |

Generator operation with alert noise → widen to `48` / `52`.

### 5.4 Phase current (PDU input)

| Macro | Default | What it does |
|---|---|---|
| `{$MSPDU.PHASE.CURRENT.MAX}` | `16` A | **One macro drives two thresholds:** 80 % → WARNING, 100 % → HIGH |

A 32 A PDU → `32` yields WARNING at 25.6 A and HIGH at 32 A.

### 5.5 Power factor (phase)

| Macro | Default | What it does |
|---|---|---|
| `{$MSPDU.PF.MIN}` | `0.6` | INFO threshold for low PF |
| `{$MSPDU.PF.CURRENT.MIN}` | `1` A | **False-positive guard** — below this current PF is ignored (PF = 0 with no load is normal) |

### 5.6 Outlets — switch state

| Macro | Default | Context | What it does |
|---|---|---|---|
| `{$MSPDU.OUTLET.STATE.ON}` | `ON` | no | Value meaning "outlet energised". The item normalises the reading to UPPERCASE (`on`, `On`, `ON` → `ON`). If the PDU returns `1` — change to `1` |
| `{$MSPDU.OUTLET.OFF.ALERT}` | `1` | **yes** | `1` = alert on a switched-off outlet, `0` = silent |

**Most common change — empty outlets:**
```
{$MSPDU.OUTLET.OFF.ALERT:"7"}  = 0
{$MSPDU.OUTLET.OFF.ALERT:"8"}  = 0
{$MSPDU.OUTLET.OFF.ALERT:"24"} = 0
```

### 5.7 Outlets — current and power

| Macro | Default | Context | What it does |
|---|---|---|---|
| `{$MSPDU.OUTLET.CURRENT.MAX}` | `16` A | **yes** | Current limit: 80 % → WARNING, 100 % → HIGH. **`0` = both triggers disabled** |
| `{$MSPDU.OUTLET.POWER.MAX}` | `3000` W | **yes** | Power limit → WARNING. `0` = disabled |
| `{$MSPDU.OUTLET.POWER.MIN}` | `0` W | **yes** | PSU failure detection: power below threshold despite ON state. **`0` = disabled (default)** |
| `{$MSPDU.OUTLET.NOLOAD.PERIOD}` | `15m` | **yes** | How long power must stay below the threshold before alerting |

> 🔧 **PDU without per-outlet current metering** (constant `0 A` or `--`) — set on the host:
> ```
> {$MSPDU.OUTLET.CURRENT.MAX} = 0
> {$MSPDU.OUTLET.POWER.MAX}   = 0
> ```
> Leave `POWER.MIN = 0` as well, because the calculated power (I × U × PF) will also be zero.

### 5.8 Outlets — power factor

| Macro | Default | What it does |
|---|---|---|
| `{$MSPDU.OUTLET.PF.MIN}` | `0.6` | INFO threshold |
| `{$MSPDU.OUTLET.PF.POWER.MIN}` | `50` W | Below this power the outlet PF is ignored |

### 5.9 Environmental sensors

| Macro | Default | What it does |
|---|---|---|
| `{$MSPDU.SENSOR1.ENABLED}` | `1` | **Safety switch.** `0` = probe not connected → no alerts |
| `{$MSPDU.SENSOR2.ENABLED}` | `0` | **Disabled** by default — enable (`1`) after physically connecting the probe |
| `{$MSPDU.OUTLET.TEMP.MAX}` | `35` °C | AVERAGE, hysteresis −2 °C |
| `{$MSPDU.OUTLET.TEMP.MIN}` | `5` °C | WARNING, hysteresis +2 °C |
| `{$MSPDU.OUTLET.HUM.MAX}` | `80` % | AVERAGE, hysteresis −5 % |
| `{$MSPDU.HUM.MIN}` | `20` % | WARNING, hysteresis +5 % |

> Without `SENSOR*.ENABLED = 0`, a disconnected probe reports `0` and raises permanent "temperature too low" and "humidity too low" alerts.

---

## 6. Trigger reference

### 6.1 PDU level

| Trigger | Severity | Condition (simplified) | Driven by | Depends on |
|---|---|---|---|---|
| No SNMP data – PDU unreachable | **HIGH** | `nodata(phase current)` | `NODATA` | — (root) |
| Phase 1 – OVERLOAD | **HIGH** | `min(3m) > MAX` | `PHASE.CURRENT.MAX` | No data |
| Phase 1 – current > 80 % of rating | WARNING | `min(5m) > MAX×0.8` | `PHASE.CURRENT.MAX` | Overload, No data |
| Voltage critically low | **HIGH** | `max(5m) < MIN.CRIT` | `VOLTAGE.MIN.CRIT` | No data |
| Voltage below normal | WARNING | `avg(5m) < MIN.WARN` | `VOLTAGE.MIN.WARN` | Voltage crit. low |
| Voltage critically high | **HIGH** | `min(3m) > MAX.CRIT` | `VOLTAGE.MAX.CRIT` | No data |
| Voltage above normal | WARNING | `avg(5m) > MAX.WARN` | `VOLTAGE.MAX.WARN` | Voltage crit. high |
| Frequency below normal | WARNING | `min(10m) < FREQ.MIN` and `>0` | `FREQ.MIN` | No data |
| Frequency above normal | WARNING | `max(10m) > FREQ.MAX` | `FREQ.MAX` | No data |
| Phase 1 – low PF | INFO | `avg(15m) < PF.MIN` **and** current `> PF.CURRENT.MIN` | `PF.MIN`, `PF.CURRENT.MIN` | No data |
| PDU has been restarted | WARNING | uptime drop and `< UPTIME.RESTART` | `UPTIME.RESTART` | — |
| Firmware version changed | INFO | value changed | — | — |
| Device name changed | INFO | value changed | — | — |

### 6.2 Sensors (×2 sets)

| Trigger | Severity | Driven by |
|---|---|---|
| Sensor N – temperature too high | AVERAGE | `OUTLET.TEMP.MAX` + `SENSORN.ENABLED` |
| Sensor N – temperature too low | WARNING | `OUTLET.TEMP.MIN` + `SENSORN.ENABLED` |
| Sensor N – humidity too high | AVERAGE | `OUTLET.HUM.MAX` + `SENSORN.ENABLED` |
| Sensor N – humidity too low | WARNING | `HUM.MIN` + `SENSORN.ENABLED` |

### 6.3 Trigger prototypes (per outlet, ×24)

| Trigger | Severity | Condition | Driven by | Depends on |
|---|---|---|---|---|
| Outlet – current OVERLOAD | **HIGH** | `CURRENT.MAX>0` and `min(3m)>CURRENT.MAX` | `OUTLET.CURRENT.MAX` | No data |
| Outlet – current > 80 % of limit | WARNING | `CURRENT.MAX>0` and `min(5m)>80 %` | `OUTLET.CURRENT.MAX` | Outlet overload, No data |
| Outlet – power above limit | WARNING | `POWER.MAX>0` and `min(5m)>POWER.MAX` | `OUTLET.POWER.MAX` | No data |
| Outlet – no load despite being ON | WARNING | `POWER.MIN>0` and `max(window)<POWER.MIN` and state `= ON` | `OUTLET.POWER.MIN`, `NOLOAD.PERIOD` | Outlet OFF, No data |
| Outlet – SWITCHED OFF | AVERAGE | state `≠ ON` and `OFF.ALERT=1` | `OUTLET.OFF.ALERT`, `OUTLET.STATE.ON` | No data |
| Outlet – switch state changed | INFO | value ≠ previous | — (manual close) | — |
| Outlet – low PF | INFO | `avg(15m)<PF.MIN` and power `> PF.POWER.MIN` | `OUTLET.PF.MIN`, `OUTLET.PF.POWER.MIN` | No data |
| Outlet – energy counter reset | INFO | `change() < 0` | — (manual close) | — |

---

## 7. Dependency map

```
No SNMP data (HIGH)
 ├── Phase 1 – OVERLOAD
 │    └── Phase 1 – current > 80 %
 ├── Voltage critically low
 │    └── Voltage below normal
 ├── Voltage critically high
 │    └── Voltage above normal
 ├── Frequency below / above normal
 ├── Phase 1 – low PF
 ├── Outlet – current OVERLOAD
 │    └── Outlet – current > 80 % of limit
 ├── Outlet – power above limit
 ├── Outlet – low PF
 └── Outlet – SWITCHED OFF
      └── Outlet – no load despite being ON
```

**Result:** a PDU power loss produces **1 alert**, not 50.

---

## 8. Recipes for common scenarios

### A. PDU without per-outlet current metering
```
{$MSPDU.OUTLET.CURRENT.MAX} = 0
{$MSPDU.OUTLET.POWER.MAX}   = 0
```

### B. Empty / spare outlets (do not alert on OFF)
```
{$MSPDU.OUTLET.OFF.ALERT:"13"} = 0
{$MSPDU.OUTLET.OFF.ALERT:"14"} = 0
```

### C. Detecting a server PSU failure (outlet 3, ~120 W)
```
{$MSPDU.OUTLET.POWER.MIN:"3"}     = 40
{$MSPDU.OUTLET.NOLOAD.PERIOD:"3"} = 10m
```
Set the threshold to **30–40 % of typical draw** — leaving headroom for idle.

### D. 32 A PDU instead of 16 A
```
{$MSPDU.PHASE.CURRENT.MAX} = 32
```

### E. Second probe connected
```
{$MSPDU.SENSOR2.ENABLED} = 1
```

### F. Strict thermal regime
```
{$MSPDU.OUTLET.TEMP.MAX} = 27
{$MSPDU.OUTLET.HUM.MAX}  = 60
```

### G. PDU returns `1`/`0` instead of `ON`/`OFF`
```
{$MSPDU.OUTLET.STATE.ON} = 1
```

### H. Example host configuration for `row1rack6pduA`
```
{$MSPDU.PHASE.CURRENT.MAX}     = 16
{$MSPDU.OUTLET.CURRENT.MAX}    = 0     # no per-outlet metering
{$MSPDU.OUTLET.POWER.MAX}      = 0
{$MSPDU.SENSOR2.ENABLED}       = 0
{$MSPDU.OUTLET.OFF.ALERT:"9"}  = 0     # empty outlet
{$MSPDU.OUTLET.OFF.ALERT:"10"} = 0
{$MSPDU.OUTLET.OFF.ALERT:"11"} = 0
```

---

## 9. Pitfalls

1. **Disabling a trigger with `= 0`** works only where the expression contains a `MACRO > 0` guard — i.e. `OUTLET.CURRENT.MAX`, `OUTLET.POWER.MAX`, `OUTLET.POWER.MIN`. Other thresholds cannot be "zeroed out"; disable those triggers in the GUI instead.
2. **Text macros are compared as strings.** `{$MSPDU.OUTLET.STATE.ON}` set to `ON ` (with a trailing space) will not match — the item applies `trim()`, the macro does not.
3. **Hysteresis cuts both ways.** Narrowing voltage thresholds to, say, 228–232 V with ±3 V margins means the trigger will almost never close. With tight thresholds, widen the margins in the recovery expression.
4. **Outlet power is calculated** (`I × U × PF`, rounded to W), not read from SNMP. Without per-outlet current metering it is always 0.
5. **Macro changes take effect immediately** — no server restart and no re-discovery required. Exception: adding a context for an outlet that has not been discovered yet.
6. **Do not edit expressions in the template** — changes are lost on YAML re-import. Make logic changes in the YAML file and version it in a repository.
7. **Trigger dependencies in YAML.** If the target trigger uses `recovery_mode: RECOVERY_EXPRESSION`, the `dependencies` block must contain **all three fields** (`name`, `expression`, `recovery_expression`) copied verbatim — otherwise the import fails with "trigger ... which does not exist".

---

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Import: "trigger ... which does not exist" | Missing `recovery_expression` in the `dependencies` block | Provide all three dependency fields (see pitfall 7) |
| Import: "invalid UUID" | UUID is not valid UUIDv4 (13th char ≠ `4`, 17th char ∉ `8,9,a,b`) | Fix the characters or generate a new UUID |
| LLD discovers no outlets | The walk returns nothing, or names are `--` | Check `snmpwalk` on OID `...7.1.5`; outlets named `--` are skipped by design |
| Flood of "outlet switched off" alerts | Empty outlets or a different ON value | `{$MSPDU.OUTLET.OFF.ALERT:"N"} = 0` or fix `{$MSPDU.OUTLET.STATE.ON}` |
| Permanent temperature/humidity alert | Probe not connected, reading is 0 | `{$MSPDU.SENSORn.ENABLED} = 0` |
| Outlet power always 0 W | PDU does not meter current per outlet | Set `OUTLET.CURRENT.MAX = 0` and `OUTLET.POWER.MAX = 0` |
| Permanent low-PF alert | Outlet/phase has no load | Raise `PF.CURRENT.MIN` or `OUTLET.PF.POWER.MIN` |

---

## 11. Validating the file before import

```bash
# 1. YAML syntax check
python3 -c "import yaml,sys; yaml.safe_load(open('mspdu.yaml')); print('YAML OK')"

# 2. UUIDs not conforming to UUIDv4 (13th char must be '4', 17th one of 8/9/a/b)
grep -oE 'uuid: [0-9a-f]{32}' mspdu.yaml \
  | grep -vE 'uuid: [0-9a-f]{12}4[0-9a-f]{3}[89ab][0-9a-f]{15}'
# expected output: no lines

# 3. Duplicate UUIDs
grep -oE 'uuid: [0-9a-f]{32}' mspdu.yaml | sort | uniq -d
# expected output: no lines

# 4. UUID count (completeness check)
grep -cE 'uuid: [0-9a-f]{32}' mspdu.yaml
```

> 💡 After a successful import, export the template from the GUI — Zabbix produces the canonical dependency form, worth committing to the repository as a reference.

---

## Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025-01 | Initial release: 24-outlet LLD, phase metrics, 2 environmental sensors, trigger dependency tree |

## License / Contact

Internal template — adapt freely to your environment.
Report issues and improvements through your internal monitoring team.
