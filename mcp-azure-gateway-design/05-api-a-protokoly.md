# 05 — API a protokoly

**Verze:** 1.0 · **Datum:** 2026-09-17 · **Stav:** Návrh
**Dotčená ADR:** **ADR-001 (dual-era)**, ADR-002 (stack), ADR-005 (Entra jako AS), ADR-006 (bez DCR, PRM), ADR-007 (authz), ADR-008 (OBO), ADR-012 (immutable katalog), ADR-015 (multi-workspace), ADR-016 (idempotence), ADR-017 (rate limiting)
**Souvisí:** [`./03-kodova-architektura.md`](./03-kodova-architektura.md) · [`./06-stavova-logika-a-workflow.md`](./06-stavova-logika-a-workflow.md) · [`./07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md)

---

## 1. Přehled protokolové situace: dvě éry MCP

MCP prošel v revizi **`2026-07-28`** změnou, která není kompatibilní se vším předchozím. Spec sama pro to zavádí terminologii: **Modern** = revize předávající verzi, identitu a schopnosti jako metadata každého požadavku (`2026-07-28` a novější); **Legacy** = revize navazující session přes `initialize` handshake (`2025-11-25` a starší) [6].

### 1.1 Tabulka rozdílů

| Aspekt | **Éra A — Legacy** (`2025-11-25`, `2025-06-18`) | **Éra B — Modern** (`2026-07-28`) | Zdroj |
|---|---|---|---|
| Handshake | `initialize` → `initialize` result → `notifications/initialized` | **žádný** | [6][14] |
| Deklarace verze | `initialize.params.protocolVersion`; poté hlavička `MCP-Protocol-Version` na všech dalších požadavcích | `_meta["io.modelcontextprotocol/protocolVersion"]` **a** hlavička `MCP-Protocol-Version` na **každém** POST; musí se shodovat | [2][7][15] |
| Neshoda hlavičky a těla | nedefinováno | `400` + `-32020` `HeaderMismatch` | [2] |
| Povinné zrcadlené hlavičky | jen `MCP-Protocol-Version` | + `Mcp-Method` (vždy), `Mcp-Name` (pro `tools/call`, `resources/read`, `prompts/get`), volitelně `Mcp-Param-{Name}` | [2] |
| Session | `MCP-Session-Id` přidělený na `initialize` result, posílaný na všech dalších požadavcích; `DELETE` ukončuje | **zrušena** — server nemá mintovat ani echovat session ID | [15][2] |
| Discovery | `initialize` result (`capabilities`, `serverInfo`, `instructions`) | povinné RPC **`server/discover`** → `supportedVersions`, `capabilities`, `_meta["io.modelcontextprotocol/serverInfo"]`, `instructions` | [9] |
| Neshoda verze | server odpoví **jinou** verzí v `initialize` result; klient se odpojí, pokud ji neumí | `400` + `-32022` `UnsupportedProtocolVersion` s `data.supported[]` a `data.requested` | [14][6] |
| GET stream (SSE) | ano — samostatný server→klient kanál | **zrušeno** → `405 Method Not Allowed` | [15][2] |
| `DELETE` na `/mcp` | ano | **zrušeno** → `405` | [2] |
| Resumability | `Last-Event-ID`, replay od offsetu | **zrušeno**; klient musí požadavek podat znovu s novým `id` | [2] |
| Server-iniciované požadavky na streamu | ano (sampling, elicitation, roots) | **zakázáno**; nahrazeno MRTR — `InputRequiredResult` + retry klientem | [2] |
| Cancellation na HTTP | `notifications/cancelled` | **zavření SSE streamu JE signál zrušení**; `notifications/cancelled` zůstává jen pro stdio | [2][11] |
| `resultType` ve výsledku | neexistuje | **povinné** — `"complete"` nebo `"input_required"` | [4][9] |
| Cachovací hinty na `*/list` | ne | `ttlMs` + `cacheScope` (`"public"`/`"private"`) | [4][10] |
| Notifikace změn | `notifications/tools/list_changed` na GET streamu | `subscriptions/listen` → dlouhoběžící SSE odpověď s vyfiltrovanými notifikacemi | [2][4] |

### 1.2 Co to znamená pro implementaci

**Realita klientů k září 2026** (⚠️ převzato z rešerše, ověřeno nepřímo — přes to, na jakou revizi odkazuje produkční dokumentace obou dodavatelů, ne přes jejich prohlášení o podporované verzi [13]):

- Anthropic ve všech konektorových dokumentech odkazuje na `modelcontextprotocol.io/specification/2025-11-25/...` a používá beta hlavičku `mcp-client-2025-11-20` [5][13].
- OpenAI nepřipíná v dostupné dokumentaci **žádnou** datovanou verzi MCP [13]. **NEOVĚŘENO.**
- **Žádný z dodavatelů nezveřejnil datum migrace na éru B.** **NEOVĚŘENO.**

Z toho plyne ADR-001. Zásadní je, že **dual-era server je spec-konformní volba, ne hack**: spec explicitně říká, že server, který chce obsloužit legacy i modern klienty, **MAY** implementovat oboje, a že dual-era server **MAY** obsluhovat obě éry současně na stejném endpointu [6]. Výběr chování je řízen tím, jak klient otevře komunikaci: požadavek s modern per-request `_meta` se obslouží bezstavově podle `2026-07-28`; požadavek `initialize` vybere legacy sémantiku [6].

Matice kompatibility ze spec [6] navíc potvrzuje, že **dual-era je jediná varianta bez ztráty klientů**:

| Klient | Server | Výsledek |
|---|---|---|
| Legacy | Modern-only | **selže** — legacy klient nemá mechanismus „fall-forward" |
| Modern | Legacy-only | **selže** |
| Legacy | **Dual-era** | funguje |
| Modern | **Dual-era** | funguje |

Praktické implementační důsledky, které se promítají do celého návrhu:

1. **Nestavět nic na session.** Idempotence se klíčuje `Idempotency-Key` odvozeným z `requestId` (ADR-016), ne session ID. Rate limiting per uživatele, ne per session (ADR-017) — a countery musí být sdílené (Redis), ne v paměti jednotlivé repliky, jinak se efektivní limit násobí počtem replik (viz [`08-provoz-observabilita-rollout.md` §7](./08-provoz-observabilita-rollout.md#7-skalovani-a-kapacita)). Session store éry A je volitelný modul (ADR-001): spec `2025-11-25` u `Mcp-Session-Id` říká **MAY**, ne MUST [15] — server éry A smí `initialize` obsloužit zcela bezstavově, a gateway, která žádné session ID nevydává, zůstává plně spec-konformní.
2. **Nestavět nic na resumability.** Retry logika nesmí předpokládat replay-from-offset ani v éře A, kde by ji spec ještě umožňovala — v éře B je zrušená [2].
3. **Nestavět nic na server-iniciovaných požadavcích.** Sampling, elicitation a roots jsou v éře B nahrazené MRTR a navíc formálně deprecated [13]. Gateway je **nepoužívá**; lidské potvrzení nevratných akcí se řeší povinným parametrem `confirm` (ADR-016), ne elicitací.
4. **`Mcp-Method` a `Mcp-Name` jsou příležitost.** Éra B zrcadlí metodu a název nástroje do hlaviček právě proto, aby zprostředkovatelé mohli směrovat bez parsování těla [2]. Front Door i APIM tak mohou dělat per-tool rate limiting bez čtení JSON. Spec ale zároveň varuje: zprostředkovatel, který vynucuje politiku podle zrcadlených hlaviček, **SHOULD** ověřit, že `MCP-Protocol-Version` označuje revizi, která validaci hlavička–tělo vyžaduje, a jinak požadavek odmítnout [2]. Pro éru A tedy tato optimalizace neplatí.

---

## 2. HTTP povrch

| Metoda | Cesta | Auth | Request | Response | Stavové kódy | Klíčové hlavičky |
|---|---|---|---|---|---|---|
| `POST` | `/mcp`, `/mcp/{workspaceSlug}` | **povinná** Bearer | jeden JSON-RPC request nebo notifikace | JSON objekt nebo SSE stream | `200`, `202` (notifikace), `400`, `401`, `403`, `404` (neznámá metoda / expirovaná session éry A), `405`, `413`, `415`, `429`, `500` | req: `Authorization`, `Content-Type: application/json`, `Accept: application/json, text/event-stream`, `MCP-Protocol-Version`, `Mcp-Method`, `Mcp-Name`, `Origin`, `Idempotency-Key`, `traceparent`<br>res: `Content-Type`, `MCP-Session-Id` (jen éra A), `x-request-id`, `X-Accel-Buffering: no` (SSE), `Cache-Control: no-store` |
| `GET` | `/mcp`, `/mcp/{workspaceSlug}` | povinná | — | SSE stream (éra A) | `200` (éra A), **`405` pro éru B** [2] | req: `MCP-Session-Id`, `Last-Event-ID`; res: `Content-Type: text/event-stream` |
| `DELETE` | `/mcp`, `/mcp/{workspaceSlug}` | povinná | — | prázdné | `204` (éra A), **`405` pro éru B** [2] | req: `MCP-Session-Id` |
| `GET` | `/.well-known/oauth-protected-resource` | **žádná** | — | PRM JSON (RFC 9728) | `200` | res: `Content-Type: application/json`, `Cache-Control: public, max-age=3600` |
| `GET` | `/.well-known/oauth-protected-resource/mcp/{workspaceSlug}` | žádná | — | PRM JSON per workspace | `200`, `404` | totéž |
| `GET` | `/healthz` | žádná (jen z VNet/FD) | — | `{"status":"ok"}` | `200`, `503` | — |
| `GET` | `/readyz` | žádná (jen z VNet/FD) | — | stav závislostí | `200`, `503` | — |
| `GET` | `/metrics` | interní | — | OTel/Prometheus expozice | `200` | — |
| `POST` | `/admin/catalog/publish` | Bearer + app role `Mcp.CatalogAdmin` | `{ "catalogVersionId": "...", "workspaces": ["..."] }` | `{ "previous": "...", "current": "..." }` | `200`, `401`, `403`, `409` (konflikt pointeru), `422` | `Idempotency-Key` |
| `POST` | `/admin/catalog/rollback` | Bearer + `Mcp.CatalogAdmin` | `{ "workspace": "...", "toVersionId": "..." }` | totéž | `200`, `401`, `403`, `404` | — |
| `POST` | `/webhooks/github` | HMAC SHA-256 | GitHub event payload | `{"accepted":true}` | `202`, `401` (vadný podpis), `422` | `X-Hub-Signature-256`, `X-GitHub-Event`, `X-GitHub-Delivery` |

**Konstrukce cesty PRM.** RFC 9728 §3.1 předepisuje: má-li identifikátor zdroje cestu, vloží se `/.well-known/{suffix}` **mezi host a cestu** [3]. Pro `https://mcp.contoso.com/mcp/acme` je tedy PRM na `https://mcp.contoso.com/.well-known/oauth-protected-resource/mcp/acme` — to je přesně tvar v architektonických rozhodnutí v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md) a je to zároveň mechanismus, kterým RFC podporuje víc zdrojů na jednom hostu [3]. Root varianta `/.well-known/oauth-protected-resource` zůstává jako fallback pro klienty, kteří zkoušejí obojí [1].

**Strojově čitelný kontrakt administrativních a webhook endpointů.** Řádky `/admin/catalog/publish`, `/admin/catalog/rollback`, `/webhooks/github`, `/healthz`, `/readyz` a `/.well-known/oauth-protected-resource*` výše mají formální OpenAPI 3.1 popis v [`kontrakty/gateway-admin.openapi.yaml`](./kontrakty/gateway-admin.openapi.yaml) — CI ho validuje proti této tabulce (viz [`08-provoz-observabilita-rollout.md` §8](./08-provoz-observabilita-rollout.md#8-nasazeni-a-cicd)), takže rozchod mezi dokumentací a kontraktem shodí build, ne až provoz. `POST /mcp[/{workspaceSlug}]` samotný v OpenAPI záměrně **není** — je to jediný JSON-RPC endpoint, jehož tvar požadavku/odpovědi se řídí metodou uvnitř těla (`tools/call`, `tools/list`, …), ne HTTP cestou/metodou jako u REST zdrojů, které OpenAPI modeluje; jeho kontrakt dávají dohromady MCP specifikace (JSON-RPC obálka, viz §3–4 níže) a schéma jednotlivých nástrojů (`inputSchema`/`outputSchema` z katalogu, viz [`kontrakty/skill-manifest.schema.json`](./kontrakty/skill-manifest.schema.json)), ne jeden statický OpenAPI dokument.

**Povinné vstupní kontroly na `POST /mcp`**, v tomto pořadí:

1. `Origin` — pokud je přítomen a neplatný, `403 Forbidden`; tělo **MAY** být JSON-RPC error bez `id` [2]. Ochrana proti DNS rebindingu.
2. `Content-Type: application/json` → jinak `415`.
3. `Accept` musí obsahovat **obojí** `application/json` i `text/event-stream` [2] → jinak `406`. Tolerance: `*/*` se akceptuje.
4. `Content-Length` ≤ 1 MiB → jinak `413`.
5. JSON-RPC rámec musí být **jeden** objekt (ne batch) a nesmí to být JSON-RPC *response* [2].

### 2.1 Kanonické pořadí zpracování požadavku

Výše uvedené kontroly jsou začátek jedné **kanonické 20krokové request pipeline**, závazné napříč tímto dokumentem i dokumenty 02, 03, 06, 07 a diagramy 07, 13, 17, 20 — pořadí a názvy kroků se nikde neliší, jen se cituje odtud nebo z [`./03-kodova-architektura.md`](./03-kodova-architektura.md#7-zpracování-requestu-krok-za-krokem), kde je stejná tabulka jako implementační normativa.

| # | Krok | Komponenta | Selhání |
|---|---|---|---|
| 1 | TLS terminace, WAF, DDoS | Front Door Premium | `403` / blokace na edge |
| 2 | Ingress: `Content-Type`, velikost těla, `Accept`, `Origin` | `edge/` | `400` / `413` / `415` |
| 3 | Korelace: `traceparent`, `requestId`, `Mcp-Session-Id` (éra A) | `telemetry/` | — |
| 4 | Detekce éry a negotiation verze | `protocol/` | `400` + `-32020`, nebo `-32022` |
| 5 | Parse JSON-RPC a validace obálky | `protocol/` | `-32700` / `-32600` |
| 6 | **Autentizace**: validace bearer tokenu | `identity/` | **HTTP 401 + `WWW-Authenticate`** |
| 7 | Sestavení `Principal` z claimů | `identity/` | `401` |
| 8 | Resolution workspace, kontrola podporované éry a capability | `protocol/` + `catalog/` | `404` / `-32021` |
| 9 | Routing metody | `protocol/` | `404` + `-32601` |
| 10 | Resolution katalogové verze (aktivní nebo `pinnedVersion`) | `catalog/` | tool error |
| 11 | Dohledání nástroje a validace argumentů proti `inputSchema` | `catalog/` | `-32602` |
| 12 | **Autorizace** (PolicyEngine, deny-by-default) | `authz/` | tool error `isError: true` |
| 13 | Rate limit a kvóta | `execution/` | tool error s `retryAfterMs` |
| 14 | **Kontrola idempotence** — až ZDE, po autorizaci | `execution/` | replay výsledku |
| 15 | Potvrzovací brána (`confirm`) dle §5.3 | `execution/` | tool error „vyžaduje potvrzení" |
| 16 | Získání OBO tokenu | `identity/` | tool error |
| 17 | Volání downstream APIM: timeout, retry, circuit breaker | `adapters/connector/` | tool error |
| 18 | Tvarování odpovědi, validace `outputSchema`, redakce, limit velikosti | `execution/` | tool error |
| 19 | **Zápis auditu** (dvoustupňový model — viz [`./03-kodova-architektura.md`](./03-kodova-architektura.md#85-selhání-auditu--dvoustupňový-model)) | `audit/` | fail-open/fail-closed dle rizika nástroje |
| 20 | Serializace odpovědi podle éry | `protocol/` | `500` |

**Bezpečnostní invariant:** idempotence cache (krok 14) se čte až **po** úspěšné autorizaci (krok 12), nikdy před ní. Klíč cache obsahuje `principal.oid`, takže výsledek nelze přehrát jiným uživatelem — formát klíče je `idem:{tenantId}:{workspaceSlug}:{principalOid}:{toolName}:{idempotencyKey}`. Autorizace se vyhodnocuje **znovu při každém přehrání** — zrušené oprávnění se projeví okamžitě, nikoli až po vypršení 24h TTL idempotence záznamu.

---

## 3. Negotiation algoritmus

### 3.1 Diagram

```mermaid
flowchart TD
    START["POST na /mcp nebo /mcp/workspaceSlug"] --> ORIGIN{"Hlavicka Origin pritomna a povolena"}
    ORIGIN -- "ne" --> E403["403 Forbidden<br/>telo muze byt JSON-RPC error bez id"]
    ORIGIN -- "ano" --> PARSE{"Telo je platny JSON-RPC ramec"}
    PARSE -- "ne" --> E32700["400 + JSON-RPC -32700 Parse error"]
    PARSE -- "ano" --> METAV{"Pritomno _meta klic io.modelcontextprotocol/protocolVersion"}

    METAV -- "ano" --> HDR{"Hlavicka MCP-Protocol-Version pritomna"}
    HDR -- "ne" --> E32020A["400 + -32020 HeaderMismatch<br/>chybi povinna hlavicka"]
    HDR -- "ano" --> MATCH{"Hodnota hlavicky se rovna hodnote v _meta"}
    MATCH -- "ne" --> E32020B["400 + -32020 HeaderMismatch<br/>hlavicka neodpovida telu"]
    MATCH -- "ano" --> MIRROR{"Mcp-Method a pripadne Mcp-Name odpovidaji telu"}
    MIRROR -- "ne" --> E32020C["400 + -32020 HeaderMismatch"]
    MIRROR -- "ano" --> SUPB{"Verze je v seznamu supportedVersions"}
    SUPB -- "ne" --> E32022["400 + -32022 UnsupportedProtocolVersion<br/>data.supported a data.requested"]
    SUPB -- "ano" --> CAPS{"clientCapabilities pritomne a dostacujici"}
    CAPS -- "chybi povinne pole" --> E32602["400 + -32602 Invalid params"]
    CAPS -- "chybi schopnost" --> E32021["400 + -32021 MissingRequiredClientCapability"]
    CAPS -- "ok" --> ERAB["StatelessAdapter<br/>era B 2026-07-28"]

    METAV -- "ne" --> ISINIT{"method je initialize"}
    ISINIT -- "ano" --> NEGA{"Pozadovana protocolVersion je podporovana"}
    NEGA -- "ano" --> ERAA["LegacyHandshakeAdapter<br/>vrati stejnou verzi a MCP-Session-Id"]
    NEGA -- "ne" --> DOWN["Vrati nejvyssi vlastni podporovanou verzi<br/>a seznam verzi v instructions"]
    DOWN --> ERAA
    ISINIT -- "ne" --> SESS{"Hlavicka MCP-Session-Id pritomna a platna"}
    SESS -- "ano" --> ERAA
    SESS -- "neplatna nebo vyprsela" --> E404["404 Not Found<br/>klient zahaji novy initialize"]
    SESS -- "ne" --> LEGHDR{"Hlavicka MCP-Protocol-Version pritomna"}
    LEGHDR -- "ano" --> ERAA
    LEGHDR -- "ne" --> FALLBACK["Predpoklad verze 2025-03-26<br/>pokud je povoleno konfiguraci"]
    FALLBACK --> SUPA{"Verze je v seznamu podporovanych"}
    SUPA -- "ano" --> ERAA
    SUPA -- "ne" --> E400["400 Bad Request<br/>nepodporovana verze protokolu"]

    ERAA --> CANON["Normalizace na CanonicalRequest"]
    ERAB --> CANON
    CANON --> CORE["Jadro - identity, authz, catalog, execution, audit"]
    CORE --> DENORM["Denormalizace CanonicalResult zpet do tvaru dane ery"]

    GETDEL["GET nebo DELETE na /mcp"] --> ERACHK{"Je pro dany workspace povolena era A"}
    ERACHK -- "ano" --> SSE["GET otevre SSE stream<br/>DELETE ukonci session"]
    ERACHK -- "ne" --> E405["405 Method Not Allowed"]
```

**Co diagram říká.** Rozhodovací strom má jeden nosný uzel: **přítomnost klíče `io.modelcontextprotocol/protocolVersion` v `_meta`**. Ten je v éře B povinný na každém požadavku [1], zatímco éra A ho nikdy neposílá. Všechno ostatní jsou už jen validace uvnitř zvolené větve.

Tři věci, které z toho plynou a nejsou zřejmé:

1. **Chybějící hlavička `MCP-Protocol-Version` u požadavku s modern `_meta` není „nepodporovaná verze", ale `HeaderMismatch`.** Spec uvádí chybějící povinnou hlavičku výslovně mezi podmínkami selhání validace vedoucími na `-32020` [2]. Záměna kódů by zmátla dual-era klienty, protože ti podle obsahu těla `400` odpovědi rozhodují, jestli fallbacknout na `initialize` [2][6].
2. **Server musí odmítnout i požadavek, kde hlavička a tělo sedí, ale `Mcp-Method` chybí.** To je v praxi nejčastější chyba vlastních klientů a nejlevnější místo, kde ji odhalit.
3. **Fallback na verzi `2025-03-26` při chybějící hlavičce je volba, ne povinnost.** Server, který podporuje klienty starší než `2025-06-18`, **MAY** požadavek bez hlavičky interpretovat jako `2025-03-26`; server, který takové klienty nepodporuje, ho **MUST** odmítnout [2]. Náš `supportedVersions` obsahuje `2025-06-18` jako nejstarší, takže konfigurační přepínač `protocol.assumeLegacyVersionWhenHeaderMissing` je ve výchozím stavu `null` = odmítat.

### 3.2 Pseudokód

```
function negotiate(envelope) -> EraAdapter | Error:

  # --- vstupní brány společné oběma érám ---
  if envelope.headers.origin present and not in workspace.allowedOrigins:
    return Http(403, jsonRpcErrorWithoutId("origin not allowed"))
  frame = parseJsonRpc(envelope.body) or return Http(400, error(-32700, "Parse error"))

  metaVersion = frame.params?._meta?["io.modelcontextprotocol/protocolVersion"]
  headerVersion = header("MCP-Protocol-Version") # case-insensitive [2]

  # ============ VĚTEV B — modern ============
  if metaVersion is present:

    if headerVersion is absent:
      return Http(400, error(-32020, "HeaderMismatch: missing MCP-Protocol-Version"))
    if headerVersion != metaVersion: # hodnoty case-sensitive [2]
      return Http(400, error(-32020, "HeaderMismatch: header/body protocol version"))

    if header("Mcp-Method") absent or != frame.method:
      return Http(400, error(-32020, "HeaderMismatch: Mcp-Method"))
    if frame.method in {tools/call, resources/read, prompts/get}:
      expected = frame.params.name ?? frame.params.uri
      got = decodeBase64Sentinel(header("Mcp-Name")) # =?base64?...?= [2]
      if got absent or got != expected:
        return Http(400, error(-32020, "HeaderMismatch: Mcp-Name"))
      for each h in headers matching "Mcp-Param-*":
        verify against statically reachable property with x-mcp-header # [4]
        on mismatch: return Http(400, error(-32020, "HeaderMismatch: Mcp-Param-*"))

    if metaVersion not in config.supportedVersions:
      return Http(400, error(-32022, "Unsupported protocol version",
        data = { supported: config.supportedVersions,
        requested: metaVersion })) # [6]

    if frame.params._meta["io.modelcontextprotocol/clientCapabilities"] is absent:
      return Http(400, error(-32602, "Invalid params: clientCapabilities required")) # [7]
    missing = requiredCapabilitiesFor(frame.method) - declaredCapabilities
    if missing not empty:
      return Http(400, error(-32021, "Missing required client capability",
        data = { requiredCapabilities: missing })) # [7]

    if frame.method not in KNOWN_METHODS:
      return Http(404, error(-32601, "Method not found")) # [2]

    return StatelessAdapter

  # ============ VĚTEV A — legacy ============
  if frame.method == "initialize":
    requested = frame.params.protocolVersion
    chosen = requested in config.supportedVersions
      ? requested
      : highestLegacyVersion(config.supportedVersions) # [14]
    return LegacyHandshakeAdapter(chosen) # odpověď nese MCP-Session-Id

  sessionId = header("MCP-Session-Id")
  if sessionId present:
    session = sessionStore.get(sessionId)
    if session is null or expired:
      return Http(404) # klient MUSÍ zahájit nový initialize [13]
    return LegacyHandshakeAdapter(session.protocolVersion)

  if headerVersion present:
    if headerVersion not in config.supportedVersions: return Http(400) # [13]
    return LegacyHandshakeAdapter(headerVersion)

  if config.assumeLegacyVersionWhenHeaderMissing != null:
    return LegacyHandshakeAdapter(config.assumeLegacyVersionWhenHeaderMissing) # [2]
  return Http(400, "missing MCP-Protocol-Version")


function handleGetOrDelete(envelope):
  if not workspace.eraAEnabled: return Http(405) # éra B: GET/DELETE zrušeny [2]... # éra A: SSE stream / ukončení session
```

**Konflikt hlavičky a `_meta`.** Spec nepřipouští žádnou „chytrou" harmonizaci: neshoda **MUST** vést na `400` + `-32020` [2]. Gateway tedy nikdy nepreferuje jednu stranu — důvod je bezpečnostní a spec ho pojmenovává: pokud load balancer směruje podle hlavičky a server vykonává podle těla, vzniká rozpor mezi dvěma zdroji pravdy [2].

**Nepodporovaná verze.** Odpověď musí nést `data.supported` se seznamem verzí, aby klient mohl zvolit společnou a požadavek zopakovat [6]. Modern-only server **SHOULD** navíc svoje podporované verze uvést i v chybě na `initialize` — legacy klient nemá jiný způsob, jak se to dozvědět [6]. Naše dual-era gateway na `initialize` odpovídá normálně, ale pokud by v budoucnu éru A vypnula, musí tuto formulaci doplnit do chybové zprávy; je to naplánováno jako součást kroku vypnutí éry A ([`./03-kodova-architektura.md`](./03-kodova-architektura.md#133-odchod-éry-a)).

### 3.3 Kanonická tabulka: situace → HTTP status → JSON-RPC kód nebo tool error

Rozsah **`-32000` až `-32019` je legacy a v nových implementacích se nepoužívá** — nové kódy se v něm nealokují. Chybějící nebo neplatný token **není JSON-RPC chyba**: je to HTTP `401` s hlavičkou `WWW-Authenticate: Bearer resource_metadata="https://mcp.contoso.com/.well-known/oauth-protected-resource"` a tělo odpovědi **není** JSON-RPC error objekt (viz §4.9). Aplikační selhání a autorizační odmítnutí jsou **tool error** (`isError: true`), ne JSON-RPC error — výsledek nese strukturovaný důvod a to, co má agent udělat dál. Toto je kanonická tabulka pro celý návrh; jinde v dokumentaci se na ni jen odkazuje. Sloupec **`DenyReason`** uvádí hodnotu, která se u autorizačně podmíněných řádků objeví v `_meta.errorCode` odpovědi — kanonická sada (jediná v celém návrhu, snake_case) je definovaná jako TypeScript union `DenyReason` v [`03-kodova-architektura.md §4`](./03-kodova-architektura.md); u řádků, které s autorizačním rozhodnutím nesouvisí, je „—".

| # | Situace | HTTP status | JSON-RPC kód nebo tool error | `DenyReason` |
|---|---|---|---|---|
| 1 | Blokováno WAF/DDoS pravidlem na edge | `403` | žádné tělo JSON-RPC (blokace mimo gateway) | — |
| 2 | Neplatný nebo nepovolený `Origin` | `403` | JSON-RPC error bez `id`, bez kódu | — |
| 3 | Nesprávný `Content-Type` | `415` | — | — |
| 4 | `Accept` neobsahuje `application/json` i `text/event-stream` | `406` | — | — |
| 5 | Tělo přesahuje limit velikosti | `413` | — | — |
| 6 | Neplatný JSON-RPC rámec (parse error) | `400` | `-32700` | — |
| 7 | Neplatná JSON-RPC obálka (chybí povinná pole, špatný typ) | `400` | `-32600` | — |
| 8 | Chybějící nebo neshodná hlavička `MCP-Protocol-Version` / `Mcp-Method` / `Mcp-Name` (éra B) | `400` | `-32020` HeaderMismatch | — |
| 9 | Nepodporovaná verze protokolu | `400` | `-32022` UnsupportedProtocolVersion | — |
| 10 | Chybějící `clientCapabilities` nebo jiné povinné `_meta` pole | `400` | `-32602` Invalid params | — |
| 11 | Chybějící požadovaná client capability | `400` | `-32021` MissingRequiredClientCapability | — |
| 12 | Chybějící nebo neplatný bearer token | **`401`** | **žádný JSON-RPC error objekt** — jen `WWW-Authenticate` | — |
| 13 | Nedostatečný scope (step-up autorizace) | `403` | žádný JSON-RPC kód — `WWW-Authenticate: error="insufficient_scope"` | — (scope problém, ne entitlement — viz §4.9) |
| 14 | Neznámá metoda | `404` | `-32601` Method not found | — |
| 15 | `GET`/`DELETE` na endpointu éry B | `405` | — | — |
| 16 | Neplatná nebo expirovaná session éry A | `404` | — (klient zahájí nový `initialize`) | — |
| 17 | Neznámý nástroj v katalogu | `200` | tool error, `isError: true` | `tool_not_in_catalog` nebo `tool_deprecated_removed` |
| 18 | Neplatné argumenty proti `inputSchema` | `200` | tool error, `isError: true` | — (validace schématu, ne autorizace) |
| 19 | Autorizační odmítnutí (deny-by-default) | `200` | tool error, `isError: true`, `errorCode:` hodnota `DenyReason` | `toolset_not_mapped`, `no_matching_entitlement`, `explicit_deny_rule`, `parameter_constraint_violated`, `risk_tier_requires_role`, `workspace_not_entitled` nebo `entitlement_expired` — podle konkrétního kroku algoritmu §5.3 dokumentu 07, viz mapování tamtéž |
| 20 | Rate limit / kvóta vyčerpána | `200` | tool error, `isError: true`, `_meta.retryAfterMs` | `rate_limit_exceeded` |
| 21 | Idempotence — replay uloženého výsledku | `200` | normální výsledek (bez chyby) | — |
| 22 | Chybějící potvrzení (`confirm`) u rizikového nástroje | `200` | tool error, `isError: true`, „vyžaduje potvrzení" | — |
| 23 | Selhání OBO výměny (`interaction_required`) | `200` | tool error, `isError: true`, `DownstreamAuthRequired` | — |
| 24 | Downstream APIM vrátil 4xx/5xx | `200` | tool error, `isError: true`, mapovaný `errorCode` (§8.3) | — |
| 25 | Downstream timeout | `200` | tool error, `isError: true`, `_meta.retryAfterMs` | — |
| 26 | Otevřený circuit breaker | `200` | tool error, `isError: true`, `errorCode: CIRCUIT_OPEN` | — |
| 27 | Selhání auditu — `failClosed` větev dvoustupňového modelu | `200` | tool error, `isError: true`, „audit nedostupný" | — |
| 28 | Neošetřená interní výjimka v gateway | `500` | `-32603` Internal error | — |
| 29 | Neplatný formát `Idempotency-Key` | `400` | `-32602` | — |
| 30 | Neplatný HMAC podpis na `/webhooks/github` | `401` | — (mimo MCP JSON-RPC, administrativní endpoint) | — |

---

## 4. Konkrétní JSON příklady

Hodnoty jsou ilustrativní; hlavičky jsou uvedené tam, kde jsou součástí kontraktu. Workspace v příkladech je `acme`, host `mcp.contoso.com`.

### 4.1 `server/discover` — éra B

```http
POST /mcp/acme HTTP/1.1
Host: mcp.contoso.com
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGc...
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Protocol-Version: 2026-07-28
Mcp-Method: server/discover
Origin: https://claude.ai

{
  "jsonrpc": "2.0",
  "id": "discover-1",
  "method": "server/discover",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientInfo": { "name": "Claude", "version": "2026.09" },
      "io.modelcontextprotocol/clientCapabilities": {}
    }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": "discover-1",
  "result": {
    "resultType": "complete",
    "supportedVersions": ["2026-07-28", "2025-11-25", "2025-06-18"],
    "capabilities": { "tools": { "listChanged": true } },
    "instructions": "Gateway do interních systémů ACME. Nástroje s prefixem crm_ pracují se zákaznickými účty, finance_ s fakturací, hr_ s personálními daty. Čtecí nástroje lze volat volně; zapisovací vyžadují potvrzení parametrem confirm. Výsledky jsou omezené oprávněními přihlášeného uživatele.",
    "ttlMs": 3600000,
    "cacheScope": "private",
    "_meta": {
      "io.modelcontextprotocol/serverInfo": { "name": "acme-mcp-gateway", "version": "1.4.2" }
    }
  }
}
```

Tři poznámky ke tvaru: `serverInfo` je v éře B **uvnitř `_meta`**, ne na kořeni výsledku [9]; `resultType` je povinný na každém výsledku [4]; `cacheScope` volíme `"private"`, protože obsah `capabilities` i budoucí `instructions` mohou být per-workspace, a `"public"` by dovolilo sdílenou cache u zprostředkovatele [10].

**`instructions` a Claude/ChatGPT.** OpenAI uvádí, že pole `instructions` je předáváno modelu a že **prvních 512 znaků by mělo dávat smysl samostatně**, protože se může zkrátit [13]. Text výše je proto napsaný tak, aby první věty nesly nejdůležitější informaci.

### 4.2 `initialize` — éra A

```http
POST /mcp/acme HTTP/1.1
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGc...
Content-Type: application/json
Accept: application/json, text/event-stream

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": { "elicitation": { "form": {} } },
    "clientInfo": { "name": "Claude", "title": "Claude", "version": "2026.09" }
  }
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
MCP-Session-Id: 00000000-0000-4000-8000-000000000001
x-request-id: 01JB9Z2K4P7Q1M

{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": { "tools": { "listChanged": true } },
    "serverInfo": { "name": "acme-mcp-gateway", "title": "ACME MCP Gateway", "version": "1.4.2" },
    "instructions": "Gateway do interních systémů ACME. Nástroje s prefixem crm_ pracují se zákaznickými účty, finance_ s fakturací, hr_ s personálními daty..."
  }
}
```

Gateway **nedeklaruje** `resources`, `prompts`, `logging` ani `completions`. Důvod je věcný: hostované konektorové cesty obou dodavatelů jsou **tools-only** — Anthropic to u Messages API konektoru uvádí explicitně („❌ MCP Resources, ❌ MCP Prompts"), u OpenAI to plyne z absence jakékoli zmínky napříč dokumentací (⚠️ NEOVĚŘENO jako explicitní vyloučení) [13]. Deklarovat schopnost, kterou nikdo nevyužije, znamená jen rozšířit útočnou plochu.

Pokud klient požádá o verzi, kterou neumíme, vrátíme **nejvyšší vlastní podporovanou** — spec to předepisuje jako chování serveru a klient se má odpojit, pokud ji neumí [14].

### 4.3 `notifications/initialized` — éra A

```http
POST /mcp/acme HTTP/1.1
MCP-Session-Id: 00000000-0000-4000-8000-000000000001
MCP-Protocol-Version: 2025-11-25
Content-Type: application/json

{ "jsonrpc": "2.0", "method": "notifications/initialized" }
```

```http
HTTP/1.1 202 Accepted
```

### 4.4 `tools/list` s kurzorem

```http
POST /mcp/acme HTTP/1.1
Host: mcp.contoso.com
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGc...
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/list

{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/list",
  "params": {
    "cursor": "eyJ2IjoiOWYyYTFjMy1iNGQiLCJvIjo1MH0",
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {}
    }
  }
}
```

`Mcp-Name` se u `tools/list` neposílá — je povinné jen pro `tools/call`, `resources/read` a `prompts/get` (§1.1).

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "resultType": "complete",
    "tools": [
      {
        "name": "crm_get_account",
        "title": "Detail zákaznického účtu",
        "description": "Vrátí detail jednoho zákaznického účtu z CRM podle jeho identifikátoru nebo IČO. Použij, když uživatel odkazuje na konkrétního zákazníka a potřebuješ jeho segment, přiřazeného obchodníka, fakturační adresu nebo stav smlouvy. Nevrací faktury ani objednávky, na ty jsou nástroje finance_list_invoices a sales_list_orders. Vrací chybu, pokud účet neexistuje nebo k němu nemáš oprávnění.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "accountId": { "type": "string", "pattern": "^ACC-[0-9]{6}$", "description": "Identifikátor účtu ve tvaru ACC-000123. Uveď buď accountId, nebo ico." },
            "ico": { "type": "string", "pattern": "^[0-9]{8}$", "description": "Osmimístné IČO. Uveď buď accountId, nebo ico." }
          },
          "oneOf": [{ "required": ["accountId"] }, { "required": ["ico"] }],
          "additionalProperties": false
        },
        "outputSchema": {
          "type": "object",
          "properties": {
            "accountId": { "type": "string" },
            "name": { "type": "string" },
            "ico": { "type": "string" },
            "segment": { "type": "string", "enum": ["enterprise", "mid_market", "smb"] },
            "ownerUpn": { "type": "string", "format": "email" },
            "contractStatus": { "type": "string", "enum": ["active", "suspended", "terminated", "prospect"] },
            "billingAddress": {
              "type": "object",
              "properties": { "street": { "type": "string" }, "city": { "type": "string" }, "zip": { "type": "string" }, "country": { "type": "string" } },
              "required": ["street", "city", "zip", "country"]
            }
          },
          "required": ["accountId", "name", "segment", "contractStatus"]
        },
        "annotations": { "title": "Detail zákaznického účtu", "readOnlyHint": true, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false }
      }
    ],
    "nextCursor": "eyJ2IjoiOWYyYTFjMy1iNGQiLCJvIjoxMDB9",
    "ttlMs": 300000,
    "cacheScope": "private"
  }
}
```

Kurzor je **neprůhledný řetězec** a klient nesmí předpokládat pevnou velikost stránky [10]. Naše implementace do něj kóduje `{ v: catalogVersionId, o: offset }` v base64url — tím je kurzor **vázaný na verzi katalogu**: pokud se mezi stránkami katalog přepne, gateway vrátí `-32602` (neplatný kurzor) a klient začne znovu. Tím se vyloučí stránkování napříč dvěma verzemi katalogu, které by agentovi ukázalo nekonzistentní sadu nástrojů.

Řazení je **deterministické** (lexikograficky podle `name`), což spec doporučuje kvůli cachování a zásahovosti prompt cache modelu [4].

### 4.5 `tools/call` — úspěch se `structuredContent`

```http
POST /mcp/acme HTTP/1.1
Authorization: Bearer eyJ0eXAi...
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: finance_list_invoices
Idempotency-Key: 01JB9Z2K4P7Q1M
Accept: application/json, text/event-stream

{
  "jsonrpc": "2.0",
  "id": 12,
  "method": "tools/call",
  "params": {
    "name": "finance_list_invoices",
    "arguments": { "accountId": "ACC-000123", "status": "overdue", "limit": 3 },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "progressToken": "p-12",
      "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
    }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 12,
  "result": {
    "resultType": "complete",
    "content": [
      { "type": "text", "text": "{\"invoices\":[{\"invoiceId\":\"INV-2026-004411\",\"issuedOn\":\"2026-06-30\",\"dueOn\":\"2026-07-30\",\"amount\":128400.00,\"currency\":\"CZK\",\"status\":\"overdue\",\"daysOverdue\":49},{\"invoiceId\":\"INV-2026-004987\",\"issuedOn\":\"2026-07-31\",\"dueOn\":\"2026-08-30\",\"amount\":96250.50,\"currency\":\"CZK\",\"status\":\"overdue\",\"daysOverdue\":18}],\"totalCount\":2,\"totalAmount\":224650.50,\"currency\":\"CZK\"}" }
    ],
    "structuredContent": {
      "invoices": [
        { "invoiceId": "INV-2026-004411", "issuedOn": "2026-06-30", "dueOn": "2026-07-30", "amount": 128400.00, "currency": "CZK", "status": "overdue", "daysOverdue": 49 },
        { "invoiceId": "INV-2026-004987", "issuedOn": "2026-07-31", "dueOn": "2026-08-30", "amount": 96250.50, "currency": "CZK", "status": "overdue", "daysOverdue": 18 }
      ],
      "totalCount": 2,
      "totalAmount": 224650.50,
      "currency": "CZK"
    },
    "isError": false,
    "_meta": {
      "io.modelcontextprotocol/serverInfo": { "name": "acme-mcp-gateway", "version": "1.4.2" },
      "com.contoso.mcpgw/catalogVersionId": "9f2a1c3-b4d",
      "com.contoso.mcpgw/truncated": false
    }
  }
}
```

**Textový blok je duplicitní serializace `structuredContent`.** Spec to označuje za doporučené chování kvůli zpětné kompatibilitě [4]; ChatGPT to navíc u konektorové cesty explicitně vyžaduje jako JSON-encoded text blok vedle `structuredContent` [13]. Naše gateway to dělá **vždy**, řízeno feature flagem `features.structuredContentEverywhere`.

Vlastní `_meta` klíče používají prefix `com.contoso.mcpgw/`. Podle spec jsou rezervované prefixy `io.modelcontextprotocol/` a obecně jakýkoli klíč, jehož **druhý label je `mcp`** (např. `io.mcp.*`) — vlastní rozšíření proto nesmí používat `io.mcp.gateway/…` ani žádnou jinou podobu s druhým labelem `mcp`. `com.contoso.mcpgw/` je v pořádku, protože druhý label je `contoso` [7]. Kanonické klíče tohoto rozšíření: `com.contoso.mcpgw/contentTrust`, `com.contoso.mcpgw/retryAfterMs`, `com.contoso.mcpgw/catalogVersionId`, `com.contoso.mcpgw/policyDecisionId`, `com.contoso.mcpgw/requiresConfirmation`, plus další klíče pod stejným prefixem (`errorCode`, `requiredRoles`, `truncated`, `sunsetInDays` — viz příklady níže).

### 4.6 `tools/call` — `isError: true` (business chyba)

```json
{
  "jsonrpc": "2.0",
  "id": 13,
  "result": {
    "resultType": "complete",
    "content": [
      { "type": "text", "text": "Účet ACC-000999 v CRM neexistuje. Zkontroluj identifikátor, nebo účet vyhledej nástrojem crm_search_accounts podle názvu či IČO." }
    ],
    "isError": true,
    "_meta": { "com.contoso.mcpgw/errorCode": "DOWNSTREAM_NOT_FOUND" }
  }
}
```

Chyba provedení nástroje patří **dovnitř úspěšného výsledku**, protože z ní model může vyvodit další krok a opravit se [4]. Text je proto psaný pro model: co se stalo + co s tím. Nikdy neobsahuje interní URL, ID požadavku do backendu ani stack trace ([`./03-kodova-architektura.md`](./03-kodova-architektura.md#83-co-se-nikdy-nesmí-dostat-ven)).

### 4.7 `tools/call` — odmítnuto autorizací

```json
{
  "jsonrpc": "2.0",
  "id": 14,
  "result": {
    "resultType": "complete",
    "content": [
      { "type": "text", "text": "Nemáš oprávnění volat nástroj finance_post_journal. Vyžaduje role Mcp.Approver a Mcp.Domain.Finance.Write ve workspace acme. Požádej o ně správce přes službu ACME Access Request. Čtecí operace nad fakturami jsou dostupné nástrojem finance_list_invoices." }
    ],
    "isError": true,
    "_meta": {
      "com.contoso.mcpgw/errorCode": "toolset_not_mapped",
      "com.contoso.mcpgw/requiredRoles": ["Mcp.Approver", "Mcp.Domain.Finance.Write"]
    }
  }
}
```

HTTP stav je **`200`**, ne `403`. Toto je vědomé rozhodnutí architektonických rozhodnutí v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md) a stojí za zdůvodnění: `403` v OAuth kontextu znamená „nedostatečný scope" a spouští v klientovi step-up autorizaci [1]. Odmítnutí konkrétního nástroje na základě **entitlementu** ale není scope problém — reautorizace ho nevyřeší, protože přiřazení role je administrativní úkon. Kdyby gateway vrátila `403`, klient by zahájil zbytečný OAuth kolotoč a uživatel by neuviděl větu, která mu řekne, co má udělat. `matchedRuleId` ani `principal.oid` se do odpovědi nedostanou — jsou v auditu.

### 4.8 Chyba nepodporované verze

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
```
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32022,
    "message": "Unsupported protocol version",
    "data": {
      "supported": ["2026-07-28", "2025-11-25", "2025-06-18"],
      "requested": "2025-03-26"
    }
  }
}
```

A pro neshodu hlavičky a těla:

```http
HTTP/1.1 400 Bad Request
```
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32020,
    "message": "Header mismatch: Mcp-Name header value 'crm_get_account' does not match body value 'crm_search_accounts'"
  }
}
```

Tvar odpovídá příkladu ve spec [2]. Text zprávy smí obsahovat názvy nástrojů, protože je klient poslal sám.

### 4.9 `401` odpověď včetně `WWW-Authenticate`

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token", error_description="Authorization required", resource_metadata="https://mcp.contoso.com/.well-known/oauth-protected-resource/mcp/acme", scope="mcp.invoke"
Content-Type: application/json
Cache-Control: no-store
x-request-id: 01JB9Z2K4P7Q1M

{ "error": "invalid_token", "error_description": "Authorization required" }
```

**Tělo této odpovědi není JSON-RPC error objekt.** Chybějící nebo neplatný token se ověřuje v kroku 6 pipeline (§2.1), tedy dřív, než gateway vůbec ví, jestli šlo o platný JSON-RPC request — přidělit mu JSON-RPC chybový kód (natož z rozsahu `-32000`…`-32019`, který je legacy a nové implementace ho nemají alokovat) by naznačovalo protokolovou chybu tam, kde jde o autentizaci na úrovni HTTP. Tělo výše je proto obyčejný OAuth/RFC 6750 chybový tvar (`error`/`error_description`), ne JSON-RPC.

Parametr `resource_metadata` zavádí RFC 9728 §5.1 [3]; parametr `scope` doporučuje MCP autorizační spec podle RFC 6750 §3 [1]. Anthropic k tomu přidává provozní požadavek: server **musí vždy vrátit `401`** (ne `200`) s touto hlavičkou, jinak Claude PRM nenajde a spadne na sondování well-known cest, což WAF nebo deploy target neservírující kořen může tiše rozbít [5][13].

Odpovídající PRM dokument:

```http
GET /.well-known/oauth-protected-resource/mcp/acme HTTP/1.1
```
```json
{
  "resource": "https://mcp.contoso.com/mcp",
  "authorization_servers": ["https://login.microsoftonline.com/<tenant-id>/v2.0"],
  "scopes_supported": ["mcp.invoke"],
  "bearer_methods_supported": ["header"],
  "resource_name": "ACME MCP Gateway — workspace acme",
  "resource_documentation": "https://intranet.acme.example/mcp-gateway"
}
```

**`resource` je na per-workspace PRM cestě stejné jako na výchozí** (`https://mcp.contoso.com/mcp`) — gateway má jeden Application ID URI a jednu app registration (§4.1); workspace se rozlišuje jen cestou dokumentu (`/mcp/{workspaceSlug}`) a případně odlišným `resource_name`/`resource_documentation`, nikdy odlišnou hodnotou `resource`. Kdyby se `resource` mezi workspacy lišil, vyžadovalo by to samostatnou audience a tedy samostatnou app registration per workspace — to K1/ADR-015 zakazuje (viz §4.1 bod 3).

Tři věci, na kterých to v Entra ID padá (ověřeno [13][18], AADSTS9010010 doloženo z dokumentace Anthropicu [5]):

1. `authorization_servers` **musí být v2.0 issuer** (`.../v2.0`), protože ten odpovídá claimu `iss` ve v2 access tokenech; v1 issuer `sts.windows.net/...` s v2 metadaty nesedí.
2. Claude použije **jen první položku** `authorization_servers` a nefallbackuje — primární issuer musí být první.
3. URL MCP serveru musí být registrovaná jako **Application ID URI na ověřené vlastní doméně**; `*.azurewebsites.net` nestačí a token request selže s `AADSTS9010010`.

`offline_access` do `scopes_supported` **nepatří** — spec to výslovně nedoporučuje, refresh token není požadavek resource serveru [1]. Claude si `offline_access` přidá sám, pokud ho AS metadata inzerují [5].

**Jediný delegovaný scope: `mcp.invoke`.** Granularita volání (co konkrétní principal smí zavolat) se neřeší přes OAuth scopy, ale přes app role a `PolicyEngine` (ADR-007, §5 tohoto dokumentu). Samostatné scopy typu `tools.read`/`tools.execute` byly zvažovány a zamítnuty ze tří důvodů: (1) **exploze počtu scopů** — s desítkami až stovkami nástrojů v katalogu by jemnější granularita znamenala buď jeden scope na skupinu nástrojů (a tedy duplicitu s app rolemi), nebo jeden scope na nástroj (nespravovatelné v Entra); (2) **nový admin consent při každé změně katalogu** — přidání nástroje do nové kategorie by vyžadovalo novou scope definici a nový consent flow pro všechny connectory, což by vázalo publikaci katalogu na cyklus schvalování v Entra; (3) **scopy nejsou dynamické** — jsou udělené při consentu a platí po celou dobu platnosti tokenu/session, zatímco entitlements (a tedy co principal smí volat) se mohou měnit mezi jednotlivými `tools/call` bez nutnosti nový token vůbec žádat. Proto scope `mcp.invoke` říká jen „tento principal smí vůbec s gateway mluvit" a o granularitu se stará výhradně `PolicyEngine` vyhodnocovaný na každém requestu (krok 12 pipeline, §5) — to umožňuje okamžitě promítnout zrušení/změnu entitlementu bez nutnosti re-consentu.

Odmítnutí konkrétního volání kvůli chybějícímu entitlementu proto **není** scope step-up — je to tool error `200`/`isError: true` s `DenyReason` v `_meta.errorCode` (§3.3 řádek 19, §5.5, §11.3, kanonická sada v [`03-kodova-architektura.md §4`](./03-kodova-architektura.md)), ne HTTP `403`/`insufficient_scope`. Řádek 13 tabulky v §3.3 (`insufficient_scope`) zůstává vyhrazen jen pro skutečnou absenci scope `mcp.invoke` na tokenu (např. špatně nakonfigurovaný klient, který si o scope vůbec nepožádal) — v běžném provozu s admin-consentovaným scope k tomu nedochází; entitlement problémy vždy jdou cestou tool error, ne HTTP step-upu.

### 4.10 `notifications/progress`

Klient si progres vyžádá `progressToken` v `_meta` požadavku [12]:

```http
POST /mcp/acme HTTP/1.1
Authorization: Bearer eyJ0eXAi...
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: finance_export_ledger

{ "jsonrpc": "2.0", "id": 21, "method": "tools/call",
  "params": { "name": "finance_export_ledger", "arguments": { "period": "2026-08", "confirm": true },
    "_meta": { "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "progressToken": "exp-21" } } }
```

Odpověď je SSE stream scoped na tento jediný požadavek [2]:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
X-Accel-Buffering: no
Cache-Control: no-store

event: message
data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"exp-21","progress":0.15,"total":1,"message":"Načítám hlavní knihu za 2026-08"}}

event: message
data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"exp-21","progress":0.62,"total":1,"message":"Zpracováno 6 200 z 10 000 řádků"}}

