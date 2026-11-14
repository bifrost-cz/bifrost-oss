<!--
zdroj: maximhq/bifrost docs/quickstart/gateway/reranking.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
prelozeno: 2026-11-14
-->

# Reranking

*Přeřazení dokumentů podle relevance k dotazu pomocí `/v1/rerank`.*

Reranking použijte k seřazení dokumentů podle relevance pro vyhledávání, retrieval a výběr kontextu.

## Příklady modelů poskytovatelů

- Cohere: `cohere/rerank-v3.5`
- vLLM: `vllm/BAAI/bge-reranker-v2-m3`
- Bedrock: `bedrock/<rerank-model-or-arn>`
- Vertex AI: `vertex/<ranking-model>`

## Základní požadavek

```bash
curl --location 'http://localhost:8080/v1/rerank' \
--header 'Content-Type: application/json' \
--data '{
  "model": "cohere/rerank-v3.5",
  "query": "What is Bifrost?",
  "documents": [
    {"text": "Bifrost is an AI gateway that unifies many LLM providers."},
    {"text": "Paris is the capital of France."},
    {"text": "Bifrost exposes an OpenAI-compatible API."}
  ]
}'
```

## Parametry požadavku

- `model` (povinné): model ve formátu `provider/model`
- `query` (povinné): dotaz, podle kterého se řadí
- `documents` (povinné): pole dokumentů s `text` (volitelně `id`, `meta`)
- `top_n` (volitelné): maximální počet výsledků
- `max_tokens_per_doc` (volitelné): limit tokenů na dokument, závisí na poskytovateli
- `priority` (volitelné): nápověda priority, závisí na poskytovateli
- `return_documents` (volitelné): zahrnout do každého výsledku obsah nalezeného dokumentu
- `fallbacks` (volitelné): záložní modely ve formátu `provider/model`

## Příklad s možnostmi

```bash
curl --location 'http://localhost:8080/v1/rerank' \
--header 'Content-Type: application/json' \
--data '{
  "model": "cohere/rerank-v3.5",
  "query": "gateway observability",
  "top_n": 2,
  "return_documents": true,
  "documents": [
    {"id": "a", "text": "Bifrost supports observability plugins like OTEL and Maxim."},
    {"id": "b", "text": "Bifrost can run in Kubernetes and ECS."},
    {"id": "c", "text": "Token counting is available at /v1/responses/input_tokens."}
  ]
}'
```

## Kompatibilita endpointů vLLM

Při použití modelu `vllm/...` posílá Bifrost požadavky na rerank nejdřív na `/v1/rerank` a pokud upstream endpoint odpoví `404`, `405` nebo `501`, automaticky to zkusí znovu na `/rerank`.

## Vlastní poskytovatelé kompatibilní s OpenAI

Vlastní poskytovatelé postavení na základním typu `openai` mohou obsluhovat reranking proti jakémukoli endpointu kompatibilnímu s OpenAI, který nabízí kontrakt rerank ve stylu Cohere. Řídí se standardním chováním `allowed_requests`: když `allowed_requests` chybí, je povolený, jinak ho musíte výslovně zapnout pomocí `rerank: true`. Požadavky jdou ve výchozím stavu na `/v1/rerank`. Cestu změníte položkou `rerank` v `request_path_overrides`. Podrobnosti konfigurace najdete v části [Custom Providers](https://docs.getbifrost.ai/providers/custom-providers).

## Tvar odpovědi

```json
{
  "results": [
    {
      "index": 0,
      "relevance_score": 0.98,
      "document": {
        "id": "a",
        "text": "Bifrost supports observability plugins like OTEL and Maxim."
      }
    },
    {
      "index": 2,
      "relevance_score": 0.63,
      "document": {
        "id": "c",
        "text": "Token counting is available at /v1/responses/input_tokens."
      }
    }
  ],
  "model": "rerank-v3.5",
  "usage": {
    "prompt_tokens": 52,
    "completion_tokens": 0,
    "total_tokens": 52
  },
  "extra_fields": {
    "request_type": "rerank",
    "provider": "cohere",
    "latency": 245,
    "chunk_index": 0
  }
}
```

## Časté chyby validace

- Chybí `query` -> `query is required for rerank`
- Prázdné `documents` -> `documents are required for rerank`
- Prázdný text dokumentu -> `document text is required for rerank at index N`
- `top_n < 1` -> `top_n must be at least 1`

## Další kroky

Když už rozumíte rerankingu, podívejte se na související témata:

### Základní témata

- **[Multimodální AI](./multimodal.md)**: zpracování obrázků, zvuku a multimediálního obsahu.
- **[Volání nástrojů](./tool-calling.md)**: modely AI mohou používat externí nástroje a funkce.
- **[Konfigurace poskytovatelů](./provider-configuration.md)**: více poskytovatelů pro redundanci.
- **[Integrace](./integrations.md)**: přímá kompatibilita se stávajícími SDK.

### Pokročilá témata

- **[Core Features](https://docs.getbifrost.ai/features/)**: pokročilé schopnosti Bifrostu.
- **[Architecture](https://docs.getbifrost.ai/architecture/)**: jak Bifrost funguje uvnitř.
- **[Deployment](https://docs.getbifrost.ai/deployment-guides)**: produkční nasazení a škálování.

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/quickstart/gateway/reranking.mdx`, commit `0fb9342`. Změny: překlad do češtiny, převod z MDX do Markdownu. Originál: https://docs.getbifrost.ai/quickstart/gateway/reranking</sub>
