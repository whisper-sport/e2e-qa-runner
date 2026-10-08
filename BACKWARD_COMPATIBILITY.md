# Zgodność wsteczna

Ten dokument wymienia **chronione powierzchnie kontraktowe** projektu i sposób zmiany każdej z nich. Skille review sprawdzają zmiany względem tego pliku; skille implementujące ostrzegają, gdy zmiana go narusza.

## Stan obecny

Repo jest na etapie dokumentów projektowych: nie ma jeszcze kodu, wydanych wersji ani konsumentów korzystających z narzędzia. Dziś żadna powierzchnia nie jest wydana, więc żadna zmiana nie łamie zgodności. Poniższe powierzchnie są **planowane** na podstawie README i briefów; status „chroniona” uzyskują w chwili pierwszego wydania, które je zawiera. Uaktualnij ten plik razem z PR-em, który wprowadza daną powierzchnię.

## Powierzchnie

| Powierzchnia | Źródło | Status |
|---|---|---|
| Format manifestu środowiska konsumenta | Spec #1 | planowana |
| Kontrakt seeda (nazwane stany danych, role) | Spec #1 | planowana |
| Kontrakt kont QA i logowania | Spec #1 | planowana |
| Interfejs uruchamiania (CLI, komendy, flagi, kody wyjścia) | Spec #1 | planowana |
| Deskryptor środowiska dla wykonawcy QA | Spec #1 | planowana |
| Format scenariuszy MD | Spec #2 | planowana |
| Format raportu i werdyktu (kontrakt dowodu) | Spec #3 | planowana |

### Manifest środowiska

Manifest żyje w repozytorium konsumenta, poza naszą kontrolą, dlatego to najważniejsza powierzchnia.

- **Zmiana łamiąca:** usunięcie lub zmiana nazwy pola, zmiana znaczenia albo typu pola, nowe pole wymagane bez wartości domyślnej, zmiana domyślnego zachowania.
- **Wymagana ścieżka:** wersjonowanie manifestu (pole wersji schematu); stara wersja jest czytana co najmniej przez jedno wydanie z ostrzeżeniem o przestarzałości; nota migracyjna w changelogu z przykładem przed/po; podbicie wersji głównej narzędzia.

### Kontrakt seeda i kont QA

- **Zmiana łamiąca:** zmiana nazwy lub semantyki nazwanego stanu danych albo roli, zmiana sposobu wywołania seeda lub logowania wymagająca zmian po stronie konsumenta.
- **Wymagana ścieżka:** jak dla manifestu. Zmiana nie może wymagać od konsumenta umieszczenia poświadczeń w repo.

### Interfejs uruchamiania

- **Zmiana łamiąca:** usunięcie lub zmiana nazwy komendy albo flagi, zmiana znaczenia kodu wyjścia, zmiana formatu wyjścia maszynowego.
- **Wymagana ścieżka:** stara forma działa przez co najmniej jedno wydanie z ostrzeżeniem; nota w changelogu; podbicie wersji głównej.

### Format scenariuszy MD

Scenariusze regresji żyją w katalogach konsumentów.

- **Zmiana łamiąca:** zmiana składni kroku, asercji albo metadanych, przez którą istniejący scenariusz przestaje się wykonywać lub zmienia wynik.
- **Wymagana ścieżka:** parser akceptuje starą składnię przez co najmniej jedno wydanie; narzędzie migracji albo instrukcja przepisania; podbicie wersji głównej.

### Raport i werdykt

Raport czytają ludzie i automaty (orkiestrator, przyszłe UI historii runów).

- **Zmiana łamiąca:** zmiana pól maszynowych raportu, zbioru wartości werdyktu albo reguły, kiedy krok jest PASS. Złagodzenie kontraktu dowodu (PASS bez sprawdzonego efektu) jest niedopuszczalne niezależnie od ścieżki migracji.
- **Wymagana ścieżka:** wersjonowany format raportu; nowe pola tylko addytywnie w ramach wersji; podbicie wersji głównej przy zmianie istniejących pól.

## Zasady ogólne

- Przed pierwszym wydaniem (0.x) zmiany łamiące są dopuszczalne, ale wymagają noty w changelogu.
- Od wersji 1.0 obowiązuje SemVer: zmiana łamiąca = podbicie wersji głównej.
- Przykłady w notach migracyjnych są fikcyjne, zgodnie z zasadą publicznego repo z `CLAUDE.md`.