event: message
data: {"jsonrpc":"2.0","id":21,"result":{"resultType":"complete","content":[{"type":"text","text":"Export dokončen: 10 000 řádků, 4,2 MB."}],"structuredContent":{"rowCount":10000,"sizeBytes":4404019,"downloadUrl":"https://…"},"isError":false}}
```

Hodnota `progress` **musí monotónně růst**, i když `total` není známý; `progress` i `total` smí být desetinné; obě strany **SHOULD** notifikace rate-limitovat [12]. Naše pravidlo je v [`./03-kodova-architektura.md`](./03-kodova-architektura.md#105-backpressure). Finální odpověď stream ukončuje [2].

### 4.11 `notifications/cancelled`

**Éra A** — klient pošle notifikaci s ID rušeného požadavku [11]:

```json
{ "jsonrpc": "2.0", "method": "notifications/cancelled",
  "params": { "requestId": 21, "reason": "User requested cancellation" } }
```
```http
HTTP/1.1 202 Accepted
```

Server má zpracování zastavit, uvolnit zdroje a **neposílat odpověď** na zrušený požadavek; notifikaci na neznámý nebo už dokončený požadavek smí ignorovat. `initialize` se rušit nesmí [11].

**Éra B** — `notifications/cancelled` na Streamable HTTP **neexistuje**. Zrušením je samo **zavření SSE response streamu**, což server **MUST** takto interpretovat a nesmí pro daný požadavek poslat už žádnou zprávu [2]. Gateway to implementuje tak, že `res.on('close')` abortuje `AbortController` z `ExecutionContext`; downstream volání dostane `AbortSignal` a audit se zapíše s `outcome: 'cancelled'`.

**Praktický dopad:** u éry B nelze zrušit požadavek, který server odpověděl jako jeden JSON objekt (ne stream). Nástroje s `riskTier: high` a delším během proto gateway odpovídá **vždy** SSE streamem, i když nemá co poslat před finální odpovědí — jen tak zůstane zrušení možné.

---

## 5. Design nástrojů

### 5.1 Pojmenování

Konvence z architektonických rozhodnutí v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md): **`{domain}_{action}_{object}`**, snake_case, max 64 znaků. Typ zdroje (skill vs. connector) je interní detail a **do jména nepatří** — agent nemá důvod ho vědět a při migraci nástroje z connectoru na skill by se změnilo jméno.

Spec doporučuje 1–128 znaků, case-sensitive, jen `A-Za-z0-9_.-`, bez mezer a čárek [4]. Náš strop 64 znaků je přísnější záměrně: jména se dostávají do kontextu modelu při každém `tools/list` a delší jména platíme tokeny.

| Doména | Prefix | Příklad |
|---|---|---|
| CRM | `crm_` | `crm_get_account`, `crm_search_accounts` |
| Fakturace | `finance_` | `finance_list_invoices`, `finance_post_journal` |
| Prodej | `sales_` | `sales_list_orders`, `sales_create_quote` |
| Personalistika | `hr_` | `hr_get_employee_profile` |
| IT služby | `itsm_` | `itsm_create_incident` |
| Znalostní báze | `kb_` | `kb_search_articles` |

Slovesa jsou uzavřená množina: `get` (jeden záznam podle ID), `search` (fulltext), `list` (filtrovaný výčet), `create`, `update`, `cancel`, `export`, `post` (nevratné zaúčtování). Uzavřená množina je vynucená linterem — agent se pak orientuje podle vzoru, ne podle jednotlivých popisů.

**Kolize.** Spec upozorňuje, že proxy agregující nástroje z více serverů může narazit na kolize a měla by mít strategii disambiguace [4]. U nás ji řeší právě doménový prefix; ingestor odmítne dvě definice se stejným `name` v jedné `CatalogVersion`.

### 5.2 Jak psát `description` pro LLM

`description` je **jediná dokumentace, kterou model má**. Nečte README, nevidí OpenAPI. Struktura, kterou vyžaduje náš linter (a která odpovídá doporučením OpenAI pro developer mode: action-oriented jména, styl „Use this when…", zdokumentované hraniční případy, enumerované parametry [13]):

1. **Věta 1 — co nástroj dělá.** Konkrétně, v činném rodě, se jménem systému. „Vrátí seznam faktur zákaznického účtu z fakturačního systému."
2. **Věta 2 — kdy ho použít.** „Použij, když se uživatel ptá na neuhrazené částky, splatnost nebo historii fakturace konkrétního zákazníka."
3. **Věta 3 — kdy ho nepoužít a čím ho nahradit.** „Nevrací objednávky ani smlouvy; na ně jsou `sales_list_orders` a `crm_get_contract`."
4. **Věta 4 — hraniční případy a chování při chybě.** „Vrací prázdný seznam, pokud účet faktury nemá. Vrací chybu, pokud účet neexistuje."

**Co do `description` nepatří:**

- Informace už vyjádřená ve schématu (typy, povinnost, rozsahy) — je v `inputSchema` a duplikace jen stojí tokeny.
- Implementační detaily: název APIM API, endpoint, technologie backendu, verze.
- Bezpečnostní instrukce typu „Nevolej tento nástroj, pokud uživatel nemá oprávnění." Autorizace je věc `PolicyEngine`, ne promptu. Instrukce v popisu je jen návrh, který lze obejít.
- Marketing a zdvořilostní fráze.
- Cokoli, co vypadá jako pokyn modelu měnit chování mimo tento nástroj — popis je **nedůvěryhodný vstup**, spec klientům nařizuje považovat anotace z neověřených serverů za nedůvěryhodné [4] a totéž platí pro naše zacházení s texty ze skill repozitáře.

**Limity velikosti** (rozhodnutí projektu, spec je nestanovuje):

| Pole | Limit | Vynucení |
|---|---|---|
| `name` | 64 znaků | linter, ingestor |
| `title` | 60 znaků | linter |
| `description` | **600 znaků** (cíl 250–450) | linter |
| `inputSchema` — `description` u vlastnosti | 200 znaků | linter |
| Součet `description` všech nástrojů v jednom tool setu | **12 000 znaků** | ingestor, varování od 9 000 |

### 5.3 `title` vs. `name`, povinné `annotations`

Pořadí zobrazovaného jména je podle schématu: **`title` → `annotations.title` → `name`** [8]. Duplikace `title` na dvou místech je historická; náš ingestor vyplňuje **obě** stejnou hodnotou, aby se klient chovající se podle staršího pravidla zobrazil správně.

Anotace jsou ve spec **volitelné hinty s defaulty** — `readOnlyHint: false`, `destructiveHint: true` (významné jen když `readOnlyHint == false`), `idempotentHint: false`, `openWorldHint: true` [4][8]. U nás jsou **všechny čtyři povinné** v manifestu; Anthropic vyžaduje deklaraci `readOnlyHint` a `destructiveHint` u každého nástroje [17] a OpenAI podle `readOnlyHint` rozhoduje, zda před spuštěním vyžádat potvrzení zápisu — neoznačené nástroje se implicitně chovají jako zapisovací [13].

Anotace mají u nás **druhý, tvrdý význam**: řídí idempotenční cache a retry pravidla (ADR-016, [`./03-kodova-architektura.md`](./03-kodova-architektura.md#102-retry)). Jsou tedy zároveň hint pro klienta i konfigurace gateway — proto je ingestor kontroluje proti HTTP metodě bindingu (`GET` ⇒ `readOnlyHint: true`; `DELETE` ⇒ `destructiveHint: true`) a nesoulad odmítne.

**`riskTier` a povinné potvrzení (`confirm`).** Kanonická enumerace je čtyřhodnotová, lowercase: `riskTier: "low" | "medium" | "high" | "critical"`. Hodnota `destructive` **není** platný `riskTier` a v tomto katalogu se nevyskytuje. Vynucení parametru `confirm` je povinné **právě když**:

> `annotations.destructiveHint === true` **NEBO** `riskTier ∈ {"high", "critical"}`.

Obě podmínky jsou nezávislé — nástroj s `destructiveHint: false` a `riskTier: high` (např. `finance_export_ledger`, čtecí, ale citlivý a nákladný) potvrzení vyžaduje stejně jako nástroj s `destructiveHint: true` a `riskTier: critical`. Naopak zapisovací nástroj s `destructiveHint: false` a `riskTier: medium` (např. `sales_create_quote`, `itsm_create_incident`) potvrzení **nevyžaduje** — vratný zápis se středním rizikem stačí autorizovat rolí `Mcp.Operator`, bez dalšího kroku. Mapování rizika na roli: `low`/`medium` → `Mcp.Operator`, `high`/`critical` → `Mcp.Approver`.

### 5.4 `inputSchema` a `outputSchema`

`inputSchema` je povinné a **nesmí být `null`**; pro bezparametrový nástroj je doporučený tvar `{ "type": "object", "additionalProperties": false }` [4]. Výchozí dialekt je JSON Schema **2020-12**, pokud není uvedeno `$schema` [4].

`outputSchema` je volitelné, ale u nás **povinné pro každý nástroj vracející strukturovaná data** (feature flag `structuredContentEverywhere`). Důvody: server pak **MUST** dodat odpovídající `structuredContent` a klient ho **SHOULD** validovat [4]; ChatGPT u konektorové cesty explicitně doporučuje deklarovat output schema [13]; a pro nás je to strojová kontrola, že mapování odpovědi z APIM nevypadlo ze tvaru.

**`$ref`:** implementace **MUST NOT** automaticky dereferencovat síťové `$ref` [4]. Naše schémata jsou proto plně inline, bez `$defs` ([`./03-kodova-architektura.md`](./03-kodova-architektura.md#5-zod-schémata-jeden-zdroj-dva-výstupy)).

**`x-mcp-header`:** rozšiřující klíč, který nechá klienta zrcadlit hodnotu parametru do hlavičky `Mcp-Param-{Name}` pro směrování bez parsování těla [4]. Omezení: neprázdný token dle RFC 9110, bez řídicích znaků, case-insensitive unikátní v rámci schématu, jen primitivní typy (`string`, `integer`, `boolean` — **`number` ne**), celá čísla v bezpečném rozsahu IEEE754, a vlastnost musí být **staticky dosažitelná** řetězcem klíčů `properties` bez `$ref`/`oneOf`/`items`/podmínek [4]. Klient, který to poruší, **MUST** nástroj z `tools/list` vyřadit. Spec zároveň varuje, aby se takto neoznačovaly citlivé parametry, protože hlavičky vidí zprostředkovatelé [4].

**Rozhodnutí projektu:** `x-mcp-header` používáme **pouze** pro parametr `region` u nástrojů s regionálně dělenými backendy. Nepoužíváme ho pro `accountId`, `employeeId` ani nic, co je osobním údajem nebo obchodním identifikátorem.

### 5.5 Počet nástrojů a dopad na kontext

Každý nástroj v `tools/list` stojí místo v kontextu modelu při **každém** volání. Ani jeden dodavatel nezveřejňuje tvrdý limit počtu nástrojů (⚠️ NEOVĚŘENO pro obě platformy [13]), ale dopad je měřitelný: model s 200 nástroji vybírá hůř než se 30.

Hrubý odhad nákladu (⚠️ TODO — změřit na reálném katalogu): nástroj s `title`, 400znakovým popisem a schématem o 5 vlastnostech ≈ **250–400 tokenů**. Katalog o 60 nástrojích tedy ≈ 15–24 tisíc tokenů v každém požadavku.

**Strategie „tool set per role" (ADR-007 + ADR-015).** `tools/list` **nevrací katalog** — vrací **projekci katalogu přes entitlementy volajícího**. Spec to výslovně připouští: sada nástrojů **MAY** záviset na autorizaci předložené na požadavku, protože přihlašovací údaje jsou per-request vstup, ne stav spojení [4].

Sloupec „Entra role" ukazuje **průnik** funkční role (`Mcp.Reader`/`Mcp.Operator`/`Mcp.Approver`/`Mcp.CatalogAdmin`) a doménové role (`Mcp.Domain.<Domena>.<Read|Write>`) — efektivní oprávnění je vždy průnik obou, viz [`./07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md). Doménová jména se řídí stejnou PascalCase konvencí i mimo domény `Crm`/`Finance`/`Hr` — zde rozšířeno o `Sales` a `Itsm`.

