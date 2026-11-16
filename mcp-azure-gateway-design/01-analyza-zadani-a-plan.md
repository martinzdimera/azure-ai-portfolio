# 01 — Analýza zadání a plán prací

Verze 1.0 · 2026-09-17 · Stav: Návrh · Vztahuje se k: ADR celého návrhu, zejména ADR-001, ADR-003, ADR-005, ADR-006, ADR-018

---

## 1. Účel a rozsah dokumentu

Dokument rozebírá zadání projektu MCP-Azure Gateway větu po větě, odděluje explicitní požadavky od implicitních předpokladů, jmenuje otevřené otázky pro zadavatele a dává je do vztahu s riziky a plánem prací. Nerozhoduje o architektuře — tu stanovují ADR v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md) a rozvádí `./02-architektura-reseni.md`. Tento dokument se soustředí na analýzu zadání a plán prací (kapitola `01`); architektonické řešení samotné je rozvedeno v `02` a navazujících dokumentech `03`–`09` a v diagramech `01`–`22`, na které se dokument dále odkazuje.

---

## 2. Rozbor zadání věta po větě

Tabulka odděluje, co zadání říká přímo, od toho, co je nutné doplnit vlastní interpretací.

| # | Věta / formulace zadání | Explicitně požadováno | Implicitně předpokládáno |
|---|---|---|---|
| 1 | „produkční MCP gateway — jeden connector endpoint, na který ukáží workspacy Claude i ChatGPT Business" | Jeden veřejně adresovatelný endpoint, sdílený oběma platformami | Že obě platformy umí ukázat na stejnou URL (liší se transportem — viz `mcp-spec.md` §4–5); že „workspace" znamená administrátorem spravovaný custom connector / Developer mode app, ne osobní připojení jednotlivce |
| 2 | „Production-grade MCP server" | Produkční kvalita — dostupnost, škálování, zabezpečení | Definice „production-grade" není v zadání kvantifikována (SLO, RTO/RPO) — návrhové hodnoty dopočítává tento dokument a [`08`](./08-provoz-observabilita-rollout.md) |
| 3 | „Autentizace + per-role autorizace — jen oprávnění uživatelé smí volat konkrétní nástroje" | Autentizace + autorizace na úrovni nástroje (tool-level), ne jen na úrovni endpointu | Existence rolí/skupin v Entra ID, které lze na nástroje mapovat (ADR-007); že „uživatel" = identita protékající až do gateway, ne jen identita klientské aplikace |
| 4 | „Adapter layer se dvěma zdroji: skills ← GitHub, connectors → APIM" | Dva typy nástrojů s odlišným zdrojem pravdy a odlišným prováděcím mechanismem | Že GitHub repozitář se skilly už existuje nebo bude založen v rámci projektu; že APIM v organizaci již existuje a je použitelné (viz předpoklad A01) |
| 5 | „Strukturované logování + audit trail každého tool invocation: kdo, co, kdy, výsledek" | Čtyři povinná pole auditu (identita, akce, čas, výsledek) u každého volání | Požadavek na neměnnost/tamper-evidence není v zadání explicitní — ADR-010 jej doplňuje jako architektonické rozhodnutí, ne jako doslovný požadavek zadání |
| 6 | „Deployment na existující Azure infrastruktuře — TLS, scaling, monitoring" | Nasazení do Azure, ne do jiného cloudu; TLS, autoscaling a monitoring jako povinné vlastnosti | Rozsah „existující infrastruktury" (VNet? APIM? Log Analytics workspace?) není specifikován — viz A02–A07 |
| 7 | „Fázový rollout: lokální PoC → pilot → celofiremně" | Tři fáze nasazení, ne big-bang | Kritéria přechodu mezi fázemi (gate) nejsou v zadání — ADR-018 je dopočítává |
| 8 | Deliverables: PoC, produkční gateway, konfigurační návod, handover dokumentace | Čtyři konkrétní výstupy s jasnou definicí „hotovo" (viz kap. 7) | Že „handover pro interní IT" znamená provoz gateway přebírá interní tým, ne externí MSP (viz A13) |

---

## 3. Tabulka předpokladů (Assumption register)

