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

## 2. Interfejs planowany dla PV_home

Do lokalnego Modbus RTU przewidziana jest para komunikacyjna A1/B1:

- pin 1 — `Signal +`,
- pin 3 — `Signal -`.

Planowany interfejs po stronie kontrolera: **galwanicznie izolowany USB↔RS485**, zasilany po USB z komputera sterującego. Nie planujemy używać zasilania z pinu 16.

Piny 5/6 pozostają przeznaczone dla Smart Metera, a piny 7–11 dla fabrycznej komunikacji z systemem baterii.

## 3. Parametry komunikacji do weryfikacji

Punkt startowy dla testu READ-ONLY:

- Modbus RTU,
- RS485 2-wire half-duplex,
- 9600 bit/s,
- 8N1,
- adres slave: do potwierdzenia w menu falownika.

Pierwszy etap projektu nie wykonuje zapisów do rejestrów falownika.

## 4. Orientacja złącza — jeszcze niepotwierdzona fotograficznie

Funkcje pinów są zapisane powyżej, ale **geometria numeracji na konkretnym konektorze nie jest jeszcze zatwierdzona**. Przed dodaniem rysunku należy wykonać zdjęcia:

1. portu COM od spodu falownika,
2. całej wtyczki,
3. wkładu od strony przewodów,
4. oznaczeń / numerów wytłoczonych na wkładzie.

Po weryfikacji dodamy rysunek orientacyjny z oznaczeniem `PIN 1`, `PIN 3` i stroną widoku.

## 5. Uwaga o oznaczeniach A/B w RS485

Różni producenci adapterów potrafią stosować odmienne nazewnictwo `A/B`. W dokumentacji PV_home zapisujemy więc również polaryzację:

- SOFAR A1 = `Signal +`,
- SOFAR B1 = `Signal -`.

Dla wybranego adaptera USB↔RS485 jego oznaczenia zostaną dopisane dopiero po wyborze konkretnego modelu.

## 6. Kolejne pinouty

W tym samym pliku będą dopisywane kolejne urządzenia, m.in.:

- izolowany adapter USB↔RS485,
- sterownik / Raspberry Pi,
- dodatkowe urządzenia pomiarowe,
- pozostałe interfejsy projektu.

Dla każdej pozycji zapisujemy model, rewizję, numerację, funkcje pinów, poziomy sygnałów, status weryfikacji oraz źródło dokumentacji.
