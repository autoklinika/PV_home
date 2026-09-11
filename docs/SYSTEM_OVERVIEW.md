# PV_home — założenia systemu, diagnostyka i integracja SOFAR Modbus

> Status dokumentu: roboczy / żywy dokument projektu  
> Ostatnia aktualizacja: 2026-09-11

## 1. Cel projektu

Projekt `PV_home` ma stworzyć lokalny, niezależny od chmury system monitorowania i docelowo inteligentnego zarządzania domową instalacją fotowoltaiczną oraz magazynem energii.

Główne powody uruchomienia projektu:

1. **Wysokie rachunki za energię mimo instalacji PV i magazynu energii.**
2. **Częste wyłączenia falownika**, w historii zdarzeń dominują alarmy `ID01`, które dla tej rodziny SOFAR odpowiadają przekroczeniu napięcia sieci (`GridOVP`). Występują też pojedyncze `ID02`, `ID03` i `ID12`.
3. **Podejrzenie problemu z napięciem / impedancją sieci AC**, powodującego cykliczne odłączanie falownika podczas produkcji i eksportu energii.
4. **Rozbieżność pomiędzy bilansem energii widocznym w SOLARMAN i danymi rozliczeniowymi PGE**, którą trzeba zmierzyć niezależnie.
5. **Optymalizacja taryfy G12W** — zimą magazyn energii może być ładowany z sieci w taniej strefie i rozładowywany w droższej strefie, o ile bilans ekonomiczny po uwzględnieniu strat i degradacji baterii jest dodatni.
6. **Lepsze wykorzystanie magazynu energii do ograniczenia problemu wysokiego napięcia**, np. przez dynamiczne zwiększanie ładowania przy rosnącym napięciu sieci zamiast dopuszczania do całkowitego wyłączenia falownika.

Projekt ma najpierw dostarczyć **wiarygodne dane pomiarowe**, a dopiero później sterowanie.

---

## 2. Aktualna instalacja

### 2.1. Fotowoltaika

- moc zainstalowana paneli: **9,13 kWp**,
- układ dachu: **wschód–zachód**.

### 2.2. Falownik

- producent: **SOFAR**,
- model: **HYD 15KTL-3PH**,
- moc znamionowa AC: **15 kW**,
- kraj / Safety Standard: **Poland**,
- numer seryjny: `SP1ES115P2Q407`.

Wersje odczytane z urządzenia / SOLARMAN:

| Parametr | Wartość |
|---|---|
| DSPM Firmware Version | `V120005` |
| DSPS Firmware Version | `V120005` |
| Standard Main Version | `1008` |
| Protocol Version | `1.36` |
| Firmware Version | `V000001_V120005` |
| Communication Processor SW | `V120005` |
| PCU Software Version | `V010022` |
| BDU Software Version | `V010010` |
| Version Type | `0` |

### 2.3. Magazyn energii

- system SOFAR BTS,
- falownik raportuje: **2 × BTS 5K online**,
- nominalna pojemność zestawu: około **10,24 kWh**,
- PCU_ID: `1`.

### 2.4. Obserwacja napięć AC

Przykładowy odczyt nocny, bez produkcji PV:

| Faza | Napięcie |
|---|---:|
| R | 230,8 V |
| S | 221,4 V |
| T | 235,9 V |

Różnica pomiędzy najniższą i najwyższą fazą wynosiła około **14,5 V**. To jest ważna obserwacja diagnostyczna — przy eksporcie PV należy rejestrować każdą fazę osobno i sprawdzić, czy któraś z nich dochodzi do progu ochrony nadnapięciowej.

---

## 3. Dane rozliczeniowe i dlaczego potrzebujemy własnego pomiaru

Z rzeczywistego rozliczenia PGE za okres 2026-01-01…2026-06-30:

- pobór z sieci: **3187 kWh**,
- oddanie do sieci: **509 kWh**.

Dla czerwca 2026:

- pobór: **70 kWh**,
- oddanie: **226 kWh**.

Jednocześnie raport SOLARMAN widoczny 2026-09-11 pokazywał około **2,68 MWh energii z sieci od początku roku**. Już samo PGE za pierwsze sześć miesięcy wykazało 3,187 MWh poboru, dlatego dane SOLARMAN dotyczące importu/eksportu wymagają weryfikacji.

### Wniosek

Własny logger Modbus ma być niezależnym źródłem surowych danych. Dobowe i miesięczne sumy z niego będziemy porównywać z:

- licznikiem / danymi PGE,
- SOLARMAN,
- licznikami energii raportowanymi przez falownik.

---

## 4. Interfejs komunikacyjny SOFAR

Falownik HYD 15KTL-3PH udostępnia lokalną komunikację **RS485 / Modbus RTU** przez złącze `COM`.