| ID | Předpoklad | Dopad, pokud neplatí | Jak ověřit | Kdo potvrdí |
|---|---|---|---|---|
| A01 | Existující APIM instance je na tieru Developer/Basic/Standard/Premium (ne Consumption) — Consumption tier nepodporuje MCP-related funkce APIM ani řadu policy scope [Zdroj 4] | Nutná migrace APIM tieru nebo přepracování ConnectorAdapter na jiný způsob volání backendů | `az apim show` / dotaz na vlastníka APIM | Vlastník/platform team APIM |
| A02 | Tým řešení má v Entra ID tenantu práva vytvořit app registration s vlastním Application ID URI na ověřené custom doméně (ne `*.azurewebsites.net`) [Zdroj 3] | Blokuje ADR-005/ADR-006; nutná eskalace na IAM tým, zpoždění řádu týdnů | Ticket na IAM tým, ověření role (Application Administrator / Cloud Application Administrator) | IAM / Identity tým |
| A03 | Existuje VNet s volným adresním prostorem min. `/22` bez kolize s existujícím IP plánem organizace | Nutná realokace adresního prostoru nebo peering do jiné VNet, zdržení fáze 1 | Konzultace se síťovým týmem, IPAM evidence | Síťový tým |
| A04 | Vlastníkem GitHub skill repozitáře je interní tým se schopností nastavit branch protection, required review a GitHub App instalaci (ADR-011) | Bez branch protection nelze vynutit vstupní podmínku publikace katalogu — bezpečnostní mezera | Potvrzení vlastnictví repa a admin práv | Vlastník skill repozitáře |
| A05 | Klasifikace dat, která nástroje (connectors) vracejí, je nanejvýš „interní" — bez zvláštní regulace (zdravotní, finanční, osobní údaje ve velkém objemu) | Nižší retence/šifrování než vyžaduje skutečná klasifikace; možné porušení compliance | Bezpečnostní/DPO review katalogu nástrojů před GA | Bezpečnostní tým / DPO |
| A06 | Síťová izolace (Private Link pro Cosmos/Redis/KeyVault/Storage, žádný public endpoint) je organizací vyžadována jako standard, i když to zadání explicitně neříká | Pokud organizace preferuje jednodušší (a levnější) public-endpoint model s IP allowlist, návrh je předimenzovaný | Potvrzení bezpečnostní politiky organizace | Bezpečnostní architekt / CISO zástupce |
| A07 | Azure Front Door (Standard/Premium) v organizaci ještě neexistuje a bude zřízen nově pro tento projekt | Pokud Front Door již existuje pro jiné účely, je nutné posoudit sdílení vs. dedikovaná instance | Dotaz na platform/network tým, `az afd profile list` | Platform team |
| A08 | Licence ChatGPT Business zahrnuje Developer mode (custom MCP apps, read/write) s možností přiřadit přístup pilotní skupině [Zdroj 6] | Bez Developer mode nelze napojit write nástroje, jen starší „Connectors" mechanismus (read-only, `search`/`fetch`) — nutná revize scope pro ChatGPT | Ověření u OpenAI admin konzole organizace | Administrátor ChatGPT Business workspace |
| A09 | Celkový cílový počet uživatelů po Fázi 2 je řádově stovky (ne tisíce) — ovlivňuje RU/s dimenzování Cosmos DB a replika count ACA | Při výrazně vyšším počtu je nutné přepočítat kapacitní plán a náklady | Odpověď zadavatele (viz G7) | Sponzor projektu |
| A10 | Počet nástrojů (tools) v katalogu při GA je řádově desítky, ne stovky | Ovlivňuje návrh paginace `tools/list`, cache TTL a UX výběru nástroje v klientech | Odpověď zadavatele (viz G7) | Sponzor projektu / vlastníci API |
| A11 | Požadovaná retence auditu odpovídá návrhu v [`04`](./04-datova-architektura.md) (Log Analytics 90 dní → archiv 2 roky, WORM blob 7 let) jako konzervativnímu výchozímu nastavení, ne specifickému regulatornímu požadavku | Pokud existuje odvětvová regulace s jinou retencí (např. finanční sektor), je nutná úprava immutability policy před prvním zápisem (retenci lze jen prodloužit, ne zkrátit — WORM char.) [Zdroj 8] | Compliance review | Compliance / právní oddělení |
| A12 | Neexistuje specifický compliance rámec (ISO 27001, SOC2, NIS2…), který by kladl dodatečné požadavky nad rámec interní bezpečnostní politiky | Chybějící kontroly (např. formální risk assessment, penetrační test s definovaným rozsahem) by musely být doplněny | Dotaz na compliance/security tým | Compliance tým |
| A13 | Provoz produkčního prostředí po handoveru přebírá interní IT tým (ne externí MSP) v režimu pracovní doby, ne 24/7 on-call | Bez 24/7 on-call neodpovídá SLO 99,9 % běžné definici; nutná úprava očekávání nebo posílení alertingu/auto-remediace | Odpověď zadavatele (viz G11) | IT manažer |
| A14 | Dostupná CI/CD platforma je GitHub Actions s možností nastavit OIDC federaci na Entra ID (workload identity federation) [Zdroj 7] | Pokud organizace vyžaduje Azure DevOps, je nutné přepracovat pipeline definice (funkčně ekvivalentní, ale jiný nástroj) | Potvrzení standardu CI/CD v organizaci | Platform/DevOps tým |
| A15 | Cílové SLO 99,9 % měsíčně a p95 latence dle tento návrh jsou návrhové hodnoty přijatelné pro zadavatele, ne závazné smluvní SLA vůči třetí straně | Přísnější SLA (např. 99,95 %) vyžaduje multi-AZ i pro Redis/Cosmos v single-region a případně earlier multi-region | Odpověď zadavatele (viz G10) | Sponzor projektu |
| A16 | Rozpočet na Azure zdroje není shora omezen; návrh cílí na minimální produkční konfiguraci splňující ADR, ne na nejlevnější možnou variantu | Při tvrdém rozpočtovém stropu je nutné revidovat SKU (např. Cosmos DB provisioned vs. serverless, Redis tier) | Odpověď zadavatele (viz G14) | Sponzor projektu / FinOps |
| A17 | Existující APIM vrstva už má vlastní autentizaci/autorizaci downstream API a governance (subscription keys, produkty) — gateway se na ni spoléhá, neduplikuje ji | Pokud APIM nemá žádnou vlastní ochranu, OBO token od gateway je jediná ochranná vrstva a je nutné posílit validaci na straně APIM | Review existujících APIM policies | Vlastník APIM |
| A18 | Claude i ChatGPT budou připojeny jako staticky registrovaní klienti (manuální `client_id`/`client_secret`), v souladu s ADR-006 — organizace akceptuje provozní zátěž ruční správy a rotace těchto credentials | Bez akceptace je nutné hledat alternativu (CIMD hosting gateway pro sebe jako klienta vůči Entra není řešením — Entra CIMD nepodporuje, jen MCP klient-strana) | Potvrzení procesu správy client secrets (Key Vault rotace, kdo ji provádí) | IT operátor / Identity engineer |

