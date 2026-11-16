# `kontrakty/` — strojově čitelné kontrakty

Tento adresář obsahuje formální, strojově validovatelné popisy rozhraní, která
jsou jinde v dokumentaci popsaná jen prózou a příklady. Účel je oddělit dvě
různé věci, které dokumentace jinak snadno smíchá: **co gateway dělá** (text v
`00`–`09`) a **jaký je přesný, kontrolovatelný tvar dat**, který na hranicích
systému prochází. Druhé z toho patří sem.

## Co je v adresáři a odkud pochází

| Soubor | Co formalizuje | Zdroj pravdy v textové dokumentaci |
|---|---|---|
| `skill-manifest.schema.json` | JSON Schema manifestu skillu — vstup do `skill-ingestor`, publikovaný do katalogu | [`../05-api-a-protokoly.md`](../05-api-a-protokoly.md) §9 (mapování manifestu na MCP tool) |
| `audit-event.schema.json` | JSON Schema jednoho auditního záznamu (`AuditEvent`) | [`../04-datova-architektura.md`](../04-datova-architektura.md) §6.1 (rozšířené schéma audit záznamu) a [`../07-bezpecnost-a-autorizace.md`](../07-bezpecnost-a-autorizace.md) §11 (bezpečnostně závazné chování) |
| `gateway-admin.openapi.yaml` | OpenAPI 3.1 kontrakt administrativních, webhook a discovery endpointů (`/admin/catalog/publish`, `/admin/catalog/rollback`, `/webhooks/github`, `/healthz`, `/readyz`, `/.well-known/oauth-protected-resource*`) | [`../05-api-a-protokoly.md`](../05-api-a-protokoly.md) §2 (tabulka HTTP povrchu) |

Žádný z těchto tří souborů nepřidává ani nemění pole, chování nebo hodnoty
oproti textové dokumentaci — každý ji jen formalizuje do tvaru, který lze
strojově ověřit. Kde textová dokumentace a schéma nesouhlasí, je to chyba a
opravuje se dokumentace i schéma společně, ne jen jedno z nich.

## Proč `POST /mcp` v OpenAPI není

Samotný MCP endpoint (`/mcp`, `/mcp/{workspaceSlug}`) je jediný JSON-RPC
endpoint gateway — jeho request/response tvar určuje metoda uvnitř JSON-RPC
těla (`tools/list`, `tools/call`, …), ne HTTP cesta nebo metoda, kterou OpenAPI
modeluje jako REST zdroj. Jeho kontrakt tvoří dohromady MCP specifikace
(obálka JSON-RPC, chybová taxonomie — [`../05-api-a-protokoly.md`](../05-api-a-protokoly.md)
§3) a schéma jednotlivých nástrojů v katalogu (`inputSchema`/`outputSchema`,
viz `skill-manifest.schema.json` výše) — ne jeden statický OpenAPI dokument,
který by se musel přegenerovat při každé změně katalogu. Administrativní a
webhook cesty naproti tomu jsou obyčejné REST endpointy s pevným tvarem, a
proto OpenAPI dostávají.

## CI je validuje proti dokumentaci — rozchod shodí build

Toto není jen sada souborů k nahlédnutí. CI pipeline (viz
[`../08-provoz-observabilita-rollout.md`](../08-provoz-observabilita-rollout.md)
§8, krok `KONTRAKTY` v kódové dráze) na každý pull request:

1. Ověří, že všechny tři soubory jsou syntakticky validní (JSON Schema draft
 2020-12 přes `jsonschema`, OpenAPI 3.1 přes `yaml.safe_load` + validaci
 struktury).
2. Ověří, že `skill-manifest.schema.json` přijme oba příklady manifestu z
 [`../05-api-a-protokoly.md`](../05-api-a-protokoly.md) §9.2–9.3 a odmítne
 syntetický neplatný manifest (chybějící `confirm` u `riskTier: critical`
 apod.) — regresní test proti tichému rozjetí schématu a dokumentace.
3. Ověří, že `audit-event.schema.json` přijme příklad `AuditEvent` z
 [`../04-datova-architektura.md`](../04-datova-architektura.md) §6.1.
4. Ověří, že `gateway-admin.openapi.yaml` popisuje přesně tu množinu cest,
 která je v tabulce HTTP povrchu v
 [`../05-api-a-protokoly.md`](../05-api-a-protokoly.md) §2 (kromě `/mcp*`,
 viz výše) — ani cesta navíc, ani chybějící.

Selhání kteréhokoli z těchto kroků **shodí build**, ne jen vypíše varování.
Smysl brány je, že se dokumentace a kontrakt nemohou tiše rozejít — pokud
někdo přidá pole do `AuditEvent` v textu `04`, ale zapomene na schéma zde (nebo
naopak), review to nemá šanci přehlédnout, protože CI ho zastaví dřív, než se
PR dá mergnout.

## Jak validaci spustit lokálně

```bash
python3 -c "
import json, yaml
from jsonschema import Draft202012Validator

for f in ['skill-manifest.schema.json', 'audit-event.schema.json']:
 with open(f) as fh:
 schema = json.load(fh)
 Draft202012Validator.check_schema(schema)
 print(f, 'OK — validní JSON Schema')

with open('gateway-admin.openapi.yaml') as fh:
 doc = yaml.safe_load(fh)
assert doc.get('openapi', '').startswith('3.1')
print('gateway-admin.openapi.yaml OK — validní YAML, OpenAPI 3.1')
"
```

Plná sada testů z bodů 2–4 výše žije v `packages/testkit` (mimo tento
adresář, viz [`../03-kodova-architektura.md`](../03-kodova-architektura.md)) —
zde je jen syntaktická kontrola, kterou lze spustit bez zbytku monorepa.
