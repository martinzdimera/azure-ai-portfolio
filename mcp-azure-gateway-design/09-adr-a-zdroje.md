# 09 — ADR a zdroje

**Verze:** 1.0 · **Datum:** 2026-09-17 · **Stav:** Návrh
**Týká se ADR:** všech, ADR-001 až ADR-023

Úplný katalog architektonických rozhodnutí (Architecture Decision Records) pro MCP-Azure Gateway. ADR-001 až ADR-018 jsou závazná architektonická rozhodnutí tohoto návrhu, zde rozvedená do plného formátu. ADR-019 až ADR-023 doplňují rozhodnutí, která z návrhu vyplývají nebo reagují na nezávislé srovnání návrhu (viz [`10-srovnani-navrhu.md`](./10-srovnani-navrhu.md)), ale nejsou v jádru návrhu explicitně zachycená. Formát každého záznamu: Kontext → Rozhodnutí → Odůvodnění → Důsledky → Zvažované alternativy → Stav → Zdroje.

---

## ADR-001: Dual-era MCP gateway

- **Kontext:** MCP specifikace má dvě neslučitelné generace protokolu živé současně — éru A (`2025-11-25` a starší, stavový handshake) a éru B (`2026-07-28`, bezstavová) [Z-MCP1]. Produkční dokumentace Claude i ChatGPT v září 2026 stále popisuje éru A [Z-A1], zatímco specifikace samotná deklaruje éru B jako aktuální.
- **Rozhodnutí:** Gateway implementuje obě éry současně za jedním URL, s vnitřní normalizací na kanonický interní model (`CanonicalRequest`/`CanonicalResult`). Detekce éry probíhá per-request na vstupu.
- **Odůvodnění:** Volba jen jedné éry by znamenala buď nefunkční gateway dnes (jen éra B — klienti ji nemluví), nebo technický dluh zítra (jen éra A — spec ji označuje jako legacy s plánovaným vyřazením některých mechanismů).
- **Důsledky:** (+) gateway funguje s reálnými klienty i je připravena na budoucí migraci bez přepisu jádra; (+) jádro (routing, authz, audit) je éru-agnostické; (−) vyšší implementační složitost vrstvy `protocol/`; (+) éra A **nevyžaduje** session store jako architektonickou nutnost (oprava A-3 — viz `10-srovnani-navrhu.md` §7) — specifikace `2025-11-25`, Session Management, říká: *„A server using the Streamable HTTP transport **MAY** assign a session ID at initialization time"* [Z-MCP6], tedy volitelné, ne povinné. Server éry A smí `initialize` obsloužit zcela bezstavově; session store v Redis je pro éru A **volitelný modul** (viz [`02-architektura-reseni.md` §4](./02-architektura-reseni.md) a [`05-api-a-protokoly.md` §1.2](./05-api-a-protokoly.md)), nasazovaný jen tehdy, když provozní zkušenost prokáže přínos resumability (`Last-Event-ID` replay) nebo diagnostiky dlouhotrvajících spojení — ne jako trvalá komponenta kritické cesty bez alternativy.
- **Zvažované alternativy:** Jen 2025-11-25 (zamítnuto — ignoruje směr specifikace, dluh při budoucí migraci klientů). Jen 2026-07-28 (zamítnuto — nefunkční proti současným produkčním klientům).
- **Stav:** Přijato.
- **Zdroje:** [Z-MCP1], [Z-A1], [Z-MCP6]

## ADR-002: TypeScript / Node.js 24 (Active LTS), Fastify 5 (opraveno A-1, A-2 — viz `10-srovnani-navrhu.md` §7)

- **Kontext:** Implementační jazyk a HTTP framework musí podporovat dlouhotrvající streamable-HTTP/SSE spojení a mít funkční MCP SDK podporu pro obě éry.
- **Rozhodnutí:** TypeScript na **Node.js 24 (Active LTS)**. Pro éru B **`@modelcontextprotocol/server` 2.0.0**, `@modelcontextprotocol/client` 2.0.0 a **`@modelcontextprotocol/fastify` 2.0.0** (vydáno 2026-07-27). Pro éru A **`@modelcontextprotocol/sdk` 1.30.0** (vydáno 2026-07-27). HTTP vrstva Fastify 5, Zod schémata sdílená mezi runtime validací a `inputSchema` deklarací nástrojů.
- **Odůvodnění — runtime a jeho životní cyklus.** Node.js 22 je od 2025-10-21 v režimu Maintenance s EOL 2027-04-30. Node.js 24 je Active LTS, ale **jen do 2026-10-20** — necelý měsíc po datu tohoto návrhu (2026-09-17) — kdy sám přechází do Maintenance (EOL 2028-04-30); Active LTS štafetu přebírá Node.js 26 od 2026-10-28 [Z-AZ14]. To musí být řečeno explicitně, ne zamlčeno: v okamžiku produkčního nasazení (po Fázi 0–1, řádově měsíce od data tohoto dokumentu) poběží gateway na runtimu, který právě přešel nebo se chystá přejít do Maintenance. **Strategie:** Maintenance status není blokující — Node.js v Maintenance nadále dostává bezpečnostní opravy až do EOL, takže provoz zůstává podporovaný po celou dobu Fáze 1–2. Návrh proto Maintenance u Node.js 24 **přijímá vědomě**, ne jako přehlédnutí, a zařazuje vyhodnocení povýšení na Node.js 26 (Active LTS od 2026-10-28) jako revizní bod do plánu prací (viz [`01-analyza-zadani-a-plan.md`](./01-analyza-zadani-a-plan.md), po Gate 1), ne jako podmínku go-live. Node.js 26 nebyl zvolen rovnou, protože k datu tohoto dokumentu je ve fázi `Current`, ne `Active LTS` — nasazovat produkční systém rovnou na non-LTS řadu by bylo vyšší riziko než krátké, dopředu známé okno v Maintenance na řadě, která už LTS certifikaci má.
- **Odůvodnění — volba jazyka vůči C#/.NET.** Předchozí verze tohoto ADR zamítala C#/.NET s odůvodněním „méně zralé MCP SDK, neověřeno" — to bylo **nepravdivé tvrzení** a je zde opraveno. Oficiální **MCP C# SDK** (balíček `ModelContextProtocol` na NuGetu, udržovaný ve spolupráci s Microsoftem) je k datu tohoto dokumentu na **GA verzi 2.2.0** (stabilní vydání, publikováno 2026-08-13; řada zahrnuje i GA 2.0.0 a 2.1.0) [Z-CS1], a release poznámky k v2 řadě popisují zarovnání s revizí `2026-07-28` s automatickým fallbackem na `2025-11-25` [Z-CS2] — tedy **jeden balíček pokrývající obě protokolové éry**, na rozdíl od volby v tomto ADR, kde éru A a éru B pokrývají dvě nezávisle verzované linie (`@modelcontextprotocol/sdk` 1.x vs. `@modelcontextprotocol/server`/`client`/`fastify` 2.x). **C#/.NET je proto plnohodnotná alternativa, ne řešení druhé kategorie.** Skutečné důvody volby TypeScriptu jsou jiné: (1) Zod schémata slouží zároveň jako runtime validace argumentů nástroje i jako zdroj `inputSchema` deklarace — jeden zdroj pravdy bez generování/duplikace, což by v C# vyžadovalo buď zdrojové generátory nad JSON Schema, nebo ruční synchronizaci dvou reprezentací; (2) existence oficiálního `@modelcontextprotocol/fastify` 2.0.0 dává hotovou integraci s Fastify, frameworkem zvoleným v tomto ADR pro HTTP vrstvu; (3) šířka ekosystému knihoven pro deklarativní `transform` (JSONata/JMESPath, viz [`05-api-a-protokoly.md`](./05-api-a-protokoly.md) §9) je v Node.js ekosystému vyzrálejší; (4) předpokládaná kompetence týmu, který bude gateway po handoveru udržovat (`⚠️ TODO — potvrdit se zadavatelem, viz A14/A18 v [`01-analyza-zadani-a-plan.md`](./01-analyza-zadani-a-plan.md)`). **C#/.NET by byla lepší volba, kdykoli platí jedna nebo více z těchto podmínek:** tým má silnější nebo výhradní kompetenci v.NET; organizace už provozuje.NET backendy, do kterých má gateway logicky zapadnout (sdílené knihovny, interní NuGet feed, společný CI/CD standard); nebo je žádoucí ergonomie Azure SDK pro.NET — Managed Identity, Key Vault a Cosmos DB SDK mají v.NET historicky nejplnější a nejlépe udržovanou podporu ze všech jazyků na Azure. V takovém případě je jediný C# balíček pokrývající obě protokolové éry čistě technická výhoda, ne nevýhoda, kterou by bylo nutné omlouvat. Statické typy a sdílená Zod schémata (v C# variantě analogicky silné typy a zdrojové generátory) v obou jazycích shodně snižují riziko rozjetí mezi deklarovaným a skutečným chováním nástroje (viz hrozba tool poisoning/schema confusion v [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) §2).
- **Důsledky:** (+) jedno schéma zdroj pravdy pro validaci i dokumentaci nástroje; (+) silná typová kontrola napříč `packages/contracts`; (+) aktivně podporovaný Node.js runtime po celou dobu Fáze 0–1; (−) závislost na zralosti a vzájemném souladu dvou paralelně verzovaných linií MCP SDK (`sdk` 1.x pro éru A, `server`/`client`/`fastify` 2.x pro éru B) — riziko, které volba C#/.NET jedním balíčkem nemá; (−) podpora `@modelcontextprotocol/sdk` 1.x končí řádově v 1. pololetí 2027 a adaptér éry A na ni vyžaduje plán před tímto datem (viz riziko R15 v `01`); (−) Node.js 24 přechází do Maintenance 2026-10-20, necelý měsíc po datu tohoto návrhu — přijato vědomě, bez dopadu na bezpečnostní podporu do EOL 2028-04-30, s revizí povýšení na Node.js 26 zařazenou do plánu prací.
- **Zvažované alternativy:** Python/FastMCP (zamítnuto — slabší streaming ergonomie na Container Apps). **C#/.NET — plnohodnotná alternativa, nezamítnutá na základě zralosti SDK** (viz odůvodnění výše); v tomto návrhu nezvolena kvůli sdíleným Zod schématům, existenci `@modelcontextprotocol/fastify` a předpokládané kompetenci týmu, ne kvůli nedostatku C# SDK — volbu jazyka by měl v praxi potvrdit tým, který bude gateway po handoveru udržovat (viz `01`). Node.js 22 LTS (zamítnuto — od 2025-10-21 v režimu Maintenance, kratší zbývající podpora než Node.js 24 i po zohlednění vlastního přechodu Node.js 24 do Maintenance 2026-10-20).
- **Stav:** Přijato.
- **Zdroje:** [Z-MCP1], [Z-AZ14], [Z-CS1], [Z-CS2]