---

## 4. Gap-list — otevřené otázky pro zadavatele

Otázky jsou seřazené podle blokujícího dopadu na architekturu a plán. U každé je uvedena výchozí odpověď, se kterou návrh pracuje, pokud zadavatel neodpoví.

| Priorita | Otázka | Výchozí odpověď, pokud nepotvrzeno |
|---|---|---|
| **BLOKUJÍCÍ** | Jaký je SKU existující APIM instance? Je Consumption tier vyloučen? | Počítáme s tierem Standard v2 nebo vyšším; Consumption tier je z návrhu vyloučen (A01) |
| **BLOKUJÍCÍ** | Existuje VNet, do které má gateway zapadnout (peering na hub), nebo se buduje samostatná síť? Jaký je volný adresní prostor? | Buduje se nová dedikovaná VNet `10.60.0.0/22` (návrh) v novém resource groupu, peering řeší síťový tým dodatečně (A03) |
| **BLOKUJÍCÍ** | Kdo v Entra ID tenantu má práva zřídit app registration s vlastním Application ID URI a přiřazovat app roles? | Žádost jde na IAM tým s časovou rezervou min. 2 týdny před začátkem Fáze 1 (A02) |
| **BLOKUJÍCÍ** | Je síťová izolace (Private Link, žádný public endpoint pro Cosmos/Redis/KeyVault/Storage) povinná, nebo je přípustný public endpoint s IP allowlistem? | Ano, Private Link je povinný — bezpečnostní výchozí nastavení ADR-003 (A06) |
| **VYSOKÝ** | Jaká je klasifikace dat, která nástroje vracejí (osobní údaje, finanční data, obchodní tajemství)? | Klasifikace „interní", bez zvláštní regulace; audit retence zůstává konzervativně 7 let (A05, A11) |
| **VYSOKÝ** | Existuje specifický compliance rámec (ISO 27001, SOC2, NIS2, odvětvová regulace)? | Žádný specifický rámec nad rámec interní bezpečnostní politiky (A12) |
| **VYSOKÝ** | Kolik nástrojů (tools) a kolik downstream API v APIM se plánuje připojit do konce Fáze 2? Kolik uživatelů celkem? | Řádově desítky nástrojů, řádově stovky uživatelů (A09, A10) |
| **VYSOKÝ** | Zahrnuje licence ChatGPT Business Developer mode (custom MCP apps, read/write) pro pilotní skupinu? | Ano, administrátor aktivuje Developer mode pro pilotní skupinu (A08) |
| **STŘEDNÍ** | Kdo je formálním vlastníkem GitHub skill repozitáře a má autoritu nastavit branch protection? | Vlastníkem je tým platformy/gateway, branch protection aktivní od Fáze 0 (A04) |
| **STŘEDNÍ** | Jaké SLO a error-budget politiku organizace očekává? | 99,9 % měsíčně dle tento návrh, bez formálního error-budget procesu v Fázi 1 (A15) |
| **STŘEDNÍ** | Provozuje produkci interní IT samostatně, v jakém on-call režimu (business hours vs. 24/7)? | Interní IT, business hours v Fázi 1, rozšíření na 24/7 posouzeno pro Fázi 2 dle kritičnosti nástrojů (A13) |
| **STŘEDNÍ** | Existuje standardizovaná CI/CD platforma (GitHub Actions vs. Azure DevOps)? | GitHub Actions s OIDC federací na Entra (A14) |
| **NÍZKÝ** | Je požadována multi-region DR (aktivní/pasivní), nebo stačí single-region se zálohami? | Single-region ve Fázích 0–1, multi-region DR posouzeno jako rozšíření Fáze 2 (viz `./02-architektura-reseni.md#8-multi-region--dr-pohled`) |
| **NÍZKÝ** | Je stanoven rozpočtový strop na Azure zdroje? | Žádný známý strop; ceny dopočítá Azure Pricing Calculator (⚠️ TODO — ověřit), návrh cílí na minimální produkční konfiguraci (A16) |

