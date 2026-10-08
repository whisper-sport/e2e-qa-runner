# CLAUDE.md

Instrukcje dla agentów pracujących w tym repozytorium.

## Czym jest to repo

Publiczne (Apache-2.0), projekt-agnostyczne narzędzie do E2E QA zmian wytworzonych przez agentów. Kolejność dostaw: `README.md`. Brief bieżącego etapu: `docs/specs/briefs/`. Eksperyment, który ukształtował źródło scenariuszy: `docs/research/2026-10-08-scenariusze-z-ac-probe.md`.

## Twarde zasady

- **Repo jest publiczne od pierwszego dnia.** Nie commituj nazw, adresów, kont, haseł, SHA, tras API ani opisów defektów żadnego projektu-konsumenta. Dotyczy to zwłaszcza podatności: nawet naprawionych, a tym bardziej otwartych. Wiedza projektowa żyje w repozytoriach konsumentów. Tutaj trafiają wyłącznie uogólnione wnioski i fikcyjne przykłady.
- **Agnostyczność.** Rdzeń nie zna żadnego projektu. Wszystko, co specyficzne (porty, seed, konta, sposób logowania), opisuje manifest konsumenta.
- **Nie zgadujemy abstrakcji na zapas.** Kontrakty projektujemy tak, żeby się nie zamykały na inne projekty, ale budujemy pod realne potrzeby pierwszego konsumenta.
- **Jedna funkcja = jeden spec.** Etapy z README to osobne specyfikacje.
- **Ślepota generatora to izolacja techniczna, nie instrukcja.** Generator scenariuszy nie może widzieć pamięci agenta, trackera ani zgłoszeń defektów (przepis w `docs/research/`).
- **Kontrakt dowodu.** PASS wymaga sprawdzenia efektu (DOM, sieć, baza). Krok, który przechodzi dopiero drogą okrężną, to FAIL.

## Konwencje

- Dokumentacja po polsku, identyfikatory i kod po angielsku.
- Nazwy plików: kebab-case.
- Commity: jednolinijkowy subject bez cudzysłowów.