| Persona (příklad) | Entra role | Tool set | Počet nástrojů | Odhad tokenů |
|---|---|---|---|---|
| Obchodník — čtení | `Mcp.Reader` + `Mcp.Domain.Sales.Read` | `crm_*` read + `kb_*` | 8 | ~2 500 |
| Obchodník — zápis | `Mcp.Operator` + `Mcp.Domain.Sales.Write` + `Mcp.Domain.Crm.Write` | + `sales_create_quote`, `crm_update_account_owner` | 11 | ~3 600 |
| Fakturace — čtení | `Mcp.Reader` + `Mcp.Domain.Finance.Read` | `finance_*` read + `crm_get_account` | 7 | ~2 200 |
| Fakturace — schvalování | `Mcp.Approver` + `Mcp.Domain.Finance.Write` | + `finance_post_journal` (`riskTier: critical`), `finance_export_ledger` (`riskTier: high`) | 9 | ~3 000 |
| Podpora | `Mcp.Operator` + `Mcp.Domain.Itsm.Write` | `itsm_*`, `kb_*`, `crm_get_account` | 9 | ~2 900 |
| Správce katalogu | `Mcp.CatalogAdmin` | vše + admin nástroje | 40+ | ~14 000 |

Fakturační schvalovací persona potřebuje `Mcp.Approver`, ne `Mcp.Operator`: `finance_post_journal` je `riskTier: critical` a `finance_export_ledger` je `riskTier: high`, a mapování rizika na roli (§5.3) je `low`/`medium` → `Mcp.Operator`, `high`/`critical` → `Mcp.Approver`.

