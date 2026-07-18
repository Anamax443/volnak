# HANDOFF — deník stavu: Volnák

Append-only. Nejnovější záznam nahoru. Slouží k pokračování z jiného počítače / po pauze.

## 2026-07-18 — STATUS.html + dokumentace + repo public
- Přidán **STATUS.html** (vizuální přehled stavu ve stylu faxx-dox: brána, fáze F0→F3, rozhodovací log, otevřené otázky).
- Srovnána dokumentace: README odkazuje na STATUS, docs/BUILD.md označen jako design-fáze.
- Repo přepnuto na **public** (`Anamax443/volnak`).
- **GitHub Pages** zapnuté (main/root) → STATUS živě na <https://anamax443.github.io/volnak/> (index.html redirect → STATUS.html).
- Vše pushnuto na `main`.

## 2026-07-18 — F0 návrh: repo + architektura
- Projekt založen podle standardu (scaffold z `project-standard`).
- Jméno **Volnák** (z „volno / kdo má volno") — repo `Anamax443/volnak`, private.
- **Hotové:** kompletní architektonický dokument v [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — tenancy nad saas-foundation, dvourychlostní identita hráče, model viditelnosti (busy-only + visibility_grants), konektorová vrstva, sekce GDPR + ToS zdrojů, fázování F0→F3.
- **Klíčová rozhodnutí:** žádný FB/IG scraping (jen legální cesty); MVP jede na volných jménech; GDPR first-class od momentu, kdy je to platforma pro cizí.
- **Rozpracované:** —
- **Zbývá (před F1):** rozhodnout bázi (saas-foundation vs rezervace-app), vybrat konektory pro MVP, doplnit docs/BUILD.md, aplikovat „Klasika" showcase kit až se začne stavět.
- Kontext: vzniklo z osobní potřeby hlídat kolize koncertů spoluhráčů; návrh prošel dvěma chaty + revizí.
