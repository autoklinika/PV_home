# PV_home — pinout i dokumentacja złączy

> Dokument referencyjny projektu. Ostatnia aktualizacja: 2026-09-11.
>
> Numerację i orientację złącza należy potwierdzić na konkretnym egzemplarzu przed jakąkolwiek pracą serwisową. Ten plik opisuje funkcje niskonapięciowego portu komunikacyjnego COM; nie opisuje połączeń AC ani baterii HV.

## 1. SOFAR HYD 15KTL-3PH — port COM (16 pin)

Falownik: **SOFAR HYD 15KTL-3PH**, SN `SP1ES115P2Q407`, Protocol `1.36`, DSPM/DSPS `V120005`.

| Pin | Oznaczenie | Funkcja | Status w PV_home |
|---:|---|---|---|
| 1 | RS485 A1-1 | RS485 Signal +, monitoring / EMS | planowany Modbus |
| 2 | RS485 A1-2 | RS485 Signal +, monitoring / EMS | linia równoległa / daisy-chain |
| 3 | RS485 B1-1 | RS485 Signal -, monitoring / EMS | planowany Modbus |
| 4 | RS485 B1-2 | RS485 Signal -, monitoring / EMS | linia równoległa / daisy-chain |
| 5 | RS485 A2 | RS485 + Smart Meter | istniejąca komunikacja — bez zmian |
| 6 | RS485 B2 | RS485 - Smart Meter | istniejąca komunikacja — bez zmian |
| 7 | CAN0_H | CAN High systemu baterii | istniejąca komunikacja — bez zmian |
| 8 | CAN0_L | CAN Low systemu baterii | istniejąca komunikacja — bez zmian |
| 9 | GND.S | masa sygnałowa | nieplanowana w pierwszym teście |
| 10 | 485TX0+ | RS485 + systemu baterii | istniejąca komunikacja — bez zmian |
| 11 | 485TX0- | RS485 - systemu baterii | istniejąca komunikacja — bez zmian |
| 12 | GND.S | masa sygnałowa | nieplanowana w pierwszym teście |
| 13 | BAT_Temp | sygnał temperatury baterii / wejście pomocnicze wg wariantu | bez zmian |
| 14 | DCT1 | sygnał / wyjście pomocnicze 1 | do późniejszej analizy |
| 15 | DCT2 | sygnał / wyjście pomocnicze 2 | do późniejszej analizy |
| 16 | VCC_12V | pomocnicze 12 V; wg dokumentacji producenta max. 0,5 A | nieużywane w pierwszym etapie |

### Para używana przez PV_home

Do lokalnego Modbus RTU przewidziana jest para komunikacyjna A1/B1:

- pin 1 — `Signal +`,
- pin 3 — `Signal -`.

Piny 5/6 pozostają przeznaczone dla Smart Metera, a piny 7–11 dla fabrycznej komunikacji z systemem baterii.

## 2. DFRobot DFR0845 — izolowany RS485 ↔ UART

Wybrany interfejs dla PV_home:

```text
DFRobot DFR0845
Gravity: Active Isolated RS485 to UART Module
```

Moduł został wcześniej fizycznie zwalidowany w projekcie WVC. Producent deklaruje separację galwaniczną strony UART i RS485; izolacja modułu jest podawana jako **3000 VDC**. Strona logiczna może być zasilana napięciem **3,3–5 V**.

### 2.1. Strona UART / logiczna

| Oznaczenie DFR0845 | Funkcja | Połączenie w PV_home |
|---|---|---|
| `T` | UART TX po stronie modułu | Raspberry Pi `TX` → `T` |
| `R` | UART RX po stronie modułu | Raspberry Pi `RX` ← `R` |
| `-` | GND strony logicznej | GND Raspberry Pi / zasilania logiki |
| `+` | VCC strony logicznej | 3,3 V lub inne zgodne z projektem zasilanie 3,3–5 V |

Najważniejsza zasada zwalidowana w WVC:

```text
Raspberry Pi TX -> T DFR0845
Raspberry Pi RX <- R DFR0845
```

Oznaczeń `T` i `R` **nie krzyżujemy**.

### 2.2. Strona izolowana RS485

| Oznaczenie DFR0845 | Funkcja | Połączenie w PV_home |
|---|---|---|
| `A` | RS485 signal A | do linii A1 falownika |
| `B` | RS485 signal B | do linii B1 falownika |
| `GND` | izolowana masa strony RS485 | domyślnie nie łączyć z GND strony UART; użycie do ustalenia po testach |
| `12V` | pomocnicze wyjście 12 V / 2 W | nieużywane |

Producent DFR0845 opisuje `GND` przy `A/B` jako **RS485 side isolated ground**. Nie jest to ta sama masa co `-` po stronie UART.