## ADR-003: Ingress = Front Door Premium → Private Link → Container Apps

- **Kontext:** Zadání popisuje APIM jako cíl volání connectorů („forwards connector calls to the existing APIM"), ne jako vstupní bránu. Dvojitý hop přes APIM na vstupu by přidal latenci a APIM logging response body by rozbíjel MCP streaming [Z-AZ1].
- **Rozhodnutí:** Ingress = Azure Front Door Premium s WAF → Private Link → Azure Container Apps (internal ingress). APIM zůstává výhradně downstream.
- **Odůvodnění:** Front Door Premium je jediná varianta podporující Private Link origin (Standard tier toto nemá) [Z-AZ2], WAF na edge blíž ke klientovi, ne až za druhým hopem. Oddělení ingress a downstream odpovídá skutečné roli APIM v této architektuře.
- **Důsledky:** (+) nižší latence, žádné riziko rozbití streamingu diagnostickým loggingem APIM na vstupu; (+) jasné oddělení odpovědností; (−) organizace s povinným API governance gatem na vstupu musí akceptovat, že APIM zde negatuje tuto roli (viz varianta B níže).
- **Zvažované alternativy:** APIM jako ingress — popsáno jako varianta B pro organizace s povinným API governance gatem na vstupu, zamítnuto jako primární volba kvůli výše uvedeným důvodům. Application Gateway WAF — zamítnuto, regionální rozsah a bez Private Link origin podpory na non-Premium úrovni [Z-AZ2].
- **Stav:** Přijato.
- **Zdroje:** [Z-AZ1], [Z-AZ2]

## ADR-004: Compute = Azure Container Apps

- **Kontext:** MCP streamable HTTP (zejména éra A se SSE) vyžaduje dlouhotrvající spojení s konfigurovatelným idle timeoutem, blue/green nasazení a autoscaling.
- **Rozhodnutí:** Azure Container Apps, dedikovaný workload profile (ne jen Consumption pro produkci), min. 2 repliky v produkci, s **Premium ingress módem výslovně zapnutým** a `idleRequestTimeout = 600` s (10 minut) pro provoz éry A / SSE.
- **Odůvodnění:** Výchozí (non-Premium) ingress mode Container Apps má idle timeout pevných 240 s — stejně jako App Service (jehož Load Balancer má tvrdý strop ~230–240 s [Z-AZ3]) — a **sám o sobě by tedy hlavní argument této volby negoval**: bez explicitně zapnutého Premium ingress by volba mezi ACA a App Service stála jen na revisions, KEDA a scale-to-zero. Výhodu dává až **Premium ingress mode**, kde je `idleRequestTimeout` konfigurovatelný v rozsahu **4–30 minut (240–1800 s)** [Z-AZ13]. Container Apps dále nabízí nativní revisions pro blue-green/canary traffic split [Z-AZ5], KEDA-based scale rules, VNet a Private Endpoint bez nutnosti Isolated tier — tyto výhody platí nezávisle na volbě ingress módu, ale samotný delší idle timeout je podmíněn Premium ingress.
- **Důsledky:** (+) s Premium ingress nejdelší konfigurovatelný idle timeout mezi zvažovanými platformami; (+) nativní podpora canary rolloutu využitá v [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §8; (−) Premium ingress má vlastní cenový dopad (`⚠️ TODO — ověřit v Azure Pricing Calculator`); (−) provozní model KEDA scale rules vyžaduje zátěžové testování k určení správných prahů (viz otevřené otázky dokumentu 08).
- **Zvažované alternativy:** App Service Linux (zamítnuto — tvrdý ~230s LB timeout limituje dlouhé streamy; ACA bez Premium ingress by měla stejný problém). Azure Functions (zamítnuto jako primární — vhodné spíš pro krátké operace, obdobný timeout strop). AKS (zamítnuto — provozní režie nepřiměřená rozsahu).
- **Stav:** Přijato.
- **Zdroje:** [Z-AZ3], [Z-AZ4], [Z-AZ5], [Z-AZ13]

## ADR-005: Authorization Server = Microsoft Entra ID

- **Kontext:** Organizace již provozuje Entra ID jako identity provider s existující Conditional Access, MFA a lifecycle správou uživatelů.
- **Rozhodnutí:** Entra ID je jediný Authorization Server, gateway je čistě OAuth 2.1 Resource Server bez vlastní AS logiky.
- **Odůvodnění:** Využití existující infrastruktury eliminuje duplikaci identity správy a odpovídající compliance zátěž vlastního AS (rotace klíčů, správa uživatelů, MFA enforcement by musely být znovu implementovány).
- **Důsledky:** (+) žádná duplicitní identity infrastruktura; (+) přímé využití Conditional Access/MFA; (−) plně vázáno na omezení Entra ID jako AS, konkrétně absenci DCR (viz ADR-006).
- **Zvažované alternativy:** Vlastní AS (Duende IdentityServer, Keycloak) — zamítnuto, duplikace identity a dodatečná compliance zátěž bez odpovídajícího přínosu.
- **Stav:** Přijato.
- **Zdroje:** [Z-MS1]

## ADR-006: Bez Dynamic Client Registration

- **Kontext:** Entra ID nepublikuje RFC 7591 registration endpoint — potvrzeno napříč pěti nezávislými MS Learn zdroji [Z-MS2], [Z-MS3], [Z-MS4]. MCP spec navíc DCR deprecuje ve prospěch CIMD [Z-MCP2], kterou Entra rovněž nepodporuje jako AS mechanismus.
- **Rozhodnutí:** Claude a ChatGPT connectory se registrují staticky jako samostatné Entra app registrations; gateway publikuje RFC 9728 Protected Resource Metadata na `/.well-known/oauth-protected-resource`.
- **Odůvodnění:** Toto je platformní nutnost, ne designová preference — Entra strukturálně DCR nenabízí. Statická registrace navíc odpovídá doporučení Anthropicu pro servery s vysokou návštěvností (vyhnout se neomezenému růstu klientů) [Z-A2].
- **Důsledky:** (+) žádné riziko nekontrolovaného růstu registrovaných klientů; (+) jednodušší bezpečnostní model (žádný `/register` endpoint k obraně); (−) každý nový klient/connector vyžaduje ruční administrativní krok (app registration + distribuce `client_id`/`secret`), viz [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) §4.
- **Zvažované alternativy:** DCR proxy/shim před Entrou — zamítnuto, bezpečnostní riziko nekontrolovaného růstu klientů a provozní dluh vlastního shim komponentu bez odpovídajícího přínosu.
- **Stav:** Přijato.
- **Zdroje:** [Z-MS2], [Z-MS3], [Z-MS4], [Z-MCP2], [Z-A2]

## ADR-007: Dvouvrstvá deny-by-default autorizace

- **Kontext:** Autorizace musí škálovat na stovky nástrojů a desítky rolí, s auditovatelným rozhodnutím per volání. Samotný `groups` claim má overage problém (200 skupin JWT limit [Z-MS5]) a app roles samotné neškálují na jemnozrnné, parametrové constrainty.
- **Rozhodnutí:** Hrubá vrstva — Entra app roles v `roles` claimu mapované na tool sety. Jemná vrstva — policy engine nad katalogem s tool-level entitlements, parametrovými constrainty a data scope. Deny-by-default, explicitní deny > allow, specifičtější selektor vyhrává mezi allow pravidly. Každé rozhodnutí je auditované.
- **Odůvodnění:** Plné rozpracování algoritmu, formátu pravidel a decision matrix je v [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) §5 — jádro tohoto dokumentu.
- **Důsledky:** (+) škáluje na velký počet nástrojů bez nutnosti jednu app roli per kombinaci; (+) auditovatelnost per-rozhodnutí umožňuje ladění „proč mi to zakázalo"; (−) dvouvrstvá evaluace přidává latenci oproti jedné ploché kontrole (mitigováno tím, že hrubá vrstva odfiltruje většinu požadavků levně před jemnou).
- **Zvažované alternativy:** Jen `groups` claim (zamítnuto — overage problém při velkém počtu skupin [Z-MS5]). Jen APIM policy (zamítnuto — APIM je downstream dle ADR-003, ne vhodné místo pro centrální autorizační rozhodnutí napříč skill i connector nástroji). OPA sidecar — popsáno jako budoucí rozšíření, ne zamítnuto natrvalo, pokud složitost pravidel v budoucnu přeroste možnosti vlastního policy engine.
- **Stav:** Přijato.
- **Zdroje:** [Z-MS5], [Z-MS6]

## ADR-008: OBO pro downstream identitu, Managed Identity pro infrastrukturu

- **Kontext:** Přeposlání příchozího uživatelského tokenu beze změny do downstream API (token passthrough) je confused deputy zranitelnost — downstream by nemohlo spolehlivě rozlišit, pro koho byl token vydán.
- **Rozhodnutí:** On-Behalf-Of (OBO) flow pro connector volání do APIM, kde downstream API potřebuje uživatelský kontext. Managed Identity (app-only) pro infrastrukturní volání (Key Vault, Cosmos, Storage, GitHub App token exchange). Token passthrough je zakázán bezvýjimečně.
- **Odůvodnění:** OBO zachovává uživatelskou identitu až do backendu (audit a downstream authz zůstávají smysluplné), zatímco token audience izolace (nový token, nová audience per downstream) strukturálně brání confused deputy scénáři — viz [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) §2 hrozba #1 a §3.3. OBO používá výhradně delegated scopes, ne app roles — role zůstávají vázané na uživatele, ne na aplikaci jednající jeho jménem [Z-MS7].
- **Důsledky:** (+) plná auditní stopa až do downstream; (+) žádné riziko cross-API elevation přes sdílený token; (−) OBO token cache a refresh logika je dodatečná implementační složitost (`identity/` modul), viz stav `OBO token lifecycle` v [`06-stavova-logika-a-workflow.md`](./06-stavova-logika-a-workflow.md).
- **Zvažované alternativy:** Token passthrough (zamítnuto — confused deputy, viz ADR odůvodnění výše). Jen Managed Identity i pro uživatelský kontext (zamítnuto — ztráta uživatelské identity na downstream, znemožňuje per-uživatelský audit a authz na straně downstream API).
- **Stav:** Přijato.
- **Zdroje:** [Z-MS7], [Z-MS1]

## ADR-009: Cosmos DB pro katalog, Managed Redis pro cache/session/rate-limit

- **Kontext:** `tools/list` je voláno při každé interakci klienta a musí mít nízkou latenci čtení; katalog i entitlements potřebují durabilitu a nativní invalidační mechanismus.
- **Rozhodnutí:** Azure Cosmos DB for NoSQL (session consistency, partition key `/tenantId`, TTL na cache dokumentech, change feed pro invalidaci cache) pro katalog a entitlements. Azure Managed Redis pro hot cache, rate-limit countery a session store éry A.
- **Odůvodnění:** Change feed poskytuje nativní invalidační kanál bez nutnosti vlastního pub/sub mechanismu [Z-AZ6]. Session consistency je rozumný kompromis mezi latencí čtení a čerstvostí dat pro tento use case. Redis pokrývá krátkodobý, vysoce frekventovaný stav (rate limit token bucket, éra A session), kde durabilita Cosmos není potřeba a latence Redis je řádově nižší.
- **Důsledky:** (+) nízká latence `tools/list` čtení; (+) change feed eliminuje potřebu vlastního cache-invalidation protokolu; (−) dvě odlišné datové technologie k provozování a monitorování místo jedné.
- **Zvažované alternativy:** PostgreSQL Flexible Server — uvedeno jako plnohodnotná alternativa pro týmy se zavedeným SQL provozem, ne zamítnuto principiálně. Jen Redis bez Cosmos (zamítnuto — chybí durabilita a auditovatelnost katalogových dat, Redis TTL by nechtěně mazal historii).
- **Stav:** Přijato.
- **Zdroje:** [Z-AZ6], [Z-AZ7]

## ADR-010: Dual-sink, hash-chained audit trail

- **Kontext:** Zadání vyžaduje „who, what, when, result" audit každého tool invocation; compliance obvykle vyžaduje neměnnost záznamu, ne jen jeho existenci.
- **Rozhodnutí:** Dva nezávislé sinky — Log Analytics custom table přes DCR (pro dotazování a alerting) a Azure Blob Storage s immutability policy (WORM archiv). Každý záznam nese `prevHash` + `hash` (SHA-256 nad kanonizovaným JSON), s denním checkpoint záznamem kořenového hashe. Plný mechanismus a diagram v [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md).
- **Odůvodnění:** Jeden sink samotný nestačí — Log Analytics samotný je mazatelný/má retenční limity, databáze samotná je mutovatelná. Kombinace dvou nezávislých, odlišně spravovaných sinků plus kryptografický hash chain poskytuje tamper-evidence (ne tamper-proof v absolutním smyslu, ale detekci manipulace).
- **Důsledky:** (+) nezávislá verifikace integrity nezávisle na kterémkoli jednotlivém sinku; (+) WORM politika (až 146 000 dní retence [Z-AZ8]) pokrývá dlouhodobé compliance požadavky; (−) dvojnásobný zápisový výkon a složitost outbox patternu k zajištění konzistence mezi sinky.
- **Zvažované alternativy:** Jen App Insights (zamítnuto — mazatelné, retenční limity). Jen databáze (zamítnuto — mutovatelná bez dodatečného hash-chain mechanismu).
- **Stav:** Přijato.
- **Zdroje:** [Z-AZ8], [Z-AZ9]

## ADR-011: Pull + push hybrid skill ingestion

- **Kontext:** Publikace katalogu z Git repozitáře potřebuje být rychlá (reagovat na push) i odolná (nezávislá na spolehlivosti webhook doručení).
- **Rozhodnutí:** GitHub App webhook (push na chráněné větve / release) → validace → build katalogové verze → atomická publikace, nezávisle doplněné periodickým reconcile (každých 15 minut) proti aktuálnímu Git SHA. Commit signature a branch protection jsou vstupní podmínkou publikace.
- **Odůvodnění:** Webhook poskytuje nízkou latenci reakce, reconcile poskytuje záchrannou síť proti ztracenému/nedoručenému webhook eventu — bez reconcile by výpadek jediného webhook doručení mohl nechat katalog neomezeně zastaralý bez jakéhokoli alertu.
- **Důsledky:** (+) rychlá i odolná publikace; (+) měřitelná metrika čerstvosti katalogu (`mcp_catalog_staleness_seconds`, viz [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §2); (−) dvě paralelní cesty k publikaci (webhook a reconcile) vyžadují idempotentní implementaci `publish/` kroku, aby souběh nezpůsobil race condition.
- **Zvažované alternativy:** Jen polling (zamítnuto — nepřijatelná latence mezi mergem a publikací). Jen webhook (zamítnuto — ztráta eventů bez záchranné sítě). Submodule/build-time bake katalogu do image (zamítnuto — vyžaduje redeploy kódu na každou změnu skillu, poruší oddělení kódové a katalogové pipeline, viz ADR-021).
- **Stav:** Přijato.
- **Zdroje:** [Z-A3] (obecný princip GitHub App webhooků, HMAC ověření — dobře zavedená praxe, ne specifický MS Learn zdroj)

## ADR-012: Immutable, verzovaný katalog

- **Kontext:** Pilot potřebuje testovat nové nástroje bez zásahu do kódu; produkce potřebuje okamžitý rollback bez redeploy.
- **Rozhodnutí:** `CatalogVersion` je neměnný snapshot (`id = gitSha + buildHash`). Publikace je atomický přepis ukazatele `activePointer`. Rollback = přepnutí ukazatele. `pinnedVersion` per workspace umožňuje katalogový canary nezávislý na deploymentu kódu.
- **Odůvodnění:** Immutabilita eliminuje celou třídu bugů „katalog se za běhu tiše změnil" a činí rollback triviální a rychlou operací (přepnutí ukazatele, ne rebuild). `pinnedVersion` umožňuje přesně to, co potřebuje fázovaný rollout (ADR-018) — pilotní skupina na jedné verzi, zbytek na jiné, bez forku kódu.
- **Důsledky:** (+) okamžitý, bezpečný rollback; (+) pilot bez rizika pro zbytek organizace; (−) úložný nárok roste s každou publikovanou verzí (mitigace: `Archived` stav a retenční politika starých verzí, mimo přímý rozsah tohoto ADR).
- **Zvažované alternativy:** In-place mutace katalogu (zamítnuto — znemožňuje spolehlivý rollback a auditovatelnost „co bylo aktivní kdy").
- **Stav:** Přijato.
- **Zdroje:** — (odvozeno z požadavků zadání, ne z externího zdroje)

## ADR-013: Sandboxovaný skill — žádný spuštěný kód

- **Kontext:** Skill repozitář je externí vstup do produkčního systému s nízkou bariérou přispívání — atraktivní útočný vektor.
- **Rozhodnutí:** Skill smí deklarovat jen `http` volání na whitelistovaný APIM backend, nebo `transform` v deklarativním JSONata/JMESPath výrazu. Žádný spuštěný uživatelský kód v procesu gateway. Budoucí potřeba kódu poběží v odděleném izolovaném workeru (Container Apps Job), nikdy v request cestě gateway.
- **Odůvodnění:** RCE v gateway procesu by znamenal kompromitaci všech tokenů a přístupu, které gateway proces drží (Managed Identity, OBO cache) — nepřijatelné riziko vzhledem k nízké bariéře přispívání do skill repa. Deklarativní transform pokrývá naprostou většinu legitimních use case (přemapování polí, jednoduché podmínky) bez nutnosti Turing-complete jazyka.
- **Důsledky:** (+) celá třída RCE zranitelností strukturálně vyloučena; (+) validace manifestu je plně statická, ne runtime sandboxing s vlastními hranicemi úniku; (−) omezuje vyjadřovací sílu skillů — komplexní transformace vyžadující skutečnou výpočetní logiku nejsou možné bez budoucího rozšíření.
- **Zvažované alternativy:** `vm2`/`isolated-vm` inline sandboxing (zamítnuto — historicky opakované úniky z JS sandboxů, nepřijatelné riziko v request cestě s přístupem k tokenům). WASM — uvedeno jako budoucí rozšíření, ne zamítnuto natrvalo, pokud poptávka po výpočetní síle přeroste možnosti deklarativního transform.
- **Stav:** Přijato.
- **Zdroje:** — (bezpečnostní princip least-privilege/attack-surface reduction, obecně zavedená praxe)

## ADR-014: OpenTelemetry → Application Insights

- **Kontext:** Multi-vrstvá architektura (edge → protocol → identity → authz → adapters → downstream) potřebuje end-to-end korelaci pro diagnostiku incidentů.
- **Rozhodnutí:** OpenTelemetry jako instrumentační standard, Application Insights jako backend. Jeden `traceId` propagovaný z příchozího requestu přes OBO až do APIM (`traceparent` hlavička). RED metriky plus business metriky (viz [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §2).
- **Odůvodnění:** OTel je vendor-neutrální standard s nativní podporou trace propagace do MCP `_meta` (`traceparent`/`tracestate`/`baggage`, SEP-414 [Z-MCP3]) — umožňuje korelaci napříč gateway i případnými budoucími MCP-native downstream servery bez vlastního korelačního mechanismu.
- **Důsledky:** (+) jeden `traceId` sleditelný od klienta až po downstream odpověď; (+) standardizovaná atributová sada usnadňuje budoucí přechod na jiný APM backend, pokud by byl potřeba; (−) instrumentace všech vrstev je dodatečná implementační práce, ne automatická.
- **Zvažované alternativy:** Jen App Insights SDK bez OTel vrstvy (zamítnuto — vendor lock-in do proprietárního instrumentačního modelu, horší interoperabilita s MCP `_meta` trace propagací). Jen logy bez distribuovaného trasování (zamítnuto — nedostatečné pro diagnostiku napříč vrstvami popsanou v runbookách dokumentu 08).
- **Stav:** Přijato.
- **Zdroje:** [Z-MCP3], [Z-AZ10]

## ADR-015: Multi-workspace model

- **Kontext:** Jeden gateway deployment obsluhuje Claude i ChatGPT Business (a potenciálně další) jako odlišné „workspacy" s odlišným chováním klienta, katalogem a limity.
- **Rozhodnutí:** Workspace identifikován kombinací audience + app role + volitelně dedikovaná cesta `/mcp/{workspaceSlug}`. Katalog, entitlements, rate limity a `pinnedVersion` jsou per-workspace.
- **Odůvodnění:** Zadání explicitně počítá s oběma platformami jako klienty jednoho endpointu; jejich klientské chování (transport preference, autentizační detaily, limity) se dostatečně liší, aby vyžadovalo per-workspace konfiguraci, ne jeden globální nastavení pro všechny.
- **Důsledky:** (+) nezávislá konfigurace a rollout per platforma; (+) izolace — incident/rollback v jednom workspace neovlivní druhý; (−) datový model a routovací logika nesou dodatečnou dimenzi (`workspaceSlug`) napříč prakticky každou entitou.
- **Zvažované alternativy:** Jeden globální katalog pro všechny (zamítnuto — neumožňuje odlišné tool sety/limity per platforma, které zadání implicitně vyžaduje).
- **Stav:** Přijato.
- **Zdroje:** [Z-A4], [Z-O1]

## ADR-016: Idempotence a `confirm` pro zápisy

- **Kontext:** Agenti retryují požadavky (síťové chyby, timeouty); duplicitní zápis do produkčního downstream API je nejdražší třída incidentu.
- **Rozhodnutí:** Každý tool call nese `Idempotency-Key` odvozený z `requestId`. Write nástroje (`destructiveHint: true` nebo `readOnlyHint: false`) procházejí idempotence cache (Redis, TTL 24 h). Explicitní `confirm` parametr je povinný právě když `annotations.destructiveHint === true` **NEBO** `riskTier ∈ {"high", "critical"}`.
- **Odůvodnění:** Idempotence cache zajišťuje, že opakovaný request se stejným klíčem vrátí uložený výsledek místo opakovaného provedení. `confirm` je nezávislá, serverem vynucená obrana proti automatickému řetězení destruktivních nebo obtížně vratných akcí bez lidského potvrzení — nezávisle na tom, zda klientská UX vrstva svůj vlastní dialog zobrazí (viz [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) §6).
- **Důsledky:** (+) bezpečná retry sémantika bez rizika duplicitního zápisu; (+) server-side obrana proti destruktivním akcím nezávislá na klientovi; (−) dodatečná paměťová zátěž Redis (mitigace: TTL 24 h a per-principal kvóta, viz [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) §2 hrozba #20).
- **Zvažované alternativy:** Bez idempotence (zamítnuto — nepřijatelné riziko duplicitních zápisů v agentním prostředí s automatickými retry).
- **Stav:** Přijato.
- **Zdroje:** [Z-MCP4]

## ADR-017: Třívrstvý rate limiting

- **Kontext:** Downstream systémy potřebují ochranu před přetížením; agenti neumí spolehlivě interpretovat holé HTTP 429 uvnitř JSON-RPC streamu.
- **Rozhodnutí:** Front Door WAF (hrubá, IP/geo) → gateway (per-user, per-tool, token bucket v Redis) → APIM policy (`rate-limit-by-key`, `quota-by-key`) na downstream. Gateway vrací MCP chybu s `retryAfterMs` v `_meta`, ne holé HTTP 429 uvnitř JSON-RPC odpovědi.
- **Odůvodnění:** Tři vrstvy pokrývají tři odlišné hrozby — plošný útok/scraping (Front Door), per-uživatelské zneužití konkrétního nástroje (gateway), ochranu samotného downstream API bez ohledu na to, co se stane výš v řetězci (APIM). MCP-native chybová signalizace (`retryAfterMs` v `_meta`) je pro agenta actionable, zatímco holý HTTP 429 uvnitř streamu by mohl být obtížně interpretovatelný.
- **Důsledky:** (+) ochrana na třech nezávislých úrovních, výpadek jedné nezanechává systém bez ochrany; (+) agent dostává strojově čitelný pokyn, kdy zkusit znovu; (−) tři nezávisle konfigurované limity vyžadují koordinaci, aby se vzájemně nerozporovaly (např. gateway limit vyšší než APIM limit je zbytečný).
- **Zvažované alternativy:** Jen APIM (zamítnuto — nechrání gateway samotnou před nadměrným voláním skill nástrojů, které přes APIM vůbec neprochází).
- **Stav:** Přijato.
- **Zdroje:** [Z-AZ1], [Z-MCP5]

## ADR-018: Třífázový rollout s tvrdými branami

- **Kontext:** Zadání i obecná praxe snižování rizika vyžadují postupné zavádění, ne big bang nasazení produkčního přístupu k firemním datům.
- **Rozhodnutí:** **Fáze −1 interoperabilní spike** (nejmenší možný MCP server na veřejné HTTPS URL s OAuth a jedním triviálním nástrojem, připojený ke skutečnému Claude workspace **a** skutečnému ChatGPT Business workspace — doplněno po nezávislém srovnání návrhu, viz [`10-srovnani-navrhu.md`](./10-srovnani-navrhu.md) §6/§8) → Fáze 0 lokální PoC (stdio + docker compose, mock APIM, fake katalog) → Fáze 1 pilot (≤ 25 uživatelů, Entra security group `MCP-Gateway-Pilot`, read-only tool set, izolace pilotní skupiny je **autorizační, ne traffic-splitting** — všichni uživatelé, pilotní i budoucí, běží na stejné nasazené revizi gateway; přístup mimo pilotní skupinu je uzavřen tím, že app role není přiřazena, ne canary traffic split) → Fáze 2 celofiremně (write nástroje postupně, per-role). Každá brána má měřitelná exit kritéria. Plné rozpracování v [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §9.
- **Odůvodnění:** Postupné rozšiřování expozice s měřitelnými branami umožňuje zachytit problémy (technické i UX) v prostředí s omezeným dopadem, než se rozšíří na celou organizaci — obzvlášť důležité pro write nástroje, kde chyba má nevratný dopad. Fáze −1 je samostatný předstupeň, ne součást Fáze 0, protože ověřuje předpoklady **mimo kontrolu tohoto návrhu** — kterou revizi protokolu klienti skutečně mluví, zda OAuth bez DCR projde s reálným tenantem, jaké jsou skutečné timeouty — dřív, než na nich vznikne cokoli dalšího (Fáze 0 už staví na mock APIM/fake katalogu, tedy testuje vnitřní logiku, ne vnější předpoklady). Selhání Gate −1 je jediná brána v rolloutu, jejíž důsledkem není zúžení rozsahu v rámci stejné architektury, ale přehodnocení samotného návrhu (viz Gate −1 exit kritéria a diagram stavů v `08` §9).
- **Důsledky:** (+) nízké riziko plošného incidentu; (+) měřitelná exit kritéria vynucují disciplínu, ne subjektivní „vypadá to OK"; (+) Fáze −1 zachytává nesoulad s realitou klientů v řádu dnů, ne až po ≈165–180 člověkodnech práce na Fázi 0–2 ([`01-analyza-zadani-a-plan.md` §6](./01-analyza-zadani-a-plan.md#6-plán-prací-work-breakdown-structure)); (−) delší celkový čas do plného produkčního nasazení oproti big bang přístupu.
- **Zvažované alternativy:** Big bang (zamítnuto — nepřiměřené riziko vzhledem k produkčnímu charakteru přístupu k firemním datům a nástrojům). Zahrnout interoperabilní ověření jako součást Fáze 0 místo samostatné fáze (zamítnuto — Fáze 0 běží proti mock APIM a fake katalogu, takže by nesprávně budila dojem, že interoperabilita s reálnými klienty byla ověřena, když ve skutečnosti ověřena nebyla).
- **Stav:** Přijato.
- **Zdroje:** — (odvozeno z požadavků zadání a obecné praxe fázovaného rolloutu; Fáze −1 doplněna po nezávislém srovnání návrhu, [`10-srovnani-navrhu.md`](./10-srovnani-navrhu.md) §6/§8)

## ADR-019: Jazyk dokumentace — čeština s anglickými technickými termíny

- **Kontext:** Cílová čtenářská skupina zahrnuje architekty, vývojáře i interní IT tým, primárně česky mluvící organizace. Technické termíny (názvy Azure služeb, claimy, hlavičky, kódy, názvy souborů) mají ustálené anglické názvy s jednoznačným mapováním na oficiální dokumentaci.
- **Rozhodnutí:** Veškerá architektonická dokumentace v češtině. Technické termíny, názvy Azure služeb, názvy claimů, hlaviček, chybových kódů a souborů zůstávají v původním anglickém tvaru, nepřekládají se.
- **Odůvodnění:** Čeština snižuje bariéru pro širší interní IT tým (viz handover v [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §11), zatímco zachování anglických technických termínů zajišťuje jednoznačné, dohledatelné mapování na oficiální Microsoft/MCP/Anthropic/OpenAI dokumentaci — počeštění by zavedlo riziko nejednoznačnosti (např. který český ekvivalent odpovídá `readOnlyHint`?).
- **Důsledky:** (+) přístupné širšímu internímu publiku; (+) beze ztráty přesnosti při odkazování na oficiální zdroje; (−) míšení jazyků v jedné větě může být pro netechnické čtenáře zpočátku nezvyklé.
- **Zvažované alternativy:** Plně anglická dokumentace (zamítnuto — bariéra pro širší IT tým bez silné angličtiny). Plně počeštěné termíny včetně technických názvů (zamítnuto — ztráta jednoznačného mapování na oficiální dokumentaci a zdrojový kód).
- **Stav:** Přijato.
- **Zdroje:** — (rozhodnutí o formě dokumentace, bez externího zdroje)

## ADR-020: Formát diagramů — Mermaid jako kód

- **Kontext:** Architektura vyžaduje 22 diagramů napříč kontextem, komponentami, sekvencemi a stavovými automaty, podléhajících revizím a dlouhodobé údržbě.
- **Rozhodnutí:** Všechny diagramy v Mermaid, uložené jako samostatné `.mmd` soubory v `diagramy/` a zároveň vložené do textu v ```` ```mermaid ```` blocích s doprovodným odstavcem výkladu.
- **Odůvodnění:** Diagramy jako text (diagrams-as-code) jsou diffovatelné v PR review stejně jako kód, nevyžadují proprietární editor, renderují se nativně v GitHubu a mnoha dokumentačních platformách, a lze je strojově validovat syntaktickým parserem Mermaid jako součást CI (viz gating v [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §8).
- **Důsledky:** (+) verzovatelné, diffovatelné, validovatelné; (+) žádný lock-in do konkrétního kreslicího nástroje; (−) složité diagramy s mnoha uzly mohou narazit na limity čitelnosti/renderování (viz zkušenost s diagramy 18 a 19 v tomto souboru dokumentů, kde bylo nutné zkrátit popisky a přepnout orientaci pro úspěšné vyrenderování).
- **Zvažované alternativy:** draw.io/diagrams.net (zamítnuto — binární/XML formát, horší diff, závislost na externím nástroji). PlantUML (zamítnuto — menší nativní podpora renderování v cílových dokumentačních platformách oproti Mermaid). Visio (zamítnuto — proprietární formát, netextové, nevhodné pro verzování v Gitu).
- **Stav:** Přijato.
- **Zdroje:** — (odvozeno z pravidel psaní–8)

## ADR-021: Monorepo pro kód, oddělený repozitář pro katalog skillů

- **Kontext:** Komponentová dekompozice (architektonických rozhodnutí v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md)) ukazuje `apps/gateway`, `apps/ingestor` a sdílené `packages/contracts`, `packages/mcp-core`, `packages/testkit` jako těsně provázané celky se sdílenými Zod schématy. Skill katalog má naopak odlišný lifecycle, vlastníky a review proces (ADR-011, ADR-013).
- **Rozhodnutí:** Monorepo pro `apps/gateway` + `apps/ingestor` + `packages/*` (kód). Samostatný, oddělený repozitář pro skill katalog — mimo monorepo s kódem.
- **Odůvodnění:** Sdílená `packages/contracts` mezi gateway a ingestorem vyžadují atomické změny napříč oběma konzumenty — monorepo eliminuje cross-repo verzovací drift a umožňuje jeden PR, jeden CI běh pro změnu, která se dotýká obou aplikací zároveň. Skill repo má naopak zásadně odlišný governance model — review mohou dělat i doménoví vlastníci dat bez vývojářských oprávnění ke kódu gateway (viz [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) §7 CODEOWNERS) — smísení do jednoho repozitáře by komplikovalo řízení přístupových práv.
- **Důsledky:** (+) atomické změny napříč gateway/ingestor bez verzovacího pekla; (+) jasně oddělené, jednodušší access-control hranice pro skill repo; (−) dvě sady CI/CD pipeline definic a repozitářových nastavení k udržování místo jedné.
- **Zvažované alternativy:** Plný polyrepo (`gateway`, `ingestor`, `contracts` jako samostatné repozitáře) — zamítnuto, přináší verzovací drift mezi `contracts` a jeho konzumenty a komplikuje atomické změny napříč hranicí balíčku. Jeden monorepo včetně skill repa — zamítnuto, smísilo by odlišné access-control a review požadavky vývojářského týmu a doménových vlastníků dat.
- **Stav:** Přijato.
- **Zdroje:** — (odvozeno z komponentové dekompozice v [`02-architektura-reseni.md`](./02-architektura-reseni.md))

## ADR-022: IaC nástroj — Bicep s Azure Verified Modules, Terraform jako rovnocenná alternativa

- **Kontext:** Nasazení na Azure infrastrukturu vyžaduje volbu IaC nástroje pro reprodukovatelné, revidovatelné nasazení Container Apps, Cosmos DB, Key Vault, sítě a dalších zdrojů.
- **Rozhodnutí:** Doporučeno Bicep s Azure Verified Modules (AVM) jako primární cesta. Terraform je uveden jako rovnocenná alternativa, pokud organizace už provozuje Terraform pipeline pro jinou infrastrukturu a preferuje konzistenci napříč nástroji.
- **Odůvodnění:** Bicep nemá vlastní state file ke správě (ARM je zdroj pravdy), což eliminuje celou třídu state-drift problémů. AVM poskytuje oficiálně udržované, testované moduly přímo pro Container Apps s podporou managed identities, RBAC role assignments, diagnostic settings a scale settings [Z-AZ11] — nižší implementační režie než psaní vlastních modulů od nuly. Terraform zůstává rovnocennou volbou tam, kde organizace řídí multi-cloud nebo cross-team infrastrukturu jedním nástrojem — konzistence napříč týmy může v takovém kontextu převážit výhody nativní ARM integrace; AVM moduly existují paralelně i pro Terraform.
- **Důsledky:** (+) žádná state file správa u Bicep varianty; (+) AVM snižuje implementační režii a rizika chybné konfigurace RBAC/diagnostiky oproti psaní vlastních šablon; (−) tým musí zvládat buď Bicep, nebo (při volbě alternativy) Terraform HCL — dvojí standard mezi projekty organizace, pokud se různé týmy rozhodnou odlišně.
- **Zvažované alternativy:** ARM JSON přímo (zamítnuto — Bicep je jeho čitelnější nadstavba bez ztráty funkcí, není důvod psát nižší abstrakci ručně). Pulumi/CDK for Azure (zamítnuto jako primární volba — menší adopce v cílové organizaci, `⚠️ TODO — ověřit skutečné zvyklosti organizace`). Ruční konfigurace přes Azure Portal (zamítnuto — netestovatelné, nereprodukovatelné, žádná verze v Gitu).
- **Stav:** Přijato (s explicitně rovnocennou alternativou).
- **Zdroje:** [Z-AZ11], [Z-AZ12]

## ADR-023: Vztah k SEP-2640 „Skills Extension" (Skills over MCP) — sledovat, revize po Fázi 1

- **Kontext:** MCP pracovní skupina **Skills Over MCP** vyvíjí od 2026-02 standardizovaný mechanismus pro objevování a distribuci „agent skills" (strukturovaných pracovních instrukcí) přes MCP samotný [Z-SEP1]. Její aktuální směr, **SEP-2640 „Skills Extension"** (Resources-based, Extensions Track, identifikátor rozšíření `io.modelcontextprotocol/skills`), byl k datu tohoto dokumentu (2026-09-17) **finalizován a sloučen 2026-09-13** — definuje konvenci `skill://<cesta>/SKILL.md` nad existující MCP Resources primitivou, ne nové MCP metody (žádné vyhrazené `skills/list`/`skills/get`) [Z-SEP2]. Tento návrh řeší distribuci skillů vlastním mechanismem (ADR-013, viz [`05-api-a-protokoly.md` §9](./05-api-a-protokoly.md)) — YAML manifest v odděleném Git repozitáři, publikovaný `skill-ingestor` do katalogu a vystavený jako běžné MCP `tools/*`, ne jako `resources/*`.
- **Rozhodnutí:** **Nepřejímat SEP-2640 nyní.** Sledovat jeho vývoj a klientskou podporu jako trackovanou položku (viz riziko v [`01-analyza-zadani-a-plan.md`](./01-analyza-zadani-a-plan.md)), s formální revizí po Fázi 1 (pilot) — nikoli blokovat současný rozsah na jeho přijetí.
- **Odůvodnění:** SEP-2640 řeší jinou vrstvu problému, než tento návrh potřebuje vyřešit primárně. Definuje, jak klient skill **objeví a stáhne** jako resource (dokument s instrukcemi), ne jak se skill **bezpečně provede** jako akce s efektem na produkční data. ADR-013 tohoto návrhu řeší přesně to druhé — sandboxing, `riskTier`, entitlements, idempotenci a audit — nic z toho SEP-2640 sám neřeší, protože skill je v jeho pojetí vždy text/instrukce pro model, ne strukturovaný, autorizovaně volatelný nástroj s `inputSchema`/`outputSchema` a vlastním bindingem na APIM (viz [`kontrakty/skill-manifest.schema.json`](./kontrakty/skill-manifest.schema.json)). Přijetí SEP-2640 by tedy nenahradilo `adapters/skill`, nanejvýš by doplnilo způsob, jak gateway vedle stávajících `tools/*` **navíc** vystaví manifesty jako MCP resources — přídavná, ne nahrazující změna. Protože SEP-2640 byl finalizován teprve čtyři dny před datem tohoto dokumentu, klientská podpora (Claude, ChatGPT) není k dispozici ani zdokumentovaná — přijmout ho nyní by znamenalo stavět na stejném typu neověřeného předpokladu o chování klienta, jaký Gate −1 (viz [`08-provoz-observabilita-rollout.md` §9](./08-provoz-observabilita-rollout.md#9-rollout-adr-018)) u zbytku návrhu záměrně eliminuje dřív, než na něm vznikne implementační závazek.
- **Důsledky:** (+) žádná změna současného rozsahu Fáze 0–1; (+) sledovaná položka bez okamžitého architektonického závazku; (−) pokud SEP-2640 v budoucnu získá širokou klientskou podporu, katalog skillů této gateway zůstane pro klienty čitelný jen jako `tools/*`, ne přes `skill://` — potenciální budoucí duplicita mezi dvěma distribučními cestami téhož obsahu, pokud se organizace rozhodne exponovat oboje; (−) manifest schema tohoto návrhu ([`05-api-a-protokoly.md` §9.4](./05-api-a-protokoly.md), [`kontrakty/skill-manifest.schema.json`](./kontrakty/skill-manifest.schema.json)) není navržené jako přímo převeditelné na formát `skill://`/`SKILL.md` bez transformace — pokud se organizace rozhodne adaptér stavět, je to dodatečná práce, ne bezešvý přechod.
- **Zvažované alternativy:** Adoptovat SEP-2640 jako primární mechanismus místo vlastního `adapters/skill` (zamítnuto nyní — řeší jinou vrstvu problému, viz odůvodnění, a klientská podpora je neověřená čtyři dny po finalizaci SEP). Navrhnout `skill://` fasádu/adaptér nad existujícím katalogem souběžně s tímto návrhem (zváženo jako budoucí rozšíření, ne součást tohoto vydání — zařazeno jako revizní bod po Fázi 1, ne jako závazek této dokumentace).
- **Stav:** Přijato (sledovací rozhodnutí — formální revize po Fázi 1).
- **Zdroje:** [Z-SEP1], [Z-SEP2]

---

## Kompletní seznam zdrojů

### MCP specifikace

1. [Z-MCP1] MCP Versioning (2026-07-28) — https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning (přístup 2026-09-17)
2. [Z-MCP2] MCP Deprecated features (DCR → CIMD) — https://modelcontextprotocol.io/specification/2026-07-28/deprecated (přístup 2026-09-17)
3. [Z-MCP3] MCP `_meta` a OTel trace propagace — https://modelcontextprotocol.io/specification/2026-07-28/basic/index#meta (přístup 2026-09-17)
4. [Z-MCP4] MCP Streamable HTTP transport — https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http (přístup 2026-09-17)
5. [Z-MCP5] MCP Authorization specification — https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization (přístup 2026-09-17)
6. MCP Authorization Server Discovery — https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/authorization-server-discovery (přístup 2026-09-17)
7. MCP Client Registration (CIMD) — https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration (přístup 2026-09-17)
8. MCP Tool definitions — https://modelcontextprotocol.io/specification/2026-07-28/server/tools (přístup 2026-09-17)
9. MCP `server/discover` — https://modelcontextprotocol.io/specification/2026-07-28/server/discover (přístup 2026-09-17)
10. MCP Pagination — https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/pagination (přístup 2026-09-17)
11. MCP Cancellation pattern — https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/cancellation (přístup 2026-09-17)
12. MCP Progress pattern — https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/progress (přístup 2026-09-17)
13. MCP MRTR pattern — https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr (přístup 2026-09-17)
14. MCP Schema — https://modelcontextprotocol.io/specification/2026-07-28/schema (přístup 2026-09-17)
15. MCP Changelog — https://modelcontextprotocol.io/specification/2026-07-28/changelog (přístup 2026-09-17)
16. MCP Sitemap (přehled revizí) — https://modelcontextprotocol.io/sitemap.xml (přístup 2026-09-17)

### RFC a standardy

17. RFC 6750 — Bearer Token Usage — https://www.rfc-editor.org/rfc/rfc6750 (přístup 2026-09-17)
18. RFC 8414 — Authorization Server Metadata — https://www.rfc-editor.org/rfc/rfc8414 (přístup 2026-09-17)
19. RFC 7591 — Dynamic Client Registration — https://www.rfc-editor.org/rfc/rfc7591 (přístup 2026-09-17)
20. RFC 8707 — Resource Indicators for OAuth 2.0 — https://www.rfc-editor.org/rfc/rfc8707 (přístup 2026-09-17)
21. RFC 9728 — OAuth 2.0 Protected Resource Metadata — https://www.rfc-editor.org/rfc/rfc9728 (přístup 2026-09-17)
22. RFC 9207 — OAuth 2.0 Authorization Server Issuer Identification — https://www.rfc-editor.org/rfc/rfc9207 (přístup 2026-09-17)
23. RFC 8252 — OAuth 2.0 for Native Apps (loopback redirect) — https://www.rfc-editor.org/rfc/rfc8252 (přístup 2026-09-17)
24. OAuth 2.1 (draft-ietf-oauth-v2-1) — https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1 (přístup 2026-09-17)
25. OAuth Client ID Metadata Documents (draft-ietf-oauth-client-id-metadata-document) — https://datatracker.ietf.org/doc/html/draft-ietf-oauth-client-id-metadata-document (přístup 2026-09-17)
26. OpenID Connect Discovery 1.0 — https://openid.net/specs/openid-connect-discovery-1_0.html (přístup 2026-09-17)

### Microsoft Learn

27. [Z-MS1] Secure a Model Context Protocol (MCP) server with Microsoft Entra ID — https://learn.microsoft.com/entra/agent-id/secure-mcp-server-with-entra-id (přístup 2026-09-17)
28. [Z-MS2] Configure dynamic client registration — https://learn.microsoft.com/microsoft-365/copilot/extensibility/plugin-authentication-dynamic-client-registration (přístup 2026-09-17)
29. [Z-MS3] Configure OAuth 2.0 authentication — https://learn.microsoft.com/microsoft-365/copilot/extensibility/plugin-authentication-oauth (přístup 2026-09-17)
30. [Z-MS4] Register the remote Power BI MCP server with external MCP clients — https://learn.microsoft.com/power-bi/developer/mcp/remote-mcp-server-external-clients (přístup 2026-09-17)
31. [Z-MS5] Configure group claims and app roles in tokens (overage) — https://learn.microsoft.com/security/zero-trust/develop/configure-tokens-group-claims-app-roles (přístup 2026-09-17)
32. [Z-MS6] Custom RBAC for developers — app roles vs. scopes vs. groups — https://learn.microsoft.com/entra/identity-platform/custom-rbac-for-developers (přístup 2026-09-17)
33. [Z-MS7] On-Behalf-Of flow — https://learn.microsoft.com/entra/identity-platform/v2-oauth2-on-behalf-of-flow (přístup 2026-09-17)
34. [Z-AZ1] MCP server overview v API Management — https://learn.microsoft.com/azure/api-management/mcp-server-overview (přístup 2026-09-17)
35. [Z-AZ2] WAF platform comparison (Front Door vs. Application Gateway) — https://learn.microsoft.com/azure/networking/design-guide/web-application-firewall (přístup 2026-09-17)
36. [Z-AZ3] App Service scale-out limity a LB timeout — https://learn.microsoft.com/azure/app-service/manage-scale-up (přístup 2026-09-17)
37. [Z-AZ4] Azure Functions MCP tutorial — https://learn.microsoft.com/azure/azure-functions/functions-mcp-tutorial (přístup 2026-09-17)
38. [Z-AZ5] Update and deploy changes / Blue-green deployment v Container Apps — https://learn.microsoft.com/azure/container-apps/revisions (přístup 2026-09-17)
39. [Z-AZ6] Cosmos DB Change Feed — https://learn.microsoft.com/azure/cosmos-db/nosql/change-feed-modes (přístup 2026-09-17)
40. [Z-AZ7] Cosmos DB Request Units v multi-region — https://learn.microsoft.com/azure/cosmos-db/request-units (přístup 2026-09-17)
41. [Z-AZ8] Immutable storage for Azure Blob Storage — https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview (přístup 2026-09-17)
42. [Z-AZ9] Azure Monitor Logs — table plans a retence — https://learn.microsoft.com/azure/azure-monitor/logs/data-platform-logs (přístup 2026-09-17)
43. [Z-AZ10] OpenTelemetry v Azure Monitor — https://learn.microsoft.com/azure/azure-monitor/app/opentelemetry-enable (přístup 2026-09-17)
44. [Z-AZ11] Azure Verified Modules — https://azure.github.io/Azure-Verified-Modules/ (přístup 2026-09-17)
45. [Z-AZ12] Deploy Bicep přes GitHub Actions s OIDC — https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-github-actions (přístup 2026-09-17)
46. [Z-AZ13] Premium ingress for Azure Container Apps — https://learn.microsoft.com/azure/container-apps/premium-ingress (přístup 2026-09-17)
47. Secure backend services client certificate authentication (mTLS APIM→backend) — https://learn.microsoft.com/azure/api-management/api-management-howto-mutual-certificates (přístup 2026-09-17)
48. Key Vault RBAC — výchozí model — https://learn.microsoft.com/azure/key-vault/general/access-control-default (přístup 2026-09-17)
49. Key Vault jako Event Grid source (near-expiry eventy) — https://learn.microsoft.com/azure/event-grid/event-schema-key-vault (přístup 2026-09-17)
50. Set scaling rules v Container Apps (KEDA HTTP) — https://learn.microsoft.com/azure/container-apps/scale-app (přístup 2026-09-17)
51. Traffic splitting v Container Apps — https://learn.microsoft.com/azure/container-apps/traffic-splitting (přístup 2026-09-17)
52. Conditional Access policy templates — https://learn.microsoft.com/entra/identity/conditional-access/concept-conditional-access-policy-common (přístup 2026-09-17)
53. Workload identity federation (OIDC pro CI/CD) — https://learn.microsoft.com/entra/workload-id/workload-identity-federation-create-trust (přístup 2026-09-17)

### Dokumentace Anthropic

54. [Z-A1] Anthropic — MCP connector (Messages API) — https://platform.claude.com/docs/en/agents-and-tools/mcp-connector (přístup 2026-09-17)
55. [Z-A2] Anthropic — Authenticating MCP connectors — https://claude.com/docs/connectors/building/authentication (přístup 2026-09-17)
56. [Z-A3] Anthropic — Remote MCP custom connectors — https://claude.com/docs/connectors/custom/remote-mcp (přístup 2026-09-17)
57. [Z-A4] Anthropic — Authorize MCP connectors for entire organization — https://support.claude.com/en/articles/15537633-authorize-mcp-connectors-for-your-entire-organization (přístup 2026-09-17)
58. Anthropic — Use connectors to extend Claude's capabilities — https://support.claude.com/en/articles/11176164-use-connectors-to-extend-claude-s-capabilities (přístup 2026-09-17)
59. Anthropic — Get started with custom connectors — https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp (přístup 2026-09-17)
60. Anthropic — API IP addresses — https://platform.claude.com/docs/en/api/ip-addresses (přístup 2026-09-17)
61. Anthropic — MCP building overview — https://claude.com/docs/connectors/building/mcp (přístup 2026-09-17)

### Dokumentace OpenAI

62. [Z-O1] OpenAI — Developer mode and MCP apps in ChatGPT — https://help.openai.com/en/articles/12584461-developer-mode-and-mcp-apps-in-chatgpt (přístup 2026-09-17)
63. OpenAI — Developer mode guide — https://developers.openai.com/api/docs/guides/developer-mode (přístup 2026-09-17)
64. OpenAI — MCP support (search/fetch, Responses API) — https://developers.openai.com/api/docs/mcp (přístup 2026-09-17)
65. OpenAI — Tools, connectors and MCP guide — https://developers.openai.com/api/docs/guides/tools-connectors-mcp (přístup 2026-09-17)
66. OpenAI — Connectors in ChatGPT (admin controls) — https://help.openai.com/en/articles/11487775-connectors-in-chatgpt (přístup 2026-09-17)

### Ostatní

67. MCP TypeScript SDK README — https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md (přístup 2026-09-17)
68. GitHub — Securing webhooks (HMAC signature verification, obecná praxe pro GitHub Apps) — https://docs.github.com/webhooks/using-webhooks/validating-webhook-deliveries (přístup 2026-09-17)
69. [Z-AZ14] Node.js Release schedule (LTS/Maintenance/EOL data) — https://raw.githubusercontent.com/nodejs/Release/main/README.md (přístup 2026-09-17)

### SLA závislostí pro composite SLA výpočet (08 §1.1, oprava S17)

70. Azure Front Door SLA (99,99 %) — https://azure.microsoft.com/en-us/support/legal/sla/front-door/ (přístup 2026-09-17)
71. Microsoft Entra ID SLA (99,99 %) — Microsoft Entra blog, „99.99% uptime for Azure Active Directory" — https://techcommunity.microsoft.com/blog/microsoft-entra-blog/99-99-uptime-for-azure-active-directory/1999628 (přístup 2026-09-17)
72. Azure Cosmos DB SLA (99,99 % single-region) — https://azure.microsoft.com/en-us/support/legal/sla/cosmos-db/v1_0/ (přístup 2026-09-17)
73. Azure Cache for Redis SLA (99,9 % connectivity, Basic/Standard/Premium) — https://azure.microsoft.com/en-us/support/legal/sla/summary/ — `⚠️ TODO` hodnota specificky pro Azure Managed Redis nepotvrzena, použit klasický Cache for Redis jako konzervativní zástupná hodnota (viz `08` §1.1)
74. Azure API Management SLA (99,9 % baseline Basic/Standard/Premium) — https://azure.microsoft.com/en-us/support/legal/sla/summary/ (přístup 2026-09-17)
75. Reliability in Azure Container Apps (SLA závisí na zvolené konfiguraci, ne pevné číslo) — https://learn.microsoft.com/en-us/azure/reliability/reliability-container-apps (přístup 2026-09-17)

### Doplňkové zdroje (opravy po nezávislém srovnání návrhu, viz `10-srovnani-navrhu.md`)

76. [Z-CS1] NuGet Gallery — balíček `ModelContextProtocol` (oficiální MCP C# SDK, udržovaný ve spolupráci s Microsoftem), GA 2.2.0, publikováno 2026-08-13 — https://www.nuget.org/packages/ModelContextProtocol/ (přístup 2026-09-17)
77. [Z-CS2] GitHub — `modelcontextprotocol/csharp-sdk`, oficiální repozitář a release notes v2 řady (zarovnání s revizí `2026-07-28`, fallback na `2025-11-25`) — https://github.com/modelcontextprotocol/csharp-sdk (přístup 2026-09-17)
78. [Z-MCP6] MCP Specification 2025-11-25 — Streamable HTTP transport, Session Management (`Mcp-Session-Id` **MAY** být přidělen, ne MUST) — https://modelcontextprotocol.io/specification/2025-11-25/basic/transports#session-management (přístup 2026-09-17)
79. [Z-SEP1] MCP „Skills Over MCP" Working Group — charta, mise, stav prací — https://modelcontextprotocol.io/community/working-groups/skills-over-mcp (přístup 2026-09-17)
80. [Z-SEP2] SEP-2640 „Skills Extension" (Resources-based, Extensions Track) — stav Merged/Final 2026-09-13, konvence `skill://` nad MCP Resources — https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2640 (přístup 2026-09-17)
81. Node.js Release schedule (raw `schedule.json`) — přesná data LTS/Maintenance/EOL pro v24 a v26 použitá v ADR-002 — https://raw.githubusercontent.com/nodejs/Release/main/schedule.json (přístup 2026-09-17)

---

## Seznam všech ⚠️ TODO a neověřených tvrzení

Souhrn otevřených položek napříč všemi dokumenty návrhu (01–09), pokrývající jak podrobně rozpracované položky z dokumentů 07–09, tak přehled otevřených bodů ze zbylých dokumentů. Před finálním schválením návrhu doporučujeme sloučit vše do jednoho řízeného provozního backlogu.

**Podrobné položky (dokumenty 07–09):**

| # | Položka | Kde se vyskytuje | Jak ověřit | Kdo ověří |
|---|---|---|---|---|
| 1 | JWKS cache TTL a tolerance clock skew | [`07`](./07-bezpecnost-a-autorizace.md) §3.5–3.6 | Určit dle zvolené JWT knihovny v Node.js implementaci a organizační politiky | Implementační tým (backend) |
| 2 | Limity velikosti výsledku nástroje per `riskTier` | [`07`](./07-bezpecnost-a-autorizace.md) §6 | Zátěžový test s reprezentativními payloady | Platformní/QA tým |
| 3 | Redirect URI pro ChatGPT Business developer mode connector | [`07`](./07-bezpecnost-a-autorizace.md) §4.3, [`08`](./08-provoz-observabilita-rollout.md) §10 | Ověřit v aktuálním UI ChatGPT Business při zakládání connectoru | Platformní tým při onboardingu |
| 4 | Nutnost mTLS mezi gateway a APIM nad rámec OBO tokenu | [`07`](./07-bezpecnost-a-autorizace.md) §8 | Konzultace s vlastníkem APIM vrstvy a organizační politikou API governance | Architekt / vlastník APIM |
| 5 | Konkrétní compliance framework organizace (ISO 27001, CIS Azure Foundations Benchmark, jiný) | [`07`](./07-bezpecnost-a-autorizace.md) §9 | Potvrdit s bezpečnostním/compliance týmem organizace | CISO / compliance tým |
| 6 | Právní posouzení napětí mezi WORM retencí a právem na výmaz (GDPR) | [`07`](./07-bezpecnost-a-autorizace.md) §9 | Formální právní review | DPO organizace |
| 7 | Nutnost PIM just-in-time aktivace pro `Mcp.CatalogAdmin` | [`07`](./07-bezpecnost-a-autorizace.md) §4.6 | Rozhodnutí bezpečnostního týmu na základě rizikového apetitu | Bezpečnostní tým |
| 8 | RPO/RTO cíle a konkrétní DR model (aktivní/pasivní) | [`08`](./08-provoz-observabilita-rollout.md) §6 Runbook 6, otevřené otázky | Architektonické rozhodnutí navazující na architektonických rozhodnutí v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md) | Architekt (řešení) |
| 9 | Cílová hodnota SLI „úspěšnost tool volání" | [`08`](./08-provoz-observabilita-rollout.md) §1 | Stanovit dle charakteru konkrétních downstream API | Product owner |
| 10 | Vzorkovací poměr logů pro vysokoobjemové read nástroje | [`08`](./08-provoz-observabilita-rollout.md) §3 | Doladit po pilotní fázi na základě skutečného objemu | SRE tým |
| 11 | Hodnota `concurrentRequests` KEDA pravidla a kapacitní model repliky | [`08`](./08-provoz-observabilita-rollout.md) §7 | Zátěžový test | SRE/platformní tým |
| 12 | Doba čekání mezi kroky canary traffic split | [`08`](./08-provoz-observabilita-rollout.md) §8 | Potvrdit provozní zkušeností z pilotní fáze | SRE tým |
| 13 | Konkrétní ceny/náklady komponent | [`08`](./08-provoz-observabilita-rollout.md) §4, §12 (nákladová struktura po komponentách) | Azure Pricing Calculator | FinOps/platformní tým |
| 14 | Konkrétní kontakty a eskalační matice | [`08`](./08-provoz-observabilita-rollout.md) §11 | Doplnit dle organizační směrnice | IT management |
| 15 | Cílový podíl aktivních pilotních uživatelů pro Gate 1 | [`08`](./08-provoz-observabilita-rollout.md) §9 | Stanovit s produktovým vlastníkem před zahájením pilotu | Product owner |
| 16 | Skutečná preference organizace mezi Pulumi/CDK a Bicep/Terraform | [`09`](./09-adr-a-zdroje.md) ADR-022 | Ověřit zvyklosti platformního týmu organizace | Platformní tým |
| 17 | Konkrétní délka trvání Fáze 0 (PoC) | [`08`](./08-provoz-observabilita-rollout.md) §9 | Potvrdit s projektovým vedením | Projektový vlastník |
| 18 | Cenový dopad Container Apps Premium ingress módu | ADR-004 výše, [`08`](./08-provoz-observabilita-rollout.md) §7, §8, §4, §12 | Azure Pricing Calculator | FinOps/platformní tým |
| 19 | Konec podpory `@modelcontextprotocol/sdk` v1.x (řádově 1. pol. 2027) a rozhodnutí o dalším postupu adaptéru éry A | ADR-002 výše, [`01`](./01-analyza-zadani-a-plan.md) riziko R15 | Sledovat oficiální oznámení konce podpory v1.x linie; revize dle plánu (bod P2.9 v `01`) | Tech lead gateway |
| 20 | Potvrzení výchozích hodnot `auditFailureMode` (`failOpen`/`failClosed` dle typu nástroje) compliance týmem | [`08`](./08-provoz-observabilita-rollout.md) §1, §6 Runbook 7 | Formální compliance review | Compliance tým |
| 21 | Realistická dostupnost gateway (composite SLA) proti naivní násobné kompozici 99,77 % | [`08`](./08-provoz-observabilita-rollout.md) §1.1 | Game-day test simulující výpadek každé závislosti zvlášť, před Gate 1 | SRE/platformní tým |
| 22 | Přesné SLA pro zvolenou konfiguraci Container Apps (zone-redundant, min. 2 instance) a pro Azure Managed Redis konkrétně | [`08`](./08-provoz-observabilita-rollout.md) §1.1 | Azure dokumentace/support v době nasazení | Platformní tým |
| 23 | Parametry exponenciálního backoff (počáteční interval, násobitel) pro retry `auditChainCursor` konfliktu | [`04`](./04-datova-architektura.md) §6.3 | Zátěžový test před Gate 1 | Implementační tým (backend) |
| 24 | Schválení bezpečnostním architektem: přijatelný zbytkový rozsah dat pro nástroje s `openWorldHint: true` a proces schvalování výjimek z preventivní kontroly proti exfiltraci | [`07`](./07-bezpecnost-a-autorizace.md) §2 (hrozba #4), §6 | Rozhodnutí sponzora/bezpečnostního architekta před nasazením nad firemní data | Bezpečnostní architekt |
| 25 | Chování rate limiteru při výpadku Redis — **rozhodnuto**: fail-closed, per-replika kvóta `globalLimit / maxReplicas`, přepínač `rateLimitFallbackMode` (výchozí `strict`); | viz [`04`](./04-datova-architektura.md) §5.2 a [`08`](./08-provoz-observabilita-rollout.md) §7.1 | Zbývá potvrdit hodnotu `maxReplicas` a ověřit chování zátěžovým testem s vypnutým Redis před Gate 1 | Bezpečnostní architekt + provozní tým |
| 26 | Délka trvání Gate −1 (interoperabilní spike) a přesné datum/tenant pro testovací Entra app registration | [`08`](./08-provoz-observabilita-rollout.md) §9 | Potvrdit s projektovým vedením a IAM týmem před zahájením projektu | Projektový vlastník / IAM tým |
| 27 | Limity velikosti výsledku nástroje per `riskTier` (duplicitně s položkou 2) — potřebné i pro dokončení `kontrakty/audit-event.schema.json` pole `result.sizeBytes` jako měřitelný strop, ne jen popisné pole | [`07`](./07-bezpecnost-a-autorizace.md) §6, [`kontrakty/audit-event.schema.json`](./kontrakty/audit-event.schema.json) | Zátěžový test s reprezentativními payloady | Platformní/QA tým |
| 28 | Revize klientské podpory SEP-2640 „Skills Extension" po Fázi 1 a rozhodnutí o adaptéru | ADR-023 výše | Sledovat MCP Skills Over MCP WG a klientskou dokumentaci Anthropic/OpenAI | Tech lead gateway |

**Doplňkové položky z dokumentů 01–06** (přehledově, podrobnosti v příslušných dokumentech):

| Dokument | Otevřené položky |
|---|---|
| `02` | RU model Cosmos DB (provisioned/serverless) a Redis SKU dle A09/A10; geo-redundance WORM blobu v multi-region; konkrétní RPO/RTO; dosažitelnost APIM přes Private Link; NAT Gateway vs. Azure Firewall pro egress |
| `03` | Ověření Redis klienta proti Azure Managed Redis; Front Door origin timeout; hodnoty KEDA/kapacitního modelu (viz i `08` §7); cílové skóre „blind" testu popisů nástrojů; volba tunelovací služby pro test proti reálnému Claude konektoru |
| `04` | Vstupní kapacitní předpoklady (uživatelé, tool calls/den); TTL/GC archivovaných verzí katalogu; TTL JWKS cache (viz i `07`); právní posouzení crypto-shreddingu; potvrzení retenčních lhůt compliance; retence `toolUsageStats`; přesnost/měření Cosmos PITR a WORM redundance |
| `05` | Náklad na tokenovou cenu katalogu při větším počtu nástrojů (viz i S1 v review) |
| `06` | Minimální doba pilotu a prahové hodnoty Gate 2 v dnech/metrikách; přesná délka platnosti Entra access tokenu; pokrytí Continuous Access Evaluation (CAE) pro app registration |
| `07` | Viz podrobná tabulka výše (položky 1–7) |

---

## Otevřené otázky

1. Sloučení TODO seznamů ze všech deseti dokumentů (00–09) do jednoho řízeného provozního backlogu je vhodný krok před finálním schválením návrhu.
2. Zda organizace preferuje Bicep nebo Terraform (ADR-022) by měla být potvrzena před zahájením implementace IaC vrstvy, ne až v jejím průběhu.
3. Rozsah a přesné znění ADR-019/020/021/022 může vyžadovat revizi ze strany vedoucího architekta projektu, protože jde o doplňky nad rámec původního zadání, ne jeho doslovný přepis.

---

## Zdroje

Viz sekce „Kompletní seznam zdrojů" výše — číslované položky 1–75 pokrývají všechny citace použité v tomto dokumentu i v [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) a [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md).