Tím je snižování kontextu **vedlejším produktem deny-by-default autorizace**, ne samostatným mechanismem — a je nemožné je obejít, protože nejde o filtr v UI, ale o autorizační rozhodnutí. Sekundární mechanismy: Claude Messages API umí `defer_loading` + Tool Search Tool pro velké sady [13]; `mcp_toolset.configs` umožňuje allowlist/denylist per server [13]. Obojí je na straně klienta a bere se jako doplněk, ne náhrada.

---

## 6. Reprezentativní katalog nástrojů

Dvanáct nástrojů jako vzor pro interní IT. Sloupec „kind" odlišuje `connector` (HTTP na APIM) od `skill` (deklarativní transformace nad jiným nástrojem, ADR-013).

| # | `name` | kind | R/W | `riskTier` | `readOnly` | `destructive` | `idempotent` | Entitlement tag |
|---|---|---|---|---|---|---|---|---|
| 1 | `crm_get_account` | connector | R | low | ✔ | ✘ | ✔ | `crm.read` |
| 2 | `crm_search_accounts` | connector | R | low | ✔ | ✘ | ✔ | `crm.read` |
| 3 | `crm_update_account_owner` | connector | W | medium | ✘ | ✘ | ✔ | `crm.write` |
| 4 | `finance_list_invoices` | connector | R | low | ✔ | ✘ | ✔ | `finance.read` |
| 5 | `finance_get_invoice_pdf` | connector | R | medium | ✔ | ✘ | ✔ | `finance.read` |
| 6 | `finance_post_journal` | connector | W | **critical** | ✘ | **✔** | ✘ | `finance.post` |
| 7 | `finance_export_ledger` | connector | R | high | ✔ | ✘ | ✔ | `finance.export` |
| 8 | `sales_list_orders` | connector | R | low | ✔ | ✘ | ✔ | `sales.read` |
| 9 | `sales_create_quote` | connector | W | medium | ✘ | ✘ | ✘ | `sales.write` |
| 10 | `itsm_create_incident` | connector | W | medium | ✘ | ✘ | ✘ | `itsm.write` |
| 11 | `kb_search_articles` | connector | R | low | ✔ | ✘ | ✔ | `kb.read` |
| 12 | `crm_account_health_summary` | **skill** | R | low | ✔ | ✘ | ✔ | `crm.read` |