---

## 5. Rizika

| ID | Riziko | Pravděpodobnost | Dopad | Mitigace | Vlastník |
|---|---|---|---|---|---|
| R01 | Rozpolcenost MCP spec na dvě éry (2025-11-25 vs. 2026-07-28) a nejistá/neověřená migrace Claude i ChatGPT na novou éru [Zdroj 1] | Střední | Střední | Dual-era adaptér (ADR-001), kontraktní testy proti oběma erám v CI, monitoring `MCP-Protocol-Version` v provozu jako signál migrace klientů | Tech lead gateway |
| R02 | Entra ID nepodporuje Dynamic Client Registration (RFC 7591) [Zdroj 3] — statická registrace klientů roste s počtem workspace/connectorů bez formálního procesu | Vysoká | Střední | Standardizovaný onboarding runbook, rotace client secret přes Key Vault, alerting na blížící se expiraci | IT operátor |
| R03 | Prompt injection přes výsledky nástrojů (škodlivý obsah v tool response manipuluje model k neoprávněné následné akci) | Vysoká | Vysoký | Přísný `outputSchema`, redakce/sanitizace v `execution/`, `readOnlyHint`/`destructiveHint` jako závazné anotace, žádné řetězení akcí bez explicitního `confirm` (ADR-016) | Bezpečnostní architekt |
| R04 | Confused deputy — nesprávné předání identity při OBO způsobí, že downstream API provede akci jménem jiného principal, než volal nástroj | Nízká | Vysoký | ADR-008: zákaz token passthrough, audience-scoped tokeny (RFC 8707), izolační testy mezi tenanty/workspaci | Identity engineer |
| R05 | Exfiltrace dat přes nástroj — read-only nástroj vrací objem/rozsah dat mimo scope role volajícího | Střední | Vysoký | Parametrové constrainty a data scope v `authz/entitlements`, audit + alerting na anomální objem výsledku, `riskTier` na nástroji | Bezpečnostní architekt |
| R06 | Škodlivý nebo chybný skill commitnutý do repozitáře (pokus o RCE, nebezpečný cílový endpoint) | Nízká | Vysoký | ADR-013 sandbox (jen `http`/`transform`, žádný uživatelský kód v gateway), validace v `ingestor/validate`, branch protection + povinný review (A04) | Správce katalogu |
| R07 | Nedostatečný rate limiting způsobí, že retry storm od agentů přetíží downstream produkční API v APIM | Střední | Vysoký | Třívrstvý rate limiting (ADR-017), circuit breaker na `adapters/connector`, idempotence cache (ADR-016) | Platform ops |
| R08 | Jeden gateway endpoint jako SPOF pro obě platformy (Claude i ChatGPT) | Nízká | Vysoký | Min. 2 replicas, zónová redundance, health probes, autoscaling; DR plán viz `./02-architektura-reseni.md#8-multi-region--dr-pohled` | Platform ops |
| R09 | Shadow IT — týmy obcházejí gateway a volají APIM/backend přímo, čímž obchází audit i autorizaci | Střední | Střední | APIM policy vyžadující claim/hlavičku prokazující původ z gateway, governance komunikace, postupné omezení přímého přístupu k citlivějším operacím | Sponzor projektu |
| R10 | Katalog drift — aktivní `CatalogVersion` neodpovídá poslednímu Git SHA kvůli ztracenému webhooku | Střední | Střední | Periodický reconcile 15 min jako pojistka (ADR-011), alert na stáří katalogu vs. Git SHA | Platform ops |
| R11 | Selhání refreshe OBO tokenu uprostřed dlouho běžící streaming operace éry A přeruší tool call | Střední | Střední | Proaktivní refresh před expirací, token cache s TTL, retry s novým JSON-RPC id | Identity engineer |
| R12 | Nedostatečná kapacita downstream APIM/backend API na zvýšený objem po go-live (thundering herd) | Střední | Vysoký | Kapacitní test před Gate 1/Gate 2, postupné otevírání write nástrojů dle ADR-018 | Vlastník APIM |
| R13 | Interní IT nemá provozní zkušenost s Container Apps/KEDA/Cosmos DB a handover dokumentace nestačí k samostatnému provozu | Střední | Střední | Runbooky, shadowing v průběhu Fáze 1, formální školení před Gate 1 | Tech lead gateway / IT manažer |
| R14 | Groups claim overage (>200 skupin u JWT tokenů) [Zdroj 3] způsobí chybějící claim a nesprávné autorizační rozhodnutí u uživatelů s mnoha skupinami | Nízká | Střední | Primárním autorizačním mechanismem jsou app roles, ne groups claim (ADR-007); test se syntetickým uživatelem s vysokým počtem skupin | Identity engineer |
| R15 | Konec podpory `@modelcontextprotocol/sdk` v1.x (éra A) je deklarován na minimálně 6 měsíců po vydání v2 linie (`@modelcontextprotocol/server`/`client`/`fastify` 2.0.0, vydáno 2026-07-27), tedy řádově v 1. pololetí 2027 — viz ADR-002 v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md). Pokud do té doby Claude ani ChatGPT nepřejdou plně na éru B, gateway drží adaptér éry A na neudržovaném SDK | Střední | Střední | Sledovat oznámení konce podpory v1.x SDK jako trigger; do plánu zařazena revize (viz kapitola 6, bod P2.9); alternativa — vlastní tenký adaptér éry A nezávislý na `@modelcontextprotocol/sdk`, pokud migrace klientů na éru B nestihne termín | Tech lead gateway |
| R16 | „Skills over MCP" (SEP-2640, MCP Skills Extension) formalizuje distribuci skillů přes `skill://` Resources jako standardní součást protokolu — mechanismus, který se svým účelem překrývá s vlastním katalogem/skill manifestem tohoto návrhu (ADR-013). Rozšíření bylo sloučeno/finalizováno 2026-09-13 [Z-SEP2], čtyři dny před datem tohoto dokumentu — nejde tedy o vzdálenou budoucnost, ale o právě ustálený standard, jehož podporu klienti mohou začít nasazovat v horizontu měsíců, ne let. Pokud Claude/ChatGPT přijmou SEP-2640 jako preferovaný způsob distribuce skillů, vlastní katalogový mechanismus tohoto návrhu riskuje, že se stane paralelní, nestandardní cestou vedle nativně podporované — dvojí údržba bez jasného vítěze (doplněno po nezávislém srovnání návrhu, viz [`10-srovnani-navrhu.md`](./10-srovnani-navrhu.md) §7 a ADR-023 v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md)) | Nízká–Střední (nová specifikace, přijetí klienty zatím neověřeno) | Střední | Sledovat (watch), ne implementovat předčasně — revize po Fázi 1 (viz ADR-023); do plánu zařazena revize podpory klientů (kapitola 6, bod P2.10); alternativa při přijetí klienty — tenký adaptér mapující `skill://` Resources na existující katalog místo paralelní implementace | Tech lead gateway |