Producent przewiduje również **Passive Mode** przeznaczony do współpracy z zewnętrznym systemem EMS (Energy Management System). Oznacza to, że docelowe sterowanie zewnętrznym kontrolerem nie jest koncepcją opartą na modyfikacji firmware falownika — jest to tryb przewidziany przez producenta.

### Bazowe parametry komunikacji

Do pierwszych testów przyjmujemy:

- protokół: **Modbus RTU**,
- warstwa fizyczna: **RS485 2-wire, half-duplex**,
- prędkość: **9600 bit/s**,
- format: **8N1**,
- adres urządzenia: do potwierdzenia w menu falownika przed pierwszym odczytem.

### Ważne

Pełna mapa **rejestrów zapisywalnych** musi zostać potwierdzona dla dokładnej kombinacji:

- HYD 15KTL-3PH,
- FW `V120005`,
- Protocol `1.36`,
- wersji sprzętowej urządzenia.

Do czasu potwierdzenia oficjalnej mapy rejestrów nie wykonujemy żadnych zapisów opartych wyłącznie na nieoficjalnych mapach z Internetu.

---

## 5. Planowana architektura

```text
                    Internet / prognoza pogody
                              │
                              ▼
                     ┌─────────────────┐
                     │    PV_home EMS  │
                     │ Raspberry Pi /  │
                     │ lokalny serwer  │
                     └────────┬────────┘
                              │
                     izolowany USB-RS485
                              │
                     Modbus RTU / RS485
                              │
                              ▼
                     ┌─────────────────┐
                     │ SOFAR HYD 15KTL │
                     │      -3PH       │
                     └──────┬──────────┘
                            │
           ┌────────────────┼────────────────┐
           ▼                ▼                ▼
          PV             BTS 10 kWh       sieć/dom
```

SOLARMAN pozostaje dodatkowym narzędziem podglądu, ale **nie będzie podstawą sterowania EMS**.

---

## 6. Etapy wdrożenia

### Etap 1 — tylko odczyt (READ-ONLY)

Pierwszy działający system ma **wyłącznie odczytywać dane**. Żaden rejestr sterujący nie może być zapisywany.

Planowane dane:

- napięcie L1/L2/L3,
- częstotliwość sieci,
- prądy i moce faz,
- PV1/PV2: napięcie, prąd, moc,
- całkowita moc PV,
- moc domu / load,
- import / eksport,
- SOC baterii,
- napięcie i prąd baterii,
- moc ładowania / rozładowania,
- temperatury dostępne w Modbus,
- stan pracy falownika,
- aktywne alarmy / błędy,
- liczniki energii PV, grid i battery.

### Częstotliwość logowania

Docelowo dla diagnostyki napięcia i zdarzeń ID01:

- podstawowa telemetria: **1–2 s**,
- agregaty długoterminowe: 1 min / 15 min / 1 h / 1 dzień,
- po zdarzeniu alarmowym przechowywać szczegółowe dane sprzed i po zdarzeniu.

Przykładowy rekord diagnostyczny:

```text
12:43:12  L1=242.2  L2=244.1  L3=248.1  PV=5.8kW  Export=3.9kW
12:43:20  L3=249.6  PV=6.1kW  Export=4.2kW
12:43:27  L3=251.4  PV=6.3kW  Export=4.4kW
12:43:33  ID01
12:43:34  PV=0
```

### Etap 2 — laboratoryjny Passive Mode

Dopiero po zweryfikowaniu odczytu oraz mapy rejestrów:

1. przejście do Passive Mode,
2. wymuszenie małej mocy ładowania, np. 0,5 kW,
3. test krótkotrwały,
4. kontrola reakcji Smart Metera, baterii i sieci,
5. powrót do Self-use,
6. stopniowe testy większych mocy.

Każdy zapis musi być logowany.

### Etap 3 — EMS taryfowy G12W

Przyszły EMS ma podejmować decyzję na podstawie:

- aktualnego SOC,
- zużycia domu,
- produkcji PV,
- prognozy produkcji PV na kolejny dzień,
- taryfy G12W i dnia tygodnia,
- napięcia każdej fazy,
- maksymalnej mocy ładowania / rozładowania,
- przyjętego kosztu degradacji baterii.

Przykład zimowy:

- słaba prognoza PV → większe nocne doładowanie z taniej strefy,
- dobra prognoza PV → pozostawienie większej wolnej pojemności na energię słoneczną,
- droga strefa → zasilanie domu z baterii, jeśli ekonomicznie uzasadnione.

### Etap 4 — ochrona przed cyklicznym GridOVP

Cel nie polega na zmianie progów ochrony sieci.

EMS ma reagować **przed** wyłączeniem falownika, np.:

