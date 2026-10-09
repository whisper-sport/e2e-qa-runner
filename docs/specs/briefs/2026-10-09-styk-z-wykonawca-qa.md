# Spec #1b: kontrakt scenariusza MD i styk środowiska z wykonawcą QA

- Date: 2026-10-09 (zakres rozszerzony tego samego dnia po próbie na sucho)
- Category: feature
- Priority signal: high. Bez tego cel „przeklikaj feature według podanych kroków” jest nieosiągalny, a środowisko z #1a wymaga człowieka
- Risk signal: medium. Zależność od wnętrza skilli `om-*` albo koszt własnej nakładki wykonawcy
- Routing: Next: om-spec-writing "Kontrakt scenariusza MD i styk środowiska z wykonawcą QA — brief: docs/specs/briefs/2026-10-09-styk-z-wykonawca-qa.md"
- Wydzielony ze speca #1 decyzją Q1 (2026-10-09). Zależy od: [`docs/specs/2026-10-08-manifest-srodowiska-multi-repo.md`](../2026-10-08-manifest-srodowiska-multi-repo.md) (spec #1a, `env.json` v1)

## Problem

Spec #1a stawia środowisko i opisuje je w `env.json`, ale nikt go automatycznie nie przechodzi. Brakuje dwóch rzeczy:

1. **Kontraktu scenariusza.** Nie da się powiedzieć „przeklikaj ten feature według tych kroków” tak, żeby narzędzie samo wiedziało, jakie środowisko postawić.
2. **Styku z wykonawcą.** Istniejący wykonawca (`om-auto-qa-pr` w trybie lokalnym z `agent-browser`) potrafi przejść zmianę w przeglądarce. Sam jednak stawia środowisko przez `om-prepare-test-env` i zna tylko jeden `baseUrl`.

Próba na sucho (2026-10-09) pokazała, że realne scenariusze obejmują **kilka powierzchni naraz**: panel web, mobile-web, API i bazę. Jeden `baseUrl` ich nie opisze.

## Zakres

