<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/plugins/getting-started.mdx, docs/plugins/sequencing.mdx, docs/deployment-guides/config-json/plugins.mdx, docs/plugins/writing-wasm-plugin.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Pluginy v kostce

*Co pluginy v Bifrostu umí, které jsou vestavěné a kdy se vyplatí napsat vlastní.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis je v dokumentaci výrobce: [Plugins: Getting Started](https://docs.getbifrost.ai/plugins/getting-started), [Plugin Sequencing](https://docs.getbifrost.ai/plugins/sequencing) a [config.json: Plugins](https://docs.getbifrost.ai/deployment-guides/config-json/plugins).

## Co je plugin

Plugin zachytí požadavek nebo odpověď na cestě branou a může je upravit, zalogovat, ověřit, obohatit, nebo požadavek rovnou ukončit vlastní odpovědí. Typické použití:

- úprava požadavků a odpovědí (doplnění hlaviček, přepis obsahu),
- vlastní validace podle firemních pravidel,
- vlastní cache nebo směrování,
- napojení na interní systémy (logování, monitoring, schvalování).

## Vestavěné pluginy

Část funkcí Bifrostu jsou pluginy, které už máte:

| Plugin | Jak se zapíná |
|---|---|
| Telemetrie (Prometheus `/metrics`) | vždy aktivní |
| Logování | `client.enable_logging` a nastavený `logs_store` |
| Governance (virtuální klíče, rozpočty, směrování) | vždy aktivní (OSS) |
| Sémantická cache (`semantic_cache`) | položka v poli `plugins` |
| OpenTelemetry (`otel`) | položka v poli `plugins` |
| Maxim (`maxim`) | položka v poli `plugins` |
| Datadog (`datadog`) | položka v poli `plugins`, jen Enterprise |

Pole `plugins` v `config.json` tedy obsahuje jen pluginy, které se zapínají výslovně, a vaše vlastní. Viz [Sémantická cache](./semanticka-cache.md) a [Observabilita](./observabilita.md).

## Vlastní pluginy: jaké jsou možnosti

| Typ | Stav | Poznámka |
|---|---|---|
| **Go plugin** (`.so`) | doporučený | Nativní kód v procesu brány, plný přístup k typům Bifrostu, bez režie. Vyžaduje dynamicky linkovanou binárku Bifrostu. |
| **WASM plugin** | deprecated | Stávající běží dál, nové nepište. |
| **Webhooky** | výrobce je chystá | Má jít o další cestu pro pluginy v jiných jazycích než Go. |

> [!IMPORTANT]
> Výchozí buildy Bifrostu včetně oficiální image `maximhq/bifrost` jsou linkované staticky a vlastní Go plugin **nenačtou**. Potřebujete vlastní dynamický build. Postup je v návodu [Dynamická image](./dynamicka-image.md).

## Hooky a jejich pořadí

Plugin exportuje funkce (hooky), které Bifrost volá v tomto pořadí:

| Pořadí | Hook | Kdy běží |
|---|---|---|
| 1 | `HTTPTransportPreAuthHook` | HTTP požadavek před ověřením (od v2.0), třeba pro odvození virtuálního klíče |
| 2 | `HTTPTransportPreHook` | HTTP požadavek po ověření |
| 3 | `PreRequestHook` | jednou na požadavek, rozhodnutí o poskytovateli, modelu a fallbackách |
| 4 | `PreLLMHook` / `PreMCPHook` | před každým pokusem u poskytovatele, může požadavek ukončit vlastní odpovědí |
| 5 | volání poskytovatele | pokud ho plugin neukončil |
| 6 | `PostLLMHook` / `PostMCPHook` | po každém pokusu, v opačném pořadí než pre-hooky |
| 7 | `HTTPTransportPostHook` | HTTP odpověď |
| 8 | `HTTPTransportResponseHeadersHook` | poslední chvíle pro úpravu hlaviček, i u streamingu |
| 9 | `HTTPTransportStreamChunkHook` | každý kus streamované odpovědi |

Plugin vždy exportuje `GetName` a `Cleanup`, obvykle i `Init` pro načtení konfigurace. Ostatní hooky jsou volitelné a exportujete jen ty, které potřebujete. Překlep v názvu hooku chybu nevyvolá, hook se prostě nezavolá. HTTP hooky běží jen v bráně (`bifrost-http`), ne při použití Bifrostu jako Go SDK.

## Kde v řetězci plugin běží

Vlastní pluginy se řadí do tří skupin vůči vestavěným:

| Skupina | Pre-hooky | Post-hooky |
|---|---|---|
| `pre_builtin` | první | poslední |
| `builtin` | druhé | druhé |
| `post_builtin` | třetí | první |

V rámci skupiny rozhoduje `order` (nižší dřív). Ve webovém rozhraní: **Plugins** → **Edit Plugin Sequence**, přetáhněte plugin nad blok **Built-in Plugins** nebo pod něj → **Save Sequence**. Ověřovací plugin patří do `pre_builtin`, logování výsledné odpovědi do `post_builtin`.

> [!WARNING]
> Nastavení `path` u vlastního pluginu přes API načte do procesu brány nativní kód. Proto vyžaduje přihlášenou relaci administrátora, i když je jinak ověřování dashboardu vypnuté. Vzdálenou cestu (`http`, `https`) Bifrost stáhne jen z veřejné adresy.

## Vestavěný, nebo vlastní?

Než začnete psát plugin, ověřte, jestli potřebu nepokryje vestavěná funkce: směrování podle hlaviček nebo rozpočtu řeší [Routing Rules](https://docs.getbifrost.ai/providers/routing-rules), omezení modelů a nástrojů [virtuální klíče](./virtualni-klice.md), cache [sémantická cache](./semanticka-cache.md). Vlastní plugin má smysl pro logiku specifickou pro vaši firmu, například detekci českých osobních údajů, napojení na interní schvalování nebo vlastní účtování nákladů.

## Další kroky

- **[Vlastní plugin v Go](./vlastni-plugin-go.md):** kostra pluginu, sestavení a načtení.
- **[Dynamická image](./dynamicka-image.md):** Bifrost, který vlastní plugin načte, v Dockeru i Kubernetes.
- **Podrobnosti u výrobce:** [Writing Go Plugins](https://docs.getbifrost.ai/plugins/writing-go-plugin), [Plugin Sequencing](https://docs.getbifrost.ai/plugins/sequencing), [Plugin Migration Guide](https://docs.getbifrost.ai/plugins/migration-guide), [ukázkové pluginy](https://github.com/maximhq/bifrost/tree/main/examples/plugins).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubory `docs/plugins/getting-started.mdx`, `docs/plugins/sequencing.mdx`, `docs/deployment-guides/config-json/plugins.mdx` a `docs/plugins/writing-wasm-plugin.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu. Originály: https://docs.getbifrost.ai/plugins/getting-started, https://docs.getbifrost.ai/plugins/sequencing a https://docs.getbifrost.ai/deployment-guides/config-json/plugins</sub>
