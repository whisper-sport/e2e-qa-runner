# Spec #1a: manifest środowiska multi-repo z kontraktem seeda i kont QA (container-first)

- Status: **projekt; przebudowa container-first (2026-10-09); wszystkie pytania zamknięte (Q1–Q13)**
- Brief: [`docs/specs/briefs/2026-10-08-manifest-srodowiska.md`](briefs/2026-10-08-manifest-srodowiska.md)
- Następny spec (#1b, kontrakt scenariusza i styk z wykonawcą QA): [`docs/specs/briefs/2026-10-09-styk-z-wykonawca-qa.md`](briefs/2026-10-09-styk-z-wykonawca-qa.md)
- Wejście z eksperymentu: [`docs/research/2026-10-08-scenariusze-z-ac-probe.md`](../research/2026-10-08-scenariusze-z-ac-probe.md), sekcja „Problemy środowiska”
- Wszystkie przykłady dotyczą fikcyjnego konsumenta **acme** (`api`, `web`, `mobile-web`, `ml-service`).

## 📝 TLDR

Zespół, którego zmiany wytwarzają agenci, nie ma dziś kim i czym przejść zmiany end-to-end. Po runie agenta nikt nie stawia całego ekosystemu z kilku repozytoriów, a ręczne skrypty środowiska są zaszyte pod jedną funkcję. **Proponujemy** CLI `e2e-qa` (Node/TypeScript, `npx`):

- czyta manifest YAML konsumenta;
- stawia lokalnie stos, w którym **każda usługa jest kontenerem**: obraz z rejestru dla zadanego SHA albo zbudowany z repo (własny cache albo checkout operatora);
- doprowadza dane do nazwanego stanu i przygotowuje konta QA poleceniami wykonywanymi w kontenerach;
- opisuje gotowe środowisko w maszynowym `env.json`.

Środowisko jest użyteczne samo w sobie, także dla ludzi. Przekazanie go wykonawcy QA to spec #1b. Generator scenariuszy, raport z kontraktem dowodu, pętla naprawy i UI to specy #2–#4.

## 📝 Decyzje

### Bramka Open Questions (2026-10-09)

| # | Pytanie | Decyzja |
|---|---|---|
| Q1 | Podział speca | Dwa specy: **#1a** (ten) = manifest, seed i konta QA; **#1b** = kontrakt scenariusza i styk z wykonawcą (brief obok) |
| Q2 | Runtime | Node/TypeScript, dystrybucja przez `npx` |
| Q3 | Format manifestu | YAML walidowany JSON Schema |
| Q3b | Miejsce manifestu | Dowolne: narzędzie przyjmuje ścieżkę; konwencja leży poza rdzeniem |
| Q4 | Styk z wykonawcą | **Dotyczy #1b; otwarte ponownie** (scenariusz obejmuje kilka powierzchni). #1a dostarcza dane przez `env.json` z wieloma `targets` |
| Q5 | Wejście runu | Rdzeń: refy `repo → ref` (narzędzie nie zna trackera). Nakładka: PR rozwiązywany do refu |
| Q5b | Paczka PR-ów | Gotowy ref: gałąź integracyjną dostarcza operator lub orkiestrator; narzędzie nie scala |
| Q6 | Warianty stosu | Kontrakt teraz (schemat przewiduje warianty), implementacja kilku wariantów później |
| Q7 | Stany seeda | Słownik zalecany: standardowe nazwy z gwarancjami plus dowolne stany własne |
| Q8 | Sesja kont QA | Tryb i powierzchnia per persona: logowanie przez UI (referencje poświadczeń) albo hook sesji konsumenta |
| Q9 | Budżet pamięci | Twarda bramka z jawną flagą obejścia |
| Q10 | Kod repozytoriów | Ścieżka lokalna, gdy podana; inaczej własny cache z worktree per run |

### Container-first (2026-10-09)

| # | Decyzja |
|---|---|
| C1 | Każda usługa to kontener. Tryb `runtime: host` wypada z v1 (zob. „Poza zakresem”). Znikają supervisor, procesy hosta, `prepare`, `requires` i pomiar RSS. Żywotność runu wynika z projektów compose i `state.json` |
| C2 | Źródło obrazu usługi aplikacyjnej: `image` (szablon tagu, np. z SHA) **albo** `build` (Dockerfile z worktree z cache lub ze ścieżki lokalnej, łącznie z niezacommitowanymi zmianami). Domyślnie najpierw `image`, a gdy go brak, `build`. Pochodzenie i digest trafiają do `env.json`. Flaga wymusza jedno źródło |
| C3 | Kontrakt „obrazu zdatnego do QA” dla konsumenta: konfiguracja w runtime, healthcheck, migracje w obrazie, hooki QA przez `exec`, nieaktywne poza trybem QA |
| C4 | `files` to szablony montowane do kontenera z katalogu runu. Narzędzie niczego nie zapisuje do repozytoriów |
| C5 | `mem_limit` jest egzekwowany; bramka liczy tylko kontenery |
| C6 | Zdalny Docker (`DOCKER_HOST=ssh://…`) to rozszerzenie poza v1 |

### Pytania z przeglądu przebudowy (2026-10-09)

| # | Pytanie | Decyzja |
|---|---|---|
| Q11 | Sonda `health: { http }` (healthcheck compose działa w kontenerze) | Wymóg w kontrakcie obrazu: obraz z `health: { http }` musi mieć `curl` albo `wget` (obrazy alpine i nginx mają `wget` z busyboxa); brak to błąd. `health.command` zostaje dla obrazów bez tych narzędzi (np. distroless) |
| Q12 | Tryb `auto`, gdy rejestr odpowiada „denied”/401 | Zależnie od logowania: gdy operator ma zapisane logowanie do tego rejestru, 401/denied oznacza brak obrazu, więc narzędzie buduje lokalnie i zapisuje powód w `env.json`. Bez logowania: `blocked: registry-auth` z podpowiedzią `docker login` |
| Q13 | Platforma obrazu niezgodna z hostem | Build lokalny: platformę sprawdza się przy kontroli rejestru; przy niezgodności tryb `auto` buduje natywnie i zapisuje powód w `env.json`. To będzie norma przy runnerach CI `amd64` i hostach `arm64` |

## 📝 Problem Statement

Uogólnione z eksperymentu i z briefu (szczegóły konsumenta zostają w jego repozytoriach):

- Skrypt środowiska zaszyty pod jedną funkcję: bramki gałęzi, sztywne porty, jeden projekt compose. Nie da się go odpalić na dowolnych wersjach.
- Regresja paczki kilkunastu PR-ów na drugim stosie skończyła się OOM-em, kolizjami portów, limitami logowania i brakiem OTP w dev.
- Seed woła `docker compose exec` na sztywno, fikstury trafiają w ograniczenia schematu, a feature flagi wymagają ręcznej edycji zagnieżdżonego JSON-a.
- Logowanie ma pułapki UI i bramki po pierwszym logowaniu. Testujący (człowiek albo agent) traci czas i limit prób logowania na rekonesans.
- Proxy frontendu wskazuje sztywny adres API, a bundlery frontendu wypiekają adresy przy budowaniu.
- Strażnik ścieżek agenta blokuje wszystko poza katalogiem roboczym.

Skutek: testowanie zmian agentów zależy od jednej osoby, a paczki wydań czekają.

## 📝 Proposed Solution

Wejścia i wyjście:

- **Manifest** (konsument): stała topologia: repozytoria, usługi z obrazami, zależności, migracje, hooki QA, stany seeda, przełączniki, konta i logowanie, dostęp do danych.
- **Plik runu** (operator albo orkiestrator): część zmienna: refy lub ścieżki lokalne, stan danych, wartości przełączników, opcjonalne wymuszenie źródła obrazów.
- **Konfiguracja operatora** (opcjonalna, w katalogu roboczym): kolejka, budżet, retencja.
- **Wyjście:** działający stos kontenerów, `env.json` (maszynowy opis środowiska) i `e2e-qa status` dla człowieka.

Przebieg `e2e-qa up run.yaml`:

1. Wczytanie manifestu ze ścieżki z runu i walidacja (schemat + reguły semantyczne).
2. Preflight: Docker i Compose v2 działają (oraz Buildx, gdy będzie budowanie), dostępny jest `git`, stan i przełączniki z runu są zdefiniowane.
3. Przygotowanie kodu. Dla każdego repo: ścieżka lokalna (tylko odczyt) albo fetch do cache (z blokadą per repo) i worktree na podany ref. PR-y z nakładki są wcześniej rozwiązywane do refów.
4. Kolejka „jeden run naraz”, potem twarda bramka pamięci.
5. Obrazy: dla każdej usługi aplikacyjnej rejestr albo budowanie (z cache budowania Dockera); zapis źródła i digestu.
6. Przydział portów publikowanych, sekretów i render `files` do katalogu runu. Wygenerowanie pliku compose (etykiety runu, `mem_limit`, healthchecki, jednorazowe usługi migracji, montowania).
7. `docker compose up --wait`: Compose sam pilnuje kolejności (`service_healthy`, `service_completed_successfully` dla migracji) i zdrowia.
8. Hooki QA przez `exec`: seed, przełączniki `command`, reset limitów logowania, sesje dla person w trybie `session`.
9. Zapis `env/<variant>.json` i plików poświadczeń; wypisanie linii wyniku.

`e2e-qa down` sprząta tylko projekty compose tego runu (po etykietach) i jego katalogi.

**Odrzucone alternatywy.** Z briefu: workflow w orkiestratorze lub same skille, „nic nie budować”, serwer/PaaS na start, własny wykonawca przeglądarkowy. Z bramki: jeden spec dla środowiska i wykonawcy, scalanie paczek przez narzędzie, zamknięty słownik stanów, wyłącznie logowanie przez UI. Z decyzji container-first: **tryb hybrydowy (usługi aplikacyjne na hoście).** Dawał szybki start bez obrazów, ale wymagał supervisora, grup procesów, `prepare` na cudzych checkoutach, zapisu plików do repozytoriów operatora i bramki pamięci opartej na deklaracjach. Kontenery dają izolację, egzekwowane limity i jedną ścieżkę na serwer. Kosztem jest wymóg obrazu zdatnego do QA po stronie konsumenta.

## 📝 Research: container-first u liderów

| Narzędzie | Co robią dobrze | Co bierzemy | Czego nie bierzemy |
|---|---|---|---|
| Docker Compose | `-p` izoluje projekty; `depends_on` z `service_healthy` i `service_completed_successfully`; `up --wait`; `mem_limit`; etykiety; `logs --timestamps` | Compose jako silnik: kolejność, zdrowie, migracje jako usługi jednorazowe, logi i sprzątanie po etykietach. Narzędzie generuje plik compose, a nie orkiestruje samo | `profiles`, `include`, `watch` (konsument nie pisze compose, tylko manifest) |
| Dev Containers (`devcontainer.json`) | Usługa z `image` albo `build`; nazwane hooki cyklu życia (`postCreateCommand`, `postStartCommand`) wykonywane w kontenerze | Wybór `image`/`build` per usługa; hooki QA jako polecenia w kontenerze o stałych nazwach | Integracja z IDE, features, montowanie całego repo jako workspace |
| Tilt | `docker_build` per usługa z cache, przebudowa tylko zmienionych obrazów | Budowanie z cache Dockera i `cache-from` obrazu z rejestru | Live update, dashboard, Kubernetes |
| Testcontainers | Strategie oczekiwania, losowe porty, sprzątanie po etykietach (Ryuk) | Porty publikowane przydzielane per run, sprzątanie wyłącznie po etykiecie `e2e-qa.run` | Cykl życia sterowany z kodu testów |
| Seedery Rails/Laravel, `cy.task` | Nazwane seedery wołane poleceniem aplikacji | Stan danych = polecenie w obrazie z nazwą stanu | Fikstury SQL pisane przez narzędzie |
| Playwright `storageState` / `cy.session` | Logowanie raz, zapis sesji, reużycie | Tryb `session`: hook zwraca sesję, narzędzie ją buforuje | Logowanie programowe w rdzeniu |

Wniosek: kolejność startu, zdrowie, limity i logi daje Compose, a wybór obrazu albo budowania daje wzorzec Dev Containers. Wartością tego narzędzia zostaje: **wiele repozytoriów w zadanych refach → obrazy**, **semantyczny słownik stanów i person** oraz **kontrakt obrazu zdatnego do QA**.

## 📝 Architecture

```mermaid
flowchart LR
    classDef newC fill:#2f6feb,color:#fff,stroke:#1b4fb0
    classDef extC fill:#e5e7eb,color:#111,stroke:#9ca3af
    classDef planC fill:#fff,color:#111,stroke:#2f6feb,stroke-dasharray:4

    n1["Plik runu"]:::newC
    n2["Manifest YAML (konsument)"]:::extC
    n3["e2e-qa CLI"]:::newC
    n4["Cache repo / ścieżki lokalne (tylko odczyt)"]:::newC
    n5["Rejestr obrazów"]:::extC
    n6["docker build (cache)"]:::extC
    n7["Wygenerowany compose + files"]:::newC
    n8["Docker Compose: kontenery, migracje, zdrowie"]:::extC
    n9["Hooki QA w obrazach (exec)"]:::extC
    n10["env.json"]:::newC
    n11["#1b: scenariusz i wykonawca"]:::planC

    n1 --> n3
    n2 --> n3
    n3 --> n4
    n3 -->|"image"| n5
    n4 -->|"build"| n6
    n3 --> n7
    n7 --> n8
    n3 -->|"exec"| n9
    n3 --> n10
    n10 -.-> n11
```

Niebieskie elementy są nowe, szare już istnieją, a przerywana ramka to praca planowana. Narzędzie nie uruchamia żadnego procesu aplikacji na hoście. Rdzeń nie zna żadnego wykonawcy ani trackera. Jedynym punktem styku dla #1b jest `env.json`.

| Moduł | Odpowiedzialność |
|---|---|
| `manifest` | Parsowanie YAML, JSON Schema (`schema/manifest.v1.json`, `schema/run.v1.json`), reguły semantyczne |
| `repos` | Ścieżki lokalne (tylko odczyt) albo cache lustrzany, fetch i worktree; nakładka PR → ref |
| `images` | Rozwiązanie szablonu tagu, sprawdzenie rejestru, pull z limitem czasu, budowanie z cache, digest |
| `compose` | Przydział portów, placeholdery, render `files`, generowanie pliku compose (etykiety, limity, healthchecki, migracje), `up --wait`, `down` |
| `gate` | Kolejka i twarda bramka pamięci |
| `state` | Słownik stanów, hooki QA przez `exec`, persony, przełączniki, sesje, OTP, dostęp do danych |
| `output` | `env.json`, pliki poświadczeń, `status`, `logs` z maskowaniem |

## 📝 Kontrakt obrazu zdatnego do QA (dla konsumenta)

Narzędzie zakłada, że obraz każdej usługi aplikacyjnej spełnia pięć warunków. `validate` sprawdza te, które da się sprawdzić statycznie z manifestu. `images` i `up` sprawdzają obraz (`HEALTHCHECK`), a reszta wychodzi przy hookach jako czytelne błędy.

1. **Konfiguracja w runtime.** Adresy innych usług, originy, flagi i sekrety obraz czyta przy starcie kontenera ze zmiennych środowiskowych albo z montowanych plików. Żaden adres nie jest zaszyty przy budowaniu.
   - Frontend serwowany statycznie stosuje wzorzec **runtime config**: entrypoint kontenera generuje `env.js` albo `config.json` ze zmiennych, a aplikacja ładuje go przed startem. Alternatywnie narzędzie montuje gotowy `config.json` przez `files`.
   - **Build-time env bundlerów łamie kontrakt.** Zmienne wypiekane przy budowaniu (np. `EXPO_PUBLIC_*`, `import.meta.env` w Vite, `process.env.*` zastępowane przez bundler) dają obraz przypięty do jednego adresu, więc nie da się go użyć z portami przydzielanymi per run. Takie repo wymaga przejścia na runtime config, zanim jego usługa wejdzie do manifestu.
   - Z tego samego powodu `build.args` nie mogą zawierać placeholderów `${services.*}` ani `${secret.*}` (błąd walidacji).
2. **Healthcheck.** Obraz ma `HEALTHCHECK` albo manifest podaje `health` (`http` wymaga `curl` albo `wget` w obrazie, Q11). Brak obu wykrywa narzędzie po rozwiązaniu obrazu (`images`/`up`, przez `docker image inspect`) i zgłasza `error: no-healthcheck`, chyba że manifest jawnie deklaruje `health: { none: true }`. Wtedy gotowość = kontener działa, a narzędzie wypisuje ostrzeżenie. **Sonda `health: { http }`** wykonuje się w kontenerze przez `curl -fsS` albo `wget -q -O-` (w tej kolejności), więc obraz musi mieć jedno z nich. Obrazy alpine i nginx mają `wget` z busyboxa. Obraz bez obu narzędzi (np. distroless) używa `health: { command: [...] }` (Q11).
3. **Migracje jako polecenie w obrazie.** `migrate.command` uruchamia się jako jednorazowa usługa z tego samego obrazu, przed startem usługi.
4. **Hooki QA jako polecenia w obrazie**, wykonywane przez `docker compose exec -T` w działającym kontenerze usługi: `seed`, `login.otp`, `login.session`, `login.rateLimitReset`, przełączniki `command`, `dataAccess`. Hook działa w cgroup usługi, więc jego pamięć liczy się do `mem_limit` tej usługi. `memory` usługi z hookami musi mieć zapas na seed.
5. **Hooki QA nieaktywne poza trybem QA.** Narzędzie ustawia w każdym kontenerze aplikacyjnym `E2E_QA_MODE=1`. Polecenia QA muszą odmawiać działania, gdy tej zmiennej nie ma, z **zarezerwowanym kodem wyjścia 78** (`EX_CONFIG`) i komunikatem na stderr. Dzięki temu narzędzie odróżnia „hook nieaktywny” (78) od „brak polecenia w obrazie” (126/127 zwrócone przez `exec`) i od zwykłego błędu hooka (każdy inny kod). Obraz produkcyjny może je więc zawierać bez ryzyka, a najlepiej w ogóle ich nie zawiera (osobny target `qa` w Dockerfile). To warunek bezpieczeństwa: polecenia tworzące konta, czytające OTP czy wydające sesje nie mogą być aktywne na produkcji.

Opcjonalnie, dla szybszego budowania lokalnego: CI publikuje obrazy z inline cache (`BUILDKIT_INLINE_CACHE=1`), żeby `--cache-from` działało także przy domyślnym sterowniku `docker` w Buildx.

## 📝 Data Model

Wszystkie encje to pliki (YAML lub JSON). Narzędzie nie ma bazy danych.

```mermaid
flowchart LR
    classDef newEntity fill:#2f6feb,color:#fff,stroke:#1b4fb0

    entity_1["Manifest"]:::newEntity
    entity_2["Repo"]:::newEntity
    entity_3["Service"]:::newEntity
    entity_4["ImageSource (image | build)"]:::newEntity
    entity_5["SeedState"]:::newEntity
    entity_6["Persona"]:::newEntity
    entity_7["Toggle"]:::newEntity
    entity_8["RunFile"]:::newEntity
    entity_9["Variant"]:::newEntity
    entity_10["RunState (state.json)"]:::newEntity
    entity_11["EnvDescription (env.json)"]:::newEntity

    entity_1 -->|1-n| entity_2
    entity_1 -->|1-n| entity_3
    entity_3 -->|1-1| entity_4
    entity_4 -->|n-1| entity_2
    entity_1 -->|1-n| entity_5
    entity_1 -->|1-n| entity_6
    entity_5 -->|n-n| entity_6
    entity_1 -->|1-n| entity_7
    entity_8 -->|n-1| entity_1
    entity_8 -->|1-n| entity_9
    entity_8 -->|n-1| entity_5
    entity_10 -->|1-1| entity_8
    entity_10 -->|1-n| entity_11
```

### Manifest (`version: 1`)

Przykład fikcyjny, skrócony:

```yaml
# yaml-language-server: $schema=../node_modules/e2e-qa-runner/schema/manifest.v1.json
version: 1
project: acme

repos:
  api:    { url: git@git.example.com:acme/api.git,        defaultRef: main, prRef: "refs/pull/{n}/head" }
  web:    { url: git@git.example.com:acme/web.git,        defaultRef: main, prRef: "refs/pull/{n}/head" }
  mobile: { url: git@git.example.com:acme/mobile.git,     defaultRef: main }
  ml:     { url: git@git.example.com:acme/ml-service.git, defaultRef: main }

resources:
  docker: { memory: 6g }   # budżet VM Dockera dla całego runu

services:
  postgres:
    image: postgres:16
    port: 5432                       # port w kontenerze; port publikowany przydziela narzędzie
    memory: 512m
    env: { POSTGRES_PASSWORD: "${secret.postgres}", POSTGRES_DB: acme }   # sekret jednorazowy, per run
    url: "postgres://postgres:${secret.postgres}@${self.host}:${self.port}/acme"
    urls:                            # nazwane dialekty tej samej bazy (np. sterownik async i sync)
      async: "postgresql+asyncpg://postgres:${secret.postgres}@${self.host}:${self.port}/acme"
    health: { command: [pg_isready, -U, postgres] }
  redis:
    image: redis:7
    port: 6379
    memory: 128m
    url: "redis://${self.host}:${self.port}"
    health: { command: [redis-cli, ping] }

  ml:
    image: "registry.example.com/acme/ml-service:${repo.ml.sha}"   # obraz z CI, gdy istnieje
    build: { repo: ml, dockerfile: Dockerfile, target: qa }        # w przeciwnym razie budowanie
    port: 8000
    memory: 1g
    env: { DATABASE_URL: "${services.postgres.urls.async}", DB_SCHEMA: ml }
    migrate: { command: [uv, run, migrate], env: { DATABASE_URL: "${services.postgres.url}" } }
    dependsOn: [postgres]
    health: { http: /health }
  api:
    image: "registry.example.com/acme/api:${repo.api.sha}"
    build: { repo: api, dockerfile: Dockerfile, target: qa }
    port: 3000
    memory: 700m
    env:
      DATABASE_URL: "${services.postgres.url}"
      REDIS_URL: "${services.redis.url}"
      ML_URL: "${services.ml.url}"
      # odwołanie „w górę grafu”: web i mobile-web zależą od api, a api zna ich originy (CORS)
      CORS_ORIGINS: "${services.web.origins},${services.mobile-web.origins}"
    files:
      - mountPath: /app/config/flags.json   # cel przełącznika billing-v2 (apply: file)
        format: json
        content: { "billing": { "v2": false } }
    migrate: { command: [node, dist/migrate.js], timeout: 5m }
    dependsOn: [postgres, redis, ml]
    health: { http: /health }
  web:
    image: "registry.example.com/acme/web:${repo.web.sha}"
    build: { repo: web, dockerfile: Dockerfile }
    port: 8080
    memory: 256m
    env: { ACME_API_URL: "${services.api.publicUrl}" }   # entrypoint generuje env.js (runtime config)
    dependsOn: [api]
    health: { http: / }
    target: web
    browser: { locale: pl-PL }
  mobile-web:
    image: "registry.example.com/acme/mobile-web:${repo.mobile.sha}"
    build: { repo: mobile, dockerfile: Dockerfile.web }
    port: 8080
    memory: 256m
    files:                           # gotowy runtime config montowany do kontenera
      - mountPath: /usr/share/nginx/html/config.json
        format: json
        content: { "apiUrl": "${services.api.publicUrl}" }
    dependsOn: [api]
    health: { http: / }
    target: mobile-web               # web build aplikacji mobilnej ≠ native
    browser: { locale: pl-PL, viewport: { width: 390, height: 844, mobile: true } }

personas:                            # każda persona użyta w stanach; tryb logowania i powierzchnia per persona
  admin:             { login: ui }                        # surface: domyślnie login.surface (web)
  member:            { login: ui, surface: mobile-web }   # w aplikacji: telefon + hasło
  scoped-member:     { login: session }
  unassigned-member: { login: ui }
  outsider:          { login: ui }
  auditor:           { login: ui }    # persona własna konsumenta

seed:
  run: { in: api, command: [node, dist/qa/seed.js], timeout: 5m }
  states:
    baseline: {}
    first-run: {}
    scoped-access: {}
    no-assignments: {}
    expired-data: {}
    audit-backlog:                    # stan własny: opis i persony są obowiązkowe
      description: Dziesięć zamkniętych projektów czeka na audyt
      personas: [admin, auditor, outsider]

glossary: { tenant: organizacja, resource: projekt, group: zespół }

toggles:
  new-dashboard:
    default: false
    apply: { env: { service: web, name: ACME_FF_NEW_DASHBOARD } }
  billing-v2:
    default: false
    apply: { file: { service: api, mountPath: /app/config/flags.json, pointer: /billing/v2 } }
  maintenance-banner:
    default: false
    apply: { command: { in: api, command: [node, dist/qa/flag.js, maintenance-banner, "${toggle.value}"] } }

login:
  surface: web                       # domyślna powierzchnia; persona może ją nadpisać
  notes:                             # lista (wszystkie powierzchnie) albo mapa per powierzchnia
    web:
      - Po wejściu na stronę logowania zamknij baner cookies; zasłania przycisk „Dalej”.
    mobile-web:
      - Identyfikatorem jest numer telefonu; po wpisaniu przewiń w dół, klawiatura zasłania przycisk.
  otp: { in: api, command: [node, dist/qa/otp.js, "${identity}"] }
  session: { in: api, command: [node, dist/qa/session.js, "${persona.username}"], ttl: 30m }
  rateLimitReset: { in: api, command: [node, dist/qa/reset-login-limits.js] }

dataAccess:                          # wymaganie 9; przepis i gwarancja „tylko odczyt” należą do konsumenta
  db: { in: postgres, command: [psql, -U, qa_readonly, -d, acme, -v, ON_ERROR_STOP=1, -f, "-"] }   # zapytanie na stdin

# qa: klucz zarezerwowany dla speca #1b
```

Reguły manifestu:

- **Źródło obrazu.** Usługa infrastrukturalna ma tylko `image` (stały tag). Usługa aplikacyjna ma `image` (szablon tagu), `build` albo oba.
  - Szablon tagu może używać `${repo.<r>.sha}` (pełne SHA), `${repo.<r>.shortSha}` i `${repo.<r>.ref}` (ref oczyszczony do znaków dozwolonych w tagu).
  - `build` to `{ repo, context?, dockerfile?, target?, args? }`. `context` jest względny do katalogu repo (domyślnie `.`).
  - Kolejność wyboru źródła, flagi wymuszania i reguły dla ścieżek lokalnych opisuje sekcja „Obrazy: wybór źródła i budowanie”.
- **Porty nigdy nie są stałe.** `port` to port w kontenerze. Narzędzie publikuje go na `127.0.0.1:<port przydzielony per run>`. Kontenery rozmawiają ze sobą po nazwach usług w sieci projektu compose.
- **Placeholdery** rozwiązują się w **widoku kontenera** (wszystkie usługi to kontenery):
  - lista: `${services.<n>.url|urls.<nazwa>|host|port}` (sieć compose: `host` = nazwa usługi, `port` = port w kontenerze), `${services.<n>.publicUrl|publicPort|origins}` (widok przeglądarki: `127.0.0.1` i port publikowany), `${self.host|port}`, `${repo.<r>.sha|shortSha|ref}`, `${secret.<n>}`, `${run.id}`, `${variant.name}`, `${persona.username}`, `${identity}`, `${toggle.value}`;
  - **`url` vs `publicUrl`.** Konfiguracja, którą czyta przeglądarka (runtime config frontendu), musi używać `publicUrl`. Konfiguracja, którą czyta kontener (API → baza), używa `url`. Walidacja ostrzega, gdy `env` lub `files` usługi z `target` odwołuje się do `url` zamiast `publicUrl`;
  - **placeholdery nie tworzą krawędzi grafu.** Krawędzie tworzy wyłącznie `dependsOn`. Wszystkie porty publikowane i sekrety przydziela się przed `up`, więc usługa może odwołać się do usługi, która od niej zależy (np. originy CORS w `api`). Cykl powstaje tylko w `dependsOn`;
  - `${services.<n>.origins}` jest zawsze w widoku przeglądarki: `http://127.0.0.1:<port>,http://localhost:<port>`. Przeglądarka traktuje obie formy jako różne originy. `env.json` publikuje URL-e w formie `127.0.0.1` i tej formy powinni używać testujący. W pliku `format: json` wartość będąca w całości tym placeholderem renderuje się jako tablica JSON;
  - `${secret.<n>}` w v1 jest zawsze **generowany** per run (losowy, jednorazowy). Pobieranie sekretów z zewnętrznych źródeł jest poza zakresem;
  - nierozwiązany placeholder to błąd walidacji, a nie pusty string.
- **`url`** usługi to szablon (domyślnie `http://${self.host}:${self.port}`). Usługi bez protokołu HTTP muszą go podać. **`urls`** to opcjonalne nazwane warianty tego samego adresu (np. dialekty sterownika async i sync). Każdy szablon ma dwa widoki: kontenera (`url`) i hosta (`publicUrl`, z `127.0.0.1` i portem publikowanym). URL z sekretem jest w `env.json` maskowany w obu widokach i dostaje dwie referencje: `urlEnv` (`E2E_QA_SERVICE_<USŁUGA>_URL[_<NAZWA>]`, widok kontenera) i `publicUrlEnv` (`E2E_QA_SERVICE_<USŁUGA>_PUBLIC_URL[_<NAZWA>]`, widok hosta).
- **Polecenia są tablicami (exec form)**, nigdy stringiem interpretowanym przez powłokę. Dotyczy `migrate.command`, `health.command` i każdego hooka QA. Wartości z wejścia w czasie działania (`${identity}`, `${persona.username}`, `${toggle.value}`) muszą stanowić **cały** element tablicy. Narzędzie podstawia je jako jeden argument, odrzuca wartość zaczynającą się od `-` i przekazuje je też w zmiennych `E2E_QA_IDENTITY`, `E2E_QA_PERSONA_USERNAME`, `E2E_QA_TOGGLE_VALUE`. Konsument, który potrzebuje powłoki, wywołuje własny skrypt z obrazu.
- **Hook QA** ma postać `{ in: <usługa>, command: [...], timeout? }` (domyślny `timeout` 10 min) i wykonuje się przez `docker compose -p <projekt> exec -T` (bez TTY, ze stdin) w działającym kontenerze. Konsument nigdy nie zaszywa nazwy projektu compose. Wartości sekretne (hasła person) trafiają do `exec` po nazwie zmiennej (`-e NAZWA`, z wartością w środowisku procesu CLI), nigdy w argv, więc nie są widoczne na liście procesów hosta.
- **`migrate`** to `{ command: [...], env?, timeout? }`. Narzędzie generuje z niego jednorazową usługę compose `<usługa>-migrate`:
  - ten sam obraz, `env` usługi scalony z `migrate.env`, ten sam `mem_limit`;
  - **dziedziczy `dependsOn` usługi** (`service_healthy`), więc startuje dopiero po zdrowych zależnościach;
  - usługa zależy od niej warunkiem `service_completed_successfully`.

  Seed startuje po `up --wait`, czyli po wszystkich migracjach. Obsługa usług jednorazowych w `up --wait` zależy od wersji Compose. Krok 11 planu ustala minimalną wersję testem, a preflight ją egzekwuje.
- **`files`** to szablony plików konfiguracyjnych renderowane do `runs/<runId>/files/<variant>/<usługa>/` i montowane **tylko do odczytu** do kontenera pod `mountPath` (ścieżka absolutna w kontenerze). Pozycja ma postać `{ mountPath, format: dotenv | json | text, content | template }`. `content` to mapa (dla `dotenv`/`json`) albo tekst; `template` to ścieżka do szablonu względem manifestu.
  - Narzędzie nie zapisuje niczego do repozytoriów, więc reguła o plikach ignorowanych przez git, kopiach zapasowych i przywracaniu przestała być potrzebna.
  - Uprawnienia: katalog `files/` ma 0700 (chroni przed innymi użytkownikami hosta), a same pliki 0644, żeby proces w kontenerze działający jako użytkownik inny niż root (np. serwer statyczny, node) mógł je czytać na Linuksie.
  - Pliki mogą zawierać `${secret.*}` i znikają przy `down`.
- **Każde wywołanie zewnętrzne ma limit czasu:** hooki, `docker compose pull`, sprawdzenie rejestru, `docker build`, `up --wait` i `git fetch` (z `GIT_TERMINAL_PROMPT=0`). `pull`, sprawdzenie rejestru i `fetch` mają po 2 ponowienia.
- **Przełączniki** mają trzy rodzaje `apply`: `env` (zmienna usługi przy starcie kontenera), `file` (wskaźnik JSON w pliku z `files` tej usługi, renderowany przed `up`) i `command` (hook QA z `${toggle.value}`, po seedzie).
- **`dependsOn`** przekłada się na `depends_on: condition: service_healthy` w compose.
- **`memory`** jest obowiązkowe dla każdej usługi i staje się egzekwowanym `mem_limit`. Usługa `<n>-migrate` dziedziczy limit swojej usługi.
- **`health`** to `{ http: <ścieżka> } | { command: [...] } | { none: true }` z opcjonalnym `timeout`. Bez `health` obowiązuje `HEALTHCHECK` obrazu. Brak obu to błąd przy `up` (narzędzie sprawdza to przez `docker image inspect`).
- **`target: web | mobile-web`** oznacza usługi, na których człowiek lub wykonawca widzi produkt. `mobile-web` niesie w `env.json` zastrzeżenie, że to nie jest build natywny. Opcjonalne **`browser`** (`locale` dla `navigator.language`/`Accept-Language`, `viewport { width, height, mobile }`) trafia do `env.json` jako `targets[].browser`. Obiekt jest opcjonalny i otwarty: nowe klucze dodane przez #1b nie wymagają podbicia `env.v1`.
- **`prRef`** (opcjonalne) to wzorzec refu PR-a dostępny przez git (np. `refs/pull/{n}/head` albo `refs/merge-requests/{n}/head`). Na nim opiera się nakładka PR. Rdzeń nie wywołuje API trackera.

### Obrazy: wybór źródła i budowanie

| Tryb (`--image-source`, domyślnie `auto`) | Zachowanie |
|---|---|
| `auto` | Gdy usługa ma `image` i repo pochodzi z cache albo jest czystą ścieżką lokalną: sprawdzenie rejestru (`docker buildx imagetools inspect`). Obraz istnieje → `pull`. Brak → `build` (gdy zdefiniowany), inaczej `blocked: image-missing`. Odpowiedź „denied”/401: gdy w konfiguracji Dockera operatora jest zapisane logowanie do tego rejestru (`credHelpers`/`auths`), traktowana jak brak obrazu (→ `build`); bez logowania `blocked: registry-auth` (Q12). Obraz bez wariantu dla platformy hosta (np. tylko `linux/amd64` na hoście `arm64`) → `build` natywny (Q13) |
| `registry` | Tylko rejestr; brak obrazu to `blocked: image-missing` |
| `build` | Tylko budowanie; usługa bez `build` to `blocked` |

- Plik runu może nadpisać tryb per usługa (`imageSource: { api: build }`).
- **Ścieżka lokalna z niezacommitowanymi zmianami** (`dirty`) w trybie `auto` zawsze buduje. Obraz z rejestru dla `HEAD` nie zawiera tych zmian. W trybie `registry` taka kombinacja daje `blocked` z wyjaśnieniem.
- **Budowanie** używa `docker buildx build` (BuildKit) z lokalnym cache budowania Dockera. Gdy usługa ma też `image`, narzędzie dodaje `--cache-from` z obrazem `defaultRef`, jeśli istnieje. Przy domyślnym sterowniku `docker` daje to efekt tylko wtedy, gdy CI publikuje inline cache (opcja z kontraktu obrazu). Tag lokalny to `e2eqa/<projekt>-<usługa>:<runId>`. Cache budowania nie jest czyszczony przez `down`; rośnie na dysku VM Dockera, a czyszczenie (`docker builder prune --keep-storage …`) zostaje po stronie operatora i jest opisane w dokumentacji.
- Budowanie idzie po kolei (jedno naraz), żeby nie przekroczyć pamięci VM. Równoległość można zwiększyć flagą `--build-parallel N`.
- **Kontekst budowania ze ścieżki lokalnej** jest tylko czytany. Obowiązuje `.dockerignore` repo. Narzędzie ostrzega, gdy `.dockerignore` nie wyklucza `.env*`, bo lokalne pliki operatora mogłyby trafić do obrazu.
- Uwierzytelnienie do rejestru pochodzi z `docker login` operatora; narzędzie go nie przechowuje.
- **Platforma.** Kontrola rejestru czyta listę platform obrazu (manifest list). Gdy nie ma wariantu dla platformy hosta, tryb `auto` buduje natywnie. Tryb `registry` daje wtedy `blocked: platform-mismatch`. Przy runnerach CI `amd64` i hostach `arm64` (typowe laptopy deweloperskie) to będzie norma, a nie wyjątek, dopóki CI nie publikuje obrazów wieloplatformowych.
- `env.json` zapisuje per usługa `image: { source: registry | build, reason?, ref, digest?, imageId, platform }`. `reason` wyjaśnia, dlaczego wybrano `build` w trybie `auto`: `image-missing`, `registry-denied-with-login`, `platform-mismatch`, `dirty-path` albo `forced`. `digest` (repo digest) istnieje tylko dla obrazu z rejestru. `imageId` (ID konfiguracji obrazu) istnieje zawsze i identyfikuje także obraz zbudowany lokalnie.

### Słownik stanów seeda (zalecany, v1)

Słownik nie jest abstrakcją na zapas. Każdy stan pochodzi z wymagania 5 briefu (potrzeby pierwszego konsumenta), a `first-run`, `scoped-access` i persona `outsider` odpowiadają klasom defektów A, B i C z eksperymentu. Pojęcia `tenant`, `resource` i `group` są abstrakcyjne. Konsument mapuje je na swój język w `glossary`. Stan o **standardowej nazwie** niesie gwarancje z tabeli i wymaga wymienionych person, zawsze z personą `outsider`: członkiem innego tenanta, potrzebnym do sprawdzania reguł dostępu i skutków ubocznych na cudzych danych. Konsument implementuje dowolny podzbiór słownika. Brakujące stany standardowe `validate` raportuje jako informację o pokryciu, a nie błąd.

| Stan | Gwarancja danych | Persony (oprócz `outsider`) |
|---|---|---|
| `baseline` | Typowy, wypełniony tenant: ≥2 zasoby, ≥2 grupy | `admin`, `member` |
| `first-run` | Nowy klient: tenant bez danych i bez konfiguracji | `admin` |
| `scoped-access` | ≥2 zasoby (`A`, `B`); persona ma dostęp wyłącznie do `A` | `admin`, `scoped-member` |
| `no-assignments` | Persona bez żadnych przypisań | `admin`, `unassigned-member` |
| `multi-group` | Persona w dwóch grupach o różnych uprawnieniach | `admin`, `multi-group-member` |
| `expired-data` | Dane po terminie ważności obok aktualnych | `admin`, `member` |
| `empty-group` | Grupa bez członków i zasobów | `admin` |

**Stany własne** mają dowolną nazwę spoza słownika i muszą deklarować `description` oraz `personas`. Nazwa ze słownika nie może zmienić znaczenia: stan `scoped-access` z innymi personami niż w tabeli to błąd walidacji.

Wymogi wobec person: konta bez 2FA i bramki pierwszego logowania (zgody, onboarding) już przebyte. Wyjątkiem jest stan, który jawnie testuje pierwsze logowanie i opisuje to w `description`.

**Kontrakt polecenia seeda.** Polecenie wykonuje się w kontenerze usługi z `seed.run.in`. Narzędzie przekazuje w środowisku `exec`:

- `E2E_QA_MODE=1`;
- `E2E_QA_STATE`: nazwę stanu;
- `E2E_QA_PERSONA_<NAZWA>_PASSWORD`: hasło każdej wymaganej persony, generowane raz na run i stałe przez cały run;
- `E2E_QA_SEED_OUTPUT`: ścieżkę pliku wyjścia w katalogu `/e2e-qa`. To nazwany wolumen projektu compose montowany do każdego kontenera aplikacyjnego. Wolumen, a nie bind mount, bo nie zależy od UID procesu w kontenerze i znika przy `down -v`. Narzędzie odczytuje z niego pliki przez `docker compose cp`.

`<NAZWA>` to nazwa persony wielkimi literami, w której każdy znak inny niż litera lub cyfra zamieniono na `_` (`scoped-member` → `SCOPED_MEMBER`). Ta sama nazwa zmiennej (`E2E_QA_PERSONA_<NAZWA>_PASSWORD`) obowiązuje w `credentials.env` i w polu `passwordEnv` w `env.json`; jest jedna konwencja. Seed **doprowadza dane dokładnie do stanu**: usuwa ślady poprzedniego stanu i jest idempotentny.

`e2e-qa seed <stan>` na działającym stosie to reset, a nie dopisanie:

- generuje hasła tylko dla person nowych w tym runie (istniejące zostają);
- unieważnia bufor sesji wszystkich person;
- ponownie stosuje przełączniki `command`;
- odświeża `env/<variant>.json`.

Seed zakłada konta z tymi hasłami i zapisuje do `E2E_QA_SEED_OUTPUT` JSON:

```json
{ "state": "scoped-access",
  "personas": { "admin": { "username": "qa-admin@acme.test", "identifierKind": "email" }, "scoped-member": { "username": "qa-scoped@acme.test" }, "outsider": { "username": "qa-outsider@acme.test" } },
  "resources": { "A": { "label": "Projekt Alfa" }, "B": { "label": "Projekt Beta" } } }
```

Brak wymaganej persony albo zasobu z gwarancji oznacza błąd runu. Hasła nigdy nie wracają w wyjściu seeda. `username` to identyfikator, którym persona loguje się na **swojej** powierzchni (e-mail, telefon, login). Opcjonalne `identifierKind: email | phone | username` podpowiada testującemu, którego pola użyć; dostarcza je seed, bo tylko on zna założone konto.

### Konta i sesje (tryb i powierzchnia per persona)

Każda persona ma `login` (tryb) i `surface` (powierzchnię logowania, domyślnie `login.surface`). Konsument może mieć różne powierzchnie z różnymi identyfikatorami, np. panel web z e-mailem i hasłem oraz aplikację mobile-web z telefonem i hasłem. `login.notes` może być listą wspólną albo mapą per powierzchnia. `surface` persony i klucze mapy `login.notes` muszą wskazywać usługi z `target`, inaczej walidacja zgłasza błąd.

| `login` | Co dostaje testujący | Hooki |
|---|---|---|
| `ui` | Login i referencję hasła; loguje się przez UI swojej powierzchni. Pomocniczo `login.notes` (pułapki UI) oraz `e2e-qa otp`, gdy aplikacja wymaga OTP | `login.otp` (opcjonalny), `login.rateLimitReset` |
| `session` | Gotowy artefakt sesji: plik JSON `{ cookies?, origins?/storageState?, headers?, token?, expiresAt? }` | `login.session` (obowiązkowy dla tego trybu); hook sam obsługuje OTP |

Hook sesji zapisuje wynik do `/e2e-qa/session-<persona>.json`, a narzędzie przenosi go do `secrets/` (0600) i buforuje. `e2e-qa session <persona>` zwraca ścieżkę do ważnej sesji i odświeża ją po upływie `ttl` albo `expiresAt`, co ogranicza zużycie limitów logowania. Cookies i originy w sesji muszą dotyczyć `publicUrl` powierzchni. Wstrzyknięcie sesji do przeglądarki należy do wykonawcy (spec #1b) albo do człowieka.

**OTP dla dowolnej tożsamości.** Hook `login.otp` dostaje `${identity}`. `e2e-qa otp <persona>` to przypadek szczególny, w którym `identity` jest `username` persony. `e2e-qa otp --identity <id>` obsługuje tożsamości spoza person, np. konto zarejestrowane w trakcie scenariusza. Środowisko i dane są jednorazowe, więc odczyt OTP dowolnego konta w nim jest zamierzony.

### Plik runu (`version: 1`)

```yaml
version: 1
manifest: ./acme-e2e/manifest.yaml       # ścieżka względem pliku runu
state: scoped-access
toggles: { new-dashboard: true }
imageSource: { mobile-web: build }        # opcjonalnie; globalnie: --image-source
variants:
  default:                                # v1: dokładnie jeden wariant
    refs:
      api: pr:412                         # nakładka: rozwiązywane przez repos.api.prRef
      web: integration/release-train      # gotowa gałąź integracyjna od orkiestratora
    paths:
      mobile: ../worktrees/mobile-task    # ścieżka lokalna: kontekst budowania, tylko odczyt
```

- Repo nieobecne w `refs` i w `paths` dostaje swój `defaultRef`. Repo z jednoczesnym wpisem w `refs` i w `paths` to błąd walidacji.
- Każdy wpis `refs` to **jeden** ref: gałąź, tag, SHA albo `pr:<n>`. Listy refów nie istnieją, bo narzędzie nie scala.
- `variants` to mapa nazw `[a-z0-9-]`. Schemat v1 dopuszcza wiele wariantów, ale implementacja v1 odrzuca więcej niż jeden komunikatem „nieobsługiwane w tej wersji”. Nazwy projektów compose (`e2eqa-<runId>-<variant>`), porty, katalogi i opis środowiska (`env/<variant>.json`, jedna linia `E2E_QA_ENV_<VARIANT>=…` na wariant) już teraz są per wariant, więc dodanie wariantów nie zmieni kontraktu.
- **Ścieżki lokalne są tylko do odczytu.** Narzędzie nie robi w nich checkoutu, fetcha, resetu ani żadnego zapisu. Służą wyłącznie jako kontekst budowania i źródło SHA (`HEAD` plus flaga `dirty`). Nie ma już `prepare` ani zapisu plików konfiguracyjnych do repo, bo konfiguracja idzie przez `env` i montowane `files`.

### Konfiguracja operatora (`.e2e-qa/config.yaml`, opcjonalna)

Pola: `queue.waitMinutes` (domyślnie 60), `retention.runs` (ile katalogów zakończonych runów zachować z logami, domyślnie 20), `build.parallel` (domyślnie 1) i nadpisanie budżetu pamięci. Flaga `--memory-budget` zmienia budżet. `--ignore-memory-gate` omija bramkę: wymaga podania jej jawnie przy każdym uruchomieniu, a fakt obejścia trafia do `state.json` i `env/<variant>.json`.

**Bramka pamięci.** Jest deterministyczna i liczy tylko kontenery:

- suma `memory` usług (z usługami migracji liczonymi jako maksimum, bo kończą się przed startem właściwych usług) ≤ `resources.docker.memory` ≤ całkowita pamięć VM Dockera (`MemTotal` z `docker info`).

Limity są egzekwowane przez `mem_limit`. Kontener, który go przekroczy, ginie z `OOMKilled`, a narzędzie pokazuje to w `status`. Pamięć zajęta przez cudze kontenery jest tylko **ostrzeżeniem** z liczbami. Źródło pomiarów (`docker info`, `docker stats`) jest wstrzykiwane, żeby bramkę dało się testować bez Dockera. Budowanie obrazów nie jest objęte bramką (BuildKit nie przyjmuje limitu per build). Dlatego budowanie jest domyślnie sekwencyjne.

### Katalog roboczy i stan runu

Wszystko, co tworzy narzędzie, leży pod `./.e2e-qa/` w katalogu, z którego je uruchomiono (wymaganie 10: praca pod strażnikiem ścieżek). Ścieżki lokalne operatora są tylko czytane.

```
.e2e-qa/
  cache/<repo>.git                 # klon lustrzany (bare) + blokada pliku per repo
  lock/                            # kolejka: owner.json + bilety FIFO
  runs/<runId>/
    run.yaml  state.json
    env/<variant>.json
    compose/<variant>.yaml         # bez wartości sekretów; odwołuje się do env_file w secrets/
    files/<variant>/<service>/     # katalog 0700, pliki 0644: wyrenderowane files, montowane tylko do odczytu
    secrets/                       # 0700/0600: credentials.env, env/<variant>/<service>.env, sesje, ${secret.*}
    logs/<variant>/<service>.log   # zrzut przy down, zamaskowany
    worktrees/<variant>/<repo>/
```

`state.json` zawiera:

- refy rozwiązane do SHA (z flagą `dirty` dla ścieżek lokalnych);
- źródło i digest obrazu każdej usługi;
- przydzielone porty publikowane i nazwy projektów compose;
- PID i czas startu procesu CLI tylko na czas fazy `preparing`;
- status: `preparing | running | degraded | stopped | failed`.

**Żywotność i blokada.** Nie ma procesów hosta, więc źródłem prawdy są **projekty compose** (kontenery z etykietą `e2e-qa.run=<runId>`) i `state.json`. Blokada kolejki należy do `runId`:

- Blokadę zwalnia `down`.
- W fazie `preparing` run jest porzucony, gdy nie żyje proces CLI z `owner.json` (PID **i** czas startu, więc ponownie użyty PID nie liczy się jako żywy).
- W fazie `running` run jest porzucony, gdy żaden kontener z jego etykietą nie działa (np. po restarcie Dockera bez `restart` policy).
- Kontener, który padł (`exited`, `OOMKilled`, `unhealthy`) przy działających pozostałych, zmienia status na `degraded`, widoczny w `status`. Run trzyma blokadę, dopóki operator nie zrobi `down`.

Tylko porzucony run jest sprzątany automatycznie, wyłącznie po etykiecie `e2e-qa.run`. Działający lub zdegradowany stos innego runu oznacza czekanie w kolejce. Po `waitMinutes` czekający run kończy się komunikatem z `runId` i statusem właściciela blokady oraz poleceniem `e2e-qa down <runId>`.

**Logi: odczyt na żądanie z Dockera, bez pompy.** `e2e-qa logs <usługa>` czyta `docker compose logs --timestamps --no-color` i maskuje znane sekrety przy odczycie. `--since` jest filtrowane po znacznikach czasu Dockera po stronie narzędzia, co omija niedokładność `--since` po stronie demona. `down` przed usunięciem kontenerów zrzuca zamaskowane logi do `logs/` (w zakresie `retention.runs`). Uzasadnienie: Docker i tak przechowuje logi kontenerów ze znacznikami czasu. Pompa wymagałaby procesu długożyjącego, którego usunięcie było celem C1. Koszt: do `down` surowe logi z sekretami per run leżą w magazynie logów Dockera. Sekrety są jednorazowe i znikają razem z kontenerami.

**Sekrety poza plikiem compose.** Wygenerowany `compose/<variant>.yaml` nie zawiera wartości sekretów. Zmienne usług (w tym `POSTGRES_PASSWORD` i URL-e z hasłem) trafiają do `secrets/env/<variant>/<usługa>.env` (0600), do którego compose odwołuje się przez `env_file`.

`down` zrzuca logi, robi `docker compose down -v --remove-orphans` dla projektów runu (usuwa też wolumeny `/e2e-qa`), usuwa worktree (`git worktree remove --force` + `prune`), obrazy zbudowane lokalnie dla runu (chyba że `--keep-images`; cache budowania zostaje), katalogi `files/` i `secrets/`. Logi, `state.json` i plik compose (bez sekretów) zostają w zakresie `retention.runs`.

### `env/<variant>.json`: opis środowiska (kontrakt dla #1b i innych konsumentów)

W całym specu i w briefie #1b skrót `env.json` oznacza plik `env/<variant>.json`. Schemat: `schema/env.v1.json`. Plik nie zawiera żadnej wartości sekretu. URL-e z `${secret.*}` są publikowane z zamaskowanym hasłem (`***`), a pełne wartości leżą w `credentials.env` pod referencjami `publicUrlEnv` (widok hosta) i `urlEnv` (widok kontenera). Usługi podają `publicUrl` (dla przeglądarki i człowieka) i `url` (adres w sieci compose).

```json
{ "version": 1, "runId": "…", "variant": "default", "status": "running",
  "state": { "name": "scoped-access", "standard": true, "guarantees": "…", "resources": { "A": { "label": "Projekt Alfa" } } },
  "glossary": { "tenant": "organizacja" },
  "services": {
    "web": { "publicUrl": "http://127.0.0.1:41234", "url": "http://web:8080", "target": "web",
             "image": { "source": "registry", "ref": "registry.example.com/acme/web:<sha>", "digest": "sha256:…", "imageId": "sha256:…" } },
    "mobile-web": { "publicUrl": "http://127.0.0.1:41236", "url": "http://mobile-web:8080", "target": "mobile-web",
             "image": { "source": "build", "ref": "e2eqa/acme-mobile-web:<runId>", "imageId": "sha256:…" } },
    "postgres": { "publicUrl": "postgres://postgres:***@127.0.0.1:41235/acme", "publicUrlEnv": "E2E_QA_SERVICE_POSTGRES_PUBLIC_URL",
                  "url": "postgres://postgres:***@postgres:5432/acme", "urlEnv": "E2E_QA_SERVICE_POSTGRES_URL",
                  "urls": { "async": { "publicUrl": "postgresql+asyncpg://postgres:***@127.0.0.1:41235/acme", "publicUrlEnv": "E2E_QA_SERVICE_POSTGRES_PUBLIC_URL_ASYNC",
                                       "url": "postgresql+asyncpg://postgres:***@postgres:5432/acme", "urlEnv": "E2E_QA_SERVICE_POSTGRES_URL_ASYNC" } },
                  "container": "e2eqa-…-postgres-1" } },
  "targets": [ { "service": "web", "kind": "web", "browser": { "locale": "pl-PL" } },
               { "service": "mobile-web", "kind": "mobile-web", "caveat": "web build, nie native",
                 "browser": { "locale": "pl-PL", "viewport": { "width": 390, "height": 844, "mobile": true } } } ],
  "repos": { "api": { "source": "cache", "ref": "pr:412", "sha": "<sha>" }, "mobile": { "source": "path", "sha": "<sha>", "dirty": true } },
  "personas": { "admin": { "username": "qa-admin@acme.test", "identifierKind": "email", "login": "ui", "surface": "web", "passwordEnv": "E2E_QA_PERSONA_ADMIN_PASSWORD" },
                "member": { "username": "+999000000001", "identifierKind": "phone", "login": "ui", "surface": "mobile-web", "passwordEnv": "E2E_QA_PERSONA_MEMBER_PASSWORD" },
                "scoped-member": { "username": "qa-scoped@acme.test", "login": "session", "surface": "web", "sessionCommand": "e2e-qa session scoped-member" } },
  "credentialsFile": "secrets/credentials.env",
  "login": { "notes": { "web": ["…"], "mobile-web": ["…"] }, "otp": true },
  "toggles": { "new-dashboard": true, "billing-v2": false },
  "dataAccess": ["db"],
  "memoryGate": "enforced",
  "startedAt": "…" }
```

Kontrakt poświadczeń jest zgodny z zasadą `om-*`: `env.json` zawiera tylko **referencje** (`passwordEnv`, `urlEnv`), a wartości leżą w `credentialsFile` (0600, poza gitem). Agent nie powinien czytać tego pliku, tylko ładować go do powłoki. Człowiek odczytuje hasło jawnie przez `e2e-qa creds <persona>`.

## 📝 API Contracts (CLI)

Kody wyjścia: **0** sukces, **2** błąd walidacji, **3** zablokowany (preflight, kolejka, pamięć, nieistniejący ref, brak obrazu), **4** błąd infrastruktury (budowanie, zdrowie, hook, timeout, sprzątanie niepełne).

| Polecenie | Działanie |
|---|---|
| `e2e-qa validate <manifest> [--run run.yaml]` | Schemat i reguły semantyczne (w tym statyczna część kontraktu obrazu: exec form poleceń, `build.args` bez adresów i sekretów); raport pokrycia słownika stanów |
| `e2e-qa checkout <run.yaml>` | Kroki 1–3: przygotowuje kod i wypisuje rozwiązane SHA |
| `e2e-qa images <run.yaml> [--image-source auto\|registry\|build]` | Kroki 1–3 i 5: rozwiązuje i buduje obrazy pod blokadą kolejki, bez bramki pamięci i bez startu; sprawdza `HEALTHCHECK`; wypisuje źródło, `digest` i `imageId` |
| `e2e-qa up <run.yaml> [--wait N] [--keep-on-failure] [--image-source …] [--memory-budget X] [--ignore-memory-gate]` | Kroki 1–9; wypisuje `E2E_QA_RUN_ID=…`, `E2E_QA_STATUS=running` i po jednej linii `E2E_QA_ENV_<VARIANT>=<ścieżka>` na wariant |
| `e2e-qa down [runId \| --stale] [--keep-images]` | Sprząta po etykietach i `state.json`; idempotentne; zwraca 4 i listę pozostałości, gdy coś zostało |
| `e2e-qa status [runId] [--json]` | Usługi, `publicUrl`, źródła obrazów, persony (bez haseł), zdrowie, kontenery `OOMKilled` |
| `e2e-qa seed <state>` | Reset danych do stanu; nowe hasła tylko dla nowych person, unieważnienie sesji, ponowne przełączniki `command`, odświeżenie `env.json` |
| `e2e-qa creds <persona>` | Wypisuje login i hasło (dla człowieka; nigdy do logów) |
| `e2e-qa session <persona>` | Ścieżka do ważnej sesji persony w trybie `session` (odświeża, gdy wygasła) |
| `e2e-qa otp <persona> \| --identity <id>` | Uruchamia hook `login.otp` dla persony albo dowolnej tożsamości, wypisuje kod |
| `e2e-qa logs <service> [--since 5m]` | `docker compose logs --timestamps` z maskowaniem; po `down` zrzut z `logs/` |
| `e2e-qa data <name> < query` | Przepis `dataAccess.<name>` konsumenta z zapytaniem na stdin |

Bez `runId` polecenia działają na jedynym działającym runie. Jeśli żaden nie działa albo działa kilka, kończą się błędem z listą runów.

## 📝 Edge Cases & Failure Scenarios

| Sytuacja | Zachowanie i co widzi operator |
|---|---|
| Ref albo `pr:<n>` nie istnieje, fetch odmawia (uprawnienia) | `blocked` przed kolejką, z nazwą repo i refu. `GIT_TERMINAL_PROMPT=0` zamienia pytanie o hasło w błąd |
| `pr:<n>` dla repo bez `prRef` | Błąd walidacji runu |
| Ścieżka lokalna nie jest repozytorium git albo nie istnieje | `blocked` w preflighcie |
| Ścieżka lokalna ma niezacommitowane zmiany | `auto` buduje obraz z kontekstu (ze zmianami); `dirty: true` w `env.json`; `registry` daje `blocked` |
| **Brak obrazu w rejestrze** | `auto`: budowanie, gdy jest `build`, z informacją `source: build` w `env.json`; bez `build` albo w trybie `registry`: `blocked: image-missing` z rozwiązanym tagiem |
| Rejestr wymaga logowania albo odmawia | `blocked: registry-auth` z nazwą rejestru; podpowiedź `docker login` |
| **Pull wisi albo sieć zrywa** | Limit czasu, 2 ponowienia, potem `error: pull-timeout` z nazwą obrazu |
| **Błąd budowania** | `error: build-failed` z nazwą usługi, ogonem wyjścia BuildKit (zamaskowanym) i ścieżką pełnego logu w katalogu runu; żaden kontener nie startuje |
| Budowanie przekracza limit czasu | `error: build-timeout`; proces budowania przerwany |
| `.dockerignore` nie wyklucza `.env*` w ścieżce lokalnej | Ostrzeżenie przed budowaniem |
| `build.args` zawiera `${services.*}` albo `${secret.*}` | Błąd walidacji (adres albo sekret wypieczony w obrazie) |
| **Obraz bez healthchecku** i bez `health` w manifeście | `error: no-healthcheck` po rozwiązaniu obrazu (`images`/`up`), z podpowiedzią `health` albo `health: { none: true }` |
| Sonda `health: { http }` w obrazie bez `curl`/`wget` | `error: no-http-probe-tool` po rozwiązaniu obrazu (sprawdzenie przez `docker run --rm --entrypoint` z `which`), z podpowiedzią `health: { command: [...] }` |
| Rejestr odpowiada „denied”/401 w trybie `auto` | Operator ma zapisane logowanie do rejestru → `build` z `reason: registry-denied-with-login` w `env.json`. Brak logowania → `blocked: registry-auth` z podpowiedzią `docker login <rejestr>` |
| Obraz w rejestrze bez wariantu dla platformy hosta | `auto`: `build` natywny z `reason: platform-mismatch`; `registry`: `blocked: platform-mismatch` z listą dostępnych platform |
| `health: { none: true }` | Gotowość = kontener działa; ostrzeżenie w wyjściu i w `env.json` |
| Migracja kończy się błędem | `up --wait` przerwane; `error` z nazwą `<usługa>-migrate` i ogonem logu; sprzątnięcie (chyba że `--keep-on-failure`) |
| **Hook QA nieaktywny w obrazie** (kod 78) albo nieobecny (kod 126/127 z `exec`) | `error: qa-hook-inactive` albo `qa-hook-missing` z nazwą hooka i usługi; podpowiedź: obraz z targetem `qa` albo sprawdzenie `E2E_QA_MODE` |
| Hook (np. seed) przekracza pamięć usługi | Hook działa w cgroup usługi, więc OOM może zabić kontener usługi; status `degraded` z nazwą usługi; podpowiedź: zapas w `memory` |
| Hook przekracza `timeout` albo kończy się błędem | Proces `exec` przerwany, `error` z nazwą hooka i ogonem wyjścia (zamaskowanym) |
| Wyjście seeda bez wymaganej persony lub zasobu | `error`, z nazwą braku |
| Hook sesji zwraca sesję bez wymaganych pól albo już wygasłą | `error` przy `up`; przy `e2e-qa session` jedna ponowna próba, potem błąd |
| Kontener przekracza `mem_limit` | `OOMKilled`; status `degraded` z nazwą usługi i limitem |
| Usługa z `target` odwołuje się w konfiguracji do `url` zamiast `publicUrl` | Ostrzeżenie walidacji (przeglądarka nie rozwiąże nazwy usługi z sieci compose) |
| Usługa odwołuje się do `${services.X.*}` usługi, która od niej zależy | Poprawne: porty i adresy są znane przed `up`. Cyklem jest tylko cykl w `dependsOn` |
| Wyścig o port publikowany (port zajęty między przydziałem a `up`) | Nowy port zmienia `publicUrl` i `origins` w innych usługach, więc ponowienie to pełny cykl: `down` projektu, nowy przydział, ponowny render `files` i `env_file`, `up`. Do 3 prób, potem `error` |
| Stan z runu nie istnieje w manifeście | Błąd walidacji z listą dostępnych stanów |
| Stan standardowy z innymi personami niż słownik | Błąd walidacji |
| Docker, Compose v2 w minimalnej wersji, Buildx (gdy potrzebny) albo `git` niedostępne | `blocked` w preflighcie z wymaganą i znalezioną wersją |
| Kolejka zajęta dłużej niż `waitMinutes` | `blocked: queue-timeout`; pozycja w kolejce wypisywana w trakcie czekania |
| Run trzymający blokadę jest porzucony | Najpierw `down` porzuconego runu po etykiecie, potem start. Działający stos innego runu nigdy nie jest sprzątany automatycznie |
| Bramka pamięci | `blocked: memory`, z liczbami: suma `memory` kontenerów vs `resources.docker.memory` vs `MemTotal` VM. `--ignore-memory-gate` przepuszcza run i zapisuje to |
| `otp --identity` z wartością zaczynającą się od `-` | Odrzucone przed wywołaniem hooka |
| Więcej niż jeden wariant w runie | Błąd walidacji „nieobsługiwane w tej wersji” |
| Ctrl+C, SIGTERM w trakcie `up` | Przerwanie budowania i `up`; sprzątnięcie po etykiecie; `state.json` pozwala dokończyć je przez `down --stale` |
| Wyciek sekretu do logów | Maskowanie znanych wartości przy odczycie (`logs`) i w zrzucie przy `down`; surowe logi w Dockerze znikają razem z kontenerami |

## 📝 Risks & Impact Review

- **Kontrakt obrazu zdatnego do QA to nowy, realny koszt po stronie konsumenta.** Repo z build-time env (adresy wypiekane przez bundler) albo bez healthchecku nie wejdzie do manifestu bez zmian w swoim Dockerfile i kodzie startowym. To świadoma cena container-first. Dokumentacja konsumenta musi podać wzorzec runtime config i targetu `qa`.
- **Bezpieczeństwo hooków QA w obrazach.** Polecenia tworzące konta, czytające OTP i wydające sesje trafiają do obrazów, a te mogą trafić na produkcję. Mitygacja kontraktowa: hooki odmawiają działania bez `E2E_QA_MODE=1`, a zalecany jest osobny target `qa` w Dockerfile. Narzędzie nie weryfikuje tego w obrazie. Odpowiedzialność leży po stronie konsumenta i jego review; ryzyko wymaga jawnej pozycji w dokumentacji konsumenta.
- **Czas pierwszego startu.** Budowanie obrazów z repo jest wolniejsze niż start procesu na hoście. Mitygacje: obrazy z rejestru (CI buduje per SHA), cache budowania Dockera, `--cache-from`. Skrócenie pętli dla zmian w ścieżce lokalnej (live update) jest poza v1.
- **Niezgodność platform zamieni rejestr w budowanie** (Q13). Przy runnerach CI `amd64` i hostach `arm64` tryb `auto` będzie w praktyce budował wszystkie obrazy aplikacyjne lokalnie, więc pierwszy run potrwa tyle, ile pełne budowanie każdej usługi. Kolejne runy przyspieszy cache budowania. Pełną korzyść z obrazów CI da dopiero publikowanie obrazów wieloplatformowych (`linux/amd64,linux/arm64`) po stronie konsumenta. To zalecenie trafia do dokumentacji kontraktu obrazu.
- **Interpretacja 401 przy zapisanym logowaniu** (Q12) może zamaskować wygasłe albo zbyt wąskie uprawnienia: run zbuduje obraz lokalnie zamiast użyć obrazu z CI. Mitygacja: `reason: registry-denied-with-login` w `env.json` i ostrzeżenie w wyjściu `up`/`images`.
- **Kontekst budowania ze ścieżki lokalnej** może zawierać pliki operatora (np. `.env*`). Chroni przed tym `.dockerignore` konsumenta. Narzędzie ostrzega, ale nie filtruje kontekstu samo.
- **Publiczne kontrakty, które trudno cofnąć:** `manifest.v1` (w tym `image`/`build`, `files.mountPath`, exec form poleceń), `run.v1`, słownik stanów v1, kontrakt polecenia seeda i hooka sesji, kontrakt obrazu zdatnego do QA (`E2E_QA_MODE`, `/e2e-qa`), `env.v1`. Zmiany łamiące wymagają podbicia wersji schematu. Wydanie pakietu, które wprowadza schemat v2, czyta też v1 przez co najmniej jedno kolejne wydanie minor pakietu i wypisuje ostrzeżenie migracyjne.
- **Słownik zalecany zamiast zamkniętego.** Spec #2 (generator) może polegać tylko na stanach standardowych. Stany własne będą dla niego czarnymi skrzynkami z opisem.
- **Wartość słownika jest mniejsza, niż zakładano** (wniosek z próby na sucho, 2026-10-09). Realne scenariusze częściej wymagają stanów **własnych**, czyli złożeń kilku warunków, niż pojedynczych stanów standardowych. Spec #2 powinien to uwzględnić, np. przez składanie stanów albo parametryzację.
- **Kilka dialektów URL-a tej samej usługi** (np. sterownik async i sync). Rozwiązane nazwanymi `urls`, bo składanie z `host`/`port` omija maskowanie. Koszt: kolejne pole w publicznym schemacie.
- **Brak scalania w narzędziu.** Jakość wyniku dla paczki zależy od tego, czy gałąź integracyjna operatora odpowiada temu, co trafi na gałąź główną. `env.json` zapisuje SHA i digesty obrazów, żeby wynik dało się odtworzyć.
- **Bezpieczeństwo danych runu:**
  - Hasła person, sesje, sekrety infrastruktury i wyrenderowane `files` są jednorazowe, generowane per run i trzymane w katalogach runu o uprawnieniach 0700 (sekrety w plikach 0600, plik compose bez wartości sekretów), usuwane przy `down`. Wyjście seeda i sesje przechodzą przez wolumen `/e2e-qa`, który znika przy `down -v`.
  - `env.json` zawiera wyłącznie referencje i zamaskowane URL-e.
  - Sekrety przekazywane do `exec` idą po nazwie zmiennej, nie w argv.
  - Porty są publikowane wyłącznie na `127.0.0.1`.
  - Kod OTP jest jednorazowy i dotyczy konta demo.
  - Narzędzie nie przyjmuje sekretów z manifestu (manifest jest commitowany).
- **Bramka pamięci** jest teraz egzekwowana przez `mem_limit`, ale nie obejmuje budowania obrazów. Mitygacja: budowanie sekwencyjne domyślnie. Hooki QA liczą się do limitu swojej usługi.
- **Cache budowania rośnie bez limitu** na dysku VM Dockera. Czyszczenie należy do operatora (`docker builder prune --keep-storage`). Automatyczna polityka może dojść później.
- **Zależność od wersji Compose.** Zachowanie `up --wait` z usługami jednorazowymi zmieniało się między wersjami Compose v2. Minimalną wersję ustala test w kroku 11, a preflight ją egzekwuje.
- **Zdalny Docker jako droga „na serwer”.** Ten sam plik compose może iść na zdalny host przez `DOCKER_HOST=ssh://…` albo kontekst `docker context`. To naturalna ścieżka bez Terraform czy Ansible, ale poza v1. Wymaga rozwiązania: publikacji portów i dostępu przeglądarki (tunel albo publikacja na interfejsie zdalnym), przesyłania kontekstu budowania (albo wyłącznie obrazów z rejestru), montowania `files` i `io` (bind mount ze ścieżki lokalnej nie działa na zdalnym demonie; potrzebne wolumeny albo `configs`).

## 📋 Poza zakresem

- Kontrakt scenariusza MD i styk z wykonawcą QA (Q4 otwarte ponownie): spec #1b.
- **Tryb `runtime: host`** (usługi aplikacyjne jako procesy hosta): możliwe rozszerzenie dla repozytoriów bez obrazu zdatnego do QA albo dla szybkiej pętli deweloperskiej. Wymagałby supervisora, grup procesów i zapisu konfiguracji do repo, czyli tego, co v1 usunął.
- Live update i synchronizacja plików do kontenerów.
- Zdalny Docker (`DOCKER_HOST=ssh://…`), serwer i PaaS.
- Kilka wariantów w jednym runie (kontrakt jest gotowy, implementacja później).
- Scalanie paczek PR-ów; API trackera.
- Generator scenariuszy, ocena dowodów, pętla naprawy, UI historii (specy #2–#4).
- Emulator i build natywny.

## 📋 Phasing

Każda faza zostawia działające, użyteczne narzędzie.

1. **Manifest i walidacja.** Konsument może napisać i zwalidować manifest, łącznie ze statyczną częścią kontraktu obrazu.
2. **Kod repozytoriów.** `e2e-qa checkout` przygotowuje kod z refów, PR-ów albo ścieżek lokalnych (tylko odczyt).
3. **Obrazy.** `e2e-qa images` rozwiązuje obrazy z rejestru albo je buduje, z cache, i raportuje źródło i digest.
4. **Start stosu.** `up`/`down`/`status`/`logs` przez wygenerowany compose (migracje, zdrowie, limity, `files`), kolejka i bramka pamięci, `env.json` z usługami. Człowiek może testować ręcznie na kontach z danych deweloperskich konsumenta, jeśli migracje je zakładają.
5. **Stan danych i konta.** Hooki QA przez `exec`, seed, persony, przełączniki, sesje, `creds`/`otp`/`data`, pełny `env.json`. Środowisko jest gotowe do testów ręcznych i do speca #1b.

## 📋 Implementation Plan

Testy jednostkowe: vitest. Testy integracyjne z Dockerem są oznaczone i pomijane, gdy Dockera nie ma. W CI (GitHub Actions, Linux) działają. Fikcyjny konsument testowy żyje w `examples/acme/`: maleńkie usługi HTTP w Node z Dockerfile (target `qa` z hookami, runtime config, healthcheck), postgres i redis, lokalne repozytoria bare tworzone w teście oraz lokalny rejestr (`registry:2`) w teście.

### Faza 1: manifest i walidacja

1. Szkielet pakietu: TypeScript, Node ≥20, bin `e2e-qa`, `--version`, lint i testy w CI. Test: `npx e2e-qa --version`.
2. JSON Schema `manifest.v1` i `run.v1` (z `image`/`build`, `files.mountPath`, exec form poleceń, mapą `variants`, `imageSource`, zarezerwowanym kluczem `qa`) oraz `e2e-qa validate` z czytelnymi błędami (ścieżka YAML i linia). Test: poprawny i niepoprawne manifesty `examples/acme`; manifest, w którym `api` odwołuje się do `${services.web.origins}`, a `web` zależy od `api`, przechodzi walidację.
3. Reguły semantyczne:
   - cykle w `dependsOn` (placeholdery nie tworzą krawędzi) i nierozwiązywalne placeholdery;
   - polecenia wyłącznie w exec form, a placeholdery wejściowe jako całe elementy tablicy;
   - `build.args` bez `${services.*}`/`${secret.*}`; ostrzeżenie `url` zamiast `publicUrl` w usługach z `target`;
   - `surface` person i klucze `login.notes` wskazujące usługi z `target`;
   - słownik stanów (znaczenie nazw standardowych, `description`/`personas` stanów własnych, raport pokrycia);
   - brak `memory`, nieznane przełączniki i stany w runie, `pr:` bez `prRef`, repo w `refs` i `paths` naraz, więcej niż jeden wariant.

   Test: tabela przypadków.

### Faza 2: kod repozytoriów

4. Cache lustrzany z blokadą per repo, fetch z limitem czasu i `GIT_TERMINAL_PROMPT=0`, worktree per (wariant, repo). Test: lokalne repozytoria bare i dwa równoległe fetche.
5. Nakładka `pr:<n>` przez `prRef`. Test: repo bare z refem `refs/pull/7/head`.
6. Ścieżki lokalne: weryfikacja, SHA, `dirty`, zero zapisu; polecenie `e2e-qa checkout`. Test: `git status` i lista plików (z mtime) ścieżki lokalnej identyczne przed i po.

### Faza 3: obrazy

7. Rozwiązanie szablonu tagu (`${repo.*}`), sprawdzenie rejestru z limitem czasu i ponowieniami, `pull`, digest, tryby `auto`/`registry`/`build` i nadpisanie per usługa, reguła `dirty` → `build`, obsługa 401/denied zależnie od zapisanego logowania (Q12), kontrola platformy z manifest listy (Q13), pole `reason`. Test integracyjny z lokalnym rejestrem:
   - obraz istnieje → `source: registry`; brak → `source: build`, `reason: image-missing`; `registry` bez obrazu → `blocked: image-missing`; ścieżka `dirty` → `build`, `reason: dirty-path`;
   - rejestr z uwierzytelnianiem: 401 przy zapisanym logowaniu → `build`, `reason: registry-denied-with-login`, ostrzeżenie; bez logowania → `blocked: registry-auth`;
   - obraz tylko w platformie innej niż host → `build`, `reason: platform-mismatch`; w trybie `registry` → `blocked: platform-mismatch`.
8. Budowanie obrazów przez `docker buildx build` z cache budowania Dockera i `--cache-from`, sekwencyjnie (domyślnie), z limitem czasu, ostrzeżeniem o `.dockerignore` i lokalnym tagiem `e2eqa/…:<runId>`; kontrola `HEALTHCHECK` (`docker image inspect`), `digest`/`imageId`; polecenie `e2e-qa images` pod blokadą kolejki. Kontrola narzędzia sondy HTTP (`curl`/`wget`) dla `health: { http }` (Q11). Test: obraz bez healthchecku i bez `health` daje `error: no-healthcheck`; obraz distroless z `health: { http }` daje `error: no-http-probe-tool`, a z `health: { command }` przechodzi; obraz alpine z `wget` przechodzi; drugi build tego samego kontekstu korzysta z cache (czas i log BuildKit `CACHED`); błąd w Dockerfile daje `error: build-failed` z ogonem; niezacommitowana zmiana w ścieżce lokalnej jest widoczna w zbudowanym obrazie.

### Faza 4: start stosu

9. Przydział portów publikowanych i sekretów przed `up`; placeholdery w widoku kontenera i przeglądarki (`url`, `publicUrl`, `urls`, `origins`); podstawianie wartości wejściowych w exec form. Test jednostkowy: odwołanie do usługi zależnej rozwiązuje się; `origins` zawiera obie formy; `${identity}` ze znakami powłoki trafia jako jeden argument, a wartość z wiodącym `-` jest odrzucana.
10. Render `files` do `runs/<id>/files/` (katalog 0700, pliki 0644) i montowanie tylko do odczytu pod `mountPath`; przełączniki `env` i `file`. Test na Linuksie z kontenerem działającym jako użytkownik inny niż root: kontener czyta plik z rozwiązanym `publicUrl` i nie może go zapisać. Przełącznik `file` zmienia wartość pod wskaźnikiem. Żaden plik w repozytoriach nie powstaje.
11. Generowanie pliku compose:
    - projekt `e2eqa-<runId>-<variant>`, etykiety `e2e-qa.run`, `mem_limit`, healthchecki;
    - usługi `<n>-migrate` z odziedziczonym `dependsOn` i `service_completed_successfully`, `depends_on: service_healthy`;
    - publikacja na `127.0.0.1`, nazwany wolumen `/e2e-qa`, `E2E_QA_MODE=1`;
    - zmienne usług przez `env_file` w `secrets/`, plik compose bez wartości sekretów;
    - `up --wait` z limitem czasu; ustalenie minimalnej wersji Compose i jej kontrola w preflighcie.

    Test integracyjny: kolejność postgres → `api-migrate` (startuje dopiero po zdrowym postgresie) → api → web; nieudana migracja przerywa `up`; `up --wait` kończy się sukcesem przy zakończonych usługach migracji; `grep` sekretu w `compose/` nic nie znajduje.
12. Preflight (Docker, Compose v2, Buildx, `git`) i twarda bramka pamięci (kontenery vs budżet vs `MemTotal`) z `--ignore-memory-gate`, ostrzeżenie o cudzych kontenerach, wstrzykiwane źródło pomiarów. Test jednostkowy bez Dockera: zaniżony budżet daje `blocked: memory` z liczbami; obejście jest zapisane w `state.json`. Test integracyjny: kontener przekraczający `mem_limit` daje `OOMKilled` i status `degraded`.
13. Kolejka FIFO z blokadą po `runId`, żywotność z etykiet compose i `state.json` (PID CLI z czasem startu tylko w `preparing`), status `degraded`, `up`/`down`/`status`/`down --stale`, `logs` (`--timestamps`, filtr `--since`, maskowanie przy odczycie, zrzut przy `down`), sprzątanie po etykietach, retencja `runs/`, SIGINT/SIGTERM, kody wyjścia. Test: drugi `up` czeka na działający stos i kończy się komunikatem z `runId` po `waitMinutes`; run z zatrzymanymi kontenerami zostaje sprzątnięty; zajęty port wymusza pełny cykl ponownego renderu z nowymi `publicUrl`; `down` nie dotyka kontenerów bez etykiety runu; sekret wypisany przez usługę nie pojawia się w `logs` ani w zrzucie; SIGINT w trakcie budowania zostawia czysty stan; `down` z pozostałością zwraca 4.

### Faza 5: stan danych i konta

14. Hooki QA przez `docker compose exec -T` (exec form, sekrety po nazwie zmiennej, `E2E_QA_MODE=1`, odczyt `/e2e-qa` przez `docker compose cp`, limit czasu), rozpoznanie `qa-hook-missing` (126/127) i `qa-hook-inactive` (78). Test: obraz `examples/acme` bez targetu `qa` daje `qa-hook-missing`; polecenie uruchomione bez `E2E_QA_MODE` kończy się kodem 78 i daje `qa-hook-inactive`; hasło nie pojawia się w `ps` hosta w trakcie `exec`.
15. Słownik stanów w kodzie, seed z kontraktem `E2E_QA_*` i wyjściem w `/e2e-qa`, walidacja wyjścia. Test: seed `examples/acme` dla stanów standardowych i własnego, podwójny seed (idempotencja), seed z niepełnym wyjściem.
16. Generowanie haseł person, `secrets/credentials.env` (0600, `E2E_QA_PERSONA_*`, `E2E_QA_SERVICE_*_URL[_<NAZWA>]`, `E2E_QA_SERVICE_*_PUBLIC_URL[_<NAZWA>]`), `e2e-qa creds`. Test: żadne hasło ani sekret nie pojawia się w `state.json`, `env.json`, `compose/` ani w zrzucie logów.
17. Przełączniki `command` po seedzie. Test: wartość przełącznika widoczna w odpowiedzi usługi.
18. Tryby logowania: `surface` per persona i `login.notes` per powierzchnia, `login.otp` z `${identity}`, `login.rateLimitReset`, `login.session` z buforem i `ttl`, polecenia `otp <persona>`, `otp --identity` i `session`. Test: hook sesji wywołany raz w obrębie `ttl`, ponownie po wygaśnięciu; `otp --identity` przekazuje tożsamość spoza person; persona z `surface: mobile-web` dostaje w `env.json` notatki tej powierzchni.
19. `e2e-qa seed <stan>` na działającym stosie: hasła tylko dla nowych person, unieważnienie sesji, ponowne przełączniki `command`. Test: przejście `baseline` → `scoped-access` daje nową personę z hasłem, stare hasła bez zmian, bufor sesji pusty.
20. `data` (przepis `dataAccess`, zapytanie na stdin przez `exec`). Test: `data` na `examples/acme` zwraca wynik; przepis z rolą tylko do odczytu odrzuca zapis.
21. `env.v1` pełny (usługi z `publicUrl`/`url` i źródłem obrazu z digestem, persony z powierzchnią i `identifierKind`, stan, targety z `browser`, zastrzeżenie `mobile-web`, `urls` z referencjami, SHA, stan bramki) i `status --json`. Test kontraktowy: plik przechodzi `schema/env.v1.json`, a snapshot dla `examples/acme` jest stabilny. Dokumentacja dla konsumenta: manifest, kontrakt obrazu zdatnego do QA (runtime config, healthcheck, migracje, target `qa`, `E2E_QA_MODE`), seed i hook sesji.
