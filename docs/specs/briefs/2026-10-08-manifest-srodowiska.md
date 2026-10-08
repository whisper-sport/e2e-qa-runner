# Spec #1: manifest środowiska multi-repo z kontraktem seeda i kont QA, uruchamiający istniejącego wykonawcę QA w przeglądarce

- Date: 2026-10-08
- Category: feature
- Priority signal: high — testowanie zmian agentów jest dziś wąskim gardłem (jedna osoba), a paczki wydań czekają
- Risk signal: medium — ryzyko fałszywej pewności (zielony werdykt bez pokrycia) jest większe niż ryzyko techniczne
- Routing: Next: om-spec-writing "Manifest środowiska multi-repo z kontraktem seeda i kont QA plus uruchomienie istniejącego wykonawcy QA — brief: docs/specs/briefs/2026-10-08-manifest-srodowiska.md"

## Problem

Zadania realizują autonomiczne runy agentów: orkiestrator uruchamia agenta w izolowanym worktree jednego repozytorium i kończy na PR-ze. Po runie nikt nie stawia całego ekosystemu i nie przechodzi zmiany tak, jak zrobiłby to człowiek. Zmiany często obejmują kilka repo naraz, a paczki wydań wiele. Brakuje ludzi do testowania i pewności, że wszystko działa.

U pierwszego konsumenta ręczne podejścia już zawiodły:
- regresja paczki kilkunastu PR-ów na drugim stosie compose napotkała OOM, kolizje portów, limity logowania i brak OTP w dev;
- skrypt środowiska był zaszyty pod jedną funkcję i nie działał na dowolnych wersjach.

## Agreed direction

**Osobne, projekt-agnostyczne narzędzie, open source od pierwszego dnia** (to repo, Apache-2.0). Projekt-konsument dostarcza manifest i nic poza nim. Do tego mogą dojść skille agentów oparte na zestawie `om-*`.

**Kolejność dostaw** (każdy punkt to osobny spec):

1. **← ten brief.** Manifest środowiska multi-repo (repo → wersja/gałąź → porty, przepis na start dostarczany przez projekt) z kontraktem seeda i kont QA, plus uruchomienie istniejącego wykonawcy QA (np. `om-auto-qa-pr` w trybie lokalnym z `agent-browser`) na tak postawionym środowisku. Bez UI, bez pętli naprawy, bez generatora.
2. **Generator scenariuszy MD w modelu hybrydowym:**
   - katalog regresji per projekt;
   - scenariusze z kryteriów akceptacji, generowane przez izolowanego agenta przed implementacją w trybie kontrprzykładów;
   - obowiązkowy test negatywny.
3. **Raport z kontraktem dowodu**, następnie naprawa po porażce.
4. **UI historii runów testów.**

**Odrzucone:**
- **Workflow wbudowany w orkiestrator albo same skille agentów.** Wybrane zostało osobne narzędzie.
- **„Nic nie budować” (ręczne scenariusze i ręczne odpalanie QA).** Nie rozwiązuje braku ludzi.
- **Serwer albo PaaS na start.** Później; dlatego kontrakt ma być niezależny od maszyny.
- **Scenariusze wyłącznie z AC.** Eksperyment dał 1/3 (`docs/research/2026-10-08-scenariusze-z-ac-probe.md`).
- **Jeden spec na cały epik.**

## Resolved unknowns

| Question | Answer |
|----------|--------|
| Co testujemy? | Pojedyncze zmiany i paczki zmian z wielu repo |
| Gdzie stoi środowisko | Lokalnie (Docker, VM ok. 8 GB), jeden stos testowy naraz (kolejka); kontrakt przenośny na serwer |
| Aplikacje mobilne | Target web na start, emulator później. Raport oznacza wprost, że web ≠ native |
| Źródło scenariuszy | Hybryda (spec #2) |
| Co przy porażce | Raport i naprawa, UI historii (specy #3, #4) |
| Wykonawca QA w specu #1 | Istniejący (`om-auto-qa-pr` / `agent-browser`). Bez własnego |
| Publiczność repo | Open source od pierwszego dnia. Żadnych danych projektów-konsumentów w repo |

## Wymagania dla manifestu

Wynikają z problemów środowiska zebranych w eksperymencie (`docs/research/`, sekcja „Problemy środowiska”):

1. **Mapa repo → wersja (SHA/gałąź) → port.** Bez sztywnych portów i bramek gałęzi.
2. **Kilka wariantów stosu w jednym runie** (np. dwie wersje API na dwóch bazach).
3. **Tryb hybrydowy:** usługi aplikacyjne na hoście, infrastruktura w kontenerach. Budżet pamięci i izolacja nazw projektów compose.
4. **Kolejność i zależności startu**, w tym migracje różnych usług zakładające schematy dla seeda.
5. **Kontrakt seeda:** nazwane stany danych i role RBAC dla typowych scenariuszy (użytkownik z dostępem tylko do zasobu A, bez przypisań, w dwóch grupach, dane przeterminowane, pusta grupa). Seed wywoływany bez zaszytego `docker compose exec`.
6. **Przełączniki stanu** (feature flagi) jako deklaracje w manifeście.
7. **Kontrakt kont QA i logowania:** sesja bez rekonesansu poświadczeń. Obejmuje OTP z bazy lub stubu, limity logowania i reużycie tokenów, pułapki UI logowania, bramki po pierwszym logowaniu, konta QA bez 2FA.
8. **Adresy i proxy między usługami** konfigurowane z manifestu.
9. **Dostęp do bazy i logów usług dla wykonawcy**, żeby kontrakt dowodu (spec #3) mógł sprawdzać efekt.
10. **Praca pod strażnikiem ścieżek:** wszystko w katalogu roboczym, narzędzia instalowane lokalnie.

## Non-goals

- Generator, raport z kontraktem dowodu, pętla naprawy, UI: osobne specy.
- Emulator lub build natywny; serwer lub PaaS.
- Własny wykonawca przeglądarkowy.
- Wiedza o jakimkolwiek konkretnym projekcie w rdzeniu lub w repo.

## Affected areas (if known)

Repo jest greenfield. Pierwszy konsument (wewnętrzny, wielorepozytoryjny: compose, NestJS, Angular, Expo web, FastAPI) dostarczy swój manifest we własnym repozytorium. Jego obecny skrypt środowiska działa jako kontrakt v2 skilla `om-prepare-test-env` (deskryptor `.ai/qa/test-env.json`). To punkt wyjścia do porównania.
