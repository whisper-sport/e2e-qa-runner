# Proces dostarczania zmian

## Cel

Ten plik opisuje, jak praca przechodzi od zgłoszenia do zmergowanego PR-a w tym repozytorium. Proces egzekwują skille agentów skonfigurowane w `.ai/agentic.config.json`; ludzie czytają go tutaj. PR-y celują w `main`; zgłoszenia i PR-y żyją w GitHubie, a każda operacja trackera, którą wykonują skille, jest zdefiniowana w `.ai/trackers/github.md` (edytuj ten plik, żeby rozszerzyć lub nadpisać zachowanie trackera).

Praca wchodzi dwiema drogami: jako swobodny brief zadania przekazany agentowi albo jako zgłoszenie w trackerze. Obie drogi zbiegają się w tej samej pętli review, tej samej bramce walidacji i tych samych bramkach merge'a.

Przed przyjęciem praca jest kształtowana: `om-brainstorm` zamienia pojedynczy pomysł lub pytanie w decyzję o dalszej drodze i brief, a skille specyfikacji (`om-spec-writing`, `om-auto-write-spec`) zamieniają funkcję w dokument projektowy, zanim cokolwiek powstanie. Briefy trafiają do `docs/specs/briefs/`, specyfikacje do `docs/specs/` (jedna funkcja = jeden spec, zgodnie z `CLAUDE.md`).

## Role

- **Autor**: człowiek lub agent, który pisze zmianę. Odpowiada za zgłoszenie od przejęcia do PR-a gotowego do merge'a.
- **Recenzent**: czyta diff i zatwierdza albo prosi o zmiany. Może to być człowiek albo skill `om-auto-review-pr`; w obu przypadkach obowiązuje checklista `om-code-review` oraz `CODE_REVIEW.md`. Recenzent jest też drugą osobą, której wymaga zmiana `risk-high`, i zatwierdza spec, gdy funkcja go wymaga.
- **Projektant**: odpowiada za przepływ i jego stany, zanim powstanie kod. Może to być człowiek, `om-ux-shape` do kształtowania albo `om-ux-review-pr` do przejścia po ekranach PR-a. Dotyczy wyłącznie zmian z interfejsem użytkownika (np. przyszłe UI historii runów).
- **Opiekun (maintainer)**: odpowiada za ochronę gałęzi, taksonomię etykiet, konfigurację, ten dokument oraz zainstalowane skille i ich lokalne nadpisania w `.ai/skills/`. Rozstrzyga, gdy bramki są w konflikcie. Pełni rolę release managera, dopóki zespół nie wskaże innej osoby.

## Cykl życia zgłoszenia

