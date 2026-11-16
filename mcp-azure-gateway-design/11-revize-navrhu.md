# Revize návrhu — druhý pohled

Verze 1.0 · 2026-09-17 · Stav: Revizní poznámka · Týká se: ADR-001, ADR-003, ADR-004, ADR-008, ADR-009, ADR-010, ADR-018

Nezávislý pohled na hotový návrh (dokumenty 00–10) s cílem najít místa, kde je návrh složitější, než musí být. Nejde o opravu vad — ty prošly dvěma koly review — ale o otázku, co by se dalo škrtnout nebo odložit, aniž by návrh ztratil hodnotu.

## Co drží a nemění se

Dvě éry MCP jako fakt, APIM jako downstream (ne ingress), Entra ID jako authorization server bez DCR, autorizace deny-by-default se dvěma vrstvami rolí, idempotence vyhodnocovaná až po autorizaci, Gate −1 před vším ostatním. To je jádro návrhu a je správné.

## Co zjednodušit

### 1. Hash chain v auditu (ADR-010)

`prevHash` per událost vyžaduje serializační bod per tenant napříč replikami — sdílený kurzor, contention a přesně ten transakční problém, který [`04-datova-architektura.md`](./04-datova-architektura.md) §6.6 přiznává jako zbytkové riziko. WORM blob s immutability policy dává tamper-evidence sám o sobě.

**Doporučení:** zrušit řetěz per událost. Nechat WORM + hodinový (nebo denní) Merkle root přes dávku událostí, zapsaný jako checkpoint. Je paralelní, bez sdíleného stavu, integrita ověřitelná stejně.

### 2. Úložiště (ADR-009)

Cosmos DB + Managed Redis + WORM blob + Log Analytics + Key Vault pro read-mostly katalog řádově stovek nástrojů je přestřelené. Git je zdroj pravdy — katalog může být podepsaný JSON bundle načtený do paměti každé repliky při startu a při publikaci.

Pro `writeAttempts`, `runtimeBlocks`, entitlements a audit outbox chybí u kombinace Cosmos + Redis transakční hranice. Jeden **PostgreSQL Flexible Server** ji dá (outbox pattern v téže transakci jako doménový zápis) a zároveň ubere jednu službu. Alternativní návrh v [`10-srovnani-navrhu.md`](./10-srovnani-navrhu.md) to má správně.

**Doporučení:** revidovat ADR-009 směrem k Postgres pro transakční data; Redis ponechat jen na idempotenci a rate limit; katalog jako in-memory bundle.

### 3. Obě éry od prvního dne (ADR-001)

Port `EraAdapter` je správně. Implementovat ale obě éry současně znamená dvojnásobný protokolový povrch pro revizi, kterou dnes žádný klient nemluví. Blížící se konec podpory SDK v1 je riziko k evidenci, ne důvod stavět kód, který nikdo nevolá.

**Doporučení:** éra A hned (to je, co klienti mluví), éra B až poté, co Gate −1 ukáže klienta, který ji používá.

### 4. Premium infrastruktura pro pilot (ADR-003, ADR-004)

Front Door Premium + Private Link + Container Apps Premium ingress pro pilotní skupinu 25 uživatelů. Náklady jsou v celém návrhu `⚠️ TODO` — to bude první otázka zákazníka.

**Doporučení:** infrastrukturní profil per fáze. Fáze 0–1: Container Apps external ingress s managed certifikátem, bez Front Door. Fáze 2: Front Door Premium + Private Link. Premium ingress jen pokud Gate −1 ukáže, že klienti drží spojení déle než 240 s.

### 5. Objem dokumentace

Sto tisíc slov je závazek: interní IT to nepřečte a každá změna rozjede konzistenci napříč dokumenty (ukázalo se během jediného dne). 

**Doporučení:** zákazníkovi předat [`00-README.md`](./00-README.md), [`00b-kontext-a-pripady-uziti.md`](./00b-kontext-a-pripady-uziti.md) a [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md) jako design brief (~10 stran); zbytek je referenční příloha.

## Dvě věci k ověření

- **ChatGPT Business v režimu Connectors** (ne Developer mode) vyžaduje nástroje `search` a `fetch`. Ověřit, že je katalog explicitně obsahuje — bez nich se ChatGPT workspace v tomto režimu nepřipojí.
- **OBO (ADR-008) dává smysl jen tehdy**, pokud API za APIM jsou registrovaná v Entra a přijímají user token. Pokud jedou na subscription key, celá OBO vrstva je mrtvá a stačí Managed Identity + hlavička `x-mcp-principal-oid`. To je otázka pro zákazníka, ne architektonické rozhodnutí.

## Pořadí dalších kroků

Vše níže je **před implementací** — ověřuje se model, nestaví se produkt.

1. **Gate −1 spike** (řádově den): zahoditelný minimální MCP server s OAuth na veřejné URL, připojený k reálnému Claude a ChatGPT workspace. Výstupem jsou odpovědi, ne kód.
2. **Potvrdit OBO vs. subscription key** se zákazníkem — bez kódu.
3. **Revidovat ADR-009** (Cosmos vs. Postgres) s odpověďmi z bodů 1 a 2.
4. Teprve pak fáze 0 PoC podle [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §9.