Úplné definice (výtah `tools/list`; nástroje 1 a 4 jsou v §4.4 a §4.5, zde bez opakování):

```json
[
  { "name": "crm_search_accounts", "title": "Hledání zákaznických účtů",
    "description": "Vyhledá zákaznické účty v CRM podle části názvu, IČO nebo města. Použij, když uživatel zmíní zákazníka jménem a ty neznáš jeho accountId. Vrací zkrácený záznam; pro detail použij crm_get_account. Vrací prázdný seznam, pokud nic neodpovídá.",
    "inputSchema": { "type": "object", "properties": {
        "query": { "type": "string", "minLength": 2, "maxLength": 200, "description": "Část názvu, IČO nebo města. Bez zástupných znaků." },
        "segment": { "type": "string", "enum": ["enterprise", "mid_market", "smb"], "description": "Volitelný filtr segmentu." },
        "limit": { "type": "integer", "minimum": 1, "maximum": 50, "default": 20 },
        "cursor": { "type": "string", "maxLength": 512 } },
      "required": ["query"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "accounts": { "type": "array", "items": { "type": "object", "properties": {
              "accountId": { "type": "string" }, "name": { "type": "string" }, "ico": { "type": "string" },
              "segment": { "type": "string", "enum": ["enterprise","mid_market","smb"] }, "city": { "type": "string" } },
            "required": ["accountId","name","segment"] } },
        "totalCount": { "type": "integer", "minimum": 0 }, "nextCursor": { "type": "string" } },
      "required": ["accounts","totalCount"] },
    "annotations": { "title": "Hledání zákaznických účtů", "readOnlyHint": true, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false } },

  { "name": "crm_update_account_owner", "title": "Změna vlastníka účtu",
    "description": "Nastaví nového obchodního vlastníka zákaznického účtu v CRM. Použij, když uživatel výslovně žádá přepis odpovědnosti za zákazníka. Nemění smluvní ani fakturační údaje. Vrací chybu, pokud nový vlastník není aktivní uživatel s rolí obchodníka.",
    "inputSchema": { "type": "object", "properties": {
        "accountId": { "type": "string", "pattern": "^ACC-[0-9]{6}$" },
        "newOwnerUpn": { "type": "string", "format": "email", "maxLength": 254 },
        "reason": { "type": "string", "minLength": 3, "maxLength": 500, "description": "Důvod změny pro auditní stopu." } },
      "required": ["accountId","newOwnerUpn","reason"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "accountId": { "type": "string" }, "previousOwnerUpn": { "type": "string" },
        "newOwnerUpn": { "type": "string" }, "changedAt": { "type": "string", "format": "date-time" } },
      "required": ["accountId","newOwnerUpn","changedAt"] },
    "annotations": { "title": "Změna vlastníka účtu", "readOnlyHint": false, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false } },

  { "name": "finance_get_invoice_pdf", "title": "PDF faktury",
    "description": "Vrátí odkaz ke stažení PDF konkrétní faktury z fakturačního systému. Použij, když uživatel chce fakturu vidět nebo poslat. Vrací dočasný odkaz platný 15 minut, ne obsah souboru. Vrací chybu, pokud faktura neexistuje nebo ještě není vystavená.",
    "inputSchema": { "type": "object", "properties": {
        "invoiceId": { "type": "string", "pattern": "^INV-[0-9]{4}-[0-9]{6}$" } },
      "required": ["invoiceId"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "invoiceId": { "type": "string" }, "downloadUrl": { "type": "string", "format": "uri" },
        "expiresAt": { "type": "string", "format": "date-time" }, "sizeBytes": { "type": "integer", "minimum": 0 } },
      "required": ["invoiceId","downloadUrl","expiresAt"] },
    "annotations": { "title": "PDF faktury", "readOnlyHint": true, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false } },

  { "name": "finance_post_journal", "title": "Zaúčtování účetního zápisu",
    "description": "Zaúčtuje účetní zápis do hlavní knihy. Nevratná operace. Použij výhradně na explicitní pokyn uživatele, který uvede účty, částky a období. Součet stran MÁ DÁTI a DAL se musí rovnat. Vrací chybu, pokud je období uzavřené nebo zápis nevyvážený.",
    "inputSchema": { "type": "object", "properties": {
        "period": { "type": "string", "pattern": "^[0-9]{4}-(0[1-9]|1[0-2])$", "description": "Účetní období ve tvaru RRRR-MM." },
        "lines": { "type": "array", "minItems": 2, "maxItems": 200, "items": { "type": "object", "properties": {
              "account": { "type": "string", "pattern": "^[0-9]{6}$" },
              "side": { "type": "string", "enum": ["debit", "credit"] },
              "amount": { "type": "number", "exclusiveMinimum": 0, "maximum": 100000000 },
              "currency": { "type": "string", "enum": ["CZK", "EUR", "USD"] },
              "note": { "type": "string", "maxLength": 200 } },
            "required": ["account","side","amount","currency"], "additionalProperties": false } },
        "confirm": { "type": "boolean", "const": true, "description": "Musí být true. Potvrzení nevratné operace uživatelem." } },
      "required": ["period","lines","confirm"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "journalId": { "type": "string" }, "postedAt": { "type": "string", "format": "date-time" },
        "totalDebit": { "type": "number" }, "totalCredit": { "type": "number" } },
      "required": ["journalId","postedAt","totalDebit","totalCredit"] },
    "annotations": { "title": "Zaúčtování účetního zápisu", "readOnlyHint": false, "destructiveHint": true, "idempotentHint": false, "openWorldHint": false } },

  { "name": "finance_export_ledger", "title": "Export hlavní knihy",
    "description": "Vyexportuje hlavní knihu za zvolené období do souboru. Použij, když uživatel potřebuje data pro externí zpracování nebo audit. Operace trvá desítky sekund a hlásí průběh. Vrací chybu, pokud období přesahuje 12 měsíců.",
    "inputSchema": { "type": "object", "properties": {
        "periodFrom": { "type": "string", "pattern": "^[0-9]{4}-(0[1-9]|1[0-2])$" },
        "periodTo": { "type": "string", "pattern": "^[0-9]{4}-(0[1-9]|1[0-2])$" },
        "format": { "type": "string", "enum": ["csv", "xlsx"], "default": "csv" },
        "confirm": { "type": "boolean", "const": true } },
      "required": ["periodFrom","periodTo","confirm"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "downloadUrl": { "type": "string", "format": "uri" }, "expiresAt": { "type": "string", "format": "date-time" },
        "rowCount": { "type": "integer", "minimum": 0 }, "sizeBytes": { "type": "integer", "minimum": 0 } },
      "required": ["downloadUrl","expiresAt","rowCount"] },
    "annotations": { "title": "Export hlavní knihy", "readOnlyHint": true, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false } },

  { "name": "sales_list_orders", "title": "Objednávky zákazníka",
    "description": "Vrátí objednávky zákaznického účtu z prodejního systému, volitelně filtrované stavem a obdobím. Použij při dotazech na to, co si zákazník objednal a v jakém je to stavu. Nevrací faktury, na ně je finance_list_invoices. Vrací prázdný seznam, pokud zákazník objednávky nemá.",
    "inputSchema": { "type": "object", "properties": {
        "accountId": { "type": "string", "pattern": "^ACC-[0-9]{6}$" },
        "status": { "type": "string", "enum": ["draft","confirmed","shipped","delivered","cancelled"] },
        "orderedAfter": { "type": "string", "format": "date" },
        "limit": { "type": "integer", "minimum": 1, "maximum": 100, "default": 25 },
        "cursor": { "type": "string", "maxLength": 512 } },
      "required": ["accountId"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "orders": { "type": "array", "items": { "type": "object", "properties": {
              "orderId": { "type": "string" }, "orderedOn": { "type": "string", "format": "date" },
              "status": { "type": "string", "enum": ["draft","confirmed","shipped","delivered","cancelled"] },
              "totalAmount": { "type": "number" }, "currency": { "type": "string" } },
            "required": ["orderId","orderedOn","status","totalAmount","currency"] } },
        "totalCount": { "type": "integer", "minimum": 0 }, "nextCursor": { "type": "string" } },
      "required": ["orders","totalCount"] },
    "annotations": { "title": "Objednávky zákazníka", "readOnlyHint": true, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false } },

  { "name": "sales_create_quote", "title": "Vytvoření cenové nabídky",
    "description": "Vytvoří koncept cenové nabídky pro zákaznický účet. Použij, když uživatel chce připravit nabídku s konkrétními položkami. Nabídka vzniká jako koncept, neodesílá se zákazníkovi. Vrací chybu, pokud je některý SKU neplatný nebo účet nemá aktivní smlouvu.",
    "inputSchema": { "type": "object", "properties": {
        "accountId": { "type": "string", "pattern": "^ACC-[0-9]{6}$" },
        "validUntil": { "type": "string", "format": "date" },
        "items": { "type": "array", "minItems": 1, "maxItems": 100, "items": { "type": "object", "properties": {
              "sku": { "type": "string", "pattern": "^[A-Z0-9-]{3,32}$" },
              "quantity": { "type": "integer", "minimum": 1, "maximum": 100000 },
              "discountPercent": { "type": "number", "minimum": 0, "maximum": 40 } },
            "required": ["sku","quantity"], "additionalProperties": false } },
        "note": { "type": "string", "maxLength": 1000 } },
      "required": ["accountId","items"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "quoteId": { "type": "string" }, "status": { "type": "string", "enum": ["draft"] },
        "totalAmount": { "type": "number" }, "currency": { "type": "string" },
        "createdAt": { "type": "string", "format": "date-time" } },
      "required": ["quoteId","status","totalAmount","currency","createdAt"] },
    "annotations": { "title": "Vytvoření cenové nabídky", "readOnlyHint": false, "destructiveHint": false, "idempotentHint": false, "openWorldHint": false } },

  { "name": "itsm_create_incident", "title": "Založení incidentu",
    "description": "Založí incident v systému ITSM. Použij, když uživatel hlásí poruchu nebo žádá o pomoc s IT službou. Vždy nejprve zkus najít řešení nástrojem kb_search_articles. Vrací chybu, pokud zvolená služba neexistuje.",
    "inputSchema": { "type": "object", "properties": {
        "serviceId": { "type": "string", "pattern": "^SVC-[0-9]{4}$" },
        "summary": { "type": "string", "minLength": 10, "maxLength": 200 },
        "description": { "type": "string", "minLength": 20, "maxLength": 4000 },
        "impact": { "type": "string", "enum": ["low","medium","high"], "default": "medium" } },
      "required": ["serviceId","summary","description"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "incidentId": { "type": "string" }, "state": { "type": "string" },
        "assignedGroup": { "type": "string" }, "createdAt": { "type": "string", "format": "date-time" } },
      "required": ["incidentId","state","createdAt"] },
    "annotations": { "title": "Založení incidentu", "readOnlyHint": false, "destructiveHint": false, "idempotentHint": false, "openWorldHint": false } },

  { "name": "kb_search_articles", "title": "Hledání ve znalostní bázi",
    "description": "Vyhledá články ve firemní znalostní bázi. Použij před založením incidentu a při dotazech na interní postupy. Vrací výňatky a odkazy, ne celé články. Vrací prázdný seznam, pokud nic neodpovídá.",
    "inputSchema": { "type": "object", "properties": {
        "query": { "type": "string", "minLength": 3, "maxLength": 300 },
        "space": { "type": "string", "enum": ["it","hr","finance","sales","all"], "default": "all" },
        "limit": { "type": "integer", "minimum": 1, "maximum": 20, "default": 5 } },
      "required": ["query"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "results": { "type": "array", "items": { "type": "object", "properties": {
              "id": { "type": "string" }, "title": { "type": "string" },
              "url": { "type": "string", "format": "uri" }, "snippet": { "type": "string" },
              "updatedAt": { "type": "string", "format": "date-time" } },
            "required": ["id","title","url","snippet"] } } },
      "required": ["results"] },
    "annotations": { "title": "Hledání ve znalostní bázi", "readOnlyHint": true, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false } },

  { "name": "crm_account_health_summary", "title": "Souhrn zdraví zákaznického vztahu",
    "description": "Sestaví souhrn stavu zákaznického vztahu: segment, stav smlouvy, počet a objem faktur po splatnosti a počet otevřených incidentů. Použij, když se uživatel ptá na celkový stav zákazníka nebo na rizikové účty. Nevolá se na více účtů najednou. Vrací chybu, pokud účet neexistuje.",
    "inputSchema": { "type": "object", "properties": {
        "accountId": { "type": "string", "pattern": "^ACC-[0-9]{6}$" } },
      "required": ["accountId"], "additionalProperties": false },
    "outputSchema": { "type": "object", "properties": {
        "accountId": { "type": "string" }, "name": { "type": "string" },
        "segment": { "type": "string", "enum": ["enterprise","mid_market","smb"] },
        "contractStatus": { "type": "string", "enum": ["active","suspended","terminated","prospect"] },
        "overdueInvoiceCount": { "type": "integer", "minimum": 0 },
        "overdueAmount": { "type": "number", "minimum": 0 },
        "currency": { "type": "string" },
        "openIncidentCount": { "type": "integer", "minimum": 0 },
        "healthScore": { "type": "string", "enum": ["green","amber","red"] } },
      "required": ["accountId","name","segment","contractStatus","overdueInvoiceCount","overdueAmount","healthScore"] },
    "annotations": { "title": "Souhrn zdraví zákaznického vztahu", "readOnlyHint": true, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false } }
]
```