| Etap | Co się dzieje | Kto prowadzi | Gotowe, gdy |
|---|---|---|---|
| Discovery | Pomysł, pytanie lub problem jest omawiany, zanim powstanie jakikolwiek artefakt: problem jest kwestionowany, alternatywy (łącznie z niebudowaniem niczego) są ważone, a rozmowa kończy się decyzją o dalszej drodze: odpowiedzią, zgłoszeniem, briefem do speca albo bezpośrednią zmianą. | `om-brainstorm` albo człowiek | Rozmowa rozstrzygnięta; brief zapisany, gdy praca idzie dalej |
| Intake | Zgłoszenie lub brief trafia do GitHuba z wystarczającą ilością szczegółów: co jest nie tak lub czego brakuje, dla kogo i jak wygląda „gotowe”. `om-prepare-issue` zakłada je z etykietami SDLC. | Każdy, `om-prepare-issue` | Zgłoszenie istnieje |
| Triage | Potwierdzenie, że problem jest realny, wciąż nienaprawiony na `main` i nieprzejęty ani nieobjęty otwartym PR-em. Tylko odczyt; łańcuch kończy się czysto, gdy nie ma nic do zrobienia. | `om-verify-in-repo` albo człowiek | Potwierdzone do realizacji albo zamknięte bez akcji |
| Claim | Autor przejmuje zgłoszenie, żeby równoległe agenty się wycofały. Zobacz protokół przejęcia niżej. | `om-fix` / `om-auto-create-pr` albo człowiek | Przejęcie widoczne na zgłoszeniu |
| Design | Dla zmiany widocznej dla użytkownika przepływ i jego stany są ustalone przed kodem: co ekran robi, gdy jest pusty, ładuje się, ma błąd lub brak uprawnień, i czego zmiana celowo nie robi. Zgłoszenie bez UI pomija ten etap. | `om-ux-shape` albo projektant | Przepływ i stany ustalone albo zgłoszenie nie dotyczy UI |
| Implement | Wskazanie minimalnej powierzchni zmiany (`om-root-cause`, tylko odczyt), następnie implementacja z testami regresji i uruchomienie bramki walidacji. Briefy bez zgłoszenia idą przez `om-auto-create-pr`, który planuje, implementuje fazami w izolowanym worktree i uruchamia tę samą bramkę. | `om-root-cause` + `om-fix`, `om-auto-create-pr` albo autor | Zmiana kompletna, bramka walidacji zielona |
| PR | Commit, push i otwarcie PR-a do `main` z ujednoliconymi etykietami. Na gałęzi prowadzonej ręcznie `om-check-and-commit` uruchamia bramkę, poprawia oczywiste rozjazdy i pushuje, gdy jest zielono. | `om-open-pr`, `om-auto-create-pr` albo `om-check-and-commit` | Otwarty PR z etykietami |
| Pętla review | Recenzent czyta diff według checklisty `om-code-review` i `CODE_REVIEW.md`, zatwierdza albo prosi o zmiany. Uwagi są adresowane (`om-auto-continue-pr` wznawia PR-y agentów z planu, a PR bez planu adoptuje, odtwarzając plan z jego kontekstu), po czym PR wraca do review aż do zatwierdzenia. Zmiana z UI dostaje też przejście projektowe `om-ux-review-pr`; jest ono doradcze i nie wstrzymuje merge'a. | `om-auto-review-pr` (pojedynczy PR), `om-review-prs` (przegląd wszystkich), `om-ux-review-pr` albo człowiek | Złożone zatwierdzające review |
| Merge | `om-merge-buddy` raportuje (tylko odczyt), które PR-y można zmergować teraz, a które są blisko, ale zablokowane. `om-approve-merge-pr` ponownie sprawdza każdą bramkę, zatwierdza i robi squash-merge. | `om-merge-buddy` + `om-approve-merge-pr` albo człowiek | PR zmergowany squashem do `main` |
| Porządki po merge'u | Zamknięcie zgłoszeń naprawionych przez zmergowany PR; komentarz na zgłoszeniach, których PR-y zamknięto bez merge'a; zamiana pozostałych próśb i uwag z review na zgłoszenia uzupełniające. | `om-close-fixed-issues`, `om-followup-issue-from-pr` | Tracker uzgodniony, zgłoszenia uzupełniające założone |

Po merge'u ten proces się kończy. Wydania, testy dymne i wycofywanie zmian należą do procesu wydawniczego repozytorium, nie do tego dokumentu: opiekun (albo release manager, gdy zespół go wskaże) przygotowuje changelog przez `om-auto-update-changelog` i uzgadnia tracker przez `om-close-fixed-issues`.

## Maszyna stanów etykiet

Etykiety pipeline'u wzajemnie się wykluczają: PR nosi co najwyżej jedną i mówi ona, gdzie PR jest w przepływie.

- Gotowy PR (nie draft) nosi `review`.
- Recenzent go przesuwa: prośba o zmiany → `changes-requested`; po poprawkach wraca do `review`; zatwierdzenie → `merge-queue`.
- Etykietę `qa` ustawia wyłącznie osoba testująca ręcznie, na czas testów; wynik to powrót do `merge-queue` z `qa-approved` albo `qa-failed`. Skille automatyczne proszą o QA etykietą `needs-qa`; nigdy nie ustawiają `qa`.
- `blocked` i `do-not-merge` ustawiają i zdejmują ludzie; zatrzymują przepływ, gdziekolwiek jest.

