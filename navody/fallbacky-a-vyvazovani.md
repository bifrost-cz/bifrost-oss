<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/features/retries-and-fallbacks.mdx, docs/features/governance/routing.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Fallbacky a vyvažování zátěže

*Jak udržet aplikaci v provozu při výpadku, rate limitu nebo vyčerpané kvótě poskytovatele.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis je v dokumentaci výrobce: [Retries & Fallbacks](https://docs.getbifrost.ai/features/retries-and-fallbacks) a [Routing](https://docs.getbifrost.ai/features/governance/routing).

## Tři vrstvy odolnosti

| Vrstva | Co dělá | Kde se nastavuje |
|---|---|---|
| **Opakování (retries)** | Při přechodné chybě zkusí stejný požadavek znovu u stejného poskytovatele, případně s jiným klíčem. | `network_config` poskytovatele |
| **Fallbacky** | Když poskytovatel selže i po opakováních, přejde na dalšího v řetězci. | v požadavku, nebo automaticky z virtuálního klíče |
| **Vyvažování zátěže** | Rozkládá provoz mezi poskytovatele nebo klíče podle vah. | váhy klíčů poskytovatele a váhy na virtuálním klíči |

## Opakování

Ve výchozím stavu se neopakuje nic (`max_retries: 0`). Doporučujeme nastavit aspoň 2 až 3 opakování a mít u poskytovatele víc klíčů:

```json
{
  "providers": {
    "openai": {
      "keys": [
        { "name": "openai-key-1", "value": "env.OPENAI_KEY_1", "models": ["*"], "weight": 1.0 },
        { "name": "openai-key-2", "value": "env.OPENAI_KEY_2", "models": ["*"], "weight": 1.0 },
        { "name": "openai-key-3", "value": "env.OPENAI_KEY_3", "models": ["*"], "weight": 1.0 }
      ],
      "network_config": {
        "max_retries": 3,
        "retry_backoff_initial": 500,
        "retry_backoff_max": 5000
      }
    }
  }
}
```

Ve webovém rozhraní: **Providers** → vyberte poskytovatele → **Network Config** → **Max Retries**, **Retry Backoff Initial**, **Retry Backoff Max**.

Jak Bifrost reaguje na jednotlivé chyby:

| Chyba | Opakuje? | Jiný klíč? | Čeká před dalším pokusem? |
|---|---|---|---|
| Síťová chyba, `5xx` | ano | ne, stejný klíč | ano, exponenciálně |
| Rate limit `429` | ano | ano | ano, kvóty bývají sdílené na účtu |
| Odmítnutý klíč (`401`, `403`), vyčerpaná kvóta (`402`), model nedostupný pro klíč | ano | ano, vadný klíč se pro požadavek vyřadí | ne |
| Chyba v požadavku (`400`, `422` a podobně) | ne | | |

Čekání začíná na 500 ms a zdvojuje se až do 5 s. Přechod na jiný klíč po trvalé chybě klíče (neplatný klíč, kvóta) funguje i s `max_retries: 0`, pokud má poskytovatel víc klíčů.

## Fallbacky v požadavku

Řetězec záložních modelů pošlete v poli `fallbacks` ve formátu `provider/model`:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "Explain quantum computing in simple terms"
      }
    ],
    "fallbacks": [
      "anthropic/claude-3-5-sonnet-20241022",
      "bedrock/anthropic.claude-3-sonnet-20240229-v1:0"
    ],
    "max_tokens": 1000,
    "temperature": 0.7
  }'
```

- Každý poskytovatel v řetězci dostane vlastní plný počet opakování.
- Vyhrává první úspěšná odpověď. Když selžou všichni, vrátí se chyba primárního poskytovatele.
- Fallback se nespustí, když klient požadavek zrušil nebo když fallbacky zakáže plugin. Při chybě v samotném požadavku se použijí jen fallbacky, které výslovně uvedl volající nebo pravidlo směrování.
- Kdo požadavek nakonec obsloužil, uvádí `extra_fields.routing_info` v odpovědi.

## Vyvažování a automatické fallbacky přes virtuální klíč

Na [virtuálním klíči](./virtualni-klice.md) můžete poskytovatelům dát váhy. Bifrost pak rozkládá provoz a sám sestaví řetězec fallbacků:

```
Virtual Key: vk-prod-main
├── OpenAI
│   ├── Allowed Models: [gpt-4o, gpt-4o-mini]  ← Explicit whitelist
│   └── Weight: 0.2 (20% of traffic)
└── Azure
    ├── Allowed Models: [gpt-4o]  ← Explicit whitelist
    └── Weight: 0.8 (80% of traffic)
```

- `gpt-4o`: 80 % Azure, 20 % OpenAI (oba ho mají v `allowed_models`).
- `gpt-4o-mini`: 100 % OpenAI.
- Model, který nepovoluje žádný poskytovatel, je odmítnut.

Konfigurace v `config.json`:

```json
{
  "governance": {
    "virtual_keys": [
      {
        "id": "vk-prod-main",
        "provider_configs": [
          {
            "provider": "openai",
            "allowed_models": ["gpt-4o", "gpt-4o-mini"],
            "key_ids": ["*"],
            "weight": 0.2
          },
          {
            "provider": "azure",
            "allowed_models": ["gpt-4o"],
            "key_ids": ["*"],
            "weight": 0.8
          }
        ]
      }
    ]
  }
}
```

Aby vyvažování fungovalo, posílejte v požadavku **jen název modelu**:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "x-bf-vk: vk-prod-main" \
  -d '{"model": "gpt-4o", "messages": [{"role": "user", "content": "Hello!"}]}'
```

S prefixem poskytovatele (`openai/gpt-4o`) vyvažování obejdete a míříte přímo na daného poskytovatele:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "x-bf-vk: vk-prod-main" \
  -d '{"model": "openai/gpt-4o", "messages": [{"role": "user", "content": "Hello!"}]}'
```

Pokud požadavek nemá vlastní pole `fallbacks`, Bifrost přidá jako fallbacky ostatní poskytovatele z klíče seřazené podle váhy. Vlastní `fallbacks` v požadavku mají přednost.

> [!TIP]
> Bifrost model mezi poskytovateli sám nepřekládá. Požadavek na `gpt-4o` nepůjde na Anthropic, dokud tam `gpt-4o` výslovně nepovolíte. Pro levného a dražšího poskytovatele se stejnými modely se hodí kombinace vah a rozpočtů, viz [Rozpočty a limity](./rozpocty-a-limity.md#příklad-levný-poskytovatel-jako-primární).

Vyvažování mezi více klíči stejného poskytovatele (váhy klíčů) popisuje [Konfigurace poskytovatelů](../docs/quickstart/gateway/provider-configuration.md#vážené-vyvažování-zátěže).

## Další kroky

- **[Observabilita](./observabilita.md):** v logu každého požadavku je `attempt_trail` se všemi pokusy a klíči.
- **Podrobnosti u výrobce:** [Retries & Fallbacks](https://docs.getbifrost.ai/features/retries-and-fallbacks) (klasifikace chyb, chování pluginů při fallbacku, audit rozhodnutí), [Routing](https://docs.getbifrost.ai/features/governance/routing), [Routing Rules](https://docs.getbifrost.ai/providers/routing-rules) (směrování podle hlaviček, rozpočtu nebo složitosti požadavku).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubory `docs/features/retries-and-fallbacks.mdx` a `docs/features/governance/routing.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu. Kód je převzatý beze změny. Originály: https://docs.getbifrost.ai/features/retries-and-fallbacks a https://docs.getbifrost.ai/features/governance/routing</sub>
