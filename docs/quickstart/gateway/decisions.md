<!--
zdroj: maximhq/bifrost docs/quickstart/gateway/decisions.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
prelozeno: 2026-11-14
-->

# Decisions

*Položte otázky ke vstupu a získejte typované odpovědi pomocí `/v1/decisions`.*

Rozhodovací modely (decision models) odpovídají na typované otázky ke vstupu: jaká je pravděpodobnost, že něco platí, která možnost sedí nebo kam vstup spadá na hodnoticí škále. Pošlete `input` a seznam `questions` a zpět dostanete na každou otázku jednu typovanou odpověď, ve stejném pořadí.

## Endpointy

Bifrost obsluhuje rozhodnutí na třech cestách. Každá z nich dosáhne na **všechny** níže uvedené poskytovatele rozhodnutí. Liší se jen tvarem požadavku a odpovědi, takže zvolte tu, která odpovídá vašemu klientovi.

| Cesta | Tvar | Použití z | Kam jde samotný název modelu |
|-------|-------|-------------|-------------------------|
| `POST /v1/decisions` | Jednotný tvar Bifrostu (tato stránka) | `curl`, jakýkoli HTTP klient, [Go SDK](https://docs.getbifrost.ai/quickstart/go-sdk/decisions) | K nastavenému poskytovateli, který ho obsluhuje, vybranému podle pravidel směrování nebo katalogu modelů |
| `POST /openai/v1/decisions` | Decisions API od OpenAI | [OpenAI SDK](https://docs.getbifrost.ai/integrations/openai-sdk/overview#decisions-api) | `openai` |
| `POST /typesafe/v1/systemone` | System One API od TypeSafe | [TypeSafe SDK](https://docs.getbifrost.ai/integrations/typesafe-sdk) | `typesafe` |

## Poskytovatelé

Pokud chcete poskytovatele zvolit výslovně, zapisujte modely jako `provider/model`. Přepnutí poskytovatele mění jen `model`.

| Poskytovatel | Příklady modelů | Endpoint v Bifrostu | Obrázky |
|----------|----------------|------------------|--------|
| [TypeSafe](https://docs.getbifrost.ai/providers/supported-providers/typesafe) | `typesafe/jev-latest`, `typesafe/jev-1.14.0` | `/v1/decisions` nebo `/typesafe/v1/systemone` (nativní API TypeSafe) | Ano, odkazem (viz [Obrázky odkazem](#obrázky-odkazem)) |
| [OpenAI](https://docs.getbifrost.ai/providers/supported-providers/openai#15-decisions) | `openai/gpt-6-luna` | `/v1/decisions` nebo `/openai/v1/decisions` (nativní API OpenAI) | Ano |
| [OpenRouter](https://docs.getbifrost.ai/providers/supported-providers/openrouter#decisions) | `openrouter/~typesafe/jev-latest` | `/v1/decisions` | Záleží na modelu |
| [Cloudflare](https://docs.getbifrost.ai/providers/supported-providers/cloudflare#4-decisions) | `cloudflare/clef`, `cloudflare/clef-flash`, `cloudflare/clef-omni` | `/v1/decisions` | Ano |
| [Perplexity](https://docs.getbifrost.ai/providers/supported-providers/perplexity#3-decisions) | `perplexity/pplx-decider-v1.1-27b`, `perplexity/pplx-decider-v1-27b` | `/v1/decisions` | Ano |
| [Ollama](https://docs.getbifrost.ai/providers/supported-providers/ollama#6-decisions) | `ollama/nimble`, `ollama/clef-flash`, `ollama/tev1:0.8b` | `/v1/decisions` | Ano, jen base64 |
| [Azure](https://docs.getbifrost.ai/providers/supported-providers/azure#10-decisions) | `azure/Microsoft-Decision-1` (alias klíče ho namapuje na vaše nasazení) | `/v1/decisions` | Ne |
| [Databricks](https://docs.getbifrost.ai/providers/supported-providers/databricks#decisions) | `databricks/ai_decide`, `databricks/system.ai.openjev-qwen35-4b`, `databricks/system.ai.typesafeai-jev-1-14-0` | `/v1/decisions` | Modely System One, odkazem (viz [Obrázky odkazem](#obrázky-odkazem)) |
| [vLLM](https://docs.getbifrost.ai/providers/supported-providers/vllm#decisions) | `vllm/clef-flash`, `vllm/d1-3b`, jakýkoli načtený model | `/v1/decisions` | Záleží na modelu |
| [SGLang](https://docs.getbifrost.ai/providers/supported-providers/sgl#decisions) | `sgl/clef-flash`, `sgl/kev-4b`, jakýkoli načtený model | `/v1/decisions` | Ano (Clef) |
| [Laya, Nimble](https://docs.getbifrost.ai/providers/self-hosted-decision-models) | `laya/english`, `nimble/nimble-latest` | `/v1/decisions` | Ne |

Každý poskytovatel odpovídá také na `/openai/v1/decisions` a `/typesafe/v1/systemone`, když pošlete název `provider/model`, takže se SDK OpenAI nebo TypeSafe dostane ke kterémukoli z nich. Nastavení popisuje stránka každého poskytovatele.

Chatovací modely (například `openai/gpt-4o`) endpoint pro rozhodnutí nemají. Ve výchozím stavu vrátí `400` s kódem `unsupported_operation`. Se zapnutým nastavením kompatibility `convert_decision_to_responses` Bifrost odpověď emuluje přes Responses API modelu. Viz [Decision-to-Responses Emulation](https://docs.getbifrost.ai/features/compat-plugin#4-decision-to-responses-emulation).

## Základní požadavek

```bash
curl --location 'http://localhost:8080/v1/decisions' \
--header 'Content-Type: application/json' \
--data '{
  "model": "typesafe/jev-1.13.0",
  "input": "Customer message: I was double charged and nobody replied to my two emails. I want a refund today.",
  "questions": [
    { "type": "predicate", "name": "is_frustrated", "instructions": "Is the customer frustrated?" },
    {
      "type": "choice",
      "name": "category",
      "instructions": "Pick the ticket category",
      "choices": [
        { "value": "billing", "description": "charges and refunds" },
        { "value": "bug", "description": "product defects" },
        { "value": "other", "description": "anything else" }
      ]
    },
    {
      "type": "score",
      "name": "urgency",
      "instructions": "Rate how urgently this needs a human reply",
      "levels": [
        { "label": "low", "description": "can wait a week" },
        { "label": "medium", "description": "should be answered soon" },
        { "label": "high", "description": "needs a reply today" }
      ]
    }
  ]
}'
```

## Vstup

`input` je to, čeho se otázky týkají. Má jednu ze čtyř podob:

**Text:** řetězec.

```json
"input": "Customer message: I was double charged and nobody replied."
```

**Zprávy:** pole `{ "role", "content" }`. `content` je řetězec nebo seznam částí. Části použijte k posílání obrázků:

```json
"input": [
  {
    "role": "user",
    "content": [
      { "type": "input_text", "text": "Is this receipt itemized?" },
      { "type": "input_image", "image_url": "data:image/png;base64,iVBORw0KGgo...", "detail": "high" }
    ]
  }
]
```

- `image_url` je data URL v base64 nebo URL `https`. Ollama čte jen base64.
- `detail` je volitelné: `low`, `high`, `auto` nebo `original`.
- Části jsou `input_text`, `input_image` a `input_json` (`{ "type": "input_json", "data": { ... } }`, objekt nebo pole) ve zprávách uživatele a `output_text` a `refusal` ve zprávách asistenta. Poskytovatel, který čte jen text (Azure, `ai_decide` od Databricks, Laya, Nimble), obrázek odmítne s `400`, takže požadavek může převzít [záložní model](#záložní-modely-fallbacks), který obrázky čte. TypeSafe a modely System One od Databricks čtou obrázky jen odkazem, jak je popsáno níže, a obrázek vložený přímo odmítnou stejně. vLLM a SGLang v0.5.22 nebo novější předávají obrázky modelu, který odpoví nebo vrátí vlastní `400`. SGLang v0.5.21 obrázky ignoruje, takže může odpovědět bez nich, místo aby vrátil `400`.
- `text` v části `input_text` je řetězec nebo strukturovaná data (objekt nebo pole), která modely System One čtou beze změny a OpenAI dostane jako text JSON.

### Historie konverzace

Pokud chcete posoudit konverzaci, pošlete její zprávy spolu s voláními funkcí, voláními vlastních nástrojů a jejich výstupy, ve správném pořadí. Každý výstup uvádí volání, na které odpovídá, pomocí `call_id`:

```json
"input": [
  { "role": "user", "content": "What's the weather in Paris?" },
  { "type": "function_call", "call_id": "call_1", "name": "get_weather", "arguments": "{\"city\":\"Paris\"}" },
  { "type": "function_call_output", "call_id": "call_1", "output": "18C and sunny" },
  { "role": "assistant", "content": [{ "type": "output_text", "text": "It's 18C and sunny in Paris." }] }
]
```

Výstup nástroje je řetězec nebo seznam částí `input_text` a `input_image`. Rozhodovací modely OpenAI čtou každou položku tak, jak byla poslána. Ostatní modely čtou konverzaci jako text JSON, takže obrázek ve výstupu nástroje poskytovatel, který čte jen text, odmítne stejně jako obrázek ve zprávě.

### Obrázky odkazem

TypeSafe umístí obrázek všude, kde je zapsané jeho ID: ve vstupu, v instrukcích otázky nebo v kritériích choice, score a predicate. Každý obrázek pošlete jako část `input_image` s `id` a souborem zakódovaným v base64 v `data` a `id` napište tam, kam obrázek patří. Formát se určí z bajtů. Jediná zpráva s jednou částí `input_text` a takovými částmi pošle tento text jako stav pro TypeSafe, takže strukturovaná data zůstanou strukturovaná:

```json
"input": [
  {
    "role": "user",
    "content": [
      { "type": "input_text", "text": { "camera": "r-cam" } },
      { "type": "input_image", "id": "r-cam", "data": "/9j/4AAQ..." },
      { "type": "input_image", "id": "r-thumb", "data": "iVBORw0..." }
    ]
  }
]
```

Zde je `r-cam` důkazní materiál a `r-thumb` může definovat možnost volby (`{ "value": "thumbs_up", "description": "r-thumb" }`), aniž by se stal součástí důkazů. Bifrost přidá hlavičku, kterou TypeSafe pro obrázky vyžaduje. Obrázky poslané odkazem fungují jen s TypeSafe a modely System One od Databricks. Ostatní poskytovatelé je odmítnou, včetně vLLM a SGLang, které by jinak obrázek ignorovaly a odpověděly bez něj. Viz [TypeSafe: Images and Box](https://docs.getbifrost.ai/providers/supported-providers/typesafe#images-and-box).

Pokud chcete soubory poslat tak, jak jsou, místo base64, použijte požadavek multipart: JSON patří do části s názvem `request`, každý obrázek do vlastní části se souborem a část obrázku odkazuje na svou část se souborem pomocí `name` místo `data`:

```bash
curl http://localhost:8080/v1/decisions \
  -F 'request={
    "model": "typesafe/jev-1.14.0",
    "input": [{"role": "user", "content": [
      {"type": "input_text", "text": {"camera": "r-cam"}},
      {"type": "input_image", "id": "r-cam", "name": "camera-photo"}
    ]}],
    "questions": [{"type": "box", "name": "hand", "instructions": "the hand in `camera`"}]
  };type=application/json' \
  -F 'camera-photo=@camera.jpg;type=image/jpeg'
```

`name` v požadavku JSON, `name` bez odpovídající části se souborem i část se souborem, na kterou neodkazuje žádná část obrázku, se odmítnou s `400`.

**Strukturovaná data:** objekt, který se modelům System One (TypeSafe, Clef, Perplexity, Ollama, Azure, Databricks, vLLM, SGLang, Laya, Nimble) posílá beze změny a OpenAI jako text JSON.

```json
"input": { "ticket_id": 4812, "message": "I was double charged.", "plan": "pro" }
```

**`null`:** žádný vstup. OpenAI ho odmítne s `400`, Clef s `422` a Bifrost ho odmítne s `400` pro vLLM a SGLang, které by jinak odpověděly bez důkazů.

## Otázky

| Pole | Význam |
|-------|---------|
| `type` | `predicate`: pravděpodobnost, že tvrzení platí. `choice`: která možnost sedí. `score`: úroveň na seřazené škále (může padnout mezi úrovně). `box`: kde se v obrázku nachází předmět instrukcí (TypeSafe Jev 1.14: `jev-preview`, `jev-1.14.0`) |
| `name` | Volitelné. Vrací se v odpovědi, abyste ji mohli přiřadit |
| `instructions` | Řetězec nebo strukturovaná data (objekt nebo pole): modelům System One se posílají beze změny, OpenAI jako text JSON |
| `criteria` | Jen predicate, volitelné: co znamená `true` a `false`. U modelů bez pole pro kritéria se připojí k instrukcím |
| `choices` | Jen choice: `value` každé možnosti (řetězec nebo boolean) a volitelný `description` |
| `levels` | Jen score: úrovně v pořadí, každá s `label` a volitelným `description` |

`safety_identifier` (volitelné) je stálé ID koncového uživatele. Posílá se OpenAI a OpenRouteru jako `user`, ostatní poskytovatelé ho nepřijímají.

Pole jen pro konkrétního poskytovatele, například `min_confidence` u Laya, patří na nejvyšší úroveň těla s hlavičkou `x-bf-passthrough-extra-params: true`.

## Odpověď

```json
{
  "model": "jev-1.13.0",
  "answers": [
    { "type": "predicate", "name": "is_frustrated", "probability": 0.98 },
    {
      "type": "choice",
      "name": "category",
      "choice": "billing",
      "confidence": 1,
      "probabilities": [
        { "value": "billing", "probability": 1 },
        { "value": "bug", "probability": 0 },
        { "value": "other", "probability": 0 }
      ]
    },
    {
      "type": "score",
      "name": "urgency",
      "score": 2,
      "confidence": 1,
      "probabilities": [
        { "value": 0, "label": "low", "probability": 0 },
        { "value": 1, "label": "medium", "probability": 0 },
        { "value": 2, "label": "high", "probability": 1 }
      ],
      "legend": { "0": "low: can wait a week", "1": "medium: should be answered soon", "2": "high: needs a reply today" }
    }
  ],
  "usage": { "prompt_tokens": 437, "completion_tokens": 72, "total_tokens": 509 }
}
```

Na každou otázku připadá jedna odpověď, v pořadí otázek a s `type` otázky. Odpověď typu box nese `box`: jeho okraje `top`, `bottom`, `left` a `right`, každý jako zlomek 0 až 1 obrázku měřený od levého horního rohu. `confidence`, `probabilities` a `legend` se objeví, když je model poskytne. Model, který na otázku odpovědět odmítne, na ni odpoví s `"type": "refusal"`.

### Pole specifická pro poskytovatele

Někteří poskytovatelé vracejí další pole. Objeví se jen tehdy, když je daný poskytovatel poskytne.

| Poskytovatel | Kde | Pole |
|----------|-------|--------|
| Laya | Odpověď na otázku | `answer_confidence` (kalibrovaná pravděpodobnost uvedené odpovědi), `action` a při nastaveném `min_confidence` také `abstention` (`passed`, `abstained` nebo `unevaluated`), `abstention_threshold`, `low_confidence` |
| Laya | Odpověď | `routing`: checkpoint, který odpověděl, a proč |
| Laya | `usage` | `state_tokens`, `state_tokens_dropped`, `truncated`, `truncated_questions` |

## Záložní modely (fallbacks)

Přidejte `fallbacks`, pokud se mají při selhání postupně zkoušet další modely, včetně případu, kdy model odmítne vstup, který neumí přečíst (například obrázek poslaný TypeSafe). Záložní modely mohou být od jiných poskytovatelů:

```json
{
  "model": "typesafe/jev-latest",
  "fallbacks": ["cloudflare/clef-flash", "openai/gpt-6-luna"],
  "input": "Customer message: I was double charged and nobody replied to my two emails.",
  "questions": [
    { "type": "predicate", "name": "is_frustrated", "instructions": "Is the customer frustrated?" }
  ]
}
```

## Validace

Tyto požadavky Bifrost odmítne s `400` ještě před voláním jakéhokoli poskytovatele:

- neznámý `type` otázky,
- otázka `choice` bez `choices`,
- dvě otázky se stejným `name`.

Počet úrovní score kontroluje každý poskytovatel (TypeSafe, Azure a Databricks přijímají 2 až 10, Perplexity 1 až 10).

Část vstupu, kterou jeden model neumí přečíst, způsobí selhání jen tohoto pokusu, takže ji záložní model může ještě obsloužit. Pokud selžou všechny pokusy, dostanete chybu prvního pokusu.

## Mapový tvar (zastaralé)

> [!WARNING]
> Mapový tvar je zastaralý. Stále funguje, ale přejděte na výše uvedený tvar požadavku.

Požadavky v dřívějším mapovém tvaru se dál obsluhují a odpověď přijde ve stejném tvaru:

```json
{
  "model": "typesafe/jev-1.13.0",
  "state": "Customer message: I was double charged and nobody replied to my two emails. I want a refund today.",
  "questions": {
    "is_frustrated": { "kind": "noul", "instructions": "Is the customer frustrated?" },
    "category": {
      "kind": "choice",
      "instructions": "Pick the ticket category",
      "criteria": { "billing": "charges and refunds", "bug": "product defects", "other": "anything else" }
    },
    "urgency": {
      "kind": "score",
      "instructions": "Rate how urgently this needs a human reply",
      "criteria": ["can wait a week", "should be answered soon", "needs a reply today"]
    }
  }
}
```

Odpověď má odpovědi pod klíči podle názvu otázky, každou s `kind` a `value`:

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "is_frustrated": { "kind": "noul", "value": 0.98 },
    "category": { "kind": "choice", "value": "billing", "confidence": 1, "probabilities": { "billing": 1, "bug": 0, "other": 0 } },
    "urgency": { "kind": "score", "value": 2, "confidence": 1, "probabilities": { "0": 0, "1": 0, "2": 1 } }
  }
}
```

Bifrost tělo považuje za mapový tvar, když je `questions` objekt s klíči podle názvu místo seznamu. `state` poslaný spolu s `input` se ignoruje.

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/quickstart/gateway/decisions.mdx`, commit `0fb9342`. Změny: překlad do češtiny, převod z MDX do Markdownu. Originál: https://docs.getbifrost.ai/quickstart/gateway/decisions</sub>
