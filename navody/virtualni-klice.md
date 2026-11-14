<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/features/governance/virtual-keys.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Virtuální klíče

*Jak dát týmům a aplikacím vlastní klíč k bráně s omezením modelů, rozpočtem a limity.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis všech možností je v [dokumentaci výrobce](https://docs.getbifrost.ai/features/governance/virtual-keys).

## K čemu jsou

Virtuální klíč (virtual key, VK) je hlavní prvek governance v Bifrostu. Aplikace ani lidé nedostávají skutečné klíče poskytovatelů, ale virtuální klíč `sk-bf-...`, ke kterému jsou navázaná pravidla:

- **přístup:** které poskytovatele a modely smí klíč používat,
- **náklady:** rozpočty klíče, případně i týmu nebo zákazníka, ke kterému patří,
- **limity:** počet požadavků a tokenů za období,
- **omezení klíčů:** které klíče poskytovatelů smí brána pro daný VK použít,
- **stav a platnost:** klíč jde okamžitě vypnout nebo mu nastavit expiraci.

Klíč patří buď jednomu týmu (team), nebo jednomu zákazníkovi (customer), nebo nikomu. Obojí najednou nejde.

## Jak klíč poslat

Bifrost přijme virtuální klíč v kterékoli z těchto hlaviček, takže funguje s SDK různých výrobců bez úprav:

- `x-bf-vk`: vlastní hlavička Bifrostu,
- `Authorization: Bearer sk-bf-...`: styl OpenAI,
- `x-api-key`: styl Anthropic,
- `x-goog-api-key`: styl Google Gemini,
- `api-key`: styl Azure OpenAI.

Starší klíče bez prefixu `sk-bf-` fungují jen v hlavičce `x-bf-vk`.

## Vytvoření klíče

### Ve webovém rozhraní

1. Otevřete **Virtual Keys** a klikněte na **Add Virtual Key**.
2. V části **Provider Configurations** přidejte poskytovatele, které klíč smí používat, a u každého povolené modely.
3. Volitelně nastavte rozpočet (**Max Limit**, **Reset Duration**), limity (**Token Limit**, **Request Limit**), přiřazení k týmu nebo zákazníkovi a expiraci (**Expiry**).
4. Klikněte na **Create Virtual Key**.

> [!WARNING]
> Nový klíč je ve výchozím stavu **deny-by-default**: smí jen poskytovatele uvedené v `provider_configs`. Klíč bez poskytovatelů neprojde nikam. Pokud chcete povolit všechny nastavené poskytovatele včetně těch přidaných později, zapněte **Allow all providers** (`allow_all_providers`).

### Přes API

Klíč pro tým s dvěma poskytovateli, rozpočtem 100 USD měsíčně a limity:

```bash
curl -X POST http://localhost:8080/api/governance/virtual-keys \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Engineering Team API",
    "description": "Main API key for engineering team",
    "provider_configs": [
      {
        "provider": "openai",
        "weight": 0.5,
        "allowed_models": ["gpt-4o-mini"],
        "key_ids": ["8c52039e-38c6-48b2-8016-0bd884b7befb"]
      },
      {
        "provider": "anthropic",
        "weight": 0.5,
        "allowed_models": ["claude-3-sonnet-20240229"],
        "key_ids": ["*"]
      }
    ],
    "team_id": "team-eng-001",
    "budgets": [
      {
        "max_limit": 100.00,
        "reset_duration": "1M"
      }
    ],
    "rate_limit": {
      "token_max_limit": 10000,
      "token_reset_duration": "1h",
      "request_max_limit": 100,
      "request_reset_duration": "1m"
    },
    "is_active": true
  }'
```

Pole `key_ids` určuje, které klíče poskytovatele smí VK použít. `["*"]` znamená všechny, prázdné pole nebo chybějící pole znamená žádný.

### V config.json

Nejjednodušší klíč, který smí všechny nastavené poskytovatele:

```json
{
  "governance": {
    "virtual_keys": [
      {
        "id": "vk-all",
        "name": "All Providers Key",
        "value": "sk-bf-*",
        "is_active": true,
        "allow_all_providers": true
      }
    ]
  }
}
```

## Použití klíče

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-bf-vk: sk-bf-*" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## Povinné virtuální klíče

Ve výchozím stavu je governance volitelná: požadavek bez virtuálního klíče projde, jen bez kontrol a směrování. Pro produkci doporučujeme klíče vynutit.

- **Ve webovém rozhraní:** **Settings** → **Security** → zapněte **Enforce Virtual Keys on Inference** (v Enterprise se přepínač jmenuje **Enable Auth on Inference**) → **Save Changes**.
- **V config.json:**

```json
{
  "client": {
    "enforce_auth_on_inference": true
  }
}
```

Když je zapnuté ověřování inferenčních volání, hlavička `Authorization` slouží k ověření a virtuální klíč se posílá v `x-bf-vk`. Postup zapnutí ověřování popisuje [Nastavení ověřování](../docs/quickstart/gateway/setting-up-auth.md).

## Expirace

Klíči lze nastavit čas vypršení (`expires_at`, ve formátu RFC3339). Po něm brána požadavky s klíčem odmítá kódem `403`, klíč ale zůstává v systému kvůli auditu a jde kdykoli obnovit prodloužením nebo zrušením expirace. Volitelně je možné vypršené klíče jednou denně automaticky mazat (**Settings** → **Security** → **Delete Expired Virtual Keys**).

## Nejčastější chybové odpovědi

| Kód | Co se stalo |
|---|---|
| `401` | Chybí přihlašovací údaj a virtuální klíče jsou povinné. |
| `402` | Vyčerpaný rozpočet (klíče, týmu nebo zákazníka). |
| `403` | Klíč je neaktivní nebo vypršel, případně nepovoluje daného poskytovatele, model nebo MCP nástroj. |
| `429` | Překročený limit požadavků nebo tokenů. |

## Další kroky

- **[Rozpočty a limity](./rozpocty-a-limity.md):** jak se kontrolují rozpočty v hierarchii zákazník, tým, klíč.
- **[Fallbacky a vyvažování zátěže](./fallbacky-a-vyvazovani.md):** váhy poskytovatelů na klíči a automatické fallbacky.
- **Podrobnosti u výrobce:** [Virtual Keys](https://docs.getbifrost.ai/features/governance/virtual-keys) (týmy a zákazníci, navýšení rozpočtu, metadata klíče, přístup k relay, logování obsahu po klíčích), [Routing](https://docs.getbifrost.ai/features/governance/routing), [MCP Tool Filtering](https://docs.getbifrost.ai/features/governance/mcp-tools).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/features/governance/virtual-keys.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu. Kód je převzatý beze změny. Originál: https://docs.getbifrost.ai/features/governance/virtual-keys</sub>
