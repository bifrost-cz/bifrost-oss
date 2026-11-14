<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/features/semantic-caching.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Sémantická cache

*Jak vracet uložené odpovědi na stejné nebo podobné dotazy bez volání poskytovatele.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis všech možností je v [dokumentaci výrobce](https://docs.getbifrost.ai/features/semantic-caching).

## Dva režimy

| | Jen přímá shoda (direct) | Přímá shoda + sémantická |
|---|---|---|
| **Co najde** | Přesně stejný (normalizovaný) požadavek | Stejný i významově podobný požadavek |
| **Embeddingový model** | Není potřeba | Je potřeba |
| **Cena při minutí** | Nic navíc | Jedno volání embeddingového modelu |
| **Hodí se pro** | Opakované stejné dotazy | Různě formulované dotazy uživatelů |
| **`dimension`** | `1` | Skutečná velikost vektoru modelu |

V obou režimech se nejdřív zkouší přímá shoda, sémantické hledání běží až při jejím minutí. Ve webovém rozhraní se funkce jmenuje **Local Cache** (**Settings** → **Caching**).

Nezaměňujte s prompt cachingem: tam znovu použije začátek promptu poskytovatel, volání proběhne a platí se. Sémantická cache poskytovatele nevolá vůbec. Obojí může běžet současně.

## Předpoklady

1. **Vektorové úložiště** pro oba režimy: Redis nebo Valkey (doporučené pro přímou shodu), Weaviate, Qdrant nebo Pinecone. Musí být zapnuté v `config.json`, jinak přepínač v rozhraní zůstane neaktivní:

```json
{
  "vector_store": {
    "enabled": true,
    "type": "redis",
    "config": {
      "addr": "localhost:6379"
    }
  }
}
```

2. **Poskytovatel embeddingů**, jen pro sémantický režim. Klíče se převezmou z jeho konfigurace v Bifrostu. Pro on-premise provoz to může být i lokální embeddingový model, například přes poskytovatele Ollama nebo vLLM.

## Zapnutí

### Jen přímá shoda

Nejlevnější varianta bez embeddingů: `dimension: 1` a bez `provider`.

```json
{
  "plugins": [
    {
      "enabled": true,
      "name": "semantic_cache",
      "config": {
        "dimension": 1,
        "ttl": "5m",
        "cache_by_model": true,
        "cache_by_provider": true
      }
    }
  ]
}
```

> [!WARNING]
> Když s `dimension: 1` zároveň uvedete `provider`, Bifrost to bere jako sémantický režim.

### Sémantický režim

```json
{ 
  "vector_store": {...},
  "plugins": [
    {
      "enabled": true,
      "name": "semantic_cache",
      "config": {
        "provider": "openai",
        "embedding_model": "text-embedding-3-small",
        "dimension": 1536,

        "ttl": "5m",
        "threshold": 0.8,

        "conversation_history_threshold": 3,
        "exclude_system_prompt": false,

        "cache_by_model": true,
        "cache_by_provider": true,

        "vector_store_namespace": "BifrostSemanticCachePlugin",
        "default_cache_key": "",
        "cache_tool_call_responses": false,
        "disable_identity_scoping": false
      }
    }
  ]
}
```

Nejdůležitější pole:

| Pole | Výchozí | Význam |
|---|---|---|
| `provider`, `embedding_model` | | Poskytovatel a model pro embeddingy, jen sémantický režim. |
| `dimension` | | Velikost vektoru. Musí přesně odpovídat modelu (například `1536` pro `text-embedding-3-small`). |
| `ttl` | `5m` | Jak dlouho položka platí (`"30s"`, `"1h"` nebo počet sekund). |
| `threshold` | `0.8` | Minimální kosinová podobnost pro sémantickou shodu. |
| `conversation_history_threshold` | `3` | Konverzace s víc zprávami se neukládají. |
| `default_cache_key` | prázdné | Klíč cache pro požadavky bez hlavičky. Prázdný znamená, že se bez hlavičky necachuje. |
| `cache_tool_call_responses` | `false` | Ukládat i odpovědi s voláním nástrojů. |

Ve webovém rozhraní: **Settings** → **Caching** → **Enable Caching**, zvolte režim, u sémantického vyplňte poskytovatele, model a dimenzi → **Save Changes**. Změny platí hned, bez restartu.

## Klíč cache je povinný

Nejčastější chyba při prvním nasazení: cache se použije **jen** u požadavku s hlavičkou `x-bf-cache-key` (nebo při nastaveném `default_cache_key`).

```bash
# This request WILL be cached
curl -H "x-bf-cache-key: session-123" ...

# This request will NOT be cached (no header, no default_cache_key)
curl ...
```

Klíč určuje, kdo se s kým o cache dělí. Hrubý klíč (název funkce) dává víc zásahů napříč uživateli, jemný klíč (ID uživatele nebo relace) drží cache soukromou. Navíc je každá položka automaticky omezená na identitu, která ji zapsala (virtuální klíč a uživatel). Dva různé virtuální klíče tak nikdy nedostanou odpověď uloženou tím druhým.

Další hlavičky pro jednotlivé požadavky:

| Hlavička | Účinek |
|---|---|
| `x-bf-cache-ttl` | Jiná platnost pro tento požadavek. |
| `x-bf-cache-threshold` | Přísnější práh podobnosti. |
| `x-bf-cache-type` | Jen `direct`, nebo jen `semantic`. |
| `x-bf-cache-no-store` | Odpověď neukládat (zásahy z cache se vrací dál). |

## Jak poznat zásah

Každá odpověď, u které cache hledala, nese hlavičky `x-bifrost-cache-hit` (`true` nebo `false`), `x-bifrost-cache-id` a při zásahu `x-bifrost-cache-hit-type` (`direct` nebo `semantic`). Stejné údaje jsou v těle odpovědi v `extra_fields.cache_debug`.

Smazání položek:

```bash
# Clear a specific cached entry by cache ID
curl -X DELETE http://localhost:8080/api/cache/clear/550e8500-e29b-41d4-a725-446655440001

# Clear all entries for a cache key
curl -X DELETE http://localhost:8080/api/cache/clear-by-key/support-session-456

# URL-encode a key that contains spaces, colons or slashes ("tenant a:session/42")
curl -X DELETE http://localhost:8080/api/cache/clear-by-key/tenant%20a%3Asession%2F42
```

> [!TIP]
> Vyhledávání v cache přidává latenci ke každému požadavku, u kterého cache běží: u přímé shody jednu cestu do úložiště, u sémantické navíc volání embeddingového modelu při každém minutí. Zápis do cache je asynchronní a odpověď nezdržuje.

## Další kroky

- **Podrobnosti u výrobce:** [Semantic Caching](https://docs.getbifrost.ai/features/semantic-caching) (Helm, Go SDK, životní cyklus položek), [Vector Store](https://docs.getbifrost.ai/architecture/framework/vector-store), [Prompt Caching](https://docs.getbifrost.ai/features/prompt-caching).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/features/semantic-caching.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu. Kód je převzatý beze změny. Originál: https://docs.getbifrost.ai/features/semantic-caching</sub>