Ścieżka end-to-end: **plik scenariusza → środowisko (#1a) → wykonawca → werdykt z dowodami.**

- **Kontrakt scenariusza MD.** Nagłówek (front matter) deklaruje:
  - `state`: stan seeda z manifestu (standardowy albo własny);
  - `toggles`: wartości przełączników;
  - `personas`: persony używane w krokach;
  - `targets`: powierzchnie UI, czyli usługi z `target` w manifeście #1a (`web`, `mobile-web`);
  - `access`: pozostałe usługi, z których korzystają kroki (np. API przez HTTP, baza przez `dataAccess`). To osobne pojęcie, żeby nie mieszać go z `target` z #1a;
  - opcjonalnie `refs`.

  Treść to kroki do przejścia. Narzędzie składa z nagłówka plik runu #1a, stawia środowisko i uruchamia wykonawcę.
- **Wiele targetów w jednym scenariuszu.** Wykonawca dostaje wszystkie zadeklarowane powierzchnie, persony (z powierzchnią logowania każdej) i dostęp do API i bazy.
- **Parametry przeglądarki per target.** Język interfejsu (`navigator.language`) i viewport mobilny pochodzą z `env.json` (`targets[].browser`, już w kontrakcie #1a), a nie ze scenariusza.
- **Werdykt z dowodami** w zakresie, w jakim daje go wykonawca. Ocena dowodów i kontrakt dowodu to nadal spec #3.

## Resolved unknowns

| Question | Answer |
|----------|--------|
| Runtime | Ten sam CLI co #1a (Node/TypeScript, `npx`) (Q2) |
| Sesja kont | Tryb i powierzchnia per persona z #1a: `ui` → poświadczenia i referencja hasła; `session` → artefakt sesji do wstrzyknięcia przez wykonawcę (Q8) |
| Wejście środowiska | `env.json` v1 z #1a; repozytoria ze źródłem `cache` albo `path` (Q10) |
| Warianty | Jeden wariant; kontrakt przewiduje wiele (Q6) |
| Parametry przeglądarki | `targets[].browser` w `env.json`, nie w scenariuszu |

## Q4 otwarte ponownie: kształt styku z wykonawcą

Decyzja z 2026-10-09 („deskryptor `test-env.json` v1, zero zmian w `om-*`”) **nie wystarcza**, bo v1 ma jeden `baseUrl`, a scenariusz obejmuje kilka powierzchni. Dopuszczalne opcje:

- **(a) Zmiany w `om-auto-qa-pr` / `om-prepare-test-env`:** deskryptor z wieloma targetami, tryb „użyj gotowego środowiska” (`--env reuse`), wejście w postaci scenariusza zamiast diffu.
- **(b) Własna nakładka wykonawcy w tym repo:** sterowanie `agent-browser` przez jego deskryptor, bez `om-auto-qa-pr`. To nie jest własny wykonawca przeglądarkowy (non-goal), tylko własny prowadzący scenariusz.
- **(c) Hybryda:** deskryptor v1 per target (jedno wywołanie `om-auto-qa-pr` na powierzchnię) ze wspólnym kontekstem. Ryzyko: kroki przechodzące między powierzchniami (np. akcja w panelu, efekt w aplikacji) rozpadają się na osobne wywołania.

## Ustalenia z rozpoznania (wejście do speca)

Pochodzą z lektury skilli `om-auto-qa-pr` i `om-prepare-test-env` i dotyczą opcji (a) i (c):

- `om-auto-qa-pr` nigdy nie stawia aplikacji sam. Woła `om-prepare-test-env`, który **najpierw uruchamia zapisany `test-env-up.sh`**. Skrypt **bez** markera `om-prepare-test-env: generated entrypoint` jest traktowany jako własne narzędzie repo i nie jest nadpisywany. To punkt wpięcia dla adaptera, który woła `e2e-qa attach` i wypisuje linie wyniku kontraktu entrypointu v2.
- `startedByThisRepo: false` oraz `--keep-env` sprawiają, że wykonawca nie zrywa środowiska.
- Tryb lokalny `om-auto-qa-pr` sam wyprowadza scenariusz z `git diff <base>...HEAD`. Kontrakt scenariusza MD odwraca to źródło: kroki są dane, nie wyprowadzane. To argument za opcją (a) albo (b).
- Mitygacje znalezione w przeglądzie, aktualne dla każdej opcji:
  - adaptery i pliki dopisane do worktree nie mogą pojawić się w `git status` (`info/exclude`, `skip-worktree`); na ścieżkach lokalnych tylko pliki ignorowane, jak `files` w #1a;
  - shim z absolutną ścieżką do binarki zamiast `npx`;
  - limit czasu wykonawcy;
  - detektor `environment-compromised`;
  - brak raportu oznacza błąd, nigdy PASS;
  - powierzchnia bez pokrycia daje `not-covered`, nigdy ciszę;
  - `mobile-web` niesie zastrzeżenie „web ≠ native”.

## Open Questions dla speca #1b

- **Q4 (ponownie):** opcja (a), (b) czy (c)?
- **Los klucza `qa` w manifeście** (zarezerwowanego w #1a). Przy styku per diff służył mapowaniu `qa.<repo>.surface`. Przy kontrakcie scenariusza może okazać się zbędny albo przejąć domyślne `targets`/`access` dla repozytoriów bez scenariusza.
- **Spięcie z orkiestratorem:** kto po runie agenta tworzy plik runu albo scenariusza (orkiestrator, skill agenta, człowiek)? Worktree orkiestratora jako ścieżka lokalna wymaga `prepare: true` albo wcześniej przygotowanych zależności. Kto za to odpowiada?
- **Skąd scenariusz:** ręcznie pisany MD, katalog regresji (#2) czy oba? Format kroków: wolny tekst czy struktura (akcja, oczekiwany efekt, target)?
- **Wstrzyknięcie sesji** persony w trybie `session` do `agent-browser`.
- **Polecenie wykonawcy:** stałe czy konfigurowalne z adapterem per agent CLI?

## Non-goals

Ocena dowodów i raport z kontraktem dowodu (#3), generator scenariuszy (#2), własny silnik przeglądarki, emulator i build natywny.
