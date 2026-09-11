# PV_home — założenia ekonomiczne i źródła cen

> Status: roboczy / żywy dokument projektu  
> Ostatnia aktualizacja: 2026-09-12

## 1. Cel

Warstwa ekonomiczna `PV_home` ma podejmować decyzje EMS na podstawie **rzeczywistych kosztów energii**, a nie wyłącznie na podstawie mocy, SOC lub prostych harmonogramów.

System ma rozróżniać co najmniej:

- koszt zakupu energii z sieci w aktualnej strefie G12W,
- zmienne koszty dystrybucji zależne od pobranych kWh,
- wartość energii oddawanej do sieci w net-billingu,
- straty ładowania/rozładowania magazynu,
- przyjęty koszt degradacji baterii,
- koszty stałe, które nie powinny wpływać na decyzję o pojedynczej kWh.

Celem jest minimalizacja **realnego kosztu energii w zł**, przy zachowaniu ograniczeń technicznych i bezpieczeństwa instalacji.

---

## 2. Oficjalne źródła cen

### 2.1. Zakup energii — PGE Obrót

Podstawowym źródłem ceny energii kupowanej z sieci ma być aktualna, oficjalna taryfa / cennik **PGE Obrót** dla grupy `G12W`.

Dla taryfy obowiązującej w 2026 r. PGE publikuje dla G12W dwie strefy:

| Strefa | Cena netto bez akcyzy wg tabeli PGE | Cena brutto z VAT i akcyzą |
|---|---:|---:|
| S1 — dzienna / droższa | 0,5821 zł/kWh | 0,7221 zł/kWh |
| S2 — nocna / tańsza | 0,4235 zł/kWh | 0,5271 zł/kWh |

Na fakturze użytkownika wartości netto z doliczoną akcyzą odpowiadają odpowiednio około `0,5871 zł/kWh` i `0,4285 zł/kWh`.

**Wartości nie mogą być zaszyte na stałe w kodzie.** EMS ma posiadać wersjonowaną konfigurację taryfy z datą obowiązywania.

Oficjalne źródło: PGE Obrót — taryfa dla odbiorców z grup taryfowych G oraz publikowane ceny obowiązujące w danym okresie.

### 2.2. Dystrybucja — PGE Dystrybucja

Do kosztu zakupu 1 kWh należy doliczać te składniki dystrybucyjne, które rzeczywiście zależą od pobranej energii, w szczególności:

- opłatę sieciową zmienną,
- opłatę jakościową,
- opłatę OZE,
- opłatę kogeneracyjną,
- inne zmienne składniki, jeżeli pojawią się w aktualnej taryfie.

Opłaty stałe, np. abonament, opłata sieciowa stała lub miesięczna opłata mocowa dla gospodarstwa domowego, należy przechowywać do raportowania całkowitego kosztu, ale **nie traktować jako kosztu marginalnego pojedynczej kWh** w bieżącym algorytmie sterowania.

Oficjalne źródło: **PGE Dystrybucja — aktualna taryfa usług dystrybucji**.

### 2.3. Wartość energii oddanej — PSE

Źródłem cen stosowanych w net-billingu ma być **PSE**.

System powinien obsługiwać:

- `RCEm` — Rynkową miesięczną cenę energii elektrycznej,
- `RCE` — Rynkową cenę energii, jeżeli dla danego sposobu rozliczenia użytkownika jest stosowana cena godzinowa.

PSE publikuje oficjalne RCEm z datą publikacji i ewentualnymi późniejszymi korektami. Przykładowo dla 2026 r. wartości opublikowane przez PSE obejmują m.in.:

| Miesiąc 2026 | RCEm |
|---|---:|
| styczeń | 551,96 zł/MWh |
| luty | 339,01 zł/MWh, później skorygowana do 331,39 zł/MWh |
| marzec | 191,95 zł/MWh |
| kwiecień | 132,92 zł/MWh |
| maj | 191,37 zł/MWh |
| czerwiec | 273,20 zł/MWh |
| lipiec | 262,88 zł/MWh |
| sierpień | 294,53 zł/MWh |

**RCEm jest ceną publikowaną po zakończeniu miesiąca**, więc nie może być jedynym wejściem do sterowania w czasie rzeczywistym. Po publikacji służy jako oficjalna wartość do rozliczenia i korekty modeli ekonomicznych.

Oficjalne źródło: **PSE — RCEm / RCE**.

---

## 3. Zasada pobierania cen przez PV_home

Docelowo Raspberry Pi nie powinno wymagać ręcznego wpisywania cen po każdej zmianie taryfy.

Warstwa `pricing` ma:

1. pobierać dane wyłącznie z oficjalnych źródeł,
2. zapisywać lokalnie wartość, źródło, datę publikacji i zakres obowiązywania,
3. przechowywać poprzednie wersje cen,
4. wykrywać zmianę taryfy lub korektę RCEm,
5. nie nadpisywać historii — korekta ma tworzyć nową wersję rekordu,
6. działać z lokalnego cache, gdy Internet jest niedostępny,
7. oznaczać dane jako `stale`, jeżeli nie udało się ich zaktualizować w oczekiwanym terminie.

Brak Internetu **nie może blokować pracy falownika ani podstawowej logiki bezpieczeństwa**.

---

## 4. Koszt marginalny importu

Dla bieżącej decyzji EMS interesuje nas koszt uniknięcia kolejnej 1 kWh pobranej z sieci.

W uproszczeniu:

```text
import_cost_per_kWh =
    energia_czynna_PGE
  + zmienna_dystrybucja
  + opłata_jakościowa
  + OZE
  + kogeneracja
  + podatki związane z tymi składnikami
```

Koszty stałe nie są dodawane do bieżącej decyzji `charge/discharge`, ponieważ ich wysokość zwykle nie zmieni się od przesunięcia pojedynczej kWh między godzinami.

Do raportów miesięcznych i rocznych należy jednak liczyć osobno:

- koszt energii,
- koszty zmienne dystrybucji,
- koszty stałe,
- koszt całkowity.

---

## 5. Wartość eksportu

Wartość 1 kWh wysłanej do sieci nie jest równa cenie 1 kWh kupowanej z sieci.

EMS ma znać:

```text
export_value_per_kWh
```

wyznaczoną zgodnie z aktualnym sposobem rozliczenia prosumenckiego użytkownika.

Dla okresów rozliczanych przez RCEm dokładna cena bieżącego miesiąca jest znana dopiero po publikacji PSE. Dlatego sterowanie w czasie rzeczywistym powinno używać:

- aktualnej RCE, jeżeli jest właściwa dla sposobu rozliczenia i dostępna,
- albo konserwatywnego estymatora wartości eksportu,
- a po publikacji RCEm wykonywać rozliczenie retrospektywne i korektę statystyk.

Sposób naliczania depozytu prosumenckiego i ewentualne współczynniki ustawowe mają być **konfigurowalne i wersjonowane**, a nie wpisane na stałe w logikę EMS.

---

## 6. Arbitraż G12W z magazynem energii

Zimą `PV_home` może wykorzystywać różnicę kosztów pomiędzy tanią i drogą strefą G12W.

Decyzja o ładowaniu z sieci nie może jednak bazować tylko na różnicy taryfowej.

Minimalny model:

```text
cost_from_battery =
    cheap_zone_import_cost / round_trip_efficiency
  + battery_degradation_cost
```

Ładowanie z sieci w taniej strefie i późniejsze rozładowanie jest ekonomicznie uzasadnione tylko wtedy, gdy:

```text
expensive_zone_import_cost > cost_from_battery + safety_margin
```

Dodatkowo EMS musi uwzględniać:

- SOC,
- prognozę PV na następny dzień,
- spodziewane zużycie domu,
- maksymalną moc ładowania / rozładowania,
- rezerwę energii dla EPS, jeśli zostanie przyjęta,
- potrzebę pozostawienia wolnej pojemności na PV,
- napięcie sieci i możliwość wykorzystania baterii do ograniczenia `GridOVP`.

Nie zakładamy automatycznego ładowania do 100% każdej nocy.

---

## 7. Priorytety ekonomiczne EMS

Docelowa funkcja celu powinna preferować kolejno:

1. bezpośrednią autokonsumpcję PV,
2. wykorzystanie baterii do uniknięcia drogiego importu, jeżeli jest ekonomicznie uzasadnione,
3. zachowanie odpowiedniego miejsca w baterii na przyszłą produkcję PV,
4. ładowanie z taniej strefy tylko wtedy, gdy prognoza i koszt całego cyklu to uzasadniają,
5. eksport nadwyżki wtedy, gdy magazynowanie / późniejsze użycie nie daje większej wartości,
6. ograniczenie eksportu, gdy jest to potrzebne do zapobiegania `GridOVP`.

Wartość ekonomiczna nie może mieć pierwszeństwa przed:

- ograniczeniami BMS,
- zabezpieczeniami falownika,
- kodem sieciowym `Poland`,
- limitami mocy i temperatury,
- wymaganiami operatora sieci.

---

## 8. Dane historyczne i audyt decyzji

Każda automatyczna decyzja ekonomiczna powinna być możliwa do odtworzenia.

Dla decyzji EMS zapisujemy co najmniej:

- timestamp,
- SOC,
- PV / load / grid power,
- aktywną strefę G12W,
- koszt importu użyty przez algorytm,
- estymowaną wartość eksportu,
- prognozę PV,
- sprawność przyjętą do obliczeń,
- koszt degradacji baterii,
- decyzję i zadaną moc,
- powód decyzji.

Pozwoli to po miesiącu odpowiedzieć nie tylko `ile zaoszczędziliśmy`, ale również **dlaczego EMS podjął konkretną decyzję**.

---

## 9. Źródła referencyjne

Preferowane źródła nadrzędne:

- PGE Obrót — aktualna taryfa i ceny energii dla grupy G12W,
- PGE Dystrybucja — aktualna taryfa usług dystrybucji,
- PSE — RCEm i RCE,
- rzeczywiste faktury / dane pomiarowe PGE użytkownika do walidacji.

Źródła zewnętrzne, kalkulatory i serwisy porównawcze mogą służyć wyłącznie pomocniczo. Nie są źródłem ceny używanej do automatycznego sterowania.