Tři vzorové prvky, které se mají opakovat v každém dalším nástroji:

- **`confirm` s `"const": true`** u každého nástroje, kde `destructiveHint === true` nebo `riskTier ∈ {high, critical}` (§5.3) — ne u každého zapisovacího nástroje bez rozdílu. Model tak musí potvrzení uvést explicitně a nemůže ho „zapomenout"; JSON Schema ho vynutí dřív, než se cokoli zapíše (ADR-016).
- **`additionalProperties: false`** všude. Bez toho se do downstream volání dostane cokoli, co model vymyslí.
- **`pattern` místo volného `string`** u všech identifikátorů. Je to zároveň validace a nejlevnější obrana proti injection (§7).

---

## 7. Parametry a jejich validace

| Aspekt | Pravidlo | Vynucení |
|---|---|---|
| Povinné vs. volitelné | Povinné jen to, bez čeho nelze operaci provést. Vše ostatní volitelné s rozumným defaultem. | `required` v JSON Schema |
| Defaulty | Uvedené v `default` **a** popsané v `description` — model default nevidí vždy. | linter |
| Enumerace | Vždy, když je množina hodnot konečná. OpenAI to uvádí jako přímou pomoc při výběru nástroje [13]. | `enum` |
| Formáty | `date`, `date-time`, `email`, `uri` místo volného `string` | `format` |
| Limity délky | Každý `string` má `maxLength`; každé pole `maxItems`; každé číslo `minimum`/`maximum` | linter odmítne schéma bez limitu |
| Stránkování | `limit` s `maximum` a `cursor` s `maxLength: 512` | konvence |
| Neznámé vlastnosti | `additionalProperties: false` vždy | linter |

### 7.1 Kanonizace

Před validací i před předáním downstream se argumenty normalizují, deterministicky a ve stejném pořadí:

1. Unicode **NFC** normalizace všech řetězců.
2. Ořez okrajových bílých znaků; odstranění řídicích znaků `U+0000`–`U+001F` kromě `\t`, `\n`, `\r`.
3. Odstranění BOM a **bidi override** znaků (`U+202A`–`U+202E`, `U+2066`–`U+2069`) — ochrana proti záměně významu textu v logu i v UI.
4. Číselné řetězce na čísla jen tam, kde to schéma připouští; žádná implicitní koerce.
5. Setřídění klíčů objektu pro výpočet `argumentsHash` (audit) — samotné pořadí se downstream nemění.

Kanonizovaná podoba je **zároveň vstupem do `Idempotency-Key`**, aby `{"a":1,"b":2}` a `{"b":2,"a":1}` byly totéž volání.

### 7.2 Ochrana proti injection v parametrech předávaných downstream

Základní princip: **gateway nikdy neskládá downstream požadavek řetězením řetězců.**

| Místo, kam parametr jde | Riziko | Opatření |
|---|---|---|
| Cesta URL (`/accounts/{accountId}`) | path traversal, přepis routy | dosazení jen po validaci `pattern`, poté `encodeURIComponent`; `..`, `/`, `%2e%2e` odmítnuty už schématem |
| Query string | injection parametrů, duplicitní klíče | sestavení přes `URLSearchParams`, nikdy konkatenací; allowlist povolených klíčů z bindingu |
| HTTP hlavičky (`x-mcp-*`, `Mcp-Param-*`) | **CRLF injection / response splitting** | zákaz `\r` a `\n` už v kanonizaci; navíc kontrola proti RFC 9110 token syntaxi před odesláním [4] |
| Tělo JSON požadavku | injection do backendu | tělo se staví ze **validované** struktury, nikdy z textu; `requestMapping` je JSONata výraz, ne šablona řetězce |
| SQL/NoSQL v backendu | SQL injection | není naše vrstva, ale `pattern` + `enum` + `maxLength` dramaticky zužují prostor; ingestor odmítne parametr bez limitu |
| Zpětný tok do promptu modelu | **prompt injection z dat backendu** | výstup se nikdy neinterpretuje jako instrukce; `structuredContent` je strojový, textový blok je jeho serializace; obsah třetích stran se předává jako data, ne jako pokyny |

Poslední řádek je nejpodceňovanější: nejde o injection **do** downstreamu, ale **z** downstreamu zpět do modelu. Nástroj typu `kb_search_articles` vrací text, který mohl napsat kdokoli. Proto výsledky nikdy nevkládáme do `instructions` ani do `description`, jen do `content`/`structuredContent`, a v `description` nástroje je explicitní věta, že vrácené články jsou obsah, ne pokyny.

---

## 8. Downstream kontrakt na APIM

### 8.1 Tvar volání

```http
POST /crm/v2/accounts/ACC-000123/owner HTTP/1.1
Host: acme-apim.azure-api.net
Authorization: Bearer <OBO access token pro scope api://crm-api/.default>
Content-Type: application/json
Accept: application/json
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: acme=mcpgw:1
x-request-id: 01JB9Z2K4P7Q1M
x-mcp-tool: crm_update_account_owner
x-mcp-tool-version: 9f2a1c3-b4d
x-mcp-principal-oid: 11111111-1111-1111-1111-111111111111
x-mcp-workspace: acme
x-mcp-client-type: claude
Idempotency-Key: 01JB9Z2K4P7Q1M
User-Agent: acme-mcp-gateway/1.4.2

{ "newOwnerUpn": "jan.novak@acme.example", "reason": "Reorganizace regionu Sever" }
```

**Co se nikdy neposílá:** příchozí token klienta (ADR-008 — token passthrough je confused deputy), `upn` ani jméno uživatele (stačí `oid`, zbytek je v auditu gateway), argumenty mimo ty, které binding deklaruje.

### 8.2 Subscription key vs. OAuth

| Scénář | Mechanismus | Kdy |
|---|---|---|
| Downstream API rozumí uživatelskému kontextu | **OBO access token** v `Authorization` (ADR-008) | výchozí volba pro connectory |
| Downstream API je čistě systémové (bez per-user autorizace) | **Managed Identity** app-only token | reporting, číselníky |
| Legacy API v APIM, které umí jen předplatné | **subscription key** v `Ocp-Apim-Subscription-Key`, tajemství z Key Vaultu | jen s výjimkou schválenou bezpečností |

Subscription key je **vždy doplněk, nikdy náhrada autorizace** — identita uživatele se v tom případě předává hlavičkou `x-mcp-principal-oid` a autorizační rozhodnutí zůstává výhradně na gateway. APIM produkty a předplatná se používají pro kvóty a governance, ne pro authn (⚠️ převzato z rešerše [18]).

⚠️ **Provozní past APIM:** pokud je diagnostické logování zapnuté na globálním scope, musí být „Number of payload bytes to log" pro Frontend Response nastaveno na **0**, jinak APIM bufferuje odpověď a rozbije MCP streaming; stejně tak policy nesmí sahat na `context.Response.Body` [18]. To se týká i případu, kdy APIM stojí jen jako downstream — buffering se projeví na `finance_export_ledger` a dalších dlouhoběžících nástrojích.

### 8.3 Mapování HTTP stavů na MCP výsledky

| HTTP z APIM | MCP výsledek | `errorCode` | Text pro agenta (vzor) | Retry |
|---|---|---|---|---|
| `200`/`201`/`204` | `isError: false` + `structuredContent` | — | — | — |
| `400` | `isError: true` | `DOWNSTREAM_BAD_REQUEST` | „Backend odmítl požadavek jako neplatný: {mapovaný důvod}. Zkontroluj hodnoty parametrů." | ne |
| `401` | `isError: true` po **jednom** obnovení tokenu | `DOWNSTREAM_AUTH_FAILED` | „Ověření vůči backendu selhalo. Odpoj a znovu připoj konektor." | 1× po invalidaci token cache |
| `403` | `isError: true` | `DOWNSTREAM_FORBIDDEN` | „Nemáš v {systém} oprávnění k této operaci. Oprávnění se řídí tvým účtem v cílovém systému, ne nastavením gateway." | ne |
| `404` | `isError: true` | `DOWNSTREAM_NOT_FOUND` | „{Objekt} {id} v {systém} neexistuje." | ne |
| `409` | `isError: true` | `DOWNSTREAM_CONFLICT` | „Operaci nelze provést kvůli konfliktu stavu: {důvod}. Načti aktuální stav a zkus to znovu." | **nikdy** |
| `422` | `isError: true` | `DOWNSTREAM_UNPROCESSABLE` | „Backend data přijal, ale odmítl je zpracovat: {důvod}." | **nikdy** |
| `429` | `isError: true` + `_meta.retryAfterMs` | `RATE_LIMITED` | „Limit volání {systém} byl vyčerpán. Zkus to za {n} s." | ano, `Retry-After` má přednost |
| `500`, `502`, `503`, `504` | `isError: true` + `retryAfterMs` | `DOWNSTREAM_UNAVAILABLE` | „{Systém} je dočasně nedostupný. Zkus to za chvíli." | jen idempotentní (§10.2 dok. 03) |
| timeout bez odpovědi | `isError: true` + `retryAfterMs` | `DOWNSTREAM_TIMEOUT` | „{Systém} neodpověděl v časovém limitu. Operace mohla proběhnout — ověř stav, než ji zopakuješ." | jen idempotentní |
| otevřený circuit breaker | `isError: true` + `retryAfterMs` | `CIRCUIT_OPEN` | „{Systém} je dočasně vyřazen kvůli opakovaným chybám." | ne |

Sloupec „Text pro agenta" je vzor; `{mapovaný důvod}` pochází z **allowlistovaného** pole chybové odpovědi backendu (typicky `error.title` nebo `error.detail` dle RFC 9457), nikdy z celého těla. Jméno systému je z katalogu, ne z URL.

Formulace u timeoutu je záměrná: agent musí vědět, že **nevratná** operace mohla proběhnout. U nástrojů s `destructiveHint: true` a `idempotentHint: false` se retry nikdy neprovádí automaticky.

