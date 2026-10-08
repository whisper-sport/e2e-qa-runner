# e2e-qa-runner

Projekt-agnostyczne narzędzie do end-to-end QA zmian wytworzonych przez agentów AI. Po zakończeniu zadania (pojedynczego runu agenta albo paczki zmian z wielu repozytoriów) narzędzie:

1. stawia całe środowisko testowe,
2. przeklikuje zmianę tak, jak zrobiłby to człowiek: agent wykonuje scenariusze MD krok po kroku w przeglądarce,
3. zbiera dowody i wydaje werdykt.

Status: **wczesny etap — dokumenty projektowe, brak kodu.** Licencja: Apache-2.0.

## Dlaczego

Agenci potrafią dziś dowieźć implementację i otworzyć PR, ale ktoś musi jeszcze postawić ekosystem i sprawdzić zmianę end-to-end. W małych zespołach to wąskie gardło: testuje jedna osoba albo nikt. Do tego dochodzi pułapka fałszywej pewności. Zielony wynik scenariuszy, które testują zmianę, a nie tezy produktu, niczego nie dowodzi.

## Założenia

- **Projekt-agnostyczne.** Rdzeń nie zna żadnego konkretnego projektu. Projekt-konsument opisuje się manifestem: repozytoria, wersje, porty, przepis na start, seed, konta QA.
- **Wiele repozytoriów.** Jedna zmiana często obejmuje kilka repo (API + frontend + aplikacja mobilna), a paczka wydania obejmuje ich wiele.
- **Najpierw lokalnie** (Docker, jeden stos testowy naraz), z kontraktem przenośnym na serwer.
- **Scenariusze w modelu hybrydowym:** katalog regresji per projekt plus scenariusze z kryteriów akceptacji, które generuje izolowany agent w trybie szukania kontrprzykładów. Uzasadnienie: [`docs/research/2026-10-08-scenariusze-z-ac-probe.md`](docs/research/2026-10-08-scenariusze-z-ac-probe.md).
- **Kontrakt dowodu.** PASS wymaga sprawdzenia efektu (DOM, ruch sieciowy, baza), a nie wyniku polecenia narzędzia.

## Plan

| Etap | Stan |
|---|---|
| Eksperyment: czy scenariusze z kryteriów akceptacji łapią realne defekty | ✅ 1/3 → hybryda |
| Spec #1: manifest środowiska multi-repo z kontraktem seeda i kont QA | ⏳ następny ([brief](docs/specs/briefs/2026-10-08-manifest-srodowiska.md)) |
| Spec #2: generator scenariuszy (hybryda, tryb kontrprzykładów) | — |
| Spec #3: raport z kontraktem dowodu, następnie naprawa | — |
| Spec #4: UI historii runów | — |

## Układ repo

- `docs/specs/briefs/`: briefy wejściowe dla specyfikacji
- `docs/specs/`: specyfikacje (jedna funkcja = jeden spec)
- `docs/research/`: wyniki eksperymentów