---

## 6. Plán prací (Work Breakdown Structure)

Odhady pracnosti jsou hrubé, **±50 %**, v člověkodnech jednoho seniorního full-stack/platform inženýra; nezahrnují PM rezervu ani čekací dobu na externí týmy (IAM, síť, APIM) z kapitoly 4.

### Fáze −1 — interoperabilní spike (doplněno po nezávislém srovnání návrhu, viz [`10-srovnani-navrhu.md`](./10-srovnani-navrhu.md) §6/§8 a ADR-018 v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md))

Předchází Fázi 0 — ověřuje předpoklady o chování reálných klientů (protokolová revize, OAuth bez DCR, skutečné timeouty), na kterých staví architektura jako celek, dřív, než na nich vznikne cokoli dalšího. Podrobný popis rozsahu a exit kritérií je v [`08-provoz-observabilita-rollout.md` §9](./08-provoz-observabilita-rollout.md#9-rollout-adr-018).

| Balík | Obsah | Odhad (dny, ±50 %) |
|---|---|---|
| PM1.1 | Nejmenší možný MCP server s jedním triviálním nástrojem, nasazený na veřejnou HTTPS URL | 2 |
| PM1.2 | OAuth 2.1 Resource Server flow proti reálnému Entra ID tenantu, bez DCR (RFC 9728 PRM) | 1 |
| PM1.3 | Připojení a test proti skutečnému Claude workspace | 1 |
| PM1.4 | Připojení a test proti skutečnému ChatGPT Business workspace, měření timeoutů a chování při přerušení spojení u obou klientů, zápis zjištění a rozhodnutí Gate −1 | 1 |
| **Celkem Fáze −1** | | **≈ 5 dní** |

**Gate −1 — exit kritéria:** viz [`08-provoz-observabilita-rollout.md` §9](./08-provoz-observabilita-rollout.md#9-rollout-adr-018) („Fáze −1 — Interoperabilní spike"). Stručně: triviální nástroj úspěšně zavolán z obou reálných klientů, OAuth bez DCR funkční u obou, naměřené timeouty/chování zapsané a porovnané s hodnotami v `03` §10.1. Selhání zásadního předpokladu (ne drobná kalibrace) vede k přehodnocení návrhu, ne jen k opakování fáze se stejnou architekturou.

### Fáze 0 — lokální PoC (stdio + docker compose, mock APIM, fake katalog)

| Balík | Obsah | Odhad (dny, ±50 %) |
|---|---|---|
| P0.1 | Bootstrapping repa, `packages/contracts`, `mcp-core` skeleton | 3 |
| P0.2 | `protocol/` — legacy handshake adaptér + základní Fastify `edge/` | 5 |
| P0.3 | `identity/` — mock JWT validace (bez reálného Entra) | 3 |
| P0.4 | `catalog/` — in-memory fake katalog, `tools/list` | 3 |
| P0.5 | `adapters/skill` — minimální manifest → tool, deklarativní transform | 4 |
| P0.6 | `adapters/connector` — mock APIM přes docker-compose stub | 3 |
| P0.7 | `execution/` — základní orchestrace bez idempotence | 3 |
| P0.8 | `audit/` — log do stdout, bez hash chain | 2 |
| P0.9 | docker-compose demo + smoke test proti MCP Inspector | 3 |
| P0.10 | Příprava Gate 0 (demo, README) | 2 |
| **Celkem Fáze 0** | | **≈ 31 dní** |

**Gate 0 — exit kritéria:** funkční `tools/list` a `tools/call` přes stdio i lokální HTTP; úspěšný test proti MCP Inspector (verze doporučená v `azure-platform.md` §1); min. 1 skill a 1 connector nástroj end-to-end; bezpečnostní checklist potvrzující, že `adapters/skill` nespouští žádný uživatelský kód (ADR-013).

### Fáze 1 — pilot (≤ 25 uživatelů, Entra security group `MCP-Gateway-Pilot`, read-only tool set)

| Balík | Obsah | Odhad (dny, ±50 %) |
|---|---|---|
| P1.1 | Reálná Entra ID integrace — app registration, PRM endpoint, JWT validace přes JWKS | 8 |
| P1.2 | `authz/` — PolicyEngine deny-by-default, mapování app roles → tool sets | 8 |
| P1.3 | `catalog/` — Cosmos DB store, change feed invalidace cache | 8 |
| P1.4 | Azure Managed Redis — session éry A, idempotence, rate-limit countery | 6 |
| P1.5 | `adapters/connector` — reálné OBO volání na APIM (jen read-only operace) | 8 |
| P1.6 | `audit/` — dual-sink hash chain (Log Analytics DCR + WORM blob) | 8 |
| P1.7 | Azure IaC (Bicep/AVM): VNet, ACA environment, Private Endpoints, Front Door Premium | 10 |
| P1.8 | CI/CD (GitHub Actions, OIDC federace, revisions blue/green) | 6 |
| P1.9 | `telemetry/` — OTel → App Insights, dashboardy, alerting | 5 |
| P1.10 | `skill-ingestor` job — GitHub App webhook + 15min reconcile | 6 |
| P1.11 | Bezpečnostní review + základní penetrační test | 5 |
| P1.12 | Pilotní onboarding (25 uživatelů), konfigurační návod pro Claude/ChatGPT | 4 |
| P1.13 | Příprava a vyhodnocení Gate 1 | 3 |
| **Celkem Fáze 1** | | **≈ 85 dní** |

**Gate 1 — exit kritéria:** SLO dle tento návrh dodrženo min. 2 týdny pilotního provozu; 0 kritických bezpečnostních nálezů; audit trail kompletní a ověřitelný (hash chain bez mezery, denní checkpoint sedí); rollback katalogu úspěšně otestován na produkčních datech pilotu; ≥ 90 % pilotních uživatelů provedlo min. 1 úspěšný tool call.

### Fáze 2 — celofiremně (write nástroje postupně, per-role)

| Balík | Obsah | Odhad (dny, ±50 %) |
|---|---|---|
| P2.1 | Write nástroje — vynucení idempotence, `confirm` flow (ADR-016) | 6 |
| P2.2 | Rozšíření entitlements na další role/oddělení | 5 |
| P2.3 | Kapacitní testy a tuning autoscalingu (KEDA, min/max replicas) | 5 |
| P2.4 | Rozšíření na další interní workspace, pokud relevantní (ADR-015) | 5 |
| P2.5 | Posouzení a případná implementace multi-region DR (rozsah závisí na odpovědi na G13) | 10–25 ⚠️ |
| P2.6 | Handover dokumentace pro interní IT (runbooky, on-call) | 6 |
| P2.7 | Postupné rollout vlny dle rolí + monitoring `denied` poměru | 8 |
| P2.8 | Steady-state review a retrospektiva | 2 |
| P2.9 | Revize stavu migrace klientů na éru B a rozhodnutí o dalším postupu adaptéru éry A před koncem podpory `@modelcontextprotocol/sdk` v1.x (řádově 1. pololetí 2027, viz riziko R15 a ADR-002 v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md)) | 2 |
| P2.10 | Revize přijetí SEP-2640 („Skills over MCP") klienty a rozhodnutí watch/adaptér/čekat (riziko R16, ADR-023 v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md)) | 1 |
| **Celkem Fáze 2** | | **≈ 50–65 dní** |

**Gate 2 (Steady state) — exit kritéria:** write nástroje v provozu bez nárůstu incidentů nad baseline pilotu; handover dokumentace potvrzena interním IT (proškolení proběhlo); DR plán zdokumentován a alespoň jednou tabletop-testován.

**Celkový hrubý odhad Fáze −1 až 2:** součet jednotlivých balíků (P50, „pravděpodobný" scénář) je ≈ **170–185 člověkodnů** (Fáze −1 ≈ 5 + Fáze 0 ≈ 31 + Fáze 1 ≈ 85 + Fáze 2 ≈ 50–65). Aplikovat ±50 % na tento součet jako jediné číslo je metodicky zavádějící — chyby jednotlivých nezávislých balíků se částečně vzájemně kompenzují, takže rozpětí faktoru 3,3× není použitelné pro rozpočtování. Místo toho tři orientační scénáře:

| Scénář | Odhad (člověkodny) | Předpoklad |
|---|---|---|
| Optimistický | ≈ 145–155 | Žádné externí blokace (IAM, síť, APIM) z kapitoly 4, minimální rework, Gate −1 projde napoprvé |
| Pravděpodobný (P50) | ≈ 170–185 | Součet balíků výše, běžný rozsah reworku po review |
| Pesimistický | ≈ 235–265 | Opakované zpoždění na externí týmy, revize po bezpečnostním review, přepracování kapacitního plánu po zátěžovém testu, jedno opakování Fáze −1 po drobné korekci |

**Fáze −1 jako investice do snížení rizika ostatních odhadů:** ≈ 5 dní navíc na začátku je malá cena za snížení pravděpodobnosti, že se některý z předpokladů skrytých v odhadech Fáze 0–2 (zejména P1.1–P1.2 reálná Entra integrace a autorizace, které implicitně předpokládají known-good chování klientů) ukáže mylný až uprostřed pilotu — scénář, který by stál řádově týdny, ne dny, na přepracování. Selhání Gate −1 vedoucí k `RethinkDesign` (viz `08` §9) není zahrnuto v žádném z výše uvedených scénářů, protože jde o řízené riziko s vlastním rozhodovacím bodem, ne o kalibrovatelnou nejistotu odhadu.

Ani pesimistický scénář nezahrnuje PM rezervu a čekací doby na externí týmy identifikované v kapitole 4 — ty se sčítají navíc jako kalendářní (ne pracovní) čas.

---

## 7. Definice hotovo (Definition of Done)

**PoC (Fáze 0):** Běží lokálně přes `docker compose up` bez závislosti na produkční Azure infrastruktuře; `tools/list` a `tools/call` fungují pro min. 1 skill nástroj a 1 connector nástroj, ověřeno přes MCP Inspector; README popisuje spuštění a známá omezení.

**Produkční gateway (deployed + documented):** Nasazeno v Azure dle `./02-architektura-reseni.md`; splňuje exit kritéria Gate 1; TLS end-to-end (Front Door → Private Link → ACA); autoscaling ověřen zátěžovým testem na cílový profil zátěže; audit trail ověřitelný (hash chain, denní checkpoint); observabilita (dashboardy + alerty dle tento návrh) aktivní a napojená na reálné kontakty; architektura a provozní dokumentace uložena vedle kódu a aktuální k datu nasazení.

**Konfigurační návod pro Claude a ChatGPT Business:** Krok-za-krokem postup pro připojení custom connectoru v Claude (workspace admin, custom connector remote-MCP) i Developer-mode aplikace v ChatGPT Business, včetně přesného tvaru PRM/OAuth konfigurace; ověřeno na reálném pilotním workspace obou platforem; obsahuje sekci troubleshooting pro nejčastější chyby (401 discovery flow, mismatch `MCP-Protocol-Version`, chybějící scope).

**Handover dokumentace pro interní IT:** Runbooky pro běžné provozní úkony (rotace client secret, rollback katalogové verze, škálování replik, postup při incidentu podle severity); kontaktní matice (vlastník APIM, IAM tým, síťový tým); popis SLO a alertingu; interní IT potvrdilo srozumitelnost formou walkthroughu/školení (vázáno na R13).

---

## 8. Metriky úspěchu projektu

| Metrika | Definice | Cílová hodnota |
|---|---|---|
| Adopce | Podíl cílových uživatelů/workspaců s ≥ 1 úspěšným `tools/call` do X týdnů od go-live dané fáze | ⚠️ TODO — potvrdit se zadavatelem (závisí na A09) |
| Latence | p95 `tools/list` a p95 vlastní režie gateway u `tools/call` (bez času downstream) | `tools/list` < 300 ms, režie gateway < 150 ms |
| Poměr denials | Podíl `tools/call` s `decision.effect = deny` na celkovém objemu volání | Sledovat trend; neočekávaný skok = signál chybné konfigurace entitlements, ne cílová hodnota |
| MTTR | Střední doba do obnovy provozu po incidentu podle interní klasifikace závažnosti | ⚠️ TODO — návrh výchozí < 4 h pro nejvyšší závažnost, k potvrzení s IT |
| Pokrytí auditu | Podíl tool invocations s kompletním, hash-chain ověřitelným auditním záznamem | 100 % jako tvrdý požadavek (ADR-010); měřeno denním checkpoint záznamem |

---

## Otevřené otázky

Podrobné otevřené otázky pro zadavatele viz kapitola 4 (Gap-list) — zde jen doplňkové poznámky mimo přímý gap-list formát:

1. Tento dokument pokrývá analýzu zadání a plán prací (kapitola `01`); architektura řešení je rozvedena v `02` a navazujících dokumentech `03`–`09` a v diagramech `01`–`22`.
2. Přesné číslo pro cílový počet uživatelů a nástrojů (A09, A10) ovlivňuje dimenzování Cosmos DB RU/s a Redis SKU v `./02-architektura-reseni.md` — dokud nepřijde odpověď, návrh pracuje s řádovými odhady, ne s konkrétním provisioned kapacitním plánem.

---

## Zdroje

1. MCP specification — Versioning (dual-era, kompatibilní matice) — https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning
2. MCP specification — Authorization, Dynamic Client Registration deprecated ve prospěch CIMD — https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration
3. Microsoft Entra ID nepodporuje DCR (RFC 7591) — https://learn.microsoft.com/entra/agent-id/secure-mcp-server-with-entra-id, https://learn.microsoft.com/microsoft-365/copilot/extensibility/plugin-authentication-dynamic-client-registration; groups claim overage limit 200 (JWT) — https://learn.microsoft.com/security/zero-trust/develop/configure-tokens-group-claims-app-roles#group-overages
4. Azure API Management MCP support, tiers (Consumption nepodporován) — https://learn.microsoft.com/azure/api-management/export-rest-mcp-server
5. Anthropic Claude custom connectors — požadavky na síťovou dosažitelnost a discovery — https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp
6. OpenAI ChatGPT Developer mode (custom MCP apps, read/write, Business plán omezen na adminy) — https://help.openai.com/en/articles/12584461-developer-mode-and-mcp-apps-in-chatgpt
7. GitHub Actions OIDC federace na Entra ID (workload identity federation) — https://learn.microsoft.com/entra/workload-id/workload-identity-federation-create-trust
8. Azure Storage immutable storage (WORM), time-based retention policy — https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview
