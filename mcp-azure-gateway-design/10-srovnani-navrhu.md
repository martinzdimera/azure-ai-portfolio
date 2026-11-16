# 10 — Srovnání dvou nezávislých architektonických návrhů

**Verze:** 1.0 · **Datum:** 2026-09-17 · **Stav:** Posudek
**Předmět:** Návrh A = tato dokumentace (`00`–`09`). Návrh B = `Codex/architektura/` (`README.md`, `00-business-kontext.md` až `07-zdroje-a-rozhodnuti.md`, `kontrakty/`, `validace/`, `diagramy/`).

---

## 1. Účel a metoda

Na totéž zadání (`MCP-Azure_task_description.md`) vznikly nezávisle dva kompletní architektonické návrhy. Tento dokument je porovnává. **Není to soutěž a nemá vítěze.** Dva nezávislé návrhy téhož zadání jsou vzácná příležitost: tam, kde se shodnou, je pravděpodobnější, že rozhodnutí je robustní a ne jen zvyk jednoho autora; tam, kde se rozejdou, leží buď skutečný kompromis, který musí rozhodnout zadavatel, nebo slepé místo jednoho z nich. Cílem je najít obojí.

Metoda: (1) mapování rozhodnutí proti rozhodnutí bez ohledu na pojmenování v dokumentech; (2) u každého rozporu určení věcného důvodu — jiný předpoklad, jiná priorita, nebo chyba; (3) nezávislé ověření sporných tvrzení proti primárním zdrojům (MCP specifikace, Microsoft Learn, dokumentace Anthropic a OpenAI, npm a NuGet registry, Node.js release schedule).

Kritéria: věcná správnost; úplnost vůči zadání; poctivost (je odděleno, co je požadavek, co návrh a co nepotvrzený předpoklad?); implementovatelnost; provozuschopnost a předatelnost internímu IT. Posuzuje se stejně přísně obojí; skutečnost, že návrh A je „náš", není argument.

---

## 2. Souhrnná tabulka klíčových rozhodnutí

| # | Rozhodnutí | Návrh A | Návrh B | Shodné? | Komentář |
|---|---|---|---|---|---|
| 1 | Jazyk a runtime | TypeScript / Node.js 24, Fastify 5 | C# / ASP.NET Core na.NET 10 | ne | Obojí obhajitelné; zdůvodnění A je chybné (§7) |
| 2 | Hosting | Container Apps, workload profile, **Premium ingress**, 2+ repliky | Container Apps, 2+ repliky, standardní ingress | ano (platforma), ne (ingress) | A potřebuje Premium kvůli SSE éry A; B s 30s deadlinem ne |
| 3 | Role APIM | výhradně downstream | výhradně downstream, zákaz smyčky MCP → APIM → MCP | **ano** | Silná shoda; doloženo (§3) |
| 4 | Ingress / WAF | Front Door Premium + WAF + Private Link origin | veřejný ingress ACA; firemní edge jen jako varianta | ne | A konkrétnější a doložená; B nechává na inventuře |
| 5 | Identity provider | Entra ID jako jediný Authorization Server | Entra ID single-tenant, broker jako záložní brána | ano (primárně) | B má explicitní fallback, A ne |
| 6 | Dynamic Client Registration | bez DCR, statické app registrations | bez DCR, předem registrované credentials | **ano** | Nezávislá shoda, potvrzeno zdroji |
| 7 | Autorizační model | dvouvrstvý: app role → tool set, pak policy engine s parametrovými constrainty; deny > allow | konjunkce 7 podmínek vč. rollout a RuntimeBlock; důraz na objektové právo | částečně | Stejný princip, jiná granularita (§4.4) |
| 8 | Downstream identita | OBO + Managed Identity, passthrough zakázán | tři pojmenované režimy `delegated_obo` / `managed_identity` / `legacy_secret` | ano (princip) | B z toho dělá atribut nástroje — přenositelnější |
| 9 | Úložiště katalogu | Cosmos DB for NoSQL + Managed Redis | Blob (neměnné balíčky) + Azure SQL (index, ukazatel) | ne | §4.8 |
| 10 | Aktivace katalogu | atomický přepis `activePointer` | `EnvironmentPointer` + optimistic concurrency, `If-Match`, HTTP 412 | částečně | B má API kontrakt souběhu, A jen vlastnost úložiště |
| 11 | Úložiště auditu | dual-sink Log Analytics + WORM Blob, **hash chain**, outbox | SQL append-only + outbox + WORM Blob, bez hash chainu | částečně | §4.7 |
| 12 | Zdroj skillů | GitHub App webhook + reconcile á 15 min | CI vytvoří **podepsaný** balíček; runtime nikdy netahá z GitHubu | částečně | Shoda na „žádný live pull"; B navíc ověřuje vydavatele |
| 13 | Sandboxing skillů | žádný kód; `http` na whitelist + deklarativní `transform` (JSONata/JMESPath) | žádný kód, žádný šablonovací jazyk, pevné enum/field mapování | **ano** (princip) | §4.6 |
| 14 | Verze MCP protokolu | dual-era: `2025-11-25`/`2025-06-18` + `2026-07-28` na jednom URL | profily L (`2025-11-25`) a S (`2026-07-28`); cíl S, přechodně L | **ano** (podstata) | Obojí si všimlo dvou ér |
| 15 | Sessions v legacy éře | vydávané `Mcp-Session-Id`, session store v Redis | **žádné** session ID, žádné sticky sessions | ne | B je jednodušší a spec-konformní (§4.1) |
| 16 | Idempotence zápisů | `Idempotency-Key`, Redis cache 24 h, povinný `confirm` | `requestKey` + unikátní SQL index, tombstone 90 dnů, `UnknownOutcome` + reconcile | ne | §4.7 |
| 17 | Rate limiting | třívrstvý: WAF → gateway (Redis bucket) → APIM policy | admission limit + sdílený SQL bucket; varování před per-replica pastí | částečně | A robustnější mechanismus, B lepší varování |
| 18 | Rollout | PoC (stdio) → pilot ≤ 25, read-only → celofiremně | **interoperability spike** → PoC → pilot 25–50 → zápisy → produkce | ne | §4.9 |
| 19 | Nouzové odebrání přístupu | `deny` entitlement + čekání na cache TTL | `RuntimeBlock` při každé invokaci, TTL 0, fail-closed | ne | B deterministický |
| 20 | Business kontext | není | samostatná kapitola `00` + 3 diagramy | ne | Mezera v A (§5) |
| 21 | Strojově čitelné kontrakty | Mermaid + JSON příklady v textu | `openapi.yaml`, JSON Schema, `data-model.sql`, ukázkový katalog | ne | Mezera v A (§5) |

