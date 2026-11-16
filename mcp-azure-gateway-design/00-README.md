# 00 — README: MCP-Azure Gateway, architektonický návrh

**Verze:** 1.0 · **Datum:** 2026-09-17 · **Stav:** Návrh
**Týká se ADR:** všech (viz [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md))

## K čemu dokument slouží

Toto je rozcestník k úplnému architektonickému návrhu **MCP-Azure Gateway** — produkčního MCP serveru, na který ukazují connector endpointy Claude i ChatGPT Business zároveň. Návrh pokrývá protokolovou vrstvu, autentizaci a autorizaci, adapter vrstvu (skilly z GitHubu, connectory do existující API Management vrstvy), audit trail, nasazení na Azure a fázovaný rollout. Návrh stojí na 23 architektonických rozhodnutích (ADR-001 až ADR-023); tato dokumentace je rozvádí, dokládá zdroji a doplňuje o implementační detail. Po prvním kole review proběhlo nezávislé srovnání s alternativním návrhem ([`10-srovnani-navrhu.md`](./10-srovnani-navrhu.md)), které odhalilo mezery a faktické nepřesnosti — tato dokumentace je opravuje a doplňuje (nový dokument [`00b`](./00b-kontext-a-pripady-uziti.md), nový adresář [`kontrakty/`](./kontrakty/), ADR-023, opravy v ADR-001/ADR-002 a nová Fáze −1 rolloutu; podrobnosti viz sekce „Stav a verzování" níže).

## Jak je dokumentace členěná

Třináct textových dokumentů (z toho jeden business-kontextový, bez technického žargonu), sada Mermaid diagramů a adresář strojově čitelných kontraktů:

| Dokument | O čem je |
|---|---|
| [`00-README.md`](./00-README.md) | Tento rozcestník. |
| [`00b-kontext-a-pripady-uziti.md`](./00b-kontext-a-pripady-uziti.md) | Byznysový kontext bez technického žargonu — proč gateway vzniká, mapa konkrétních případů užití, přínosy, zainteresované strany, důsledek nerealizace (shadow IT). |
| [`01-analyza-zadani-a-plan.md`](./01-analyza-zadani-a-plan.md) | Analýza zadání, rozsah, předpoklady a plán zpracování návrhu. |
| [`02-architektura-reseni.md`](./02-architektura-reseni.md) | Celková architektura řešení — kontext, kontejnery, topologie na Azure, síťový tok. |
| [`03-kodova-architektura.md`](./03-kodova-architektura.md) | Komponentová dekompozice gateway a ingestoru, moduly, hranice, testovatelnost. |
| [`04-datova-architektura.md`](./04-datova-architektura.md) | Datový model — kontejnery Cosmos DB, entitní schémata, ERD, retenční politiky. |
| [`05-api-a-protokoly.md`](./05-api-a-protokoly.md) | HTTP povrch gateway, MCP metody, chybová taxonomie, konkrétní JSON příklady požadavků a odpovědí. |
| [`06-stavova-logika-a-workflow.md`](./06-stavova-logika-a-workflow.md) | Stavové automaty — životní cyklus katalogu, spojení, tool invocation, circuit breaker, rollout brány. |
| [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) | Model hrozeb (STRIDE), OAuth 2.1 autentizace, autorizační model (role, entitlements, constrainty), ochrana skill repozitáře, tajemství, compliance. |
| [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) | SLO/SLI, telemetrie, dashboardy, alerting, runbooky, škálování, CI/CD, fázovaný rollout, konfigurace klientů, handover pro IT. |
| [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md) | Všech 23 architektonických rozhodnutí v plném formátu, kompletní seznam zdrojů, souhrn otevřených otázek. |
| [`10-srovnani-navrhu.md`](./10-srovnani-navrhu.md) | Nezávislé srovnání s alternativním návrhem — shody, rozdíly, faktické chyby obou stran, doporučení. Zdroj oprav a doplnění popsaných v této verzi dokumentace. |
| [`11-revize-navrhu.md`](./11-revize-navrhu.md) | Revize návrhu — druhý pohled: nezávislý pohled na hotový návrh (dokumenty 00–10) s cílem najít místa, kde je návrh složitější, než musí být. |

Diagramy jsou v [`diagramy/`](./diagramy/), každý zároveň vložený do příslušného textového dokumentu s výkladem mechaniky, ne jen popisem tvarů. Strojově čitelné kontrakty (JSON Schema, OpenAPI) jsou v [`kontrakty/`](./kontrakty/) — formalizují to, co je jinde v textu jen próza a příklady, a CI je validuje proti dokumentaci na každém pull requestu (viz [`kontrakty/README.md`](./kontrakty/README.md) a [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §8).

## Pořadí čtení podle role

**Architekt** (rozhoduje o celkovém návrhu, schvaluje ADR): `00b` (byznysový kontext) → `01` → `02` → `09` (ADR přehled) → `03` → `04` → `07` §2 a §5 (model hrozeb a autorizační jádro) → `08` §9 (rollout, včetně Fáze −1).

**Vývojář** (implementuje gateway nebo ingestor): `02` → `03` → `04` → `05` → `06` → `07` §5–§6 (autorizační algoritmus, ochrana proti injection) → `08` §2–§3 (telemetrie, logování jako implementační kontrakt).

**Sponzor / byznys stakeholdeři** (rozhoduje o financování a přijetí, nepotřebuje technický detail): `00b` (kontext a případy užití) → shrnutí sedmi klíčových rozhodnutí níže → `08` §9 (fáze rolloutu a brány, srozumitelně i bez technického pozadí).

**Provozní IT** (přebírá handover, řeší incidenty): `00` (tento dokument) → `08` celý, zejména §5–§7, §9 (rollout včetně Fáze −1) a §11 (alerting, runbooky, handover) → `07` §8 (tajemství a certifikáty, co rotovat) → `09` (TODO seznam — co ještě není uzavřené).

## Shrnutí návrhu — sedm klíčových rozhodnutí

1. **Dual-era gateway (ADR-001).** Gateway mluví současně starší, stavovou generací MCP protokolu (kterou dnes reálně používají produkční Claude i ChatGPT klienti) i nejnovější bezstavovou generací specifikace — jádro systému žádnou z nich „nevidí", pracuje s jedním kanonickým interním modelem.
2. **APIM je downstream, ne vstupní brána (ADR-003).** Gateway přijímá provoz přes Front Door Premium → Private Link → Container Apps; API Management je cíl, který gateway volá pro connector nástroje, nikdy vstupní bod.
3. **Entra ID jako jediný Authorization Server, bez Dynamic Client Registration (ADR-005, ADR-006).** Gateway je čistý OAuth 2.1 Resource Server. Entra ID nepodporuje RFC 7591 DCR, proto se Claude i ChatGPT connector registrují staticky jako samostatné app registrations.
4. **Dvouvrstvá, deny-by-default autorizace (ADR-007).** Hrubá vrstva (Entra app role → tool set) rychle odfiltruje nedosažitelné nástroje; jemná vrstva (entitlement pravidla s parametrovými constrainty) rozhoduje o konkrétním volání. Explicitní deny vždy přebíjí allow. Každé rozhodnutí je auditované.
5. **Nikdy token passthrough (ADR-008).** Downstream identita jde přes On-Behalf-Of výměnu s vlastní audience per API, infrastrukturní přístup přes Managed Identity — obrana proti confused deputy útoku.
6. **Hash-chained, dual-sink audit trail (ADR-010).** Každé volání nástroje generuje auditní záznam vázaný kryptografickým hash řetězcem na předchozí, zapsaný nezávisle do Log Analytics (dotazovatelné) i WORM Blob Storage (nezměnitelné) — tamper-evidence, ne jen logování.
7. **Fázovaný rollout s tvrdými, měřitelnými branami, od interoperabilního spiku po plné nasazení (ADR-018).** Fáze −1 interoperabilní spike (ověření proti reálným Claude i ChatGPT Business workspacům, doplněno po nezávislém srovnání návrhu) → lokální PoC → pilot (≤ 25 uživatelů, jen čtení) → celofiremní nasazení s postupným uvolňováním zápisových nástrojů. Každá brána má definovaná exit kritéria a cestu zpět; selhání Fáze −1 jako jediné vede k přehodnocení návrhu, ne jen k zúžení rozsahu.

## Seznam diagramů

| Diagram | O čem je |
|---|---|
| `01-kontext-c4.mmd` | C4 kontext — gateway, Claude, ChatGPT, APIM, GitHub, Entra ID jako vnější systémy. |
| `02-kontejnery-c4.mmd` | C4 kontejnery — gateway, ingestor, datové úložiště, jejich vzájemné vazby. |
| `03-komponenty-gateway-c4.mmd` | C4 komponenty uvnitř gateway — moduly `edge/`, `protocol/`, `identity/`, `authz/` atd. |
| `04-azure-topologie.mmd` | Topologie nasazení na Azure — resource groups, síť, privátní endpointy. |
| `05-sitovy-tok.mmd` | Síťový tok požadavku od klienta po downstream. |
| `06-oauth-sekvence.mmd` | Sekvence OAuth 2.1 autorizačního flow s PKCE a `resource` indikátorem. |
| `07-tool-call-sekvence.mmd` | Sekvence zpracování `tools/call` od requestu po audit zápis. |
| `08-obo-downstream-sekvence.mmd` | Sekvence On-Behalf-Of výměny tokenu pro downstream volání. |
| `09-skill-ingestion-sekvence.mmd` | Sekvence od GitHub push po publikaci katalogové verze. |
| `10-stav-katalog.mmd` | Stavový automat životního cyklu katalogové verze. |
| `11-stav-session-era-a.mmd` | Stavový automat MCP spojení v éře A. |
| `12-stav-request-era-b.mmd` | Stavový automat bezstavového requestu v éře B. |
| `13-stav-tool-invocation.mmd` | Stavový automat jednoho tool volání včetně idempotence a retry. |
| `14-stav-circuit-breaker.mmd` | Stavový automat circuit breakeru k downstream APIM. |
| `15-datovy-model-erd.mmd` | ERD datového modelu — kontejnery Cosmos DB a jejich vztahy. |
| `16-audit-hash-chain.mmd` | Tok auditního záznamu od vzniku přes hash chain po nezávislé ověření integrity. |
| `17-authz-rozhodovaci-tok.mmd` | Rozhodovací tok autorizace — token, role, tool set, entitlements, constrainty, audit. |
| `18-cicd-pipeline.mmd` | CI/CD — oddělené dráhy pro kód a pro katalog skillů, s branami kvality. |
| `19-rollout-faze.mmd` | Fáze rolloutu od Fáze −1 (interoperabilní spike) po steady state, brány a cesty zpět (přehledový flowchart). |
| `20-dual-era-negotiation.mmd` | Vyjednávání protokolové éry mezi klientem a gateway. |
| `21-stav-obo-token.mmd` | Stavový automat životního cyklu OBO tokenu — žádost, cache, refresh, expirace a revokace (CAE). |
| `22-stav-rollout-gate.mmd` | Stavový automat rollout bran od Gate −1 po steady state — fáze nasazení, podmínky jednotlivých bran a cesty zpět, včetně cesty na přehodnocení návrhu při selhání Gate −1. |

## Stav a verzování

Všech 13 textových dokumentů (`00`, `00b`, `01`–`11`), 22 diagramů a adresář [`kontrakty/`](./kontrakty/) jsou ve verzi 1.0, datováno 2026-09-17, se stavem „Návrh" — připraveno k připomínkování a schválení, ne k produkčnímu nasazení bez dalšího review. Otevřené body, které vyžadují potvrzení mimo tento návrh (compliance, zadavatel, výsledky zátěžových testů), jsou v textu vždy označeny `⚠️ TODO` a shrnuty v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md).

**Opraveno po nezávislém srovnání návrhu** (viz [`10-srovnani-navrhu.md`](./10-srovnani-navrhu.md)): faktické opravy F1 (mylné tvrzení o nezralosti C#/.NET MCP SDK v ADR-002), F2 (Node.js 24 Maintenance od 2026-10-20) a F3 (session store éry A je volitelný modul, spec MAY ne MUST) v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md) a [`02-architektura-reseni.md`](./02-architektura-reseni.md); doplnění D1–D7 — nový [`00b`](./00b-kontext-a-pripady-uziti.md), nový [`kontrakty/`](./kontrakty/) jako CI brána, Fáze −1 v `01` a `08`, souhrnná tabulka číselných parametrů a akceptační testovací matice v `08`, upozornění na per-replika rate limiting v `05`/`08`, a ADR-023 k SEP-2640 „Skills over MCP" v `09`.

**Otevřené položky napříč celým návrhem** jsou souhrnně v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md) v sekci „Seznam všech ⚠️ TODO a neověřených tvrzení" — pokrývá mimo jiné potvrzení compliance frameworku organizace, výsledky zátěžového testu pro kapacitní plánování, redirect URI pro ChatGPT Business connector a DPO posouzení retence audit logu. Tabulka je doplněna přehledem dalších otevřených položek napříč zbylými dokumenty; před finálním schválením návrhu je vhodné sloučit vše do jednoho řízeného provozního backlogu.

## Zdroje

Viz číslovaný seznam v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md) — pokrývá MCP specifikaci, relevantní RFC, Microsoft Learn, dokumentaci Anthropic a OpenAI použité napříč celým návrhem.