| Grupa | Etykiety | Wyłączność | Znaczenie |
|---|---|---|---|
| Pipeline | `review`, `changes-requested`, `qa`, `qa-failed`, `merge-queue`, `blocked`, `do-not-merge` | jedna naraz | Stan w przepływie |
| Kategoria | `bug`, `feature`, `refactor`, `security`, `dependencies`, `documentation` | addytywne | Rodzaj zmiany |
| Meta | `needs-qa`, `skip-qa`, `qa-approved`, `qa-self-verified`, `in-progress` | addytywne | Sygnały procesu |
| Priorytet | `priority-low`, `priority-medium`, `priority-high`, `priority-extreme` | jedna naraz; brak = medium | Pilność pracy |
| Ryzyko | `risk-low`, `risk-medium`, `risk-high` | jedna naraz; brak = medium | Zasięg skutków zmiany |

Priorytet mówi, jak pilna jest praca; ryzyko mówi, jak niebezpieczne jest wdrożenie zmiany. PR dziedziczy oba po zgłoszeniu źródłowym, chyba że zakres wyraźnie się zmienił. Gdy skill automatyczny dodaje lub zmienia etykietę pipeline'u albo meta, zostawia krótki komentarz z uzasadnieniem.

Gdy priorytet nie jest ustawiony, wywnioskuj go:

- `priority-extreme`: aktywny incydent bezpieczeństwa (np. wyciek danych projektu-konsumenta do publicznego repo).
- `priority-high`: wzmocnienie bezpieczeństwa albo regresja blokująca wydanie.
- `priority-medium`: zwykłe poprawki i nowe funkcje (również domyślne odczytanie braku etykiety).
- `priority-low`: kosmetyka, sama dokumentacja, podbicia zależności, porządki uzupełniające.

Gdy ryzyko nie jest ustawione, wywnioskuj je:

- `risk-high`: format manifestu konsumenta, kontrakt seeda i kont QA, kontrakt dowodu i werdykt, obsługa poświadczeń, granica izolacji generatora scenariuszy, zmiany przekrojowe. Szczegóły w `BACKWARD_COMPATIBILITY.md`.
- `risk-medium`: zwykła zmiana w jednym obszarze, dostarczona z testami (również domyślne odczytanie braku etykiety).
- `risk-low`: sama dokumentacja, same testy, literówki, izolowana kosmetyka.

Przy sprzecznych sygnałach wybierz wyższą etykietę i uzasadnij to w komentarzu. PR `risk-high` uruchamia bramki:

| Obszar `risk-high` | Co PR musi zawierać |
|---|---|
| Format manifestu i kontrakty konsumenta | ścieżkę migracji dla istniejących manifestów, zgodnie z `BACKWARD_COMPATIBILITY.md` |
| Poświadczenia i konta QA | dowód, że sekrety nie trafiają do logów, raportów ani repo; review drugiej osoby |
| Kontrakt dowodu i werdykt | test pokazujący, że krok bez sprawdzonego efektu (DOM, sieć, baza) daje FAIL, a nie PASS |
| Izolacja generatora scenariuszy | dowód, że generator nadal nie widzi pamięci agenta, trackera ani zgłoszeń defektów |
| Dowolny `risk-high` | `om-code-review` blokuje bez powyższych dowodów, chyba że opiekun uchyli to wprost na PR-ze |

Jedna etykieta istnieje poza taksonomią: `do-not-close`, nakładana przez ludzi na zgłoszenia, których skille porządkowe nie mogą automatycznie zamknąć. Skille tylko ją czytają. W tym repo nie jest jeszcze utworzona; utwórz ją ręcznie, gdy zajdzie potrzeba.

## QA

Bramka QA jest wyłączona (`qaGate: false`): etykieta `needs-qa` jest doradcza i nie blokuje merge'a. Twardymi blokadami pozostają `qa-failed`, `do-not-merge` i `blocked`. Gdy w repo pojawi się kod z interfejsem (np. UI historii runów), rozważ włączenie bramki w `.ai/agentic.config.json` i uzupełnienie tej sekcji.

## Protokół przejęcia

Zanim agent zmieni zgłoszenie lub PR, przejmuje je trzema sygnałami: przypisuje siebie, dodaje etykietę `in-progress` i publikuje komentarz z informacją, co robi. Agent, który zastanie istniejące przejęcie, wycofuje się zamiast kolidować. PR z `in-progress` jest też pomijany przez narzędzia merge'a.

`in-progress` oznacza **aktywną pracę**. Przejęcie jest zwalniane po zakończeniu pracy, zarówno przy sukcesie, jak i porażce. Nieaktualne `in-progress` bez świeżej aktywności może zdjąć opiekun.