Timeouty a retry pravidla jsou v [`./03-kodova-architektura.md`](./03-kodova-architektura.md#10-resilience-patterns-v-kódu).

### 8.4 Diagram: OBO výměna tokenu

```mermaid
sequenceDiagram
    autonumber
    participant EX as Gateway execution
    participant TS as TokenService
    participant RC as Redis token cache
    participant MI as Managed Identity endpoint
    participant E as Microsoft Entra ID token endpoint
    participant AP as Azure API Management

    EX->>TS: acquireDownstreamToken pro principal oid a scope apim
    TS->>TS: Vypocet cache klice z hash oid, tid, scope a apiId
    TS->>RC: GET klic
    alt Cache hit a plati vic nez 300 sekund
        RC-->>TS: Ulozeny access_token a expiresAt
        TS-->>EX: Access token bez volani Entra
    else Cache miss nebo blizka expirace
        RC-->>TS: Prazdno nebo token pred expiraci
        TS->>MI: Zadost o client assertion pres federovanou managed identitu
        MI-->>TS: Client assertion JWT
        TS->>E: POST token grant_type jwt-bearer, assertion prichozi token, requested_token_use on_behalf_of, scope
        alt Vymena uspesna
            E-->>TS: Novy access_token pro APIM, expires_in, pripadne refresh_token
            TS->>RC: SET klic s TTL expires_in minus 300 sekund
            TS-->>EX: Access token
        else AADSTS50076 nebo interaction_required
            E-->>TS: Chyba vyzadujici interakci uzivatele
            TS-->>EX: Chyba DownstreamAuthRequired
            EX-->>EX: Prevod na tool error s pokynem znovu pripojit connector
        else AADSTS65001 chybejici consent
            E-->>TS: Chyba chybejiciho souhlasu na downstream API
            TS-->>EX: Chyba DownstreamConsentMissing
        end
    end
    EX->>AP: Volani operace s Authorization Bearer downstream token
    AP-->>EX: Odpoved backendu
    Note over TS,RC: Invalidace cache pri 401 z APIM a jeden opakovany pokus s cerstvym tokenem
```

**Co diagram říká.** OBO je nejdražší krok v cestě tool callu, pokud se dělá naivně — je to síťové volání do Entra na každé volání nástroje. Cache proto není optimalizace, ale podmínka splnění SLO (režie gateway < 150 ms p95, architektonických rozhodnutí v [`09-adr-a-zdroje.md`](./09-adr-a-zdroje.md)).

Tři mechanické detaily:

1. **Klíč cache zahrnuje `oid` + `tid` + `scope` + `apiId`.** Vynechání `scope` by vedlo k tomu, že token pro CRM API by se použil pro finanční API — přesně ta izolace audience, kvůli které se OBO dělá (ADR-008).
2. **Proaktivní okno 300 s.** Token se z cache nevydá, pokud mu zbývá méně než 5 minut; jinak by tool call dlouhý 60 s mohl narazit na expiraci uprostřed. Stejné okno používá Claude pro svoje tokeny vůči nám [5] — držíme symetrii, aby se chyby neschovávaly.
3. **OBO používá jen delegated scopes, ne app roles.** Role zůstávají vázané na uživatele a nepřenášejí se na aplikaci jednající jeho jménem (⚠️ převzato z rešerše [18]). Autorizační vrstva gateway proto **nemůže** být nahrazena kontrolou rolí v downstream API — role tam nedorazí. To je hlavní důvod, proč ADR-007 zavádí policy engine v gateway.

Chybové větve mají přesná mapování: `AADSTS50076`/`interaction_required` znamená, že uživatel musí projít MFA nebo Conditional Access znovu — agentovi se vrací pokyn odpojit a znovu připojit konektor. `AADSTS65001` znamená chybějící souhlas na downstream API a je to **administrativní**, ne uživatelská chyba; alert míří na platformní tým.

---

## 9. Mapování skill manifestu na MCP tool

### 9.1 YAML schéma manifestu

```yaml
# Skill manifest — verze schématu 1
apiVersion: mcp.acme.example/v1 # povinné, řídí interpretaci zbytku
kind: Tool # Tool | ToolSet
metadata:
  name: string # = MCP `name`, ^[a-z][a-z0-9]*(_[a-z0-9]+){1,5}$, max 64
  title: string # = MCP `title`, max 60
  owner: string # e-mail vlastnící skupiny, povinné
  domain: enum[crm, finance, sales, hr, itsm, kb]
  tags: [string] # volné štítky pro vyhledávání v katalogu
spec:
  description: string # = MCP `description`, max 600, struktura viz §5.2
  riskTier: enum[low, medium, high, critical]
  entitlementTags: [string] # ^[a-z]+\.[a-z]+$, min 1 — vazba na entitlements
  annotations: # všechny čtyři POVINNÉ (přísnější než spec)
    readOnlyHint: bool
    destructiveHint: bool
    idempotentHint: bool
    openWorldHint: bool
  input:
    schema: object # JSON Schema 2020-12, inline, bez $ref
  output:
    schema: object # JSON Schema 2020-12; povinné, pokud vrací data
  binding: # právě jedna z variant
    apim: # varianta A — connector
      apiId: string # ID API v APIM
      operationId: string
      method: enum[GET, POST, PUT, PATCH, DELETE]
      pathTemplate: string # /crm/v2/accounts/{accountId} — placeholdery = jména parametrů
      timeoutMs: integer # 1000..30000, nesmí překročit globální strop
      auth:
        mode: enum[obo, managed_identity, subscription_key]
        scope: string # pro obo: api://.../.default
        resource: string # pro managed_identity
        secretRef: string # pro subscription_key: název secretu z Key Vaultu
      requestMapping: string # volitelný JSONata výraz: argumenty -> tělo požadavku
      responseMapping: string # volitelný JSONata výraz: odpověď -> structuredContent
    transform: # varianta B — skill
      engine: enum[jsonata, jmespath]
      source: string # jméno jiného nástroje v témže katalogu
      expression: string # výraz, max 4000 znaků
  deprecation: # volitelné
    since: string # ISO datum
    removeAfter: string # ISO datum, min. since + 90 dní
    replacedBy: string # jméno nástupce
```

Bez ohledu na variantu platí ADR-013: **žádný uživatelský kód se v procesu gateway nespouští.** `transform` je deklarativní výraz, `apim` je HTTP volání na allowlistovaný backend.

### 9.2 Příklad 1 — read connector

```yaml
apiVersion: mcp.acme.example/v1
kind: Tool
metadata:
  name: finance_list_invoices
  title: Faktury zákazníka
  owner: finance-platform@acme.example
  domain: finance
  tags: [invoicing, receivables]
spec:
  description: >-
    Vrátí seznam faktur zákaznického účtu z fakturačního systému, volitelně
    filtrovaný stavem a obdobím. Použij, když se uživatel ptá na neuhrazené
    částky, splatnost nebo historii fakturace konkrétního zákazníka. Nevrací
    objednávky ani smlouvy, na ně jsou sales_list_orders a crm_get_account.
    Vrací prázdný seznam, pokud účet faktury nemá, a chybu, pokud účet neexistuje.
  riskTier: low
  entitlementTags: [finance.read]
  annotations:
    readOnlyHint: true
    destructiveHint: false
    idempotentHint: true
    openWorldHint: false
  input:
    schema:
      type: object
      additionalProperties: false
      required: [accountId]
      properties:
        accountId:
          type: string
          pattern: "^ACC-[0-9]{6}$"
          description: Identifikátor zákaznického účtu ve tvaru ACC-000123.
        status:
          type: string
          enum: [draft, issued, paid, overdue, cancelled]
          description: Filtr stavu faktury. Vynecháním se vrátí všechny stavy.
        issuedAfter:
          type: string
          format: date
          description: Vrátí jen faktury vystavené od tohoto data včetně.
        limit:
          type: integer
          minimum: 1
          maximum: 100
          default: 25
          description: Maximální počet faktur. Výchozí 25, strop 100.
        cursor:
          type: string
          maxLength: 512
          description: Neprůhledný kurzor z pole nextCursor předchozí odpovědi.
  output:
    schema:
      type: object
      required: [invoices, totalCount, totalAmount, currency]
      properties:
        invoices:
          type: array
          items:
            type: object
            required: [invoiceId, issuedOn, dueOn, amount, currency, status]
            properties:
              invoiceId: { type: string }
              issuedOn: { type: string, format: date }
              dueOn: { type: string, format: date }
              amount: { type: number }
              currency: { type: string, enum: [CZK, EUR, USD] }
              status: { type: string, enum: [draft, issued, paid, overdue, cancelled] }
              daysOverdue: { type: integer, minimum: 0 }
        totalCount: { type: integer, minimum: 0 }
        totalAmount: { type: number, minimum: 0 }
        currency: { type: string, enum: [CZK, EUR, USD] }
        nextCursor: { type: string }
  binding:
    apim:
      apiId: finance-billing-v3
      operationId: listInvoicesByAccount
      method: GET
      pathTemplate: /finance/v3/accounts/{accountId}/invoices
      timeoutMs: 15000
      auth:
        mode: obo
        scope: api://billing-api.acme.example/.default
      responseMapping: >-
        {
          "invoices": items.{
            "invoiceId": number,
            "issuedOn": $substring(issueDate, 0, 10),
            "dueOn": $substring(dueDate, 0, 10),
            "amount": grossAmount,
            "currency": currencyCode,
            "status": $lowercase(state),
            "daysOverdue": daysPastDue
          },
          "totalCount": meta.total,
          "totalAmount": $sum(items.grossAmount),
          "currency": items[0].currencyCode,
          "nextCursor": meta.nextToken
        }
```

Všimni si, že `requestMapping` chybí — u `GET` se parametry mapují automaticky: co je v `pathTemplate` jako placeholder, jde do cesty, zbytek do query stringu (s výjimkou `cursor`, který se mapuje na `nextToken` — proto by u složitějšího mapování byl `requestMapping` potřeba).

### 9.3 Příklad 2 — skill s transformací

```yaml
apiVersion: mcp.acme.example/v1
kind: Tool
metadata:
  name: crm_account_health_summary
  title: Souhrn zdraví zákaznického vztahu
  owner: crm-platform@acme.example
  domain: crm
  tags: [customer-success, risk]
spec:
  description: >-
    Sestaví souhrn stavu zákaznického vztahu: segment, stav smlouvy, počet
    a objem faktur po splatnosti a počet otevřených incidentů. Použij, když
    se uživatel ptá na celkový stav zákazníka nebo hledá rizikové účty.
    Nevolej ho opakovaně pro seznam účtů, je určený pro jeden účet.
    Vrací chybu, pokud účet neexistuje.
  riskTier: low
  entitlementTags: [crm.read, finance.read]
  annotations:
    readOnlyHint: true
    destructiveHint: false
    idempotentHint: true
    openWorldHint: false
  input:
    schema:
      type: object
      additionalProperties: false
      required: [accountId]
      properties:
        accountId:
          type: string
          pattern: "^ACC-[0-9]{6}$"
          description: Identifikátor zákaznického účtu ve tvaru ACC-000123.
  output:
    schema:
      type: object
      required: [accountId, name, segment, contractStatus, overdueInvoiceCount, overdueAmount, healthScore]
      properties:
        accountId: { type: string }
        name: { type: string }
        segment: { type: string, enum: [enterprise, mid_market, smb] }
        contractStatus: { type: string, enum: [active, suspended, terminated, prospect] }
        overdueInvoiceCount: { type: integer, minimum: 0 }
        overdueAmount: { type: number, minimum: 0 }
        currency: { type: string }
        openIncidentCount: { type: integer, minimum: 0 }
        healthScore: { type: string, enum: [green, amber, red] }
  binding:
    transform:
      engine: jsonata
      source: crm_get_account
      expression: >-
        (
          $inv := $tool("finance_list_invoices", { "accountId": account.accountId, "status": "overdue", "limit": 100 });
          $inc := $tool("itsm_list_incidents", { "accountId": account.accountId, "state": "open", "limit": 100 });
          $overdueCount := $count($inv.invoices);
          $overdueAmount := $sum($inv.invoices.amount);
          {
            "accountId": account.accountId,
            "name": account.name,
            "segment": account.segment,
            "contractStatus": account.contractStatus,
            "overdueInvoiceCount": $overdueCount,
            "overdueAmount": $overdueAmount,
            "currency": $inv.currency,
            "openIncidentCount": $count($inc.incidents),
            "healthScore": $overdueAmount > 500000 or $overdueCount > 5 ? "red"
              : $overdueAmount > 0 or $count($inc.incidents) > 2 ? "amber"
              : "green"
          })
```

Funkce `$tool(name, args)` je **jediné** rozšíření JSONata, které gateway registruje. Její omezení jsou tvrdá a kontroluje je ingestor i runtime:

| Omezení | Hodnota | Proč |
|---|---|---|
| Max. hloubka vnoření | 2 | zamezí rekurzi skill → skill → skill |
| Max. počet `$tool` volání na jeden skill | 5 | strop latence a nákladů |
| Volané nástroje musí být `readOnlyHint: true` | ano | skill nikdy nezapisuje |
| Entitlement | vyhodnocuje se pro **každý** vnořený nástroj zvlášť, proti **volajícímu uživateli** | skill nesmí být cesta k obejití authz |
| Rozpočet času | sdílený `ctx.deadline` celého tool callu | žádné prodlužování |
| Vnořené nástroje musí být v **téže** `CatalogVersion` | ano | determinismus (ADR-012) |

Pokud uživatel nemá entitlement na `finance_list_invoices`, skill selže s tool errorem, který to říká — **ne** s tichým vynecháním části dat. Částečně naplněný souhrn by byl horší než chyba, protože by agent vyvodil nesprávný závěr.

### 9.4 JSON Schema pro validaci manifestu

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://acme.example/schemas/mcp-tool-manifest/v1.json",
  "title": "ACME MCP Tool Manifest v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["apiVersion", "kind", "metadata", "spec"],
  "properties": {
    "apiVersion": { "const": "mcp.acme.example/v1" },
    "kind": { "enum": ["Tool", "ToolSet"] },
    "metadata": {
      "type": "object", "additionalProperties": false,
      "required": ["name", "title", "owner", "domain"],
      "properties": {
        "name": { "type": "string", "pattern": "^[a-z][a-z0-9]*(_[a-z0-9]+){1,5}$", "maxLength": 64 },
        "title": { "type": "string", "minLength": 3, "maxLength": 60 },
        "owner": { "type": "string", "format": "email" },
        "domain": { "enum": ["crm", "finance", "sales", "hr", "itsm", "kb"] },
        "tags": { "type": "array", "maxItems": 10, "items": { "type": "string", "pattern": "^[a-z0-9-]{2,32}$" } }
      }
    },
    "spec": {
      "type": "object", "additionalProperties": false,
      "required": ["description", "riskTier", "entitlementTags", "annotations", "input", "binding"],
      "properties": {
        "description": { "type": "string", "minLength": 80, "maxLength": 600 },
        "riskTier": { "enum": ["low", "medium", "high", "critical"] },
        "entitlementTags": {
          "type": "array", "minItems": 1, "maxItems": 5,
          "items": { "type": "string", "pattern": "^[a-z]+\\.[a-z]+$" }
        },
        "annotations": {
          "type": "object", "additionalProperties": false,
          "required": ["readOnlyHint", "destructiveHint", "idempotentHint", "openWorldHint"],
          "properties": {
            "readOnlyHint": { "type": "boolean" },
            "destructiveHint": { "type": "boolean" },
            "idempotentHint": { "type": "boolean" },
            "openWorldHint": { "type": "boolean" }
          }
        },
        "input": { "type": "object", "required": ["schema"], "additionalProperties": false,
          "properties": { "schema": { "$ref": "#/$defs/jsonSchemaObject" } } },
        "output": { "type": "object", "required": ["schema"], "additionalProperties": false,
          "properties": { "schema": { "$ref": "#/$defs/jsonSchemaObject" } } },
        "binding": {
          "type": "object", "additionalProperties": false, "minProperties": 1, "maxProperties": 1,
          "properties": {
            "apim": {
              "type": "object", "additionalProperties": false,
              "required": ["apiId", "operationId", "method", "pathTemplate", "timeoutMs", "auth"],
              "properties": {
                "apiId": { "type": "string", "pattern": "^[a-z0-9-]{3,64}$" },
                "operationId": { "type": "string", "pattern": "^[A-Za-z][A-Za-z0-9]{2,63}$" },
                "method": { "enum": ["GET", "POST", "PUT", "PATCH", "DELETE"] },
                "pathTemplate": { "type": "string", "pattern": "^/[A-Za-z0-9/_{}.-]{1,256}$" },
                "timeoutMs": { "type": "integer", "minimum": 1000, "maximum": 30000 },
                "auth": {
                  "type": "object", "required": ["mode"],
                  "properties": {
                    "mode": { "enum": ["obo", "managed_identity", "subscription_key"] },
                    "scope": { "type": "string", "pattern": "^api://[A-Za-z0-9./_-]+$" },
                    "resource": { "type": "string", "format": "uri" },
                    "secretRef": { "type": "string", "pattern": "^[a-zA-Z0-9-]{1,127}$" }
                  },
                  "allOf": [
                    { "if": { "properties": { "mode": { "const": "obo" } } }, "then": { "required": ["scope"] } },
                    { "if": { "properties": { "mode": { "const": "managed_identity" } } }, "then": { "required": ["resource"] } },
                    { "if": { "properties": { "mode": { "const": "subscription_key" } } }, "then": { "required": ["secretRef"] } }
                  ]
                },
                "requestMapping": { "type": "string", "maxLength": 4000 },
                "responseMapping": { "type": "string", "maxLength": 4000 }
              }
            },
            "transform": {
              "type": "object", "additionalProperties": false,
              "required": ["engine", "source", "expression"],
              "properties": {
                "engine": { "enum": ["jsonata", "jmespath"] },
                "source": { "type": "string", "pattern": "^[a-z][a-z0-9]*(_[a-z0-9]+){1,5}$" },
                "expression": { "type": "string", "minLength": 3, "maxLength": 4000 }
              }
            }
          }
        },
        "deprecation": {
          "type": "object", "additionalProperties": false, "required": ["since", "removeAfter"],
          "properties": {
            "since": { "type": "string", "format": "date" },
            "removeAfter":{ "type": "string", "format": "date" },
            "replacedBy": { "type": "string", "pattern": "^[a-z][a-z0-9]*(_[a-z0-9]+){1,5}$" }
          }
        }
      },
      "allOf": [
        { "if": { "properties": { "annotations": { "properties": { "readOnlyHint": { "const": true } } } } },
          "then": { "properties": { "annotations": { "properties": { "destructiveHint": { "const": false } } } } } },
        { "if": { "properties": { "riskTier": { "enum": ["high", "critical"] } } },
          "then": { "properties": { "input": { "properties": { "schema": {
                    "properties": { "properties": { "required": ["confirm"] } } } } } } } },
        { "if": { "properties": { "annotations": { "properties": { "destructiveHint": { "const": true } } } } },
          "then": { "properties": { "input": { "properties": { "schema": {
                    "properties": { "properties": { "required": ["confirm"] } } } } } } } }
      ]
    }
  },
  "$defs": {
    "jsonSchemaObject": {
      "type": "object",
      "required": ["type"],
      "properties": {
        "type": { "const": "object" },
        "additionalProperties": { "const": false },
        "properties": { "type": "object", "minProperties": 0, "maxProperties": 25 },
        "required": { "type": "array", "items": { "type": "string" } }
      },
      "required": ["type", "additionalProperties"]
    }
  }
}
```

Pravidla, která JSON Schema **nevyjádří** a kontroluje je linter (`tools/skill-lint`) i ingestor:

1. `apiId` a `operationId` musí existovat v exportovaném APIM kontraktu.
2. Každý placeholder v `pathTemplate` musí odpovídat vlastnosti v `input.schema`.
3. Metoda `GET` ⇒ `readOnlyHint: true`; `DELETE` ⇒ `destructiveHint: true`.
4. `riskTier ∈ {high, critical}` **nebo** `annotations.destructiveHint === true` ⇒ `confirm` s `"const": true` v `required` (§5.3).
5. Každý `string` v `input.schema` má `maxLength`, každý `array` má `maxItems`.
6. `transform.source` i všechny nástroje volané přes `$tool` existují v téže `CatalogVersion`, jsou `readOnlyHint: true` a nevytvářejí cyklus.
7. `description` má strukturu ze §5.2 (heuristika: ≥ 3 věty, obsahuje „Použij, když").
8. Součet `description` v jednom tool setu ≤ 12 000 znaků.

---

## 10. Verzování API a nástrojů

### 10.1 Co je verze

Tři nezávislé osy:

| Osa | Identifikátor | Kdo mění | Viditelnost pro agenta |
|---|---|---|---|
| Verze protokolu MCP | `2026-07-28` / `2025-11-25` | spec | `supportedVersions`, `initialize` result |
| Verze katalogu | `CatalogVersionId` = `{gitSha}-{buildHash}` | interní IT přes Git | `_meta["com.contoso.mcpgw/catalogVersionId"]` |
| Verze gateway | semver `1.4.2` | platformní tým | `serverInfo.version` |

Agent **nevidí** verze jednotlivých nástrojů a ani nemá. MCP nemá koncept verze nástroje; jméno je kontrakt.

### 10.2 Změna nástroje bez rozbití agentů

| Typ změny | Bezpečná? | Postup |
|---|---|---|
| Přidání volitelného vstupního parametru s defaultem | ✔ | rovnou |
| Zpřesnění `description` | ✔ | rovnou (ale spustit blind test, §11.6 dok. 03) |
| Přidání vlastnosti do `outputSchema`, která není v `required` | ✔ | rovnou |
| Rozšíření `enum` o novou hodnotu **na výstupu** | ✔ | rovnou |
| Rozšíření `enum` o novou hodnotu **na vstupu** | ✔ | rovnou |
| Zúžení `enum` na vstupu, zpřísnění `pattern`, snížení `maximum` | ✘ | nový nástroj + deprecation starého |
| Přidání **povinného** vstupního parametru | ✘ | nový nástroj |
| Odebrání vlastnosti z `required` na výstupu | ✘ | nový nástroj |
| Změna sémantiky při zachování schématu (např. `amount` z brutto na netto) | ✘ **nejnebezpečnější** | nový nástroj s jiným jménem, povinně |
| Změna `readOnlyHint` z `true` na `false` | ✘ | nový nástroj — klient podle toho gatuje potvrzení [13] |

Nový nástroj má **jméno s pořadovým rozlišením v objektu, ne příponu verze**: ne `finance_list_invoices_v2`, ale `finance_list_invoices_net` — jméno má popisovat, co dělá, ne kdy vzniklo. Výjimka jsou případy, kde se sémantika nemění a jde čistě o kontraktní zlom; tam je `_v2` přijatelné.

### 10.3 Deprecation policy

```yaml
  deprecation:
    since: "2026-10-01"
    removeAfter: "2027-01-15" # min. since + 90 dní, vynuceno linterem
    replacedBy: finance_list_invoices_net
