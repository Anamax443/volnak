# Architektura — Volnák

> **Stav:** F0 — návrh (design fáze). Tento dokument je zdroj pravdy pro koncept, ze kterého se pak staví.
> **Poslední aktualizace:** 2026-07-18

## 1. Přehled / TL;DR

**Volnák** je multi-tenant platforma pro **booking manažery a agenty kapel**. Každý manažer má izolovaný workspace, zadá si své kapely a jejich sestavy (hráče), a systém mu z **veřejných termínů koncertů** počítá **dostupnost hráčů** a hlásí **kolize termínů** — aby nedošlo k dvojímu obsazení téhož muzikanta.

Vzniklo z osobní potřeby (hlídat, kdy hrají spoluhráči v jiných kapelách, ať se termíny nekříží) → přerostlo v platformu pro víc manažerů.

**Klíčový princip sběru dat:** bereme termíny ze zdrojů, které *chtějí* být čtené (veřejná API, kapelní weby, ICS). **Žádný scraping Facebooku/Instagramu** (viz §7 a §8).

## 2. Uživatelé a role

- **super_admin** (provozovatel platformy — Milan) — spravuje tenanty, dohled, moderace, ventil pro sdílení a sjednocení dat. Neprovisionuje ručně uživatele.
- **tenant** = workspace jednoho manažera/agenta. Data se mezi tenanty nemíchají.
- **owner** — manažer uvnitř svého tenantu (později případně další členové).
- **hráč (musician)** — zpočátku jen datový záznam (jméno) uvnitř tenantu; teprve po vlastní registraci + napojení kalendáře se z něj stává ověřená platformní identita (viz §4).

Registrace je **self-service**: kdokoli se přihlásí e-mailem (magic-link / OTP), při prvním loginu se mu automaticky založí workspace, kde je ownerem. `tenant.status` (`active` / `pending`) nechává prostor pro pozdější schvalování/pozvánkový režim proti botům.

## 3. Tenancy model

- **Jedna D1 databáze**, **row-level tenancy**: `tenant_id` na každé doménové tabulce.
- Ne databáze-per-tenant — levné, jednoduché na provoz, super admin snadno udělá pohled napříč.
- **Disciplína:** žádný dotaz nesmí `tenant_id` vynechat. Nikdy ne ručně psané SQL v handlerech — vždy přes **tenký data layer / repository**, který tenant kontext injektuje automaticky. Prosáknutí dat mezi konkurenčními manažery je hlavní riziko tohoto modelu.

### Staví nad `saas-foundation`
Tenancy, auth, role a session **nepíšeme od nuly** — přebíráme z `Anamax443/saas-foundation` (stejná báze jako Faxxar). Před stavbou ověřit, co přesně foundation už řeší:
- row-level tenancy + data layer s injektovaným `tenant_id`,
- self-service registrace + magic-link/OTP + session,
- role a `tenant.status`.

Co foundation pokrývá, tady neřešíme — Volnák dodává jen **doménu** (musicians, rosters, bands, gigs, connectors, visibility). Alternativa ke zvážení: `Anamax443/rezervace-app` už je „multi-tenant rezervační systém" — zkontrolovat, jestli tam není hotovější tenant vzor k recyklaci.

> **OTEVŘENÁ OTÁZKA:** postavit nad `saas-foundation`, nebo vyjít z `rezervace-app`? Rozhodnout před F1.

## 4. Identita hráče — dvourychlostní model

Tentýž muzikant může hrát pod víc manažery, takže hráč **není** lokální záznam jednoho workspace. Ale plná globální identita je zbytečně drahá pro MVP. Řešení = dva stavy téhož, které schéma umí obojí **i přechod**:

1. **Lokální jmenný záznam (default, MVP).** Manažer si do sestavy napíše jména volným textem: `Zadáci → Eva Trnková, Kudla, Gregory`. Žádný souhlas, kontakt ani kalendář hráče není potřeba. `"Kudla"` u manažera A a `"Kudla"` u B jsou zatím jen dva shodné stringy — matching se **neřeší**.
2. **Ověřená globální identita (nástavba, F2).** Teprve když se hráč sám přihlásí e-mailem a napojí kalendář, vznikne skutečná platformní osoba (`musicians`), ke které se volné jmenné záznamy **volitelně** přilinkují (`roster_entries.musician_id`). Kanonický klíč globální identity = **e-mail hráče**. Teprve pak se rozsvítí sdílené kalendáře a sdílená busy okna.