---

## 3. Kde se návrhy shodují

Shoda dvou nezávisle vzniklých návrhů není důkaz, ale je to nenulový signál: dva autoři došli ke stejnému závěru z týchž veřejných zdrojů, aniž by se domluvili. Shodných bodů je devět a jsou to shodou okolností ty nejrizikovější.

**APIM je downstream, ne vstupní brána.** Oba čtou zadání („forwards connector calls to the existing APIM layer") stejně. Ověřeno: dokumentace APIM sama varuje, že přístup k tělu odpovědi v MCP politikách spouští bufferování a rozbíjí streaming — přesně to, co by APIM na vstupu dělal.

**Entra ID bez DCR.** Oba nezávisle zjistily, že Entra nenabízí RFC 7591 registration endpoint a klienti se registrují staticky. Odpovídá to i doporučení Anthropicu pro servery s vysokou návštěvností.

**Skill nikdy nespouští kód.** Nejcitlivější rozhodnutí celého návrhu — skill repozitář je vstup s nízkou bariérou přispívání do procesu, který drží tokeny. Oba nezávisle dospěly ke stejnému zákazu. Nejsilnější signál srovnání; doporučuji prezentovat zadavateli jako pevné, nediskutovatelné rozhodnutí.

**Existují dvě éry MCP protokolu.** Oba si všimly, že revize `2026-07-28` odstranila `initialize` handshake a sessions a zavedla metadata v každém požadavku, a oba z toho udělaly samostatné adaptéry, ne jen jiné číslo verze. Jednodušší návrh by zde selhal tiše.

**Žádný token passthrough.** Oba nezávisle identifikovaly confused deputy a řeší jej výměnou tokenu s vlastní audience.

Dále shoda na: Container Apps, GitHubu jako autoritě definic s runtime čtením jen z publikovaných artefaktů, read-only prvním pilotu, a na tom, že počty uživatelů, region, SKU a retence nejsou zadané a nesmí se vymýšlet.

---

## 4. Kde se rozcházejí

### 4.1 Stavovost legacy éry — nejpodceněnější rozdíl

**A** implementuje éru A plnohodnotně: vydává `Mcp-Session-Id`, drží session store v Redis a v ADR-001 to uvádí jako nutnost. **B** v profilu L žádné session ID nevydává a sticky sessions nepoužívá.

**Důvod rozdílu:** A čte éru A jako „stavovou generaci" a odvozuje z toho stavové úložiště. B tutéž specifikaci čte jako volitelnost.

**Kdo má pravdu:** B. Specifikace `2025-11-25` (Session Management) říká, že server **MAY** přidělit session ID. Server éry A musí přijmout a zodpovědět `initialize`, ale nemusí si mezi požadavky nic pamatovat. Tvrzená nutnost v ADR-001 je nesprávná.

**Riziko horší volby:** session store je komponenta navíc v kritické cestě, další stav k obnově po restartu, další zdroj nedostupnosti v SLO řetězci a další věc k provozování po předání — bez protihodnoty. Zároveň zbytečně tlačí na Premium ingress a dlouhé idle timeouty s vlastním cenovým dopadem.

**Doporučení:** převzít přístup B. V ADR-001 přeformulovat éru A na bezstavovou obsluhu, Redis ponechat jen pro rate limit a idempotenci. Tím padá i hlavní argument pro Premium ingress (ADR-004), který lze degradovat na „volitelné, pokud zátěžový test ukáže potřebu".

### 4.2 Ingress

Oba mají APIM downstream; rozdíl je ve vstupu. **A** staví Front Door Premium s WAF a Private Link originem do Container Apps (doloženo: Private Link origin je výhradně Premium tier, Container Apps je podporovaný typ originu). **B** navrhuje prostý veřejný autentizovaný ingress ACA, firemní edge jen jako alternativu po inventuře.

**Lepší A**, kdykoli má organizace povinné WAF/DDoS/geo-filtrování na perimetru nebo požadavek, aby origin nebyl veřejně dosažitelný — u produkčního přístupu k firemním datům spíš pravidlo než výjimka. **Lepší B** v PoC a pilotu, kde Front Door Premium (základní poplatek řádu stovek USD měsíčně) přidává náklad a hop navíc bez měřitelného přínosu při 25 uživatelích. **Riziko horší volby:** veřejně dosažitelný MCP endpoint bez WAF je z definice veřejně adresovatelný, autentizovaný až na aplikační vrstvě a je cílem automatizovaného skenování.

**Doporučení:** etapovost — pilot bez Front Door (B), produkce s Front Door Premium + Private Link (A). Přechod je konfigurační, ne architektonický, protože gateway o něm nemusí vědět. Doplnit do `02-architektura-reseni.md`.

### 4.3 Runtime a MCP SDK

**A** volí TypeScript/Node.js 24: éru B pokrývají `@modelcontextprotocol/server` a `@modelcontextprotocol/fastify` 2.0.0, éru A `@modelcontextprotocol/sdk` 1.30.0 — **dvě paralelní linie SDK**. C#/.NET zamítá s odůvodněním „méně zralé MCP SDK, neověřeno". **B** volí C#/.NET 10 a oficiální C# SDK, ale poctivě dodává, že existence SDK neprokazuje podporu všech verzí, a připnutí nechává na PoC.

**Fakt:** odůvodnění A je nesprávné (§7, A-1). Oficiální C# SDK je na NuGetu v GA verzích 2.0.0–2.2.0 a jeho v2 linie je explicitně zaměřená na `2026-07-28` včetně `server/discover`, bezstavového modelu a automatického fallbacku na `2025-11-25` — pokrývá **obě éry jedním balíčkem**, zatímco A si vytvořila závislost na dvou nezávisle verzovaných liniích a sama si to v ADR-002 zapisuje jako negativní důsledek.

**Doporučení:** volbu jazyka rozhoduje standard týmu, ne architekt; zdůvodnění v ADR-002 je ale nutné opravit, protože zamítá C# na základě neplatného tvrzení. Umí-li tým obojí, je C# u tohoto zadání přímější cesta k dual-era gateway; umí-li jen TypeScript, volba A zůstává, jen s jiným odůvodněním a s explicitním rizikem rozjetí dvou SDK linií.

### 4.4 Autorizační model

**A:** hrubá vrstva Entra app role → tool set, jemná vrstva policy engine s parametrovými constrainty a data scope; deny > allow, specifičtější selektor vyhrává, každé rozhodnutí auditované. **B:** entitlement jako konjunkce sedmi podmínek včetně `rollout membership` a `RuntimeBlock`, s důrazem na **objektovou autorizaci** — právo na konkrétní záznam, nejen na nástroj — a s pravidlem, že tentýž filtr musí platit i pro `search`/`list` a stránkování, a že rozdílná chybová hláška nesmí prozradit existenci záznamu.

**Rozdíl jde oběma směry.** A má propracovanější jemnou vrstvu (constrainty na hodnoty parametrů, riziková třída, mapování riziko → role). B má explicitně pojmenovaný druhý stupeň a tvrdě říká, že jej nezaručuje ani managed identity, ani samotný Entra login, a že se nelze spoléhat na to, že zafiltruje backend. A tento problém neignoruje, ale nemá pro něj samostatný pojem.

**Riziko:** bez pojmenovaného druhého stupně vznikne nástroj správně autorizovaný na úrovni „smíš volat `service_asset_get`", který ale vrátí záznam, na nějž uživatel právo nemá. To je nejpravděpodobnější třída incidentu v celém systému.

**Doporučení:** kombinace. Do `07-bezpecnost-a-autorizace.md` §5 doplnit objektovou autorizaci jako samostatný pojmenovaný stupeň včetně pravidla pro `list`/`search` a stránkování. Zbytek modelu A ponechat.

### 4.5 Dynamic Client Registration a konkrétnost Entra konfigurace

Oba dospěly ke stejnému závěru (bez DCR). Rozdíl je v tom, jak daleko došly.

**A** dává úplný doložený recept: `requestedAccessTokenVersion: 2` musí být nastaveno **před** Application ID URI; Application ID URI musí být ve tvaru `https://`, ne `api://`, protože klient posílá `resource` jako kanonickou URL serveru; doména musí být ověřená vlastní, `*.azurewebsites.net` nestačí; při nesouladu Entra vrátí `AADSTS9010010`; redirect URI hostovaného Claude je `https://claude.ai/api/mcp/auth_callback`; Claude Code používá RFC 8252 loopback. Na tom staví i troubleshooting runbook.

**B** tutéž oblast uzavírá jako otevřenou bránu R01 („přímá kompatibilita není touto dokumentací prokázaná"), navrhuje experiment v testovacím tenantu a jako fallback OAuth broker federovaný s Entra.

**Kdo má pravdu:** A. Konfigurace je dokumentovaná první stranou, a to přímo ve zdroji, který B sám cituje jako S11 (Anthropic, „Authentication for connectors"). Ten obsahuje explicitní poznámku o Entra ID, Application ID URI, ověřené doméně i o chybě `AADSTS9010010`. B zdroj otevřel, ale pasáž minul.

**Riziko horší volby:** B drží otevřenou architektonickou větev (broker), která by podle jeho vlastního textu znemožnila OBO — a tím i všechny nástroje vyžadující uživatelský kontext v backendu. To je velká větev otevřená kvůli nedohledané poznámce.

**Doporučení:** základ A. Z B převzít dvě věci: (1) pravidlo, že **ChatGPT může používat callback-specific i stabilní redirect URI a nesmí se hardcodovat jediná historická varianta** — A to má jen jako TODO; (2) formulaci, že broker není runtime fallback, ale rozhodovací brána.

### 4.6 Sandboxing skillů

Oba zakazují kód; liší se v tom, co je ještě povolené. **A** povoluje deklarativní `transform` v JSONata/JMESPath. **B** povoluje jen pevné enum/field mapování a explicitně zakazuje „spustitelný šablonovací jazyk".

**Lepší A**, když je skutečně potřeba přemapovat vnořené struktury nebo normalizovat tvar odpovědi. **Lepší B** vždy, když má být povrch nástroje auditovatelný doménovým vlastníkem dat, který JSONata neumí číst: JSONata je sice Turing-neúplná, ale má funkce, `$map`/`$reduce` a dokáže spotřebovat netriviální CPU — je to výrazový jazyk vyhodnocovaný v procesu gateway nad **nedůvěryhodnými daty z backendu**. **Riziko:** DoS přes patologický výraz nad velkou odpovědí a — vážněji — schválení skillu, kterému reviewer nerozumí, takže projektuje víc polí, než si myslí.

**Doporučení:** `transform` ponechat, ale doplnit tři mantinely z ducha B: tvrdý limit na dobu vyhodnocení a velikost vstupu; **povinný `outputSchema` jako kontrakt** — co mu neodpovídá, je chyba, ne prošlá data (B to má jako `OUTPUT_INVALID`); allowlist povolených funkcí. Do `07` §7 a do validátoru manifestu.

### 4.7 Model auditu a konzistence zápisu — největší mezera návrhu A

**A** má hash-chained dual-sink audit: `prevHash`/`hash`, synchronní zápis do outboxu, asynchronní doručení do Log Analytics a WORM Blobu, denní checkpoint, nezávislý verifikační job. To je tamper-evidence, kterou B nemá.

**B** má naopak něco, co A nemá: **transakční model provedení volání**. Čtyři transakce (T1 přijetí s lease → T2 admission a dispatch → T3 dokončení → T4 export), stav `Accepted` uložený trvale **před** jakýmkoli efektem v backendu, stav `UnknownOutcome` pro ztracenou odpověď po odeslání zápisu, `downstream_idempotency_key` uložený už v T1, čtyři pojmenované režimy dohledání (`manual`, `workload_status`, `user_status`, `none`), privátní reconcile API s `If-Match` a povinným `evidenceRef`, a pravidlo, že `Cancelled` nikdy nesmí znamenat domněnku, že zrušení HTTP zrušilo backend.

Stavový automat A končí u `TimedOut` jako terminálního stavu. Chybová tabulka A sice uživateli správně říká „operace mohla proběhnout — ověř stav, než ji zopakuješ", ale **není kam ten stav zapsat, kdo ho má dohledat, ani podle čeho**. Idempotency cache v Redis řeší opakování téhož požadavku, neřeší nejistotu o efektu. **Riziko:** duplicitní zápis do produkčního systému, nebo — horší — uživateli oznámený neúspěch u operace, která proběhla. V agentním prostředí s automatickými retry je to nejdražší třída incidentu a A sama to tak v ADR-016 pojmenovává, aniž pro to má mechanismus.

**Doporučení — nejdůležitější položka posudku.** Do `06-stavova-logika-a-workflow.md` (automat 3) doplnit: trvalý stav `Accepted` zapsaný před voláním downstream, s lease; terminální `UnknownOutcome` a přechody `ReconciledSucceeded`/`ReconciledFailed`; `downstreamIdempotencyKey` uložený při přijetí, ne generovaný až při volání; reconcile endpoint a runbook (`08` §6) s povinným důkazem od operátora. Zbytek auditního modelu A je silnější než B a zůstává.

### 4.8 Úložiště: Cosmos + Redis versus SQL + Blob

**A:** Cosmos DB for NoSQL (session consistency, change feed jako invalidační kanál) + Managed Redis. **B:** Azure SQL (invocation, audit, outbox, pointer, runtime blocks) + Blob pro neměnné katalogové balíčky a oddělený Blob pro audit archiv, bez Redisu.

**Důvod:** A optimalizuje latenci `tools/list` a chce nativní invalidaci. B optimalizuje **transakčnost** — model T1–T4 vyžaduje stav invokace, auditní událost a outbox v jedné transakci, což relační databáze umí a dvojice Cosmos + Redis sama o sobě ne. **Lepší A** při vysokém objemu čtení katalogu, více regionech a týmu se zkušeností s Cosmos; **lepší B** vždy, když jsou v katalogu zápisové nástroje. **Riziko:** A provozuje dvě datové technologie a přesto nemá pojmenovanou transakční hranici, kterou pro zápisy potřebuje.

**Doporučení:** primární volbu A neměnit — Cosmos umí transakční batch v rámci jedné logické partition, což pro dvojici `invocation` + `auditOutbox` se stejným `tenantId` stačí. Ale je nutné to v `04-datova-architektura.md` **explicitně napsat jako požadavek**, ne předpokládat. Kdyby to nestačilo, je Azure SQL podle B lepší a A už v ADR-009 rovnocennou relační alternativu uvádí.

### 4.9 Pořadí rolloutu — kdy se poprvé testuje skutečný klient

**A:** Fáze 0 je lokální PoC na stdio transportu s docker compose, mock APIM a fake katalogem; exit kritéria Gate 0 jsou conformance harness, autorizační matice, verifikace hash chainu a reprodukovatelný `docker compose up`. **Skutečný Claude ani ChatGPT v Gate 0 nefigurují** — první reálné připojení je až v pilotu na Azure.

**B:** samostatná etapa 0 „inventura a interoperability spike" s podmínkou ukončení „oba klienti na jedné test URL; per-user token; správné aud/scope; odebrání přístupu". Teprve pak lokální PoC. B navíc upozorňuje, že cloudový Claude/ChatGPT nemůže volat `localhost` vývojáře a že je potřeba schválený dočasný HTTPS endpoint nebo dev Azure host.

**Kdo má pravdu:** B, jednoznačně. Největší nejistoty projektu jsou (1) kterou éru protokolu skutečně mluví produkční konektory a (2) zda OAuth tok proti Entra projde v tomto konkrétním tenantu. Obojí je levné ověřit a drahé objevit pozdě; A je odkládá za bránu, která je ověřuje jen proti vlastnímu conformance harnessu, tedy proti vlastnímu předpokladu. **Riziko:** A projde Gate 0 zeleně a v pilotu zjistí, že `resource` proti Entra selhává kvůli neověřené doméně — organizační úkol na týdny, který A sama označuje jako blokující předpoklad A02. Objevit to v pilotu znamená zastavený pilot.

**Doporučení:** do `08` §9 vložit **Gate −1: interoperability spike** před Fázi 0, s exit kritérii B: oba klienti na jedné dočasné HTTPS URL, zaznamenaná skutečná protokolová verze každého z nich, per-user token s ověřenými `aud`/`scp`/`tid`/`oid`, ověřené odebrání přístupu. Výhradně syntetická data.

---

## 5. Co má návrh B a návrh A ne — doporučení k doplnění

1. **Business kontext a mapa případů použití.** B má kapitolu `00-business-kontext.md` (uživatel, potřeba, přínos, cesta informace, use case BU-01 až BU-06, scénář krok za krokem, tabulka „co se stane, když to nejde", kritéria úspěchu pilotu) plus tři diagramy. A nemá nic. → Nový `00b-business-kontext.md`, nebo kapitola na začátek `01`. Bez toho návrh nelze předložit vlastníkovi procesu ani sponzorovi.
2. **`UnknownOutcome` a reconciliation workflow** (§4.7). → `06` automat 3, `04` (`downstreamIdempotencyKey`, `reconciliationMode`), `08` §6 runbook.
3. **Strojově čitelné kontrakty.** B dodává `openapi.yaml` (14 KB), `tool-manifest.schema.json`, `audit-event.schema.json`, `data-model.sql`, ukázkový `catalog.json`, příklady payloadů obou profilů a skripty pro jejich validaci. A má JSON jen jako bloky v textu. → Adresář `kontrakty/` a validace jako CI brána (`08` §8). Je to rozdíl mezi dokumentací a předatelným artefaktem.
4. **`RuntimeBlock` s deterministickou hranicí.** B čte nouzové deny při každé invokaci s TTL 0, fail-closed, s transakční hranicí admission, po které už nový dispatch neprojde. A nabízí `deny` entitlement s čekáním na cache TTL. → `07` §5 a `06`, jako samostatný necachovaný mechanismus vedle entitlements.
5. **Objektová autorizace jako pojmenovaný druhý stupeň** včetně `list`/`search` a stránkování (§4.4). → `07` §5.
6. **Gate −1 interoperability spike** (§4.9). → `08` §9.
7. **Tabulka návrhových parametrů pro PoC i produkci.** B má 23 pojmenovaných parametrů (`REQUEST_DEADLINE_MS`, `MAX_REQUEST_BYTES`, `CONNECTOR_TIMEOUT_MS`, lease doby, prahy audit backlogu…) se dvěma sloupci hodnot a smyslem každého. A je má roztroušené a často jako TODO. → Sjednotit do jedné tabulky v `08` §7.
8. **Akceptační matice testů.** B má C01–C21 s očekávaným výsledkem. → Doplnit do `08` §9, minimálně C04 (uživatel A žádá objekt B), C08 (20 paralelních zápisů se stejným klíčem), C09 (backend potvrdil, gateway ztratila odpověď), C19, C20.
9. **Varování před per-replica rate limitem.** B upozorňuje, že uživatelský limit definovaný per repliku se škálováním násobí. A problém nemá (sdílený Redis bucket), ale nikde to neříká — čtenář, který limit implementuje jinak, past nevidí. → Jedna věta do `08` §7.
10. **Oddělené identity pro pět rolí systému** (`gateway-runtime`, `audit-worker`, `catalog-publisher`, `catalog-admin`, `infrastructure-deployer`) s invariantem „publisher nečte business data, runtime nemění katalog, worker neaktivuje release". → `07` §8.
11. **Editovatelná topologie (Draw.io) a offline atlas** pro handover internímu IT. → Volitelné, zvážit u finální verze.

---

## 6. Co má návrh A a návrh B ne

1. **Doložená Entra konfigurace** včetně `AADSTS9010010`, tvaru `https://` Application ID URI, pořadí `requestedAccessTokenVersion: 2` a požadavku na ověřenou doménu. B to má jako otevřenou bránu. Nejcennější jednotlivá věc v návrhu A.
2. **Hash-chained tamper-evidence auditu** s denním checkpointem a nezávislou verifikací. B má append-only a WORM — odolnost proti smazání, ne proti úpravě.
3. **Model hrozeb STRIDE** s 20 pojmenovanými hrozbami, mitigacemi a reziduálním rizikem.
4. **Pokrytí novinek revize `2026-07-28`, které B vůbec nezmiňuje:** `x-mcp-header` / `Mcp-Param-*` (zrcadlení parametrů nástroje do HTTP hlaviček — klient to **MUST** podporovat, server **MUST** validovat shodu hlavička↔tělo a při neshodě vrátit `-32020`), MRTR / `InputRequiredResult` / `resultType: "input_required"` a `subscriptions/listen` jako náhrada GET streamu. A navíc identifikuje CRLF injection přes `Mcp-Param-*` a omezuje `x-mcp-header` jen na `region`. **Věcná mezera B:** nedostatečná validace hlavička↔tělo je přesně ta zranitelnost, před níž spec varuje — zprostředkovatel směruje podle hlavičky, server vykoná podle těla.
5. **Multi-workspace model** (per-workspace katalog, limity, `pinnedVersion`) včetně zdůvodnění, proč workspace nemá vlastní audience.
6. **Povinný server-side `confirm`** vázaný na `destructiveHint` nebo `riskTier ∈ {high, critical}`, nezávislý na klientském UX; B serverovou potvrzovací bránu nemá.
7. **Riziková klasifikace nástrojů (`riskTier`)** a mapování riziko → role, řídící i pořadí uvolňování zápisů.
8. **Error budget a rozbor složeného SLA závislostí**, včetně přiznání, že naivní násobení SLA cíl nesplní.
9. **Volba IaC nástroje** (Bicep s AVM, Terraform jako rovnocenná alternativa) s odůvodněním; B má jen strukturu adresářů `infra/`.

---

## 7. Faktické chyby

Ověřeno proti primárním zdrojům 2026-09-17.

### Návrh A — 3 doložené chyby

**A-1. ADR-002: „C#/.NET zamítnuto — méně zralé MCP SDK, neověřeno v době rozhodnutí."**
Nesprávné. Oficiální C# SDK je na NuGetu jako balíček `ModelContextProtocol` v GA verzích **2.0.0, 2.1.0 a 2.2.0**; jeho v2 linie je podle release notes „designed for alignment with the 2026-07-28 MCP specification release" a implementuje `server/discover`, bezstavový model, per-request metadata a automatický fallback na `2025-11-25`. Pokrývá obě éry jedním balíčkem, zatímco A si u Node.js zvolila dvě paralelně verzované linie a v témže ADR to uvádí jako negativní důsledek.
*Zdroje:* `api.nuget.org/v3-flatcontainer/modelcontextprotocol/index.json`; `github.com/modelcontextprotocol/csharp-sdk/releases`.

**A-2. ADR-002: „Node.js 24 je Active LTS … pro nově vznikající produkční systém je aktivní LTS řada konzervativnější volba."**
Tvrzení je k datu dokumentu formálně pravdivé, ale argument je vadný a vede k volbě, která bude neplatná dřív, než systém vznikne. Podle oficiálního release schedule přechází **Node.js 24 do režimu Maintenance 2026-10-20**, tedy 33 dní po datu tohoto návrhu, a Active LTS se 2026-10-28 stává Node.js 26. Vlastní logikou ADR-002 by správnou volbou byl Node.js 26.
*Zdroj:* `nodejs/Release` → `schedule.json`: `v24 {lts: 2025-10-28, maintenance: 2026-10-20, end: 2028-04-30}`, `v26 {lts: 2026-10-28}`.

**A-3. ADR-001 a `00-README.md`: „nutnost udržovat session store (Redis) jen kvůli éře A."**
Nesprávné. Specifikace `2025-11-25`, Session Management: *„A server using the Streamable HTTP transport **MAY** assign a session ID at initialization time."* Přidělení je volitelné; server éry A může být plně bezstavový. Návrh B tuto variantu volí a je spec-konformní.
*Zdroj:* `modelcontextprotocol.io/specification/2025-11-25/basic/transports`.
*Dopad:* zbytečná komponenta v kritické cestě a zbytečný tlak na Premium ingress.

### Návrh B — 1 doložená chyba

**B-1. `07-zdroje-a-rozhodnuti.md`, rozpor R01: „přímá kompatibilita [Entra ID s MCP `resource` parametrem] není touto dokumentací prokázaná."**
Tvrzení o stavu důkazů je nesprávné, a to vůči zdroji, který B sám uvádí jako S11. Stránka Anthropic „Authentication for connectors" obsahuje explicitní poznámku: *„If your authorization server is Microsoft Entra ID, you must also register the MCP server URL as an Application ID URI on your Entra app registration, or the token request fails with `AADSTS9010010`"*, spolu s upozorněním, že Entra takovou URL přijme jen na doméně ověřené tenantem, takže host typu `*.azurewebsites.net` vyžaduje nejdřív vlastní doménu. Funkční konfigurace i přesný režim selhání jsou tedy dokumentované první stranou.
*Zdroj:* `claude.com/docs/connectors/building/authentication`, sekce „Cross-host authorization servers".
*Dopad:* B kvůli tomu drží otevřenou větev (OAuth broker), která by podle jeho vlastního textu znemožnila OBO, a tím i všechny nástroje vyžadující uživatelský kontext v backendu.

### Co u návrhu B bylo ověřeno a obstálo

Aby bylo srovnání poctivé, dalších jedenáct sporných tvrzení B obstálo: `2026-07-28` je aktuální revize a odstranila session handshake; `resultType=complete` i povinné `ttlMs`/`cacheScope` u `list` platí; `Mcp-Method`/`Mcp-Name` jsou REQUIRED a jejich neshoda s tělem vede na `400` + `-32020`; server éry S vrací na GET `405` a na neznámou metodu `404` + `-32601`; zavření response streamu **MUST** být bráno jako zrušení; jeden HTTP request nese jednu JSON-RPC zprávu, ne dávku;.NET 10 je Active LTS do 2028-11-14; standardní ingress Container Apps má strop 240 s; APIM nepodporuje MCP prompts u externích serverů ani MCP ve workspaces; ChatGPT Business je mezi podporovanými účty pro developer mode; `https://claude.ai/api/mcp/auth_callback` je správné redirect URI.

### Společné slepé místo obou návrhů

Ani jeden nezmiňuje **Skills over MCP** — pracovní skupinu MCP a návrh rozšíření SEP-2640 („Skills Extension", Resources-based, Extensions Track), ke dni posudku ve stavu *In Review*, který řeší přesně to, co je v zadání pod slovem „skills": jak se strukturované pracovní instrukce objevují, distribuují a konzumují přes MCP. Oba nezávisle zvolily vlastní řešení přes nástroje (`skills_list`/`skills_get` u B, skill nástroje u A), což je dnes správné — rozšíření není ratifikované. Ale ani jeden neuvádí, že nativní mechanismus vzniká, a tedy nemá plán, co se svým řešením udělat, až vznikne. **Doporučení pro oba:** doplnit jako sledovanou položku a jako kritérium při návrhu tvaru skill manifestu, aby byl převeditelný.
*Zdroj:* `modelcontextprotocol.io/community/working-groups/skills-over-mcp`.

---

## 8. Závěr a doporučení

**Základem vzít návrh A.** Rozhoduje hloubka tam, kde je chyba nejdražší a zpětně nejhůř opravitelná: doložená Entra/OAuth konfigurace včetně přesných režimů selhání, model hrozeb, tamper-evidence auditu, pokrytí novinek revize `2026-07-28` včetně bezpečnostně relevantního `x-mcp-header`, a provozní vrstva (SLO, error budget, runbooky, handover). Návrh B je v těchto místech buď obecnější, nebo — u Entra — otevřený tam, kde odpověď existuje.

**Z návrhu B převzít jedenáct věcí z §5.** Tři z nich jsou podmínkou produkčního nasazení, ne vylepšením:

1. **Transakční model provedení volání s `UnknownOutcome` a reconciliation** (§4.7). Bez něj nelze zapnout jediný zápisový nástroj bez rizika duplicity nebo falešného potvrzení.
2. **Gate −1: interoperability spike s oběma skutečnými klienty před vším ostatním** (§4.9).
3. **Objektová autorizace jako pojmenovaný druhý stupeň** včetně `list`/`search` a stránkování (§4.4).

Dále doporučuji převzít **bezstavovou obsluhu legacy éry** (§4.1) — zjednodušuje návrh, ruší jednu komponentu a opravuje chybu A-3 — a **business kontext** (§5 bod 1), bez kterého se návrh nedá předložit sponzorovi. A **opravit v A tři faktické chyby** podle §7.

### Co nelze rozhodnout technicky a musí rozhodnout zákazník

- **Jazyk a runtime.** Po opravě A-1 nemá žádná strana technickou převahu. Rozhoduje standard týmu, který bude gateway po předání udržovat: umí interní IT číst a měnit TypeScript, nebo C#?
- **Co znamená „skill".** Oba návrhy nezávisle zvolily „instrukce v Markdownu, nikoli spustitelný kód". Pokud zadavatel myslel spustitelné skilly, mění se rozsah zásadně a oba by potřebovaly oddělený sandboxový worker. Musí padnout dřív než první řádek kódu.
- **Veřejný ingress a jeho cena.** Front Door Premium versus prostý ingress ACA je rozhodnutí o riziku a rozpočtu, ne o architektuře (§4.2).
- **Retence auditu a WORM.** A navrhuje dlouhou WORM retenci s pseudonymizací přes `oid`, B 90 dnů online / 365 archiv. Napětí mezi neměnností auditu a právem na výmaz musí posoudit DPO. Zamknutí WORM politiky je nevratné.
- **Zda vůbec budou zápisové nástroje a kdo je vlastníkem cílového API.** Obě doporučení stojí na tom, že vlastník dodá kontrakt idempotence a objektové autorizace. Bez toho se zápisy nezapínají.
- **Kdo spravuje ověřenou doménu pro Application ID URI a kdo v Entra tenantu smí založit app registration.** A to označuje jako blokující předpoklad s dobou řešení v řádu týdnů. První věc, která musí být zadaná.

---

**Poznámka k rozsahu posudku.** Posuzovány byly textové dokumenty obou návrhů, diagramy jako zdrojové soubory a u návrhu B i adresáře `kontrakty/` a `validace/`. Nebyl proveden zásah do návrhu B ani do jiných souborů návrhu A. Kde se tvrdí, že návrh B něco neobsahuje, znamená to „v posuzovaných souborech jsem to nenašel" — ne „je to špatně".
