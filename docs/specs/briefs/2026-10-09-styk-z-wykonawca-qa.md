# Spec #1b: styk środowiska z istniejącym wykonawcą QA

- Date: 2026-10-09
- Category: feature
- Priority signal: high. Bez tego środowisko z #1a wymaga człowieka do testowania
- Risk signal: medium. Zależność od niewersjonowanego wnętrza skilli `om-*`
- Routing: Next: om-spec-writing "Styk środowiska z istniejącym wykonawcą QA — brief: docs/specs/briefs/2026-10-09-styk-z-wykonawca-qa.md"
- Wydzielony ze speca #1 decyzją Q1 (2026-10-09). Zależy od: [`docs/specs/2026-10-08-manifest-srodowiska-multi-repo.md`](../2026-10-08-manifest-srodowiska-multi-repo.md) (spec #1a, `env.json` v1)

## Problem

Spec #1a stawia środowisko i opisuje je w `env.json`, ale nikt go automatycznie nie przechodzi. Istniejący wykonawca (`om-auto-qa-pr` w trybie lokalnym z `agent-browser`) potrafi przejść zmianę w przeglądarce. Sam jednak stawia środowisko przez `om-prepare-test-env` i zna tylko jeden `baseUrl`.

## Resolved unknowns

| Question | Answer |
|----------|--------|
| Kształt styku | Deskryptor `test-env.json` v1 zgodny z `om-prepare-test-env` (jeden `baseUrl`); reszta w polu rozszerzenia i w `notes`. Zero zmian w skillach `om-*` na start (Q4) |
| Runtime | Ten sam CLI co #1a (Node/TypeScript, `npx`) (Q2) |
| Sesja kont | Tryb per persona z #1a: `ui` → `credentials` + `passwordEnv`; `session` → artefakt sesji do wstrzyknięcia przez wykonawcę (Q8) |
| Wejście | `env.json` v1 z #1a; repozytoria ze źródłem `cache` albo `path` (Q10) |
| Warianty | Jeden wariant; kontrakt przewiduje wiele (Q6) |

## Ustalenia z rozpoznania (do weryfikacji w specu)

Pochodzą z lektury skilli `om-auto-qa-pr` i `om-prepare-test-env`:

- `om-auto-qa-pr` nigdy nie stawia aplikacji sam. Woła `om-prepare-test-env`, który **najpierw uruchamia zapisany `test-env-up.sh`**. Skrypt **bez** markera `om-prepare-test-env: generated entrypoint` jest traktowany jako własne narzędzie repo i nie jest nadpisywany. To naturalny punkt wpięcia: adapter, który woła `e2e-qa attach` i wypisuje linie wyniku kontraktu entrypointu v2 (`TEST_ENV_STATUS`, `TEST_ENV_BASE_URL`, `TEST_ENV_DESCRIPTOR`, `TEST_ENV_REUSED`, `BROWSER_PROVIDER`, `BROWSER_INSTALLED`).
- `startedByThisRepo: false` oraz `--keep-env` sprawiają, że wykonawca nie zrywa środowiska.
- Tryb lokalny `om-auto-qa-pr` zawęża scenariusz do `git diff <base>...HEAD`. Wywołanie powinno działać w worktree repo ze zmianą, z `baseUrl` powierzchni, na której zmianę widać. Prowadzi to do mapowania `qa.<repo>.surface` w manifeście (klucz `qa` jest zarezerwowany w schemacie #1a).
- Ryzyka i mitygacje znalezione w przeglądzie:
  - adaptery i pliki dopisane do worktree nie mogą pojawić się w `git status`: `info/exclude` dla nowych plików i `skip-worktree` dla śledzonych. Ścieżki lokalne z #1a są tylko do odczytu, więc adaptery dla nich wymagają osobnej decyzji;
  - shim z absolutną ścieżką do binarki, która postawiła run, zamiast `npx`;
  - limit czasu wykonawcy;
  - detektor `environment-compromised`: wykonawca po błędzie adaptera próbuje postawić własne środowisko;
  - brak `report.json` oznacza błąd, nigdy PASS;
  - repo bez powierzchni (`surface: none`) daje `not-covered`, nigdy ciszę;
  - wszystkie wywołania `skipped` to nie PASS;
  - `mobile-web` niesie zastrzeżenie „web ≠ native” w `notes`.
- Propozycja upstream (poza zakresem): `om-auto-qa-pr --env reuse` → `om-prepare-test-env --mode reuse`.

## Open Questions dla speca #1b

- Jak wykonawca wstrzykuje sesję persony w trybie `session` do `agent-browser`, skoro deskryptor v1 zna tylko `credentials`: przez `notes` z poleceniem czy przez rozszerzenie deskryptora?
- Adaptery w ścieżkach lokalnych: dopuszczamy dopisywanie plików wykluczonych z gita czy wymagamy źródła `cache` dla repo testowanych przez wykonawcę?
- Polecenie wykonawcy: stałe (`claude -p "/om-auto-qa-pr …"`) czy konfigurowalne z adapterem per agent CLI?

## Non-goals

Ocena dowodów i raport z kontraktem dowodu (#3), generator scenariuszy (#2), własny wykonawca przeglądarkowy, zmiany w skillach `om-*`.