MVP tedy „žije na stringech"; identita se „dopeče" později **bez přepisu modelu**.

## 5. Datový model (návrh)

Z `saas-foundation`: `tenants`, `users`, `sessions` (auth). Doménové tabulky Volnáku:

- **musicians** — globální identita hráče. `id`, `email` (unikátní klíč), `display_name`, později napojení kalendáře. Vzniká až při self-registraci (F2).
- **roster_entries** — „manažer pracuje s tímto hráčem". `tenant_id`, `display_name` (co manažer napsal), `musician_id` (nullable — prázdné do F2), tenant-specifické: role/nástroj, poznámka, sazba.
- **bands** — kapela. `tenant_id` (lokální k manažerovi, viz §6), název, zdroje termínů (viz §7).
- **band_members** — **kurátorská** vazba hráč↔kapela. Zadává **člověk** (manažer, fallback super admin), **ne** robot ze scrapu. Default lokální k manažerovi.
- **gigs** — termín koncertu kapely: `band_id`, datum, místo, zdroj, hash pro dedupe (datum+místo).
- **connectors** — konfigurace zdroje termínů pro kapelu (typ + parametry). Viz §7.
- **visibility_grants** — `from_tenant`, `to_tenant`, `musician_id`, `granted_by`, `scope`. Řídí, kdo vidí detail cizí obsazenosti. Viz §6.

## 6. Výpočet dostupnosti a model viditelnosti

### Kdy je hráč „busy"
Hráč je busy pro daného manažera, pokud **kterákoli kapela z rosteru tohoto manažera**, kde hráč hraje (`band_members`), má ten den gig.

> **POCTIVÁ HRANICE MVP:** detekce kolizí je úplná jen tak, jak dobře má manažer zmapovaný **vlastní** roster. Když Kudla hraje večer s kapelou, kterou manažer B nemá u sebe zadanou, B to neuvidí a může ho přebookovat. Cizí busy okna se napříč tenanty propíšou až po globalizaci identity (§4, F2). Toto je vědomá cena „MVP na jménech", ne chyba — musí být přiznaná, ne schovaná.

### Viditelnost detailu
- **Default = jen `busy`.** Manažer B u cizí obsazenosti vidí jen „ten den nemůže", **bez detailu** (kde, jaká kapela). Chrání to *pohodlí* (platforma nepředává konkurenční přehled zadarmo), ne *tajemství* — zdrojové gigy jsou stejně veřejné.
- **Grant zpřístupní detail.** Dotaz na obsazenost vrací `busy_dates`; detail se přibalí jen tam, kde existuje `visibility_grant` pro dvojici (žádající tenant → hráč).
- **Kdo grant vystaví:** pro MVP super admin jako ventil (per manažer + per hráč). Model je ale obecný, aby se autorita dala později posunout na hráče / bookujícího manažera (jejichž dat se to fakticky týká).

### Vazba kapela↔hráč: lokální vs globální
Default **lokální** (každý manažer si drží svou sestavu, nikdo mu do ní nemluví). Super admin může spornou sestavu povýšit/sjednotit globálně. Sedí to na izolaci tenantů + super admin jako ventil.

## 7. Konektorová vrstva (sběr termínů)

Appka **nemá napevno „scraper na weby"**. Má **sadu adaptérů**, každý pro jeden typ zdroje. Manažer u kapely vybere, odkud brát; systém použije nejlepší dostupný zdroj. Strategie: **sleduj víc zdrojů na kapelu a slučuj** (dedupe podle datum+místo).

| Zdroj | Typ přístupu | Poznámka |
|---|---|---|
| **Bandzone.cz** | scrape (strukturované) | česká scéna par excellence, veřejné, stabilní (Zadáci tam jsou) |
| **GoOut.net** | scrape / API-like | velký CZ/SK agregátor akcí a vstupenek |
| **Bandsintown API** | oficiální API | čistá data, ale menší CZ kapely tam nemusí být |
| **Songkick API** | oficiální API | dtto |
| **Ticketmaster Discovery API** | oficiální API | kde se prodávají lístky, jsou termíny strukturované |
| **Vlastní web kapely** | scrape + AI-extrakce | nejuniverzálnější fallback |
| **Klubové/regionální weby** | scrape | někdy snazší sledovat klub než kapelu |
| **ICS feed** | přímý import | když kapela/klub publikuje kalendář |

