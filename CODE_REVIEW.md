# Zasady code review

Reguły review tego repozytorium. `om-code-review` (a przez niego `om-auto-review-pr`) stosuje ten plik automatycznie, obok wbudowanej checklisty. Repo jest na etapie dokumentów projektowych; sekcje dotyczące kodu doprecyzuj, gdy pojawi się stos technologiczny.

## Priorytety

Kolejność ma znaczenie: uwaga z wyższej pozycji zawsze wyprzedza uwagę z niższej.

1. **Publiczność repo.** Każdy diff, opis PR-a, commit i komentarz sprawdzamy pod kątem danych projektów-konsumentów: nazw, adresów, kont, haseł, SHA, tras API, opisów defektów i podatności (także naprawionych). Znalezisko to zawsze **blocker**, bez wyjątków. Dozwolone są wyłącznie uogólnione wnioski i fikcyjne przykłady (np. `example.test`, `qa-user-a`).
2. **Bezpieczeństwo.** Sekrety, tokeny i treść `.env` nie mogą trafić do repo, logów, raportów ani zrzutów ekranu. Poświadczenia kont QA pochodzą z manifestu konsumenta lub zmiennych środowiska, nigdy z kodu narzędzia.
3. **Kontrakty.** Zmiana powierzchni opisanej w `BACKWARD_COMPATIBILITY.md` bez ścieżki migracji to blocker.
4. **Poprawność.** Logika, przypadki brzegowe, obsługa błędów.
5. **Jakość.** Czytelność, spójność z otoczeniem, brak abstrakcji na zapas.

## Kontrole specyficzne dla repo

### Agnostyczność rdzenia

- Rdzeń nie zna żadnego projektu. Porty, seed, konta, sposób logowania, nazwy usług i repozytoriów przychodzą z manifestu konsumenta. Zaszyta w rdzeniu wartość specyficzna dla projektu to **major**.
- Nowa abstrakcja musi odpowiadać na realną potrzebę pierwszego konsumenta. Kontrakt nie może się zamykać na inne projekty, ale nie budujemy rozszerzeń „na wszelki wypadek”.

### Kontrakt dowodu

- PASS wymaga sprawdzenia efektu: DOM, ruch sieciowy albo stan bazy. Wynik polecenia narzędzia (np. „kliknięto”) nie jest dowodem.
- Krok, który przechodzi dopiero drogą okrężną, ma dać FAIL. Kod, który w takiej sytuacji zwraca PASS albo „PASS z uwagą”, to **blocker**.
- Raport rozróżnia wprost target web od natywnego (web ≠ native).

### Ślepota generatora scenariuszy

- Generator scenariuszy jest izolowany technicznie: nie widzi pamięci agenta, trackera ani zgłoszeń defektów. Zmiana, która poszerza jego dostęp do kontekstu (nowe źródło danych, wspólny katalog roboczy, przekazanie historii), to **blocker**, dopóki spec wprost tego nie dopuszcza.
- Instrukcja w prompcie („nie patrz na…”) nie zastępuje izolacji technicznej.

### Specyfikacje i dokumentacja

- Jedna funkcja = jeden spec w `docs/specs/`. PR implementujący odwołuje się do swojego speca; PR łączący kilka funkcji powinien zostać podzielony.
- Pliki w kebab-case; specy i briefy z prefiksem daty `YYYY-MM-DD-`.
- Dokumentacja po polsku, z pełnymi znakami diakrytycznymi; identyfikatory i kod po angielsku.
- Commity: jednolinijkowy subject bez cudzysłowów.

### Środowisko i wykonanie

- Wszystko działa w katalogu roboczym; narzędzia instalowane lokalnie, bez wymogu uprawnień administratora.
- Zasoby Dockera (projekty compose, porty, wolumeny) są izolowane per run i sprzątane po nim. Brak sprzątania albo stały port to **major**.

## Bramka walidacji

Lista `validation.commands` w `.ai/agentic.config.json` jest na razie pusta (brak kodu i toolchainu). Gdy pojawi się toolchain, PR musi przechodzić pełną bramkę przed zatwierdzeniem.

## Waga uwag

| Waga | Znaczenie | Wpływ na werdykt |
|---|---|---|
| **blocker** | Wyciek danych konsumenta lub sekretu, złamany kontrakt dowodu lub izolacji generatora, zmiana kontraktu bez migracji, błąd poprawności | `changes-requested` |
| **major** | Wiedza projektowa w rdzeniu, brak sprzątania środowiska, brak testu dla nowej logiki, niezgodność ze specem | `changes-requested` |
| **minor** | Czytelność, nazewnictwo, drobne niespójności | Nie blokuje; zgłoszenie uzupełniające przy większej liczbie |
| **nit** | Preferencje stylistyczne | Nie blokuje |

Uwaga podaje plik i linię, opisuje skutek (co i dla kogo się psuje) i proponuje kierunek poprawki.
