# Manifest środowiska multi-repo z kontraktem seeda i kont QA

- Status: **szkielet — bramka Open Questions otwarta**
- Brief: [`docs/specs/briefs/2026-10-08-manifest-srodowiska.md`](briefs/2026-10-08-manifest-srodowiska.md)
- Wejście z eksperymentu: [`docs/research/2026-10-08-scenariusze-z-ac-probe.md`](../research/2026-10-08-scenariusze-z-ac-probe.md), sekcja „Problemy środowiska”

## 📝 TLDR

Zespół, którego zmiany wytwarzają agenci, nie ma dziś kim i czym przejść zmiany end-to-end: po runie agenta nikt nie stawia całego ekosystemu z kilku repozytoriów, a ręczne skrypty środowiska są zaszyte pod jedną funkcję. **Proponujemy** (stan przyszły) projekt-agnostyczne narzędzie, które z manifestu konsumenta stawia lokalnie środowisko w zadanych wersjach repozytoriów, doprowadza dane do nazwanego stanu, przygotowuje konta QA bez rekonesansu poświadczeń i przekazuje gotowe środowisko istniejącemu wykonawcy QA (`om-auto-qa-pr` w trybie lokalnym z `agent-browser`). Generator scenariuszy, raport z kontraktem dowodu, pętla naprawy i UI to osobne specy.

## ❓ Open Questions

Pytania, na które brief nie odpowiada (tabela „Resolved unknowns” jest przyjęta jako dana). Do czasu odpowiedzi spec nie przechodzi do projektu.

- **Q1. Podział speca.** Brief łączy dwie zdolności: (a) postawienie środowiska z manifestu z seedem i kontami, (b) uruchomienie na nim wykonawcy QA. (a) ma wartość sama w sobie (człowiek też może testować na tak postawionym stosie). Zostawiamy jeden spec czy dzielimy na #1a i #1b?
- **Q2. Postać i runtime narzędzia.** CLI w Node/TypeScript (dystrybucja `npx`), w Pythonie, w Go (jedna binarka), czy cienkie skrypty POSIX sh, jak entrypoint `om-prepare-test-env`?
- **Q3. Format i miejsce manifestu.** YAML, JSON (z JSON Schema) czy TOML? Leży w jednym z repozytoriów konsumenta (np. w repo „głównym”), czy w osobnym repo środowiska konsumenta?
- **Q4. Styk z wykonawcą QA.** `om-auto-qa-pr` zawsze woła `om-prepare-test-env`, a ten reużywa środowiska, gdy `.ai/qa/test-env.json` ma `status: running` i przechodzi sondy. Opcje: (a) narzędzie zapisuje deskryptor zgodny z v1 (jeden `baseUrl`, `services`, `credentials`), a dodatkowe usługi idą w `notes` lub w polu rozszerzenia; (b) proponujemy rozszerzenie deskryptora o wiele usług i kontrakt sesji (zmiana po stronie skilli `om-*`); (c) narzędzie omija `om-prepare-test-env` i woła wykonawcę z własnym kontekstem.
- **Q5. Wejście runu.** Jak opisujemy „zmianę do przetestowania”: (a) jawna lista `repo → ref` w poleceniu lub pliku runu; (b) lista PR-ów rozwiązywana do refów przez tracker; (c) obie? A dla paczki kilku PR-ów w jednym repo: narzędzie samo scala je w gałąź integracyjną, czy oczekuje gotowego refu?
- **Q6. Warianty stosu w jednym runie** (wymaganie 2: np. dwie wersje API na dwóch bazach). Wchodzą do speca #1 jako pełnoprawna funkcja (izolacja nazw projektów compose, przestrzenie portów), czy kontrakt je przewiduje, a implementacja przychodzi później?
- **Q7. Kontrakt seeda.** (a) rdzeń definiuje słownik standardowych stanów (np. `empty-tenant`, `user-scoped-to-one-resource`, `user-without-assignments`, `user-in-two-groups`, `expired-data`, `empty-group`), które konsument implementuje albo jawnie oznacza jako nieobsługiwane; (b) nazwy stanów są dowolne, a manifest tylko mapuje nazwę na polecenie konsumenta; (c) słownik jako zalecenie, nazwy dowolne.
- **Q8. Kontrakt kont i logowania.** (a) tylko referencje poświadczeń (jak dziś w `credentials` + `passwordEnv`), a wykonawca loguje się przez UI; (b) hook konsumenta zwraca gotową sesję (cookies / storage state / token), którą wykonawca wstrzykuje do przeglądarki, z reużyciem między krokami i OTP czytanym przez hook; (c) obie ścieżki, a manifest wybiera per konto.
- **Q9. Tryb hybrydowy i budżet pamięci.** Manifest deklaruje per usługa `runtime: host | container`, a narzędzie obsługuje oba. Budżet pamięci jest twardą bramką (odmowa startu, gdy suma deklarowanych limitów przekracza budżet), czy tylko ostrzeżeniem?
- **Q10. Źródło kodu repozytoriów.** Narzędzie samo klonuje i trzyma repozytoria w swoim katalogu roboczym (worktree z lokalnego klona-cache), czy przyjmuje ścieżki do istniejących lokalnych checkoutów (np. worktree orkiestratora)?

## 📝 Problem Statement

Uogólnione z eksperymentu i z briefu (szczegóły konsumenta zostają w jego repozytoriach):

- Skrypt środowiska zaszyty pod jedną funkcję: bramki gałęzi, sztywne porty, jeden projekt compose. Nie da się go odpalić na dowolnych wersjach.
- Regresja paczki kilkunastu PR-ów na drugim stosie skończyła się OOM-em, kolizjami portów, limitami logowania i brakiem OTP w dev.
- Seed woła `docker compose exec` na sztywno, fikstury trafiają w ograniczenia schematu, a feature flagi wymagają ręcznej edycji zagnieżdżonego JSON-a.
- Logowanie ma pułapki UI i bramki po pierwszym logowaniu; wykonawca traci czas i budżet prób logowania na rekonesans.
- Strażnik ścieżek agenta blokuje wszystko poza katalogiem roboczym.

Skutek: testowanie zmian agentów zależy od jednej osoby, a paczki wydań czekają.

## 📝 Proposed Solution (zarys — do uszczegółowienia po bramce)

Przykład poglądowy na fikcyjnym konsumencie **acme** (usługi `api`, `web`, `mobile-web`, `ml-service` plus infrastruktura `postgres`, `redis`):

1. Konsument utrzymuje manifest opisujący: repozytoria i ich przepisy na start, porty jako zmienne (nie stałe), zależności startu i migracje, stany seeda, przełączniki stanu, konta QA i sposób uzyskania sesji, adresy między usługami, dostęp do bazy i logów.
2. Operator (człowiek albo orkiestrator) uruchamia run: wskazuje wersje repozytoriów i nazwany stan danych.
3. Narzędzie bierze blokadę (jeden stos naraz), przygotowuje kod, stawia infrastrukturę i usługi w kolejności zależności, aplikuje stan danych i flagi, przygotowuje konta, sprawdza zdrowie i zapisuje deskryptor środowiska.
4. Narzędzie uruchamia istniejącego wykonawcę QA na tym środowisku (forma styku: Q4), po czym sprząta tylko to, co samo postawiło.

Odrzucone alternatywy (z briefu): workflow w orkiestratorze lub same skille, „nic nie budować”, serwer/PaaS na start, własny wykonawca przeglądarkowy.