1. obserwować najwyższe napięcie z L1/L2/L3,
2. przy wzroście napięcia zwiększać ładowanie baterii, jeśli dostępna jest pojemność,
3. ograniczać eksport zgodnie z możliwościami producenta,
4. preferować kontrolowane ograniczenie mocy nad cyklem `pełna moc → GridOVP → OFF → restart`.

Wartości progowe dla algorytmu będą ustalone dopiero po zebraniu rzeczywistych danych. Nie wpisujemy ich na stałe bez pomiarów.

---

## 7. Założenia bezpieczeństwa

To jest kluczowa część projektu.

1. **Nie zmieniamy** kodu kraju / Safety Standard `Poland`.
2. **Nie podnosimy** progów zabezpieczeń napięciowych ani częstotliwościowych po to, aby ukryć problem z siecią.
3. Program EMS otrzyma **allowlistę** rejestrów, do których wolno pisać.
4. Rejestry zabezpieczeń sieciowych mają być programowo zablokowane przed zapisem.
5. W pierwszym etapie cały interfejs Modbus działa READ-ONLY.
6. Komunikacja przez **galwanicznie izolowany adapter USB↔RS485**.
7. Awaria EMS / brak komunikacji nie może pozbawiać falownika jego własnych zabezpieczeń.
8. Należy przewidzieć ręczny powrót do fabrycznego / standardowego `Self-use`.
9. Każda komenda sterująca ma mieć timestamp, wartość zadaną, odpowiedź falownika i wynik operacji.
10. Przed pracą na złączu COM urządzenie należy obsługiwać zgodnie z procedurami bezpieczeństwa producenta.

---

## 8. Dane i wizualizacja

Do ustalenia podczas implementacji. Preferowane rozwiązanie:

- mała lokalna baza dla surowej telemetrii,
- retencja danych 1–2 s przez ograniczony czas,
- agregaty długoterminowe przechowywane bezterminowo,
- dashboard z wykresami:
  - PV,
  - load,
  - battery charge/discharge,
  - import/export,
  - SOC,
  - L1/L2/L3,
  - alarmy,
  - bilans dobowy i miesięczny.

Możliwe technologie: InfluxDB + Grafana, PostgreSQL/TimescaleDB lub lżejszy stos na pierwszy prototyp. Wybór nie jest jeszcze zamknięty.

---

## 9. Funkcje docelowe

Docelowo `PV_home` powinien umożliwiać:

- lokalną telemetrię bez zależności od SOLARMAN Cloud,
- diagnostykę wyłączeń falownika,
- wykrywanie asymetrii i nadmiernego napięcia faz,
- weryfikację importu / eksportu względem PGE,
- harmonogramy baterii,
- dynamiczne ładowanie z sieci w G12W,
- dynamiczne wykorzystanie prognozy PV,
- zachowywanie miejsca w baterii na produkcję PV,
- minimalizację zakupu energii w drogiej strefie,
- ograniczanie niekorzystnego eksportu przy wysokim napięciu,
- uwzględnianie kosztu degradacji baterii,
- alarmy i raporty dobowe / miesięczne,
- pełny audyt wszystkich komend sterujących.

---

## 10. Najbliższe zadania

1. Udokumentować fizyczne złącze COM na zdjęciach.
2. Potwierdzić orientację i numerację pinów na konkretnym urządzeniu.
3. Wybrać izolowany adapter USB↔RS485.
4. Potwierdzić adres Modbus falownika.
5. Uruchomić pierwszy test READ-ONLY.
6. Zweryfikować podstawowe rejestry przez porównanie z wyświetlaczem / SOLARMAN.
7. Zebrać minimum kilka dni telemetrii, w tym co najmniej jedno zdarzenie `ID01`.
8. Porównać dobowe liczniki import/export z licznikiem PGE.
9. Pozyskać / potwierdzić mapę rejestrów sterujących dla FW `V120005`, Protocol `1.36`.
10. Dopiero potem rozpocząć testy Passive Mode.

---

## 11. Źródła / dokumentacja producenta

Dokumentacja projektu powinna być weryfikowana przede wszystkim z aktualną dokumentacją SOFAR dla rodziny HYD 5–20KTL-3PH oraz dokumentacją protokołu Modbus właściwą dla konkretnego firmware.

Przydatne źródła producenta:

- SOFAR HYD 5–20KTL-3PH — instrukcja użytkownika i opis interfejsu COM,
- SOFAR — dokumentacja Passive Mode / Modbus dla HYD 3PH,
- archiwum firmware SOFAR dla HYD 5–20KTL-3PH.

Nieoficjalne implementacje open-source mogą być wykorzystywane do porównania i testów, ale **nie są źródłem nadrzędnym dla rejestrów zapisywalnych**.