### 2.3. Planowane połączenie SOFAR ↔ DFR0845

```text
SOFAR COM pin 1  RS485 A1 / Signal +  -> DFR0845 A
SOFAR COM pin 3  RS485 B1 / Signal -  -> DFR0845 B
```

Uwaga: oznaczenia `A/B` w urządzeniach RS485 bywają stosowane różnie przez producentów. SOFAR dodatkowo dokumentuje własne linie jako `Signal +` i `Signal -`. Przed uruchomieniem zapisu sterującego najpierw wykonujemy test READ-ONLY. Jeżeli fizyczna para jest poprawna, a brak odpowiedzi wskazuje wyłącznie na odwróconą polaryzację, jedyną dopuszczalną korektą na tym etapie jest zamiana A/B.

### 2.4. Zasilanie DFR0845

W WVC DFR0845 został zwalidowany przy napięciu około `3,28 V` po stronie UART. Dla PV_home dokładne źródło zasilania zostanie ustalone po wyborze konkretnego Raspberry Pi i architektury zasilania.

Nie planujemy wykorzystywać wyjścia `12V` DFR0845 do zasilania falownika ani żadnej części SOFAR.

## 3. Raspberry Pi — UART do DFR0845

Model Raspberry Pi i konkretny port UART nie są jeszcze zamknięte.

Niezależnie od wybranego modelu wymagane połączenia są następujące:

```text
RPi TX  -> DFR0845 T
RPi RX  <- DFR0845 R
RPi GND -> DFR0845 -
VCC     -> DFR0845 +
```

Jeżeli zostanie użyty CM5 / CM5 IO Board tak jak w WVC, istnieją już fizycznie zwalidowane warianty:

```text
UART0:
GPIO14 / TXD0 -> DFR0845 T
GPIO15 / RXD0 <- DFR0845 R
/dev/ttyAMA0

UART4:
GPIO12 / TXD4 -> DFR0845 T
GPIO13 / RXD4 <- DFR0845 R
/dev/ttyAMA4
```

Dla PV_home konkretna para GPIO i urządzenie `/dev/ttyAMA*` zostaną wpisane dopiero po wyborze sprzętu.

## 4. Parametry komunikacji SOFAR do weryfikacji

Punkt startowy dla testu READ-ONLY:

- Modbus RTU,
- RS485 2-wire half-duplex,
- 9600 bit/s,
- 8N1,
- adres slave: do potwierdzenia w menu falownika.

Pierwszy etap projektu nie wykonuje zapisów do rejestrów falownika.

## 5. Orientacja złącza SOFAR — jeszcze niezatwierdzona na zdjęciu referencyjnym

Funkcje pinów są zapisane powyżej, a fizyczna 16-pinowa wtyczka została już sfotografowana. Nadal trzeba jednoznacznie przypisać geometrię numeracji do konkretnego widoku konektora przed wykonaniem przewodu.

Do dokumentacji należy dodać po potwierdzeniu:

1. widok od strony styków,
2. widok od strony przewodów,
3. oznaczenie `PIN 1`,
4. oznaczenie `PIN 3`,
5. jednoznaczną informację, z której strony patrzymy na złącze.

## 6. Separacja galwaniczna — zasada projektu

DFR0845 rozdziela galwanicznie:

```text
Raspberry Pi / UART / masa logiczna
            || IZOLACJA ||
RS485 A/B / izolowana strona magistrali SOFAR
```

Nie należy lokalnie zwierać:

```text
DFR0845 '-'   (masa logiczna UART)
DFR0845 GND   (izolowana masa RS485)
```

chyba że późniejsza, konkretna analiza dokumentacji i topologii systemu wykaże taką potrzebę.

## 7. Źródła i status weryfikacji

DFR0845:

- oficjalna dokumentacja DFRobot DFR0845,
- repo `autoklinika/workshop-ventilation-controller`,
- raport walidacji `CM5_DFR0845_DUAL_UART_RS485_VALIDATION_PL.md`,
- `docs/PINOUT.md` projektu WVC.

SOFAR:

- dokumentacja HYD 5–20KTL-3PH,
- dokumentacja portu COM,
- fizyczny egzemplarz HYD 15KTL-3PH SN `SP1ES115P2Q407`.

## 8. Kolejne pinouty

W tym samym pliku będą dopisywane kolejne urządzenia, m.in.:

- konkretny Raspberry Pi / CM5 użyty w PV_home,
- zasilanie DFR0845,
- ewentualne dodatkowe urządzenia pomiarowe,
- pozostałe interfejsy projektu.

Dla każdej pozycji zapisujemy model, rewizję, numerację, funkcje pinów, poziomy sygnałów, status weryfikacji oraz źródło dokumentacji.