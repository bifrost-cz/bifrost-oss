<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/features/governance/budget-and-limits.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Rozpočty a limity

*Jak omezit útratu a provoz na úrovni zákazníka, týmu, virtuálního klíče i jednotlivého poskytovatele.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis všech možností je v [dokumentaci výrobce](https://docs.getbifrost.ai/features/governance/budget-and-limits).

## Hierarchie

Rozpočty a limity se navazují na [virtuální klíče](./virtualni-klice.md). Každá úroveň má vlastní, nezávislý rozpočet:

```
Customer (has independent budget)
    ↓ (one-to-many)
Team (has independent budget) 
    ↓ (one-to-many)
Virtual Key (has independent budget + rate limits)
    ↓ (one-to-many)
Provider Config (has independent budget + rate limits)

OR

Customer (has independent budget)
    ↓ (direct attachment)
Virtual Key (has independent budget + rate limits)
    ↓ (one-to-many)
Provider Config (has independent budget + rate limits)

OR

Virtual Key (standalone - has independent budget + rate limits)
    ↓ (one-to-many)
Provider Config (has independent budget + rate limits)
```

## Jak se rozpočty kontrolují

Při požadavku Bifrost zkontroluje **všechny** rozpočty, které se na něj vztahují. Projde jen tehdy, když má zůstatek každý z nich:

```
1. ✓ Provider Config Budget (if provider config has budget)
2. ✓ VK Budget (if VK has budget)
3. ✓ Team Budget (if VK's team has budget)  
4. ✓ Customer Budget (if team's customer has budget)
```

- Stačí jeden vyčerpaný rozpočet a požadavek je odmítnut (kód `402`).
- Cena požadavku se odečte ze všech příslušných rozpočtů najednou.
- Poskytovatel, který vyčerpal svůj rozpočet nebo limit, se vyřadí ze směrování. Ostatní poskytovatelé na stejném klíči fungují dál.

Příklad z dokumentace výrobce:

```
- Provider config budget: $4/$5 remaining ✓
- VK budget: $9/$10 remaining ✓
- Team budget: $15/$20 remaining ✓  
- Customer budget: $45/$50 remaining ✓
- Result: Allowed (no budget is exceeded)

- After request: 
    - Request cost: $2 
    - Updated Provider=$6/$5, VK=$11/$10, Team=$17/$20, Customer=$47/$50
    - Then the next request will be blocked (both provider and VK budgets exceeded).
```

Cenu Bifrost počítá sám podle ceníku modelů, počtu vstupních a výstupních tokenů, typu požadavku a toho, jestli šlo o odpověď z cache nebo dávku.

## Limity požadavků a tokenů

Vedle rozpočtů lze nastavit dva druhy limitů, které platí současně:

- **limit požadavků:** maximální počet volání za období (například 100 za minutu),
- **limit tokenů:** maximální počet tokenů (vstup i výstup) za období (například 50 000 za hodinu).

Limity se kontrolují na úrovni konfigurace poskytovatele, virtuálního klíče, týmu i zákazníka. Překročení vrací kód `429`.

## Období resetu

| Zápis | Období |
|---|---|
| `1m`, `5m` | minuta, 5 minut |
| `1h` | hodina |
| `1d` | den |
| `1w` | týden |
| `1M` | měsíc |
| `1Q` | čtvrtletí (jen rozpočty) |
| `1Y` | rok |

Ve výchozím stavu je období klouzavé: počítá se od posledního resetu. S `calendar_aligned: true` se rozpočet resetuje na začátku kalendářního období v UTC (půlnoc, pondělí, první den měsíce, čtvrtletí nebo roku). Nastavuje se na vlastníkovi rozpočtu (zákazník, tým, klíč), ne na jednotlivém rozpočtu.

## Příklad: rozpočet po poskytovatelích

Klíč marketingu smí utratit 100 USD měsíčně, z toho nejvýš 50 USD u OpenAI a 30 USD u Anthropicu:

```json
{
  "governance": {
    "virtual_keys": [
      {
        "id": "vk-mkt",
        "name": "marketing-team-vk",
        "provider_configs": [
          {"id": 30, "provider": "openai",    "allowed_models": ["*"], "key_ids": ["*"], "weight": 0.7},
          {"id": 31, "provider": "anthropic", "allowed_models": ["*"], "key_ids": ["*"], "weight": 0.3}
        ]
      }
    ],
    "budgets": [
      {"id": "b-vk-mkt", "virtual_key_id": "vk-mkt",      "max_limit": 100, "reset_duration": "1M"},
      {"id": "b-openai", "provider_config_id": 30,        "max_limit": 50,  "reset_duration": "1M"},
      {"id": "b-anth",   "provider_config_id": 31,        "max_limit": 30,  "reset_duration": "1M"}
    ]
  }
}
```

Rozpočty a limity jsou v `config.json` samostatná pole v sekci `governance`. Klíče a konfigurace poskytovatelů se na ně odkazují přes `id` (`rate_limit_id`), nebo se rozpočet odkazuje zpět (`virtual_key_id`, `provider_config_id`).

## Příklad: levný poskytovatel jako primární

Levný poskytovatel s denním rozpočtem 10 USD, dražší jako záloha s vlastním rozpočtem a limitem:

```json
{
  "governance": {
    "virtual_keys": [
      {
        "id": "vk-cost",
        "name": "cost-optimized-vk",
        "provider_configs": [
          {"id": 50, "provider": "openai-cheap",   "allowed_models": ["*"], "key_ids": ["*"], "weight": 1.0},
          {"id": 51, "provider": "openai-premium", "allowed_models": ["*"], "key_ids": ["*"], "weight": 0.0, "rate_limit_id": "rl-premium"}
        ]
      }
    ],
    "budgets": [
      {"id": "b-cheap",   "provider_config_id": 50, "max_limit": 10, "reset_duration": "1d"},
      {"id": "b-premium", "provider_config_id": 51, "max_limit": 50, "reset_duration": "1d"}
    ],
    "rate_limits": [
      {"id": "rl-premium", "request_max_limit": 100, "request_reset_duration": "1h", "token_max_limit": 50000, "token_reset_duration": "1h"}
    ]
  }
}
```

Když levný poskytovatel vyčerpá rozpočet, provoz automaticky přejde na dražší. Aby to fungovalo, neposílejte v požadavku název poskytovatele, jen model (viz [Fallbacky a vyvažování zátěže](./fallbacky-a-vyvazovani.md)).

## Dočasné navýšení rozpočtu

Rozpočtu klíče lze dočasně přidat kapacitu bez změny základního limitu: ve webovém rozhraní otevřete klíč, v části **Budget Information** klikněte na **Add override** a zvolte částku a dobu platnosti (počet období, nebo do odvolání).

> [!WARNING]
> Výrobce upozorňuje, že provoz více OSS uzlů nad sdílenou databází PostgreSQL není podporovaný. Bifrost drží rozpočty, spotřebu a konfiguraci v paměti a po startu je z databáze znovu nenačítá, takže by se uzly rozešly. Synchronizaci stavu mezi uzly řeší Enterprise ([Clustering](https://docs.getbifrost.ai/enterprise/clustering)). Podle výrobce zvládne jedna OSS instance kolem 3 000 až 5 000 požadavků za sekundu.

## Další kroky

- **[Virtuální klíče](./virtualni-klice.md):** vytvoření klíče a vynucení klíčů na bráně.
- **Podrobnosti u výrobce:** [Budget and Limits](https://docs.getbifrost.ai/features/governance/budget-and-limits) (fiskální čtvrtletí, ruční reset spotřeby, požadavky na navýšení v Enterprise), [Model Limits](https://docs.getbifrost.ai/features/governance/model-limits).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/features/governance/budget-and-limits.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu. Kód je převzatý beze změny. Originál: https://docs.getbifrost.ai/features/governance/budget-and-limits</sub>