**Priorita:** kde je API (Bandsintown, Songkick, Ticketmaster), bereme čistá data; kde není, spadneme na scrape webu s AI-extrakcí.

### Vyloučené zdroje: Facebook a Instagram
**Zásadní rozhodnutí — NEscrapovat.** Důvody:
- ToS Meta výslovně zakazuje automatizovaný sběr (i veřejného obsahu), aktivně vymáhá. Riziko **banu osobního účtu**.
- Anti-bot detekce → obcházení = křehké a stálá údržba.
- FB Events pro cizí stránky přes Graph API stejně zamčené (od 2018).

**Legální náhrady:**
- **FB:** notifikace sledovaných stránek → **e-mail** → čtení přes `gmail-mcp` (Anamax443/gmail-mcp). Info přijde kanálem, který vlastníme.
- **Instagram:** buď crossposting IG→FB (svezeme přes e-mail route), nebo oficiální **Graph API `business_discovery`** pro veřejné Business/Creator účty.

## 8. Právo a compliance

### GDPR (first-class concern)
Jakmile je to platforma pro cizí manažery, **zpracováváme osobní údaje třetích osob** (muzikanti — jméno, dostupnost, později e-mail a kalendář). Pozor na skok „je to veřejné → smím cokoli" — **veřejnost ≠ volné použití**; přeskládat veřejné gigy pojmenované osoby do komerční databáze dostupnosti je *nový účel zpracování* a potřebuje právní základ (nejspíš oprávněný zájem + balanční test) a zásady zpracování.

Dvourychlostní identita (§4) to **odstiňuje**: těžké zpracování (e-mail hráče, kalendář, sdílení „kde hraje") začíná až v F2 — a to je přesně místo pro souhlas + privacy policy. MVP na volných jménech z veřejných zdrojů je nízkoriziko.

### ToS zdrojů
API zdrojů (Bandsintown, Songkick, Ticketmaster) mají vlastní podmínky. Klauzule o **redistribuci / komerčním užití / třetích stranách** jsou pro platformu jiná liga než pro osobní použití (např. Bandsintown API je stavěné na zobrazení termínů *jedné* kapely jejím fanouškům, ne na budování konkurenční databáze dostupnosti). Před platformním nasazením každý zdroj proklepnout.

## 9. Auth flow

Z `saas-foundation`: magic-link / e-mail OTP → session v D1. Žádné heslo. Registrace = první login. Ověření e-mailu je vestavěné (magic-link ho dělá sám).

## 10. Fázování (roadmap)

- **F0 — návrh (teď):** tento dokument, jméno, repo.
- **F1 — MVP na jménech:** tenancy nad foundation, self-service login, kapely + sestavy volným textem, 1–2 konektory (Bandzone + web/AI-extrakce), výpočet busy z vlastního rosteru, kolizní přehled. Bez sdílení mezi tenanty.
- **F2 — identita + kalendáře:** self-registrace hráče, `musicians`, linkování jmenných záznamů, sdílená busy okna, `visibility_grants`, privacy policy + souhlasy.
- **F3+:** další konektory (Bandsintown/Songkick/Ticketmaster API, GoOut, ICS), dedupe napříč zdroji, export společného kalendáře (.ics), notifikace kolizí.

## 11. Externí závislosti

- **Cloudflare Workers** (runtime) + **D1** (DB) — nad `saas-foundation`.
- **gmail-mcp** (Anamax443) — FB/IG info přes e-mail.
- Konektorová API: Bandsintown, Songkick, Ticketmaster Discovery, IG Graph `business_discovery`.
- Case: AI-extrakce termínů z webů/plakátů (Claude) — sdílí logiku s `faxx-dox`.

## 12. Otevřené otázky (rozhodnout před F1)

1. **Báze:** `saas-foundation` vs `rezervace-app` jako výchozí tenant vzor?
2. **Viditelnost:** zůstat u „super admin jako ventil", nebo rovnou navrhnout souhlas hráče/bookujícího manažera?
3. **Konektory pro MVP:** stačí Bandzone + vlastní web, nebo rovnou přidat jeden oficiální API zdroj?
4. **Hosting dat + GDPR:** kde běží D1, privacy policy timing (F1 vs F2)?
