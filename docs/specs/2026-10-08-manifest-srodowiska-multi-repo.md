# Spec #1a: manifest środowiska multi-repo z kontraktem seeda i kont QA

- Status: **projekt; bramka Open Questions zamknięta (2026-10-09)**
- Brief: [`docs/specs/briefs/2026-10-08-manifest-srodowiska.md`](briefs/2026-10-08-manifest-srodowiska.md)
- Następny spec (#1b, styk z wykonawcą QA): [`docs/specs/briefs/2026-10-09-styk-z-wykonawca-qa.md`](briefs/2026-10-09-styk-z-wykonawca-qa.md)
- Wejście z eksperymentu: [`docs/research/2026-10-08-scenariusze-z-ac-probe.md`](../research/2026-10-08-scenariusze-z-ac-probe.md), sekcja „Problemy środowiska”
- Wszystkie przykłady dotyczą fikcyjnego konsumenta **acme** (`api`, `web`, `mobile-web`, `ml-service`).

## 📝 TLDR

Zespół, którego zmiany wytwarzają agenci, nie ma dziś kim i czym przejść zmiany end-to-end. Po runie agenta nikt nie stawia całego ekosystemu z kilku repozytoriów, a ręczne skrypty środowiska są zaszyte pod jedną funkcję. **Proponujemy** CLI `e2e-qa` (Node/TypeScript, `npx`):

- czyta manifest YAML konsumenta;
- stawia lokalnie stos w zadanych refach repozytoriów (z własnego cache albo z istniejących checkoutów);
- doprowadza dane do nazwanego stanu i przygotowuje konta QA;
- opisuje gotowe środowisko w maszynowym `env.json`.

Środowisko jest użyteczne samo w sobie, także dla ludzi. Przekazanie go wykonawcy QA to spec #1b. Generator scenariuszy, raport z kontraktem dowodu, pętla naprawy i UI to specy #2–#4.

## 📝 Decyzje bramki

| # | Pytanie | Decyzja (2026-10-09) |
|---|---|---|
| Q1 | Podział speca | Dwa specy: **#1a** (ten) = manifest, seed i konta QA; **#1b** = styk z wykonawcą QA (brief obok) |
| Q2 | Runtime | Node/TypeScript, dystrybucja przez `npx` |
| Q3 | Format manifestu | YAML walidowany JSON Schema |
| Q3b | Miejsce manifestu | Dowolne: narzędzie przyjmuje ścieżkę; konwencja leży poza rdzeniem |
| Q4 | Styk z wykonawcą | **Dotyczy #1b; otwarte ponownie 2026-10-09** (scenariusz obejmuje kilka powierzchni, a deskryptor v1 ma jeden `baseUrl`). #1a dostarcza dane przez `env.json` z wieloma `targets` |
| Q5 | Wejście runu | Rdzeń: refy `repo → ref` (narzędzie nie zna trackera). Nakładka: PR rozwiązywany do refu |
| Q5b | Paczka PR-ów | Gotowy ref: gałąź integracyjną dostarcza operator lub orkiestrator; narzędzie nie scala |
| Q6 | Warianty stosu | Kontrakt teraz (schemat przewiduje warianty), implementacja kilku wariantów później |
| Q7 | Stany seeda | Słownik zalecany: standardowe nazwy z gwarancjami plus dowolne stany własne |
| Q8 | Sesja kont QA | Tryb per konto: logowanie przez UI (referencje poświadczeń) albo hook sesji konsumenta |
| Q9 | Budżet pamięci | Twarda bramka z jawną flagą obejścia |
| Q10 | Kod repozytoriów | Ścieżka lokalna, gdy podana; inaczej własny cache z worktree per run |

## 📝 Problem Statement

Uogólnione z eksperymentu i z briefu (szczegóły konsumenta zostają w jego repozytoriach):

- Skrypt środowiska zaszyty pod jedną funkcję: bramki gałęzi, sztywne porty, jeden projekt compose. Nie da się go odpalić na dowolnych wersjach.
- Regresja paczki kilkunastu PR-ów na drugim stosie skończyła się OOM-em, kolizjami portów, limitami logowania i brakiem OTP w dev.
- Seed woła `docker compose exec` na sztywno, fikstury trafiają w ograniczenia schematu, a feature flagi wymagają ręcznej edycji zagnieżdżonego JSON-a.
- Logowanie ma pułapki UI i bramki po pierwszym logowaniu. Testujący (człowiek albo agent) traci czas i limit prób logowania na rekonesans.
- Proxy frontendu wskazuje sztywny adres API.
- Strażnik ścieżek agenta blokuje wszystko poza katalogiem roboczym.

Skutek: testowanie zmian agentów zależy od jednej osoby, a paczki wydań czekają.

## 📝 Proposed Solution

Wejścia i wyjście:

- **Manifest** (konsument): stała topologia: repozytoria, usługi, zależności, hooki, stany seeda, przełączniki, konta i logowanie, dostęp do danych.
- **Plik runu** (operator albo orkiestrator): część zmienna: refy lub ścieżki lokalne, stan danych, wartości przełączników.
- **Konfiguracja operatora** (opcjonalna, w katalogu roboczym): kolejka, budżety, domyślne ścieżki.
- **Wyjście:** działający stos, `env.json` (maszynowy opis środowiska) i `e2e-qa status` dla człowieka.

Przebieg `e2e-qa up run.yaml`:

1. Wczytanie manifestu ze ścieżki z runu i walidacja (schemat + reguły semantyczne).
2. Preflight: Docker działa, wymagane narzędzia hosta są dostępne, stan i przełączniki z runu są zdefiniowane.
3. Przygotowanie kodu. Dla każdego repo: ścieżka lokalna albo fetch do cache (z blokadą per repo) i worktree na podany ref. PR-y z nakładki są wcześniej rozwiązywane do refów.
4. Kolejka „jeden run naraz”, potem twarda bramka pamięci.
5. Przydział wszystkich portów i sekretów oraz render plików `files` (we wszystkich usługach, przed pierwszym `prepare`). Start supervisora runu. Potem start w kolejności grafu zależności. Przełączniki `env` i `jsonFile` są aplikowane przed startem usługi, której dotyczą. Dalej: kontenery, migracje, usługi hosta (dzieci supervisora, nie CLI), seed, przełączniki `command`, reset limitów logowania, sesje dla kont w trybie `session` i sondy zdrowia.
6. Zapis `env/<variant>.json` i plików poświadczeń; wypisanie linii wyniku.

`e2e-qa down` sprząta tylko to, co ten run postawił.

**Odrzucone alternatywy.** Z briefu: workflow w orkiestratorze lub same skille, „nic nie budować”, serwer/PaaS na start, własny wykonawca przeglądarkowy. Z bramki: jeden spec dla środowiska i wykonawcy, scalanie paczek przez narzędzie (to odpowiedzialność orkiestratora, a scalanie lokalne różniłoby się od realnego), zamknięty słownik stanów (blokowałby konsumenta), wyłącznie logowanie przez UI.

## 📝 Research: czego uczą liderzy

| Narzędzie | Co robią dobrze | Co bierzemy | Czego nie bierzemy |
|---|---|---|---|
| Garden | Graf akcji (build/deploy/run/test) z cache po hashu wersji | Graf zależności z hookami `prepare`/`migrate`/`seed` jako węzłami | Kubernetes, zdalne klastry, cache wersji (później) |
| Tilt | `local_resource` obok `docker_compose` w jednym grafie | Tryb hybrydowy: usługa `host` lub `container` w jednym manifeście | Live update, dashboard |
| Testcontainers | Losowe porty, strategie oczekiwania na gotowość | Porty zawsze przydzielane, sondy `http`/`tcp`/`command` z limitem czasu | Cykl życia sterowany z kodu testów |
| Docker Compose | `-p` izoluje projekty, `mem_limit`, `depends_on: condition: service_healthy` | Compose jako backend kontenerów (plik generowany per run) | Własny runtime kontenerów |
| Seedery Rails/Laravel, `cy.task` w Cypress | Nazwane seedery wołane poleceniem aplikacji | Stan danych = polecenie konsumenta z nazwą stanu | Fikstury SQL pisane przez narzędzie |
| Playwright `storageState` / `cy.session` | Logowanie raz, zapis sesji, reużycie | Tryb `session`: hook konsumenta zwraca sesję, narzędzie ją buforuje per run | Logowanie programowe w rdzeniu |

Wniosek: żaden z nich nie łączy **wielu repozytoriów w zadanych refach** z **semantycznym słownikiem stanów danych i person**. To jest wartość tego narzędzia. Orkiestrację kontenerów delegujemy do Compose, zamiast ją budować.

## 📝 Architecture

```mermaid
flowchart LR
    classDef newC fill:#2f6feb,color:#fff,stroke:#1b4fb0
    classDef extC fill:#e5e7eb,color:#111,stroke:#9ca3af
    classDef planC fill:#fff,color:#111,stroke:#2f6feb,stroke-dasharray:4

    n1["Plik runu"]:::newC
    n2["Manifest YAML (konsument)"]:::extC
    n3["e2e-qa CLI"]:::newC
    n4["Cache repo / ścieżki lokalne"]:::newC
    n5["Docker Compose"]:::extC
    n6["Procesy hosta"]:::newC
    n7["Hooki konsumenta (migrate, seed, otp, session, data)"]:::extC
    n8["env.json"]:::newC
    n9["#1b: styk z wykonawcą QA"]:::planC

    n1 --> n3
    n2 --> n3
    n3 --> n4
    n3 --> n5
    n3 --> n6
    n3 --> n7
    n3 --> n8
    n8 -.-> n9
```

Niebieskie elementy są nowe, szare już istnieją, a przerywana ramka to praca planowana. Rdzeń nie zna żadnego wykonawcy ani trackera. Jedynym punktem styku dla #1b jest `env.json`.

| Moduł | Odpowiedzialność |
|---|---|
| `manifest` | Parsowanie YAML, JSON Schema (`schema/manifest.v1.json`, `schema/run.v1.json`), reguły semantyczne |
| `repos` | Ścieżki lokalne (tylko odczyt) albo cache lustrzany, fetch i worktree; nakładka PR → ref |
| `graph` | Graf usług i hooków, kolejność startu, wykrywanie cykli |
| `ports` | Przydział portów, rozwiązywanie placeholderów w dwóch widokach (host, kontener) |
| `runtime-container` | Generowanie pliku compose, `pull`/`up`/`down`/`exec`, logi |
| `runtime-host` | `prepare`, start usług jako dzieci supervisora (każda we własnej grupie procesów) |
| `supervisor` | Odłączony proces per run: rodzic usług hosta, pompa logów (usługi hosta i `docker compose logs -f`) z maskowaniem i znacznikami czasu, źródło prawdy o żywotności runu |
| `gate` | Kolejka i twarda bramka pamięci |
| `state` | Słownik stanów, seed, persony, przełączniki, sesje, OTP, dostęp do danych |
| `output` | `env.json`, pliki poświadczeń, `status`, maskowanie sekretów |

## 📝 Data Model

Wszystkie encje to pliki (YAML lub JSON). Narzędzie nie ma bazy danych.

```mermaid
flowchart LR
    classDef newEntity fill:#2f6feb,color:#fff,stroke:#1b4fb0

    entity_1["Manifest"]:::newEntity
    entity_2["Repo"]:::newEntity
    entity_3["Service"]:::newEntity
    entity_4["SeedState"]:::newEntity
    entity_5["Persona"]:::newEntity
    entity_6["Toggle"]:::newEntity
    entity_7["RunFile"]:::newEntity
    entity_8["Variant"]:::newEntity
    entity_9["RunState (state.json)"]:::newEntity
    entity_10["EnvDescription (env.json)"]:::newEntity

    entity_1 -->|1-n| entity_2
    entity_1 -->|1-n| entity_3
    entity_3 -->|n-1| entity_2
    entity_1 -->|1-n| entity_4
    entity_1 -->|1-n| entity_5
    entity_4 -->|n-n| entity_5
    entity_1 -->|1-n| entity_6
    entity_7 -->|n-1| entity_1
    entity_7 -->|1-n| entity_8
    entity_7 -->|n-1| entity_4
    entity_9 -->|1-1| entity_7
    entity_9 -->|1-n| entity_10
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
  host:   { memory: 8g }

services:
  postgres:
    runtime: container
    image: postgres:16
    port: 5432                      # port wewnętrzny; zewnętrzny przydziela narzędzie
    memory: 512m
    env: { POSTGRES_PASSWORD: "${secret.postgres}", POSTGRES_DB: acme }   # sekret jednorazowy, per run
    url: "postgres://postgres:${secret.postgres}@${self.host}:${self.port}/acme"
    urls:                           # nazwane dialekty tej samej bazy (np. sterownik async i sync)
      async: "postgresql+asyncpg://postgres:${secret.postgres}@${self.host}:${self.port}/acme"
    health: { command: pg_isready -U postgres }
  redis:
    runtime: container
    image: redis:7
    port: 6379
    memory: 128m
    url: "redis://${self.host}:${self.port}"
    health: { command: redis-cli ping }
  ml:
    runtime: host
    repo: ml
    requires: [uv]
    prepare: [uv sync]
    start: uv run serve --host ${self.bind} --port ${self.port}
    memory: 1g
    env: { DATABASE_URL: "${services.postgres.urls.async}", MIGRATIONS_DATABASE_URL: "${services.postgres.url}", DB_SCHEMA: ml }
    migrate: { run: uv run migrate }
    dependsOn: [postgres]
    health: { http: /health }
  api:
    runtime: host
    repo: api
    requires: [node>=20]
    prepare: [npm ci, npm run build]
    start: npm run start
    port: { env: PORT }
    memory: 700m
    env:
      DATABASE_URL: "${services.postgres.url}"
      REDIS_URL: "${services.redis.url}"
      ML_URL: "${services.ml.url}"
      # odwołanie „w górę grafu”: web i mobile-web zależą od api, a api zna ich originy (CORS)
      CORS_ORIGINS: "${services.web.origins},${services.mobile-web.origins}"
    migrate: { run: npm run db:migrate, timeout: 5m }
    dependsOn: [postgres, redis, ml]
    health: { http: /health }
  web:
    runtime: host
    repo: web
    prepare: [npm ci]
    start: npm run dev -- --port ${self.port} --proxy-config proxy.e2e.json
    memory: 900m
    files:                           # dev-serwer czyta adres API z pliku, nie ze zmiennej procesu
      - path: proxy.e2e.json
        format: json
        content: { "/api": { "target": "${services.api.url}", "changeOrigin": true } }
    dependsOn: [api]
    health: { http: / }
    target: web
    browser: { locale: pl-PL }
  mobile-web:
    runtime: host
    repo: mobile
    prepare: [npm ci]
    start: npm run web -- --port ${self.port}
    memory: 900m
    files:                           # bundler daje plikom .env* pierwszeństwo przed zmiennymi procesu
      - path: .env.local
        format: dotenv
        content: { ACME_PUBLIC_API_URL: "${services.api.url}" }
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
  run: { in: host, repo: api, command: "npm run qa:seed", timeout: 5m }
  after: [api.migrate, ml.migrate]
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
    apply: { jsonFile: { repo: api, path: config/flags.json, pointer: /billing/v2 } }
  maintenance-banner:
    default: false
    apply: { command: { run: { in: host, repo: api, command: "npm run qa:flag -- maintenance-banner ${toggle.value}" } } }

login:
  surface: web                       # domyślna powierzchnia; persona może ją nadpisać
  notes:                             # lista (wszystkie powierzchnie) albo mapa per powierzchnia
    web:
      - Po wejściu na stronę logowania zamknij baner cookies; zasłania przycisk „Dalej”.
    mobile-web:
      - Identyfikatorem jest numer telefonu; po wpisaniu przewiń w dół, klawiatura zasłania przycisk.
  otp: { run: { in: host, repo: api, command: "npm run qa:otp -- ${identity}" } }
  session: { run: { in: host, repo: api, command: "npm run qa:session -- ${persona.username}" }, ttl: 30m }
  rateLimitReset: { run: { in: host, repo: api, command: npm run qa:reset-login-limits } }

dataAccess:                          # wymaganie 9; przepis i gwarancja „tylko odczyt” należą do konsumenta
  db:
    run: { in: postgres, command: "psql -U qa_readonly -d acme -v ON_ERROR_STOP=1 -f -" }   # zapytanie na stdin

# qa: klucz zarezerwowany dla speca #1b
```

Reguły manifestu:

- **Porty nigdy nie są stałe.** `port` to port wewnętrzny kontenera albo sposób przekazania portu usłudze hosta (`${self.port}` w poleceniu lub `port.env`). Zewnętrzny port zawsze przydziela narzędzie.
- **Placeholdery** rozwiązują się w **widoku konsumenta**:
  - lista: `${services.<n>.url|urls.<nazwa>|host|port|origins}`, `${self.host|port|bind}`, `${secret.<n>}`, `${run.id}`, `${variant.name}`, `${persona.username}`, `${identity}`, `${toggle.value}`;
  - **placeholdery nie tworzą krawędzi grafu.** Krawędzie tworzą wyłącznie `dependsOn` i `seed.after`. Wszystkie porty i sekrety przydziela się przed startem pierwszej usługi, więc usługa może odwołać się do usługi, która od niej zależy (np. lista originów CORS w `api` zawiera URL-e frontendów zależnych od `api`). Cykl powstaje tylko w `dependsOn`;
  - `${services.<n>.origins}` jest **zawsze w widoku przeglądarki**, niezależnie od widoku konsumenta. Daje oba originy usługi HTTP rozdzielone przecinkiem: `http://127.0.0.1:<port>,http://localhost:<port>`. Przeglądarka traktuje `127.0.0.1` i `localhost` jako różne originy. `env.json` publikuje URL-e w formie `127.0.0.1` i tej formy powinni używać testujący. Forma `localhost` jest na liście dla dev-serwerów i przekierowań, które same ją wybierają (uwaga: `localhost` może rozwiązać się do `::1`, a usługi hosta słuchają na IPv4). W pliku `format: json` wartość będąca w całości tym placeholderem renderuje się jako tablica JSON;
  - wartości pochodzące z wejścia w czasie działania (`${identity}`, `${persona.username}`, `${toggle.value}`) nigdy nie są interpretowane przez powłokę:
    - narzędzie wstawia je jako pojedynczy argument z cytowaniem dla ostatniej warstwy powłoki;
    - dla `in: <usługa>` buduje tablicę argumentów `docker compose exec` bez pośredniej powłoki;
    - wartości trafiają też do zmiennych `E2E_QA_IDENTITY`, `E2E_QA_PERSONA_USERNAME`, `E2E_QA_TOGGLE_VALUE`;
    - walidacja odrzuca te placeholdery umieszczone wewnątrz cudzysłowów w szablonie polecenia, a narzędzie odrzuca wartość zaczynającą się od `-` (wstrzyknięcie opcji);
  - `${secret.<n>}` w v1 jest zawsze **generowany** per run (losowy, jednorazowy). Pobieranie sekretów z zewnętrznych źródeł jest poza zakresem;
  - usługa hosta dostaje `127.0.0.1:<port zewnętrzny>`;
  - kontener dostaje nazwę usługi z sieci compose (gdy łączy się z innym kontenerem) albo `host.docker.internal:<port>` (gdy łączy się z usługą hosta). Generowany plik compose zawsze ustawia `extra_hosts: host.docker.internal:host-gateway`, więc działa to także na Linuksie i w CI;
  - `${self.bind}` ma wartość `127.0.0.1`. Usługa hosta dostaje `0.0.0.0`, a narzędzie wypisuje ostrzeżenie, tylko gdy sięga do niej kontener: przez `dependsOn` **albo** przez placeholder w `env`, `files` lub poleceniu kontenera (dla bindu liczą się oba rodzaje odwołań, choć placeholdery nie tworzą krawędzi startu);
  - nierozwiązany placeholder to błąd walidacji, a nie pusty string.
- **`url`** usługi to szablon (domyślnie `http://${self.host}:${self.port}`). Usługi bez protokołu HTTP muszą go podać. **`urls`** to opcjonalne nazwane warianty tego samego adresu, np. dialekty URL-a bazy dla sterownika async i sync. Każdy nazwany URL jest maskowany w `env.json` i dostaje własną referencję (`E2E_QA_SERVICE_<USŁUGA>_URL_<NAZWA>`, np. `E2E_QA_SERVICE_POSTGRES_URL_ASYNC`). Alternatywa, czyli składanie URL-a z `host`/`port` w `env` usługi, działa, ale omija maskowanie. Dlatego nazwane `urls` są zalecane.
- **`files`** to pliki konfiguracyjne usługi renderowane z szablonu do katalogu repo usługi **po** checkoucie i **przed** `prepare`/`start`. Pozycja ma postać `{ path, format: dotenv | json | text, content | template }`. `content` to mapa (dla `dotenv`/`json`) albo tekst; `template` to ścieżka do pliku szablonu względem manifestu. Placeholdery rozwiązują się w widoku usługi. Plik zastępuje całą istniejącą treść. Przed zapisem narzędzie robi `realpath` ścieżki i odmawia zapisu, gdy wynik wychodzi poza katalog repo albo gdy ścieżka prowadzi przez symlink. `env` nie wystarcza z dwóch powodów:
  - część bundlerów daje plikom `.env*` z repo pierwszeństwo przed zmiennymi procesu, więc aplikacja wstaje podpięta pod adres zapisany w repo, który może wskazywać zupełnie inne środowisko;
  - dev-serwery frontendu czytają adres API z pliku proxy.

  Zapis w worktree z cache jest bez ograniczeń. Ograniczenia dla ścieżek lokalnych opisuje plik runu.
- **Hook `run`** ma pola `in: host | <usługa kontenerowa>` i `timeout` (domyślnie 10 min; dla `prepare` 20 min). Dla `in: <usługa>` narzędzie samo wykonuje `docker compose -p <projekt> exec`. Konsument nigdy nie zaszywa nazwy projektu compose.
- **Każde wywołanie zewnętrzne ma limit czasu:** hooki, `docker compose pull` i `git fetch` (z `GIT_TERMINAL_PROMPT=0`, żeby nie zawisnąć na pytaniu o hasło). `pull` i `fetch` mają po 2 ponowienia.
- **Przełączniki** mają trzy rodzaje `apply`: `env` (zmienna usługi przed jej startem), `jsonFile` (wskaźnik JSON w pliku worktree z cache, przed startem usługi) i `command` (hook z `${toggle.value}`, po seedzie).
- **`requires`** to lista poleceń hosta (opcjonalnie z wersją minimalną), sprawdzana w preflighcie.
- **`dependsOn`** czeka na zdrowie zależności. `migrate` usługi wykonuje się po zdrowiu jej zależności i przed startem samej usługi. `seed.after` wymienia migracje wymagane przez seed.
- **`memory`** jest obowiązkowe dla każdej usługi (dla kontenerów staje się `mem_limit`, dla usług hosta jest deklaracją).
- **`target: web | mobile-web`** oznacza usługi, na których człowiek lub wykonawca widzi produkt. `mobile-web` niesie w `env.json` zastrzeżenie, że to nie jest build natywny. Opcjonalne **`browser`** (`locale` dla `navigator.language`/`Accept-Language`, `viewport { width, height, mobile }`) to parametry przeglądarki tej powierzchni. Trafiają do `env.json` jako `targets[].browser`, bo należą do środowiska, a nie do scenariusza. Konsumuje je spec #1b. Obiekt `browser` jest opcjonalny i otwarty: nowe klucze dodane przez #1b nie wymagają podbicia `env.v1`.
- **`prRef`** (opcjonalne) to wzorzec refu PR-a dostępny przez git (np. `refs/pull/{n}/head` albo `refs/merge-requests/{n}/head`). Na nim opiera się nakładka PR. Rdzeń nie wywołuje API trackera.

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

**Kontrakt polecenia seeda.** Narzędzie przekazuje w środowisku:

- `E2E_QA_STATE`: nazwę stanu;
- `E2E_QA_PERSONA_<NAZWA>_PASSWORD`: hasło każdej wymaganej persony, generowane raz na run i stałe przez cały run;
- `E2E_QA_SEED_OUTPUT`: ścieżkę pliku wyjścia.

`<NAZWA>` to nazwa persony wielkimi literami, w której każdy znak inny niż litera lub cyfra zamieniono na `_` (`scoped-member` → `SCOPED_MEMBER`). Ta sama nazwa zmiennej (`E2E_QA_PERSONA_<NAZWA>_PASSWORD`) obowiązuje w `credentials.env` i w polu `passwordEnv` w `env.json`; jest jedna konwencja. Seed **doprowadza dane dokładnie do stanu**: usuwa ślady poprzedniego stanu i jest idempotentny.

`e2e-qa seed <stan>` na działającym stosie to reset, a nie dopisanie:

- generuje hasła tylko dla person nowych w tym runie (istniejące zostają);
- unieważnia bufor sesji wszystkich person;
- ponownie stosuje przełączniki `command`;
- odświeża `env/<variant>.json`.

Seed zakłada konta z tymi hasłami i zapisuje do `E2E_QA_SEED_OUTPUT` JSON:

```json
{ "state": "scoped-access",
  "personas": { "admin": { "username": "qa-admin@acme.test" }, "scoped-member": { "username": "qa-scoped@acme.test" }, "outsider": { "username": "qa-outsider@acme.test" } },
  "resources": { "A": { "label": "Projekt Alfa" }, "B": { "label": "Projekt Beta" } } }
```

Brak wymaganej persony albo zasobu z gwarancji oznacza błąd runu. Hasła nigdy nie wracają w wyjściu seeda. `username` to identyfikator, którym persona loguje się na **swojej** powierzchni (e-mail, telefon, login). Opcjonalne `identifierKind: email | phone | username` w wyjściu seeda podpowiada testującemu, którego pola użyć; dostarcza je seed, bo tylko on zna założone konto.

### Konta i sesje (tryb i powierzchnia per persona)

Każda persona ma `login` (tryb) i `surface` (powierzchnię logowania, domyślnie `login.surface`). Konsument może mieć różne powierzchnie z różnymi identyfikatorami, np. panel web z e-mailem i hasłem oraz aplikację mobile-web z telefonem i hasłem. `login.notes` może być listą wspólną albo mapą per powierzchnia. `surface` persony i klucze mapy `login.notes` muszą wskazywać usługi z `target`, inaczej walidacja zgłasza błąd. `env.json` podaje każdej personie jej powierzchnię i odpowiednie notatki.

| `login` | Co dostaje testujący | Hooki |
|---|---|---|
| `ui` | Login i referencję hasła; loguje się przez UI swojej powierzchni. Pomocniczo `login.notes` (pułapki UI) oraz `e2e-qa otp`, gdy aplikacja wymaga OTP | `login.otp` (opcjonalny), `login.rateLimitReset` |
| `session` | Gotowy artefakt sesji: plik JSON `{ cookies?, origins?/storageState?, headers?, token?, expiresAt? }` | `login.session` (obowiązkowy dla tego trybu); hook sam obsługuje OTP |

Narzędzie wywołuje hook sesji przy `up` dla każdej persony w trybie `session` i buforuje wynik w pliku 0600 w katalogu runu. `e2e-qa session <persona>` zwraca ścieżkę do ważnej sesji i odświeża ją po upływie `ttl` albo `expiresAt`, co ogranicza zużycie limitów logowania. Wstrzyknięcie sesji do przeglądarki należy do wykonawcy (spec #1b) albo do człowieka.

**OTP dla dowolnej tożsamości.** Hook `login.otp` dostaje `${identity}`. `e2e-qa otp <persona>` to przypadek szczególny, w którym `identity` jest `username` persony. `e2e-qa otp --identity <id>` obsługuje tożsamości spoza person, np. konto zarejestrowane w trakcie scenariusza. Środowisko i dane są jednorazowe, więc odczyt OTP dowolnego konta w nim jest zamierzony.

### Plik runu (`version: 1`)

```yaml
version: 1
manifest: ./acme-e2e/manifest.yaml       # ścieżka względem pliku runu
state: scoped-access
toggles: { new-dashboard: true }
variants:
  default:                                # v1: dokładnie jeden wariant
    refs:
      api: pr:412                         # nakładka: rozwiązywane przez repos.api.prRef
      web: integration/release-train      # gotowa gałąź integracyjna od orkiestratora
    paths:
      mobile: { path: ../worktrees/mobile-task, prepare: false }   # ścieżka lokalna zamiast cache
```

- Repo nieobecne w `refs` i w `paths` dostaje swój `defaultRef`. Repo z jednoczesnym wpisem w `refs` i w `paths` to błąd walidacji.
- Każdy wpis `refs` to **jeden** ref: gałąź, tag, SHA albo `pr:<n>`. Listy refów nie istnieją, bo narzędzie nie scala.
- `variants` to mapa nazw `[a-z0-9-]`. Schemat v1 dopuszcza wiele wariantów, ale implementacja v1 odrzuca więcej niż jeden komunikatem „nieobsługiwane w tej wersji”. Nazwy projektów compose (`e2eqa-<runId>-<variant>`), porty, katalogi i opis środowiska (`env/<variant>.json`, jedna linia `E2E_QA_ENV_<VARIANT>=…` na wariant) już teraz są per wariant, więc dodanie wariantów nie zmieni kontraktu.
- **Ścieżki lokalne należą do operatora.** Narzędzie nie robi w nich checkoutu, fetcha ani resetu i nie zmienia plików śledzonych przez git. Zapisuje SHA `HEAD` i flagę `dirty`.
  - `prepare` domyślnie **pomija** (np. `npm ci` skasowałoby `node_modules` operatora) i wypisuje o tym linię. Włącza się je jawnie przez `prepare: true`.
  - **`files` wolno zapisać wyłącznie do pliku ignorowanego przez git.** Preflight sprawdza to przez `git check-ignore`. Istniejąca wersja pliku trafia do `runs/<runId>/secrets/backup/<repo>/<path>` (0600, bo pliki `.env*` operatora mogą zawierać jego prawdziwe sekrety), a `down` (także `down --stale`) ją przywraca. Plik, którego wcześniej nie było, `down` usuwa. `state.json` zawiera tylko ścieżki, sumy kontrolne i stan przywrócenia, nigdy treść. Plik śledzony przez git (albo nieignorowany) blokuje run w preflighcie, z nazwą pliku i podpowiedzią (dopisać wzorzec do `.gitignore` w repo konsumenta albo użyć źródła `cache`).
  - Przełącznik `jsonFile` wskazujący repo ze ścieżki lokalnej blokuje run w preflighcie.

### Konfiguracja operatora (`.e2e-qa/config.yaml`, opcjonalna)

Pola: `queue.waitMinutes` (domyślnie 60), `retention.runs` (ile katalogów zakończonych runów zachować z logami, domyślnie 20) i nadpisanie budżetów pamięci. Flaga `--memory-budget` zmienia budżet. `--ignore-memory-gate` omija bramkę: wymaga podania jej jawnie przy każdym uruchomieniu, a fakt obejścia trafia do `state.json` i `env/<variant>.json`.

**Bramka pamięci.** Jest deterministyczna i opiera się wyłącznie na deklaracjach:

- suma `memory` kontenerów ≤ `resources.docker.memory` ≤ całkowita pamięć VM Dockera (`MemTotal` z `docker info`);
- suma `memory` usług hosta ≤ `resources.host.memory`.

Pamięć zajęta przez cudze kontenery nie jest częścią bramki, tylko **ostrzeżeniem** z liczbami: pomiar jest chwilowy i nie nadaje się na twardą regułę. Źródło pomiarów (`docker info`, `docker stats`, RSS procesów) jest wstrzykiwane, żeby bramkę dało się testować bez Dockera. Po starcie narzędzie mierzy RSS usług hosta i zapisuje ostrzeżenie, gdy pomiar przekracza deklarację.

### Katalog roboczy i stan runu

Wszystko, co tworzy narzędzie, leży pod `./.e2e-qa/` w katalogu, z którego je uruchomiono (wymaganie 10: praca pod strażnikiem ścieżek). Wyjątkiem są ścieżki lokalne podane jawnie przez operatora.

```
.e2e-qa/
  cache/<repo>.git                 # klon lustrzany (bare) + blokada pliku per repo
  lock/                            # kolejka: owner.json + bilety FIFO
  runs/<runId>/
    run.yaml  state.json  supervisor.sock
    env/<variant>.json
    secrets/                       # 0600: credentials.env, sesje person, ${secret.*}
      backup/<repo>/<path>         # kopie plików ze ścieżek lokalnych nadpisanych przez files
    compose/<variant>.yaml
    logs/<variant>/<service>.log
    worktrees/<variant>/<repo>/
```

`state.json` zawiera:

- refy rozwiązane do SHA (z flagą `dirty` dla ścieżek lokalnych);
- przydzielone porty i nazwy projektów compose;
- pliki wyrenderowane z `files` (ze ścieżkami kopii zapasowych dla ścieżek lokalnych);
- PID supervisora i usług hosta, każdy z **czasem startu procesu**;
- status: `preparing | running | degraded | stopped | failed`.

Na tej podstawie da się sprzątnąć osierocony run.

**Cykl życia i blokada.** `up` kończy proces CLI, a stos działa dalej. Właścicielem usług hosta jest **supervisor runu**: odłączony proces (`setsid`), uruchamiany przez `up` jako pierwszy. Supervisor:

- jest rodzicem usług hosta (każda we własnej grupie procesów);
- pompuje ich wyjście oraz `docker compose logs -f` do `logs/` z maskowaniem sekretów i znacznikami czasu;
- przyjmuje polecenia CLI przez `supervisor.sock`.

Dzięki temu logi mają właściciela po wyjściu CLI, a `logs --since` filtruje po znacznikach czasu nadanych przez supervisora. Blokada kolejki należy do **`runId`, nie do PID-u CLI**:

- Blokadę zwalnia `down`.
- Run jest porzucony, gdy supervisor nie żyje. Żywotność weryfikujemy po PID **i** czasie startu procesu, więc PID ponownie użyty po restarcie hosta nie liczy się jako żywy.
- W fazie `preparing` run jest porzucony także wtedy, gdy nie żyje proces CLI z `owner.json`.
- Usługa, która padła przy żywym supervisorze, zmienia status runu na `degraded`, widoczny w `status`. Run trzyma blokadę, dopóki operator nie zrobi `down`.

Tylko porzucony run jest sprzątany automatycznie. Sprzątanie zabija grupy procesów wyłącznie po weryfikacji czasu startu. Działający lub zdegradowany stos innego runu oznacza czekanie w kolejce. Po `waitMinutes` czekający run kończy się komunikatem z `runId` i statusem właściciela blokady oraz poleceniem `e2e-qa down <runId>`.

`down` przywraca pliki ze ścieżek lokalnych z `secrets/backup/` (przed wszystkim innym, żeby awaria dalszego sprzątania ich nie zablokowała). Usuwa też worktree należące do runu (`git worktree remove --force` + `prune`; `--force`, bo pliki z `files` są w nich nieśledzone), kontenery, wolumeny i sieć projektu compose oraz katalog `secrets/`. Jeśli jakaś kopia nie została przywrócona (np. ochrona zmian operatora), `down` usuwa resztę `secrets/`, ale tę kopię zostawia. `retention.runs` nigdy nie usuwa katalogu runu z nieprzywróconą kopią. `status` i `down` wypisują takie pliki. Logi i `state.json` zostają w zakresie `retention.runs`.

### `env/<variant>.json`: opis środowiska (kontrakt dla #1b i innych konsumentów)

W całym specu i w briefie #1b skrót `env.json` oznacza plik `env/<variant>.json`. Schemat: `schema/env.v1.json`. Plik nie zawiera żadnej wartości sekretu. URL-e z `${secret.*}` są publikowane z zamaskowanym hasłem (`***`). Pełny URL leży w `credentials.env` pod referencją `urlEnv` (`E2E_QA_SERVICE_<NAZWA>_URL`).

```json
{ "version": 1, "runId": "…", "variant": "default", "status": "running",
  "state": { "name": "scoped-access", "standard": true, "guarantees": "…", "resources": { "A": { "label": "Projekt Alfa" } } },
  "glossary": { "tenant": "organizacja" },
  "services": { "web": { "url": "http://127.0.0.1:41234", "target": "web", "runtime": "host", "repo": "web", "log": "logs/default/web.log" },
                "postgres": { "url": "postgres://postgres:***@127.0.0.1:41235/acme", "urlEnv": "E2E_QA_SERVICE_POSTGRES_URL",
                              "urls": { "async": { "url": "postgresql+asyncpg://postgres:***@127.0.0.1:41235/acme", "urlEnv": "E2E_QA_SERVICE_POSTGRES_URL_ASYNC" } },
                              "runtime": "container", "container": "e2eqa-…-postgres-1" } },
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

Kontrakt poświadczeń jest zgodny z zasadą `om-*`: `env/<variant>.json` zawiera tylko **referencje** (`passwordEnv`, `urlEnv`), a wartości leżą w `credentialsFile` (0600, poza gitem). Agent nie powinien czytać tego pliku, tylko ładować go do powłoki. Człowiek odczytuje hasło jawnie przez `e2e-qa creds <persona>`.

## 📝 API Contracts (CLI)

Kody wyjścia: **0** sukces, **2** błąd walidacji, **3** zablokowany (preflight, kolejka, pamięć, nieistniejący ref), **4** błąd infrastruktury (zdrowie, hook, timeout, sprzątanie niepełne).

| Polecenie | Działanie |
|---|---|
| `e2e-qa validate <manifest> [--run run.yaml]` | Schemat i reguły semantyczne; raport pokrycia słownika stanów |
| `e2e-qa checkout <run.yaml>` | Kroki 1–3: przygotowuje kod i wypisuje rozwiązane SHA, bez startu usług |
| `e2e-qa up <run.yaml> [--wait N] [--keep-on-failure] [--memory-budget X] [--ignore-memory-gate]` | Kroki 1–6; wypisuje `E2E_QA_RUN_ID=…`, `E2E_QA_STATUS=running` i po jednej linii `E2E_QA_ENV_<VARIANT>=<ścieżka>` na wariant |
| `e2e-qa down [runId \| --stale]` | Sprząta tylko to, co zapisane w `state.json`; idempotentne; zwraca 4 i listę pozostałości, gdy coś zostało |
| `e2e-qa status [runId] [--json]` | Usługi, URL-e, persony (bez haseł), zdrowie, nieprzywrócone pliki ze ścieżek lokalnych |
| `e2e-qa seed <state>` | Reset danych do stanu; nowe hasła tylko dla nowych person, unieważnienie sesji, ponowne przełączniki `command`, odświeżenie `env/<variant>.json` |
| `e2e-qa creds <persona>` | Wypisuje login i hasło (dla człowieka; nigdy do logów) |
| `e2e-qa session <persona>` | Ścieżka do ważnej sesji persony w trybie `session` (odświeża, gdy wygasła) |
| `e2e-qa otp <persona> \| --identity <id>` | Uruchamia hook `login.otp` dla persony albo dowolnej tożsamości, wypisuje kod |
| `e2e-qa logs <service> [--since 5m]` | Logi z pliku (bez `docker logs --since`) |
| `e2e-qa data <name> < query` | Przepis `dataAccess.<name>` konsumenta z zapytaniem na stdin |

Bez `runId` polecenia działają na jedynym działającym runie. Jeśli żaden nie działa albo działa kilka, kończą się błędem z listą runów.

## 📝 Edge Cases & Failure Scenarios

| Sytuacja | Zachowanie i co widzi operator |
|---|---|
| Ref albo `pr:<n>` nie istnieje, fetch odmawia (uprawnienia) | `blocked` przed kolejką, z nazwą repo i refu. Uwierzytelnienie git pochodzi z konfiguracji operatora. `GIT_TERMINAL_PROMPT=0` zamienia pytanie o hasło w błąd |
| `pr:<n>` dla repo bez `prRef` | Błąd walidacji runu |
| Ścieżka lokalna nie jest repozytorium git albo nie istnieje | `blocked` w preflighcie |
| Ścieżka lokalna ma niezacommitowane zmiany | Run idzie dalej; `dirty: true` w `state.json` i `env.json` (wynik może być nieodtwarzalny) |
| Fetch lub pobieranie obrazu wisi albo sieć zrywa | Limit czasu, 2 ponowienia, potem `error` z nazwą repo lub obrazu |
| Dwa runy jednocześnie robią fetch do tego samego klona | Blokada pliku per repo w `cache/`; drugi czeka |
| Stan z runu nie istnieje w manifeście | Błąd walidacji z listą dostępnych stanów |
| Stan standardowy z innymi personami niż słownik | Błąd walidacji |
| Docker nie działa albo brak narzędzia z `requires` | `blocked` w preflighcie |
| Kolejka zajęta dłużej niż `waitMinutes` | `blocked: queue-timeout`; pozycja w kolejce wypisywana w trakcie czekania |
| Run trzymający blokadę jest porzucony | Najpierw `down` porzuconego runu z jego `state.json`, potem start. Działający stos innego runu nigdy nie jest sprzątany automatycznie |
| Bramka pamięci | `blocked: memory`, z liczbami: suma `memory` kontenerów vs `resources.docker.memory` vs `MemTotal` VM; osobno host. Pamięć cudzych kontenerów jest tylko ostrzeżeniem. `--ignore-memory-gate` przepuszcza run i zapisuje to |
| Supervisor padł | Run porzucony; następny `up` lub `down --stale` sprząta go po weryfikacji czasu startu procesów |
| Usługa hosta padła po starcie | Status `degraded` w `status`; blokada trzymana do `down` |
| Ścieżka lokalna bez `prepare: true` i bez zainstalowanych zależności | Usługa nie osiąga zdrowia; komunikat błędu podpowiada `prepare: true` |
| Wyścig o port | Do 3 prób z nowym portem, potem `error` |
| Usługa nie osiąga zdrowia w limicie czasu | `error`; ścieżka logu i ostatnie linie; sprzątnięcie (chyba że `--keep-on-failure`) |
| Hook (`prepare`, `migrate`, `seed`, `otp`, `session`, `rateLimitReset`, `dataAccess`) przekracza `timeout` albo kończy się błędem | Grupa procesów hooka zabita, `error` z nazwą hooka i ogonem wyjścia (zamaskowanym) |
| Wyjście seeda bez wymaganej persony lub zasobu | `error`, z nazwą braku |
| Hook sesji zwraca sesję bez wymaganych pól albo już wygasłą | `error` przy `up`; przy `e2e-qa session` jedna ponowna próba, potem błąd |
| `jsonFile` na repo ze ścieżki lokalnej | `blocked` w preflighcie |
| `files` na ścieżce lokalnej wskazuje plik śledzony albo nieignorowany przez git | `blocked` w preflighcie z nazwą pliku; podpowiedź: wzorzec w `.gitignore` konsumenta albo źródło `cache` |
| `files` na ścieżce lokalnej, plik ignorowany już istnieje | Kopia do `backup/`, render; `down` przywraca oryginał. Plik nieistniejący wcześniej `down` usuwa |
| Awaria przed przywróceniem kopii (crash, SIGKILL) | Przywraca je `down --stale` porzuconego runu (wołany ręcznie albo przez następny `up` przy sprzątaniu porzuconego runu), na podstawie jego własnego `state.json`; `status` pokazuje nieprzywrócone pliki |
| Operator zmienił nadpisany plik w trakcie runu | `down` nie nadpisuje jego zmian: gdy suma kontrolna różni się od wyrenderowanej, kopia zostaje w `secrets/backup/` (0600, poza retencją), a `down` wypisuje ostrzeżenie z obiema ścieżkami |
| Ścieżka w `files` wychodzi poza repo albo prowadzi przez symlink | Błąd walidacji albo `blocked` w preflighcie (`realpath`) |
| `down` po runie z `files` w worktree z cache | `git worktree remove --force` usuwa worktree z nieśledzonymi plikami; `down` zwraca 0 |
| Usługa odwołuje się do `${services.X.*}` usługi, która od niej zależy | Poprawne: porty są przydzielone przed startem. Cyklem jest tylko cykl w `dependsOn` |
| Przeglądarka otwiera `localhost`, a usługa akceptuje tylko origin `127.0.0.1` (lub odwrotnie) | Zapobiega temu `${services.X.origins}` z obiema formami; `env.json` publikuje URL-e w formie `127.0.0.1` |
| `e2e-qa otp --identity` z nieznaną tożsamością | Wynik hooka konsumenta (kod albo błąd); wartość przekazana jako jeden argument, bez interpretacji przez powłokę; wartość zaczynająca się od `-` odrzucona |
| Kontener zależy od usługi hosta (Linux, CI) | `extra_hosts: host-gateway` i `${self.bind}=0.0.0.0` dla tej usługi; ostrzeżenie o nasłuchu na wszystkich interfejsach |
| Więcej niż jeden wariant w runie | Błąd walidacji „nieobsługiwane w tej wersji” |
| Ctrl+C, SIGTERM w trakcie `up` | Sprzątnięcie w obsłudze sygnału; `state.json` pozwala dokończyć je przez `down --stale` |
| Wyciek sekretu do logów | Narzędzie maskuje znane mu wartości (hasła person, sesje, `${secret.*}`) we własnych logach i w logach usług hosta i kontenerów przed zapisem |

## 📝 Risks & Impact Review

- **Publiczne kontrakty, które trudno cofnąć:** `manifest.v1`, `run.v1`, słownik stanów v1 (znaczenie nazw standardowych), kontrakt polecenia seeda i hooka sesji, `env.v1`. Zmiany łamiące wymagają podbicia wersji schematu. Wydanie pakietu, które wprowadza schemat v2, czyta też v1 przez co najmniej jedno kolejne wydanie minor pakietu i wypisuje ostrzeżenie migracyjne.
- **Słownik zalecany zamiast zamkniętego.** Konsument nie jest blokowany, ale spec #2 (generator) może polegać tylko na stanach standardowych. Stany własne będą dla niego czarnymi skrzynkami z opisem. Pokrycie słownika raportowane przez `validate` pokazuje tę lukę.
- **Wartość słownika jest mniejsza, niż zakładano** (wniosek z próby na sucho, 2026-10-09). Realne scenariusze częściej wymagają stanów **własnych**, czyli złożeń kilku warunków, niż pojedynczych stanów standardowych. Stany własne są więc głównym mechanizmem, a słownik raczej wspólnym językiem i minimum dla generatora. Spec #2 powinien to uwzględnić, np. przez składanie stanów albo parametryzację.
- **Kilka dialektów URL-a tej samej usługi** (np. sterownik async i sync bazy). Rozwiązane nazwanymi `urls`, bo składanie z `host`/`port` omija maskowanie. Koszt: kolejne pole w publicznym schemacie.
- **`files` zapisuje do katalogów konsumenta.** W worktree z cache to bezpieczne. Na ścieżkach lokalnych ryzyko ogranicza zapis wyłącznie do plików ignorowanych przez git, kopia zapasowa i przywrócenie przy `down` (także po awarii). Kopie leżą w `secrets/backup/` (0600) i nie podlegają retencji, dopóki nie zostaną przywrócone. Pozostaje ryzyko, że proces operatora (np. jego własny dev-serwer) przeczyta wyrenderowany plik w trakcie runu. Wyrenderowane pliki mogą zawierać `${secret.*}`; leżą na dysku do `down`, tak jak `secrets/`.
- **Brak scalania w narzędziu.** Jakość wyniku dla paczki zależy od tego, czy gałąź integracyjna operatora odpowiada temu, co trafi na gałąź główną. `env.json` zapisuje SHA, żeby wynik dało się odtworzyć.
- **Ścieżki lokalne** dają szybkość (worktree orkiestratora), ale mogą być brudne i zmieniać się w trakcie runu. Mitygacja: SHA i `dirty` w opisie środowiska. Narzędzie nie zmienia tam plików śledzonych; zapisuje tylko pliki ignorowane z kopią zapasową. `prepare` uruchamia tylko na jawne żądanie.
- **Supervisor to nowy proces długożyjący.** Jego awaria oznacza porzucenie runu (usługi hosta giną razem z nim), co jest bezpieczniejsze niż sieroty bez właściciela logów.
- **Bezpieczeństwo:**
  - Hasła person, sesje i sekrety infrastruktury są jednorazowe, generowane per run i trzymane w `secrets/` (0600), usuwane przy `down`.
  - `env/<variant>.json` zawiera wyłącznie referencje i zamaskowane URL-e.
  - Kod OTP jest jednorazowy i dotyczy konta demo.
  - Hook sesji jest kodem konsumenta wykonywanym na hoście, z tym samym zaufaniem co seed.
  - Narzędzie nie przyjmuje sekretów z manifestu (manifest jest commitowany).
- **Twarda bramka pamięci opiera się na deklaracjach.** Zaniżone `memory` usługi hosta nie zostanie wykryte przed startem. Pomiar RSS po starcie daje tylko ostrzeżenie. Obejście bramki jest jawne i zapisane.
- **`${self.bind}=0.0.0.0`** wystawia usługę hosta na wszystkie interfejsy na czas runu. Ostrzeżenie w wyjściu; dotyczy tylko usług, od których zależy kontener.

## 📋 Poza zakresem

- Kontrakt scenariusza MD i styk z wykonawcą QA (kształt styku, Q4 otwarte ponownie): spec #1b.
- Kilka wariantów w jednym runie (kontrakt jest gotowy, implementacja w późniejszym speca lub fazie).
- Scalanie paczek PR-ów; API trackera.
- Generator scenariuszy, ocena dowodów, pętla naprawy, UI historii (specy #2–#4).
- Emulator i build natywny; serwer i PaaS.
- Cache przygotowania zależności między runami.

## 📋 Phasing

Każda faza zostawia działające, użyteczne narzędzie.

1. **Manifest i walidacja.** Konsument może napisać i zwalidować manifest.
2. **Kod repozytoriów.** `e2e-qa checkout` przygotowuje kod z refów, PR-ów albo ścieżek lokalnych.
3. **Start stosu.** `up`/`down`/`status` z supervisorem, kolejką, bramką pamięci i plikami konfiguracyjnymi usług oraz `env/<variant>.json` z usługami. Człowiek może testować ręcznie na kontach z danych deweloperskich konsumenta, jeśli migracje je zakładają.
4. **Stan danych i konta.** Seed, persony, przełączniki, sesje, `creds`/`otp`/`logs`/`data`, `env/<variant>.json` uzupełniony o persony i stan. Środowisko jest gotowe do testów ręcznych i do speca #1b.

## 📋 Implementation Plan

Testy jednostkowe: vitest. Testy integracyjne z Dockerem są oznaczone i pomijane, gdy Dockera nie ma. W CI (GitHub Actions, Linux) działają. Fikcyjny konsument testowy żyje w `examples/acme/`: maleńkie usługi HTTP w Node, postgres i redis, lokalne repozytoria bare tworzone w teście.

### Faza 1: manifest i walidacja

1. Szkielet pakietu: TypeScript, Node ≥20, bin `e2e-qa`, `--version`, lint i testy w CI. Test: `npx e2e-qa --version`.
2. JSON Schema `manifest.v1` i `run.v1` (z mapą `variants` i zarezerwowanym kluczem `qa`) oraz `e2e-qa validate` z czytelnymi błędami (ścieżka YAML i linia). Test: poprawny i niepoprawne manifesty `examples/acme`; manifest, w którym `api` odwołuje się do `${services.web.origins}`, a `web` zależy od `api`, przechodzi walidację.
3. Reguły semantyczne: cykle w `dependsOn` i `seed.after` (placeholdery nie tworzą krawędzi), nierozwiązywalne placeholdery, `files` (format zgodny z `content`, ścieżka wewnątrz repo), `surface` person i klucze `login.notes` wskazujące usługi z `target`, placeholdery wartości wejściowych wewnątrz cudzysłowów w poleceniach, słownik stanów (znaczenie nazw standardowych, `description`/`personas` stanów własnych, raport pokrycia), brak `memory`, nieznane przełączniki i stany w runie, `pr:` bez `prRef`, repo w `refs` i `paths` naraz, więcej niż jeden wariant. Test: tabela przypadków.

### Faza 2: kod repozytoriów

4. Cache lustrzany z blokadą per repo, fetch z limitem czasu i `GIT_TERMINAL_PROMPT=0`, worktree per (wariant, repo). Test: lokalne repozytoria bare i dwa równoległe fetche.
5. Nakładka `pr:<n>` przez `prRef`. Test: repo bare z refem `refs/pull/7/head`.
6. Ścieżki lokalne: weryfikacja, SHA, `dirty`, brak modyfikacji, `prepare` tylko przy `prepare: true`; polecenie `e2e-qa checkout`. Test: `git status` i lista plików ścieżki lokalnej identyczne przed i po.

### Faza 3: start stosu

7. Przydział wszystkich portów przed startem i rozwiązywanie placeholderów w widoku hosta i kontenera, wraz z `${secret.*}`, `${self.bind}`, `url`/`urls`, `origins` i cytowaniem wartości wejściowych w hookach. Test jednostkowy: odwołanie do usługi zależnej rozwiązuje się; `origins` zawiera obie formy w widoku przeglądarki także dla kontenera; usługa hosta, do której kontener sięga tylko przez placeholder, dostaje bind `0.0.0.0`; `${identity}` ze znakami powłoki nie wykonuje polecenia (`in: host` i `in: <usługa>`), a wartość z wiodącym `-` jest odrzucana.
8. Render `files` (`dotenv`, `json`, `text`) przed `prepare`/`start`. Kontrola `realpath` i symlinków. Na ścieżkach lokalnych `git check-ignore`, kopia do `secrets/backup/` (0600), wpis w `state.json` (bez treści), przywrócenie lub usunięcie przy `down` i `down --stale`, ochrona zmian operatora po sumie kontrolnej, wyłączenie z retencji. Test: w worktree z cache plik powstaje z rozwiązanym adresem, a `down` zwraca 0; symlink poza repo daje `blocked`; na ścieżce lokalnej plik śledzony daje `blocked`, a plik ignorowany jest po `down` identyczny bajt w bajt z oryginałem (także po zabiciu procesu i `down --stale`); plik nieistniejący wcześniej znika; `git status` ścieżki lokalnej jest identyczny przed i po.
9. Generowanie pliku compose (projekt `e2eqa-<runId>-<variant>`, `mem_limit`, healthcheck, porty, `extra_hosts: host-gateway`), `pull` z ponowieniami, `up`/`down`/`exec`. Test integracyjny na Linuksie w CI: postgres i redis startują i są zdrowe.
10. Supervisor runu: start odłączony, `supervisor.sock`, pompa logów (usługi hosta i `docker compose logs -f`) z maskowaniem i znacznikami czasu, PID z czasem startu w `state.json`. Test: znany sekret wypisany przez usługę i przez kontener nie trafia do `logs/`; supervisor przeżywa wyjście CLI.
11. Usługi hosta jako dzieci supervisora: `prepare` z limitem czasu, start we własnej grupie procesów, sondy zdrowia, `env/<variant>.json` z usługami (URL-e zamaskowane, `urlEnv`). Test: usługa HTTP z `examples/acme` żyje po wyjściu CLI i odpowiada kontenerowi przez `host.docker.internal`; `env/<variant>.json` nie zawiera wartości sekretu.
12. Graf startu z hookami `migrate` (`in: host | <usługa>`, `timeout`). Test: migracja zakładająca schemat przed zależną usługą oraz hook przekraczający limit.
13. Preflight (`docker`, `requires`) i twarda bramka pamięci (deklaracje vs budżet vs `MemTotal`) z `--ignore-memory-gate`, ostrzeżenia (cudze kontenery, RSS ponad deklarację), wstrzykiwane źródło pomiarów. Test jednostkowy bez Dockera: zaniżony budżet daje `blocked: memory` z liczbami; obejście jest zapisane w `state.json`; RSS ponad deklarację daje ostrzeżenie.
14. Kolejka FIFO z blokadą po `runId`, wykrywanie porzuconego runu (supervisor: PID + czas startu), status `degraded`, `up`/`down`/`status`/`down --stale`, sprzątanie worktree i `secrets/`, retencja `runs/`, obsługa SIGINT/SIGTERM, kody wyjścia. Test: drugi `up` czeka na działający stos i kończy się komunikatem z `runId` po `waitMinutes`; run z zabitym supervisorem zostaje sprzątnięty; PID z innym czasem startu nie jest zabijany; SIGINT w trakcie `up` zostawia czysty stan; `down` z pozostałością zwraca 4.

### Faza 4: stan danych i konta

15. Słownik stanów w kodzie, wywołanie seeda z kontraktem `E2E_QA_*` (reguła nazw zmiennych), walidacja wyjścia. Test: seed `examples/acme` dla stanów standardowych i własnego, podwójny seed (idempotencja), seed z niepełnym wyjściem.
16. Generowanie haseł person, `secrets/credentials.env` (0600, `E2E_QA_PERSONA_*`, `E2E_QA_SERVICE_*_URL` i `E2E_QA_SERVICE_*_URL_<NAZWA>`), `e2e-qa creds`. Test: żadne hasło ani sekret nie pojawia się w logach narzędzia, `state.json` ani `env/<variant>.json`.
17. Przełączniki `env` i `jsonFile` (wskaźnik JSON, tylko worktree z cache) przed startem usługi, `command` po seedzie. Test: wartość przełącznika widoczna w odpowiedzi usługi; `jsonFile` na ścieżce lokalnej daje `blocked`.
18. Tryby logowania: `surface` per persona i `login.notes` per powierzchnia, `login.otp` z `${identity}`, `login.rateLimitReset`, `login.session` z buforem i `ttl`, polecenia `otp <persona>`, `otp --identity` i `session`. Test: atrapa hooka sesji wywołana raz w obrębie `ttl`, ponownie po wygaśnięciu; `otp --identity` przekazuje tożsamość spoza person; persona z `surface: mobile-web` dostaje w `env.json` notatki tej powierzchni.
19. `e2e-qa seed <stan>` na działającym stosie: hasła tylko dla nowych person, unieważnienie sesji, ponowne przełączniki `command`. Test: przejście `baseline` → `scoped-access` daje nową personę z hasłem, stare hasła bez zmian, bufor sesji pusty.
20. `logs --since` (znaczniki czasu supervisora) i `data` (przepis `dataAccess`, zapytanie na stdin). Test: `data` na `examples/acme` zwraca wynik; przepis z rolą tylko do odczytu odrzuca zapis.
21. `env.v1` pełny (persony z powierzchnią i `identifierKind`, stan, targety z `browser`, zastrzeżenie `mobile-web`, `urls` z referencjami, SHA, stan bramki) i `status --json`. Test kontraktowy: plik przechodzi `schema/env.v1.json`, a snapshot dla `examples/acme` jest stabilny. Dokumentacja dla konsumenta: jak napisać manifest, seed i hook sesji.
