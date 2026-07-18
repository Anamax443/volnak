# Volnák

> Multi-tenant platforma pro booking manažery kapel: hlídá **dostupnost sdílených muzikantů** a **kolize koncertních termínů** z veřejných zdrojů.

## Co to dělá

Každý manažer/agent má izolovaný workspace, zadá si kapely a jejich sestavy (hráče), a Volnák mu z **veřejných termínů koncertů** počítá, kdy je který hráč obsazený — aby nedošlo k dvojímu zabookování téhož muzikanta ve dvou kapelách na stejný den.

Termíny se berou ze zdrojů, které *chtějí* být čtené (kapelní weby, Bandzone, GoOut, oficiální API Bandsintown/Songkick/Ticketmaster, ICS). **Facebook/Instagram se nescrapují** — jen legální cesty (notifikace → e-mail, IG `business_discovery`). Proč, viz [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Stav

**F0 — návrh (design fáze).** Zatím žádný kód, jen architektonický dokument. Stavět se bude nad `saas-foundation`.

📍 **Živý přehled stavu:** <https://anamax443.github.io/volnak/> (GitHub Pages)

## Stack (plánováno)

- Cloudflare Workers + D1 (nad `Anamax443/saas-foundation`)
- Auth: magic-link / OTP ze saas-foundation
- Konektorová vrstva (adaptéry na zdroje termínů) + AI-extrakce (Claude, sdíleno s `faxx-dox`)

## Požadavky

- Node + wrangler (CF), účet Cloudflare
- (F2) přístupy k API zdrojům: Bandsintown, Songkick, Ticketmaster, IG Graph

## Spuštění / build

Zatím není co spustit — projekt je v návrhové fázi. Postup vzniká v [docs/BUILD.md](docs/BUILD.md).

## Konfigurace

Tajemství nikdy do gitu — zkopíruj `*.example` na reálný soubor a vyplň lokálně.

## Dokumentace

- [STATUS.html](STATUS.html) — **vizuální přehled stavu** (fáze, rozhodovací log, otevřené otázky) — otevři v prohlížeči
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — **kompletní návrh** (tenancy, identita, konektory, GDPR/ToS, fázování)
- [docs/BUILD.md](docs/BUILD.md) — jak postavit od nuly (výrobní)
- [HANDOFF.md](HANDOFF.md) — deník stavu