Taksonomia tego repo nie zawiera etykiety `ci-monitoring`: skille, które chcą ją nałożyć po zakończeniu pracy, pomijają ją (strażnik etykiet loguje pominięcie), a kontynuację wyniku CI raportują komentarzem.

### Raportowanie jest niezależne od CI

Agenci nakładają etykiety, składają review i publikują komentarze **od razu po zakończeniu pracy**, nie czekając na zielone CI. Review złożone, gdy sprawdzenia jeszcze trwają, mówi to wprost: merge wstrzymuje ochrona gałęzi, a zatwierdzenie dotyczy kodu, nie zielonego przebiegu. Wynik CI przychodzi później jako komentarz uzupełniający, który koryguje etykietę pipeline'u, jeśli wynik zmienia werdykt.

Oczekiwanie na wynik jest ograniczone przez `ci.maxWaitMinutes` (domyślnie 40). Gdy limit minie, agent przestaje czekać, uruchamia lokalną bramkę walidacji jako własny dowód, publikuje go razem z nazwami wciąż trwających sprawdzeń i jasną informacją, że kolejnego komentarza nie będzie, po czym kończy.

Czerwony sygnał nie skraca review. Padające wymagane sprawdzenie albo konflikt są zbierane jako **blokujące uwagi** i raportowane razem z pełnym review, nigdy zamiast niego. Werdykt to wtedy wciąż `changes-requested`.

Nic z tego nie dotyka bramek merge'a. Wczesne raportowanie jest bezpieczne; wczesny merge nie.

## Kontrakt automatyzacji

Skille `om-auto-*` prowadzą ten proces bez nadzoru i dają się łączyć w łańcuch: każdy przyjmuje artefakt poprzedniego (numer zgłoszenia, ścieżkę speca albo numer PR-a z linii `PR: #<number> (link: <url>)`, którą emituje każdy skill tworzący PR) i wykrywa rozpoczętą już pracę, kontynuując ją zamiast otwierać duplikat. Zakończony autonomiczny run zostawia **gotowy** (nie draft), w pełni oetykietowany PR: jedna etykieta pipeline'u, kategoria, meta QA, jeden priorytet, jedno ryzyko, plus komentarz z podsumowaniem runu. Drafty są zarezerwowane dla stanów jawnie niekompletnych: PR-ów z samym specem, przerwanych przekazań albo autonomicznych założeń do potwierdzenia przez człowieka. Automatyzacja nakłada `qa-approved` wyłącznie w ramach wyjątku self-QA (`om-auto-qa-pr --self-qa-signoff`, zawsze razem z `qa-self-verified`, nigdy na PR-ze `risk-high`).

Ponieważ repozytorium jest publiczne, każdy artefakt automatyzacji (commit, PR, komentarz, raport, zrzut ekranu) podlega zasadzie z `CLAUDE.md`: żadnych nazw, adresów, kont, haseł, SHA, tras API ani opisów defektów projektów-konsumentów.

## Bramka walidacji

Repozytorium jest na etapie dokumentów projektowych i nie ma jeszcze kodu ani toolchainu, więc lista komend walidacji w `.ai/agentic.config.json` jest pusta. Do tego czasu review opiera się na `CODE_REVIEW.md`.

Gdy pojawi się toolchain (typecheck, lint, testy, build), dopisz komendy do `validation.commands` w kolejności wykonania i jednocześnie zaktualizuj tę sekcję. Każde niezerowe wyjście komendy oblewa bramkę i blokuje PR.

## Zmiany procesu

Ten dokument i `.ai/agentic.config.json` opisują ten sam proces: zmieniaj je razem i uruchom ponownie skill `om-setup-agent-pipeline`, gdy zmieni się toolchain albo taksonomia etykiet.

Odstępstwa per skill (dodatkowe reguły review, inny szablon opisu PR-a, dodatkowy krok bramki) należą do lokalnego skilla o tej samej nazwie w `.ai/skills/<skill-name>/SKILL.md`. Ma on pierwszeństwo przed zainstalowanym skillem i może go rozszerzać; lokalne reguły wygrywają, ale nie mogą nadać uprawnień, których zabraniają reguły bezpieczeństwa zainstalowanego skilla.
