# Czy scenariusze z kryteriów akceptacji łapią realne defekty? (eksperyment, 2026-10-08)

Eksperyment przeprowadzono na wewnętrznym, wielorepozytoryjnym systemie pierwszego konsumenta (API, panel web, aplikacja mobilna testowana na targecie web, serwis ML). Szczegóły projektowe (repozytoria, wersje, trasy, kroki reprodukcji) pozostają w repozytoriach konsumenta. Tutaj są tylko wnioski uogólnione.

## Hipoteza

Agent, który **nie zna defektów**, generuje scenariusze MD z kryteriów akceptacji funkcji (spec, issue, opis PR-a, który ją wprowadził). Wykonawca przechodzi je w przeglądarce na kodzie z defektem. Pytanie: ile znanych defektów zostanie złapanych?

Reguła decyzji ustalona przed pomiarem: poniżej 50% oznacza rewizję źródła scenariuszy.

## Próbka

Trzy realne defekty wykryte wcześniej przez ludzi, każdy w innej klasie:

| Klasa | Opis uogólniony |
|---|---|
| A. Stan pierwszoplanowy | Ekran działa dla „pełnych” danych, ale w stanie, w którym znajduje się każdy nowy klient (puste dane, brak konfiguracji), główna akcja ekranu jest niewykonalna |
| B. Rodzeństwo tras | Reguła dostępu z kryteriów akceptacji jest wdrożona na nowych trasach, ale starsza trasa oddająca ten sam zasób (agregat) jej nie stosuje |
| C. Ścieżka „bez zmian” | Zmiana deklaruje, że pewna ścieżka pozostaje bez zmian. Dokończona do końca ta ścieżka działa jednak źle i dotyka cudzych danych |

## Wynik: 1/3

| Klasa | Werdykt | Dlaczego |
|---|---|---|
| A | złapany **przypadkiem** | Generator nazwał stan pierwszoplanowy, ale sprawdził w nim tylko render. Defekt ujawnił się w innym scenariuszu, który przypadkiem zaczynał od tej samej interakcji. Zgłoszenie brzmiałoby jak drobna niedogodność, a nie jak bloker |
| B | niezłapany | Regułę dostępu sprawdzono na pięciu trasach z poprawną bramką. Trasę-rodzeństwo wywołano tylko kontem uprawnionym. Rozjazd kształtu odpowiedzi był widoczny w logu, ale żadna asercja go nie obejmowała |
| C | niezłapany | Scenariusz sam wytworzył stan wejściowy defektu i zatrzymał się na stanie pośrednim, nie dokończył ścieżki |

Wzór chybień jest wspólny: **kryteria akceptacji zawierały właściwą tezę, ale generator zastosował ją tylko do tego, co zmiana opisywała jako nowe.** Scenariusze z AC testują zmianę i jej deklarowane granice, a nie tezy produktu.

Porównanie (analiza statyczna): katalog regresji napisany po wykryciu defektów złapałby klasę C. Hybryda dałaby 2/3. Klasa B nie miała pokrycia w żadnym źródle sprzed jej wykrycia.

## Decyzja: hybryda

1. **Katalog regresji per projekt**, uruchamiany obok scenariuszy z AC.
2. **Generator w trybie kontrprzykładów**, z heurystykami, z których każda łapie jedną z chybionych klas:
   - **stan pierwszoplanowy × główna akcja:** w każdym stanie, który materiały nazywają typowym na start (pusty, cold-start, bez konfiguracji), wykonaj główne zadanie ekranu, a nie tylko render;
   - **reguła dostępu × wszystkie wejścia do zasobu:** wylicz wszystkie trasy oddające ten sam zasób (także starsze i agregaty) i odpytaj każdą rolą bez uprawnienia, porównując kształt odpowiedzi;
   - **„bez zmian” = przejdź do końca:** każdą ścieżkę deklarowaną jako niezmieniona dokończ i sprawdź skutki uboczne na innych kontach.
3. **Kontrakt dowodu.** Krok, który przechodzi dopiero drogą okrężną, to FAIL. Anomalie widoczne w logach, ale nieobjęte asercją, wykonawca zgłasza osobno.

## Izolacja generatora: wymóg techniczny

Pierwszy przebieg generacji był **skażony i został unieważniony**. Podagent uruchomiony z sesji agenta dziedziczył auto-pamięć projektu, w której były streszczenia badanych defektów. Instrukcja „nie patrz na defekty” nie wystarcza. Działający przepis dla Claude Code:

```sh
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 claude -p --tools "" --settings '{"autoMemoryEnabled": false}' < prompt.md
```

Przed właściwą generacją warto zadać zapytanie kontrolne: czy model zna frazy charakterystyczne dla defektów. Tryb `claude --bare` odcina pamięć, ale w testowanej konfiguracji nie miał dostępu do logowania.

## Problemy środowiska (wejście do speca #1)

Uogólnione z przebiegu eksperymentu:

1. Istniejący skrypt środowiska był zaszyty pod jedną funkcję (bramki gałęzi, sztywne porty, jeden projekt compose), więc nie nadawał się do pomiaru na dowolnych wersjach. Potrzebna jest mapa repo → wersja → port.
2. W jednym runie potrzebne były dwa warianty stosu (dwie wersje API, dwie bazy).
3. VM Dockera (ok. 8 GB) była zajęta innymi stosami. Usługi aplikacyjne trzeba było uruchomić na hoście, a w kontenerach tylko infrastrukturę.
4. Zależności startu (migracje jednej usługi zakładają schemat, którego wymaga seed).
5. Skrypty seeda wołały `docker compose exec` na sztywno.
6. Przełączniki stanu (feature flagi) wymagały ręcznej edycji zagnieżdżonego JSON-a.
7. Fikstury trafiały w ograniczenia schematu. Potrzebne są gotowe stany danych i role RBAC dla typowych scenariuszy.
8. Logowanie w aplikacji web miało pułapki UI (domyślny kod kraju, nakładające się arkusze, bramka zgód po pierwszym logowaniu).
9. OTP nie było wysyłane w dev, więc kod trzeba było czytać z bazy.
10. Limit prób logowania wymuszał reużycie tokenów.
11. Narzędzie przeglądarki zgłaszało sukces kliknięcia bez efektu, a drzewo dostępności dziedziczyło `disabled` z przodka.
12. Strażnik ścieżek agenta blokował dostęp poza katalog roboczy, a dokumenty wzorcowe nie były wersjonowane.
13. Drobne problemy powłoki i logów (podział słów w zsh, rozdzielczość `docker logs --since`).
14. Proxy frontendu było zaszyte na sztywny adres API.