```

Životní cyklus:

| Fáze | Trvání | Chování gateway |
|---|---|---|
| **Active** | — | normální |
| **Deprecated** | ≥ 90 dní | nástroj **zůstává** v `tools/list`; do `description` se **prependuje** věta „ZASTARALÉ od {since}, bude odstraněno po {removeAfter}. Používej místo něj {replacedBy}."; metrika `mcp_deprecated_tool_calls` s labelem `tool` a `workspace`; při každém volání varovný záznam do auditu |
| **Sunset** | 14 dní | nástroj zůstává, ale gateway přidává do výsledku `_meta["com.contoso.mcpgw/sunsetInDays"]`; alert vlastníkovi, pokud volání neklesla na nulu |
| **Removed** | — | nástroj zmizí z `tools/list`; volání vrací tool error „Nástroj byl odstraněn {date}. Použij {replacedBy}." po dobu dalších 90 dní, pak `-32602` |

MCP nemá anotaci `deprecated` — `ToolAnnotations` obsahuje jen `title`, `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` [4]. Deprecation se proto v drátovém tvaru komunikuje **výhradně textem v `description`** (to model přečte) a vlastním `_meta` klíčem pod naším prefixem (to přečte nástrojař, ne model). Zavádět nestandardní pole do `annotations` by bylo chybou — klienti anotace validují a neznámá pole ignorují nebo odmítají.

Pokud nástroj zmizí mimo tento proces (odebraný entitlement, rollback katalogu), je změna viditelná přes `notifications/tools/list_changed` — v éře A na GET streamu, v éře B přes `subscriptions/listen` [2][4]. Gateway notifikaci posílá při každém přepnutí `activePointer`.

---

## 11. Chování specifické pro klienty

### 11.1 Srovnání

| Aspekt | **Claude** (Claude.ai, Desktop, Cowork, Messages API) | **ChatGPT Business** (developer mode) | Zdroj |
|---|---|---|---|
| Transport | Streamable HTTP a SSE; **auto-detekce z URL** — URL končící `/sse` vybere starší SSE transport, jinak Streamable HTTP | SSE i Streamable HTTP | [13][16] |
| URL | **musí** začínat `https://`; server dosažitelný z veřejného internetu z rozsahů Anthropicu | veřejná HTTPS; lokální server jen přes „Secure MCP Tunnel" | [13][17] |
| Odchozí IP rozsahy | IPv4 `160.79.104.0/21` pro volání konektoru | nedokumentováno ⚠️ NEOVĚŘENO | [13] |
| Registrace klienta | `oauth_dcr`, `oauth_cimd`, `oauth_anthropic_creds`, `custom_connection`, `static_headers` (beta), `none` | CIMD (`none` nebo `private_key_jwt`), DCR, no-auth, mixed auth | [13] |
| **S Entra ID** | DCR **nefunguje** (Entra nemá RFC 7591 endpoint) → nutná statická registrace (ADR-006) | totéž | [18] |
| PKCE | vždy `S256` bez ohledu na způsob registrace | S256 | [13] |
| PRM discovery | hlavička `WWW-Authenticate` **preferovaná**; fallback sondování `/.well-known/...` na originu serveru | RFC 9728 | [13] |
| `authorization_servers` v PRM | **jen první položka**, bez fallbacku | nedokumentováno ⚠️ NEOVĚŘENO | [13] |
| Timeouty OAuth endpointů | discovery/registration/token **10 s**; refresh **30 s** | nedokumentováno ⚠️ NEOVĚŘENO | [5] |
| Refresh | reaktivní na `401` + proaktivní do 5 minut před expirací | nedokumentováno | [5] |
| `/token` content-type | **`application/x-www-form-urlencoded`**, ne JSON | standard OAuth | [5] |
| Resources / Prompts | Messages API konektor: **nepodporuje** | nezmíněno v dokumentaci ⚠️ NEOVĚŘENO | [13] |
| `readOnlyHint` | součást trust modelu; admin nastavuje per-tool „Always allow / Needs approval / Blocked" | **neoznačené nástroje implicitně vyžadují potvrzení zápisu** | [13] |
| `search`/`fetch` konvence | netýká se | pro developer mode **už není vyžadována**; platí jen pro starší „Connectors" (deep research, company knowledge) | [13] |
| `instructions` | podporováno | předáváno modelu; prvních 512 znaků má být samostatně srozumitelných | [13] |
| Org správa | Team/Enterprise: konektory org-wide, per-tool oprávnění | Business: developer mode **jen pro adminy/owners**, nelze delegovat na členy | [13] |
| Limit počtu nástrojů | nedokumentováno (Free plan: 1 konektor) ⚠️ NEOVĚŘENO | nedokumentováno ⚠️ NEOVĚŘENO | [13] |

### 11.2 Diagram: OAuth 2.1 flow

```mermaid
sequenceDiagram
    autonumber
    actor U as Uzivatel
    participant C as MCP klient Claude nebo ChatGPT
    participant G as MCP Gateway
    participant E as Microsoft Entra ID

    Note over C,G: Faze 1 - objeveni autorizacniho serveru
    C->>G: POST /mcp bez hlavicky Authorization
    G-->>C: 401 Unauthorized + WWW-Authenticate Bearer resource_metadata a scope
    Note over C: Klient cte resource_metadata primo z hlavicky
    C->>G: GET /.well-known/oauth-protected-resource/mcp/acme
    G-->>C: 200 PRM JSON - resource, authorization_servers, scopes_supported
    C->>E: GET tenant-id/v2.0/.well-known/openid-configuration
    E-->>C: AS metadata - authorization_endpoint, token_endpoint, jwks_uri, issuer
    Note over C: Overeni ze issuer v metadatech odpovida issueru z PRM

    Note over C,E: Faze 2 - PKCE authorization code flow
    C->>C: Vygeneruje code_verifier a code_challenge metodou S256
    C->>E: GET authorize s client_id, redirect_uri, code_challenge, resource, scope, state
    E->>U: Prihlaseni, MFA a vyhodnoceni Conditional Access
    U-->>E: Souhlas se scopy
    E-->>C: 302 na redirect_uri s parametry code, state a iss
    Note over C: Retezcove porovnani iss se zaznamenanym issuerem
    C->>E: POST token - grant_type authorization_code, code_verifier, resource
    E-->>C: access_token v2, refresh_token, expires_in

    Note over C,G: Faze 3 - autorizovane volani
    C->>G: POST /mcp s Authorization Bearer access_token
    G->>E: GET jwks_uri pouze pri cache miss nebo rotaci klice
    E-->>G: JWKS sada verejnych klicu
    G->>G: Overeni podpisu, aud, iss, exp, nbf, tid a claimu roles
    G-->>C: 200 OK s vysledkem MCP volani

    Note over C,E: Faze 4 - obnova tokenu
    C->>E: POST token - grant_type refresh_token, resource, scope
    E-->>C: novy access_token a pripadne rotovany refresh_token
    Note over C: Claude obnovuje reaktivne pri 401 a proaktivne do 5 minut pred expiraci
```

**Co diagram říká.** Celý flow stojí na tom, že **první volání musí selhat správně**. `401` s hlavičkou `WWW-Authenticate` je jediný krok, který klient nemůže obejít ani uhodnout — a zároveň jediný, který rozbije WAF nebo reverse proxy, když si ho vezme na starost a vrátí vlastní `401` bez té hlavičky. Anthropic to popisuje jako běžnou příčinu tichých selhání na platformách, které neservírují kořen domény [13]. Proto:

- Front Door **nesmí** mít vlastní auth ani vlastní chybové stránky pro `/mcp*` a `/.well-known/*`.
- WAF pravidlo nesmí blokovat `POST` bez `Authorization` — je to legitimní první krok protokolu.
- `/.well-known/oauth-protected-resource*` musí být dostupné **bez tokenu** a bez WAF rate limitu, který by ho odstřelil.

Druhý nosný bod je **parametr `resource` (RFC 8707)**: klient ho **MUST** posílat na authorization i token požadavku, nastavený na kanonickou URI MCP serveru, a to **bez ohledu na to, jestli AS jeho podporu deklaruje** [1]. Entra ho porovnává s Application ID URI registrace a při neshodě token nevydá [18]. Server pak **MUST** ověřit, že `aud` tokenu je on sám [1]. Tímhle řetězcem se uzavírá confused deputy: token vydaný pro jinou aplikaci u nás neprojde a náš token neprojde jinde.

Krok, který na diagramu vypadá nenápadně — **ověření `iss`** — je anti-spoofing: klient porovnává `iss` z autorizační odpovědi s issuerem, který si zapsal před přesměrováním, prostým řetězcovým porovnáním bez normalizace [1]. Pokud AS inzeruje `authorization_response_iss_parameter_supported: true` a `iss` neposílá, klient **MUST** odpověď odmítnout [1].

### 11.3 Jak se rozdíly řeší per-workspace

ADR-015 zavádí workspace jako jednotku konfigurace. Rozdíly z §11.1 jsou tedy **data v `workspaces` kontejneru**, ne větve v kódu:

```jsonc
// workspaces/acme-claude
{
  "slug": "acme-claude",
  "clientType": "claude",
  "protocolEras": ["legacy", "stateless"],
  "pinnedCatalogVersion": null,
  "behavior": {
    "alwaysEmitTextMirrorOfStructuredContent": true,
    "sseForLongRunningTools": true,
    "instructionsMaxChars": 4000,
    "allowedOrigins": ["https://claude.ai"]
  },
  "limits": { "callsPerMinutePerUser": 60, "listCallsPerMinutePerUser": 20 }
}
```
```jsonc
// workspaces/acme-chatgpt
{
  "slug": "acme-chatgpt",
  "clientType": "chatgpt",
  "protocolEras": ["legacy", "stateless"],
  "behavior": {
    "alwaysEmitTextMirrorOfStructuredContent": true,
    "sseForLongRunningTools": true,
    "instructionsMaxChars": 512, // prvních 512 znaků musí stát samostatně [13]
    "allowedOrigins": ["https://chatgpt.com", "https://chat.openai.com"]
  },
  "limits": { "callsPerMinutePerUser": 60, "listCallsPerMinutePerUser": 20 }
}
```

**Workspacy nemají vlastní audience.** Napříč celým návrhem je jediná kanonická hodnota: `aud` v access tokenu, Entra Application ID URI i RFC 8707 `resource` indikátor jsou vždy `https://mcp.contoso.com/mcp` — jedna app registration pro celou gateway, ne jedna na workspace. Workspace se rozlišuje **cestou** (`/mcp/acme-claude`, `/mcp/acme-chatgpt`) a **app rolemi** v tokenu, ne audience. Důvod je praktický: kdyby měl každý workspace vlastní Application ID URI, znamenalo by to vlastní app registration a vlastní admin consent na Entra straně pro každý nový workspace, což by odporovalo cíli mít jeden gateway endpoint pro obě platformy. „Kdo smí co" se tedy nastavuje per platformu přes to, **jaké app role** se přidělí principálům přihlašujícím se do daného workspace (například: write nástroje zatím jen rolím ve Claude workspace, ChatGPT jen s rolí `Mcp.Reader`), ne přes rozdílnou audience. Nesoulad mezi `resource` v token requestu a Application ID URI v Entra končí chybou `AADSTS9010010` a konektor se nepřipojí.

Dvě chování jsou v tabulce shodná, ale zapisujeme je explicitně, protože jejich důvod je různý: textové zrcadlo `structuredContent` je u Claude zpětná kompatibilita [4], u ChatGPT dokumentovaný požadavek konektorové cesty [13]. Kdyby jedna strana požadavek zrušila, změní se jeden řádek dat.

**Provozní konfigurace konektoru** (co administrátor zadá v UI obou platforem — `client_id`, `client_secret`, URL, per-tool oprávnění) je předmětem samostatného návodu v handover dokumentaci; zde stačí, že díky ADR-006 jde vždy o **statickou registraci** Entra app registration na obou stranách.

---

## Otevřené otázky

| # | Otázka | Kdo rozhoduje | Blokuje |
|---|---|---|---|
| 1 | Potvrdí některý z dodavatelů datum migrace na éru B? Do té doby je éra A povinná. | dodavatelé | plánování odchodu éry A |
| 2 | Pinne OpenAI někdy verzi MCP protokolu? Bez toho nelze predikovat dopad změn spec. | OpenAI | rizikový registr |
| 3 | Existuje limit počtu nástrojů nebo velikosti `tools/list` u obou platforem? | měření | velikost tool setů |
| 4 | Token cost katalogu — změřit na reálném katalogu (§5.5) | platformní tým | Gate 1 |
| 5 | Sjednotit `Mcp-Param-*` s APIM směrováním, nebo tuto optimalizaci zatím nepoužívat? | platformní tým | ne |
| 6 | Ověřená vlastní doména pro Application ID URI — kdo ji vlastní a kdo spravuje certifikát? | platformní tým | Gate 0 |
| 7 | Schválí bezpečnost `subscription_key` jako binding auth mode, nebo jen OBO a MI? | bezpečnost | fáze 1 |
| 8 | Cílové hodnoty `limits` per workspace | provoz | Gate 1 |

## Zdroje
1. MCP Specification 2026-07-28 — Authorization (PRM, `resource`, scope challenge, `iss`, error handling). https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization
2. MCP Specification 2026-07-28 — Streamable HTTP transport (hlavičky, validace, `-32020`, `405`, cancellation, backward compatibility). https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http
3. RFC 9728 — OAuth 2.0 Protected Resource Metadata (`resource_metadata`, konstrukce well-known cesty, pole dokumentu). https://www.rfc-editor.org/rfc/rfc9728
4. MCP Specification 2026-07-28 — Server / Tools (`Tool`, `ToolAnnotations`, `structuredContent`, `x-mcp-header`, error handling, řazení). https://modelcontextprotocol.io/specification/2026-07-28/server/tools
5. Anthropic — Building connectors: Authentication (timeouty, PKCE, refresh, `/token` content-type, CIMD vs. DCR). https://claude.com/docs/connectors/building/authentication
6. MCP Specification 2026-07-28 — Versioning and Compatibility (éry, `-32022`, matice kompatibility, dual-era server). https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning
7. MCP Specification 2026-07-28 — Basic / Overview (`_meta`, reserved keys, error codes, `-32021`, `-32602`). https://modelcontextprotocol.io/specification/2026-07-28/basic
8. MCP Specification 2026-07-28 — Schema (`ToolAnnotations` včetně defaultů, `Tool`, `UnsupportedProtocolVersionError`). https://modelcontextprotocol.io/specification/2026-07-28/schema
9. MCP Specification 2026-07-28 — Server / Discovery (`server/discover`, `DiscoverResult`, `serverInfo` v `_meta`). https://modelcontextprotocol.io/specification/2026-07-28/server/discover
10. MCP Specification 2026-07-28 — Pagination (`cursor`, `nextCursor`, `ttlMs`, `cacheScope`). https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/pagination
11. MCP Specification 2025-11-25 — Cancellation (`notifications/cancelled`). https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/cancellation
12. MCP Specification 2025-11-25 — Progress (`progressToken`, monotónnost, rate limiting). https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/progress
13.  — rešerše MCP a klientů (Claude, ChatGPT Business), září 2026, včetně seznamu NEOVĚŘENO.
14. MCP Specification 2025-11-25 — Lifecycle (`initialize`, `notifications/initialized`, version negotiation). https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle
15. MCP Specification 2025-11-25 — Transports (`MCP-Session-Id`, GET SSE, `DELETE`, `Last-Event-ID`, protocol version header). https://modelcontextprotocol.io/specification/2025-11-25/basic/transports
16. Anthropic — Custom connectors: remote MCP (auto-detekce transportu z URL, request-header auth). https://claude.com/docs/connectors/custom/remote-mcp
17. Anthropic — Model Context Protocol v konektorech (povinné `readOnlyHint` a `destructiveHint`). https://claude.com/docs/connectors/building/mcp
18.  — rešerše Azure platformy (Entra ID, DCR, PRM, OBO, APIM), září 2026.
