<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/features/observability/default.mdx, docs/features/telemetry.mdx, docs/features/observability/prometheus.mdx, docs/features/observability/otel.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Observabilita

*Logy požadavků, metriky pro Prometheus a trasování přes OpenTelemetry.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis je v dokumentaci výrobce: [Built-in Observability](https://docs.getbifrost.ai/features/observability/default), [Telemetry](https://docs.getbifrost.ai/features/telemetry), [Prometheus](https://docs.getbifrost.ai/features/observability/prometheus) a [OpenTelemetry](https://docs.getbifrost.ai/features/observability/otel).

## Přehled

| Co | K čemu | Výchozí stav |
|---|---|---|
| **Logy požadavků** | Každý požadavek s odpovědí, tokeny, cenou, latencí a pokusy | zapnuté, SQLite |
| **Metriky Prometheus** | Provoz, chyby, latence, tokeny, náklady, zásahy cache | zapnuté, endpoint `/metrics` |
| **OpenTelemetry** | Trasy (spans) do vašeho kolektoru, Grafany, Datadogu a dalších | vypnuté |

Všechno se zapisuje asynchronně a nezdržuje odpovědi.

## Logy požadavků

Bifrost ukládá ke každému požadavku vstupní zprávy a parametry, odpověď, poskytovatele a model, tokeny, cenu, latenci a stav. Při opakování nebo fallbacku navíc pole `attempt_trail` se všemi pokusy a použitými klíči.

Prohlížet je můžete ve webovém rozhraní (stránka logů, filtrování, vyhledávání v obsahu, náklady), přes API nebo živě přes WebSocket.

Nastavení v `config.json`:

```json
{
    "client": {
        "enable_logging": true,
        "disable_content_logging": false,
        "drop_excess_requests": false,
        "initial_pool_size": 300
    },
    "logs_store": {
        "enabled": true,
        "type": "sqlite",
        "config": {
            "path": "./logs.db"
        }
    }
}
```

- `enable_logging`: hlavní vypínač logů.
- `disable_content_logging`: neukládat obsah požadavků a odpovědí, jen metadata (latence, cena, tokeny). Hodí se tam, kde obsah nesmí opustit aplikaci nebo kde ho nechcete držet v databázi.
- `logs_store`: kam se logy ukládají.

Pro větší provoz použijte PostgreSQL (verze 16 a novější, kódování UTF8):

```json
{
    "logs_store": {
        "enabled": true,
        "type": "postgres",
        "config": {
            "host": "localhost",
            "port": "5432",
            "user": "bifrost",
            "password": "postgres",
            "db_name": "bifrost",
            "ssl_mode": "disable"
        }
    }
}
```

Dotaz na logy přes API s filtry:

```bash
curl 'http://localhost:8080/api/logs?' \
'providers=openai,anthropic&' \
'models=gpt-4o-mini&' \
'status=success,error&' \
'start_time=2024-01-15T00:00:00Z&' \
'end_time=2024-01-15T23:59:59Z&' \
'min_latency=1000&' \
'max_latency=5000&' \
'min_tokens=10&' \
'max_tokens=1000&' \
'min_cost=0.001&' \
'max_cost=10&' \
'content_search=python&' \
'limit=100&' \
'offset=0'
```

> [!TIP]
> Hlavičky s prefixem `x-bf-lh-` se automaticky uloží do metadat logu (například `x-bf-lh-tenant-id: acme` jako `tenant-id`). Tak si do logů dostanete oddělení, projekt nebo prostředí bez úprav aplikace.

## Metriky pro Prometheus

Telemetrický plugin je ve výchozím stavu zapnutý a metriky jsou na `GET /metrics` na stejném portu jako API. Mezi hlavní patří `bifrost_upstream_requests_total`, `bifrost_error_requests_total`, `bifrost_upstream_latency_seconds`, `bifrost_input_tokens_total` a `bifrost_output_tokens_total`, dále náklady v USD a zásahy cache.

```yaml
scrape_configs:
  - job_name: 'bifrost'
    static_configs:
      - targets: ['bifrost-host:8080']
    scrape_interval: 15s
```

Když je na bráně zapnuté ověřování, `/metrics` vyžaduje přihlášení. V OSS jde jen o Basic auth administrátora:

```yaml
scrape_configs:
  - job_name: 'bifrost'
    static_configs:
      - targets: ['bifrost-host:8080']
    scrape_interval: 15s
    basic_auth:
      username: '<admin_username>'
      password: '<admin_password>'
```

> [!WARNING]
> Bez TLS posílá Prometheus přihlašovací údaje nešifrovaně. Pokud Bifrost běží na HTTPS, nastavte ve scrape konfiguraci `scheme: https`.

Při více instancích za load balancerem doporučuje výrobce místo stahování metrik jejich odesílání do Push Gateway, jinak scraper nemusí zachytit všechny uzly.

## OpenTelemetry

OTel plugin posílá ke každému volání modelu span podle konvencí OpenTelemetry GenAI: poskytovatel, model, parametry, tokeny, cena, virtuální klíč a tým, počet opakování a volitelně obsah zpráv.

```json
{
  "plugins": [
    {
      "enabled": true,
      "name": "otel",
      "config": {
        "profiles": [
          {
            "service_name": "bifrost",
            "enabled": true,
            "collector_url": "http://localhost:4318/v1/traces",
            "trace_type": "genai_extension",
            "protocol": "http",
            "headers": {
              "Authorization": "env.OTEL_API_KEY"
            }
          }
        ]
      }
    }
  ]
}
```

- `collector_url`: endpoint OTLP kolektoru, `protocol`: `http` nebo `grpc`.
- Hodnoty v `headers` a `collector_url` mohou odkazovat na proměnné prostředí (`env.NAZEV`), klíče tak nemusí být v souboru.
- `disable_content_logging` v profilu vynechá obsah zpráv z exportovaných spanů.

Hotové nastavení pro Grafana Cloud, Datadog, New Relic, Honeycomb, Langfuse a další a ukázkový Docker Compose s kolektorem, Tempem, Prometheem a Grafanou najdete v [dokumentaci výrobce](https://docs.getbifrost.ai/features/observability/otel).

## Další kroky

- **[Fallbacky a vyvažování zátěže](./fallbacky-a-vyvazovani.md):** co znamenají pokusy v `attempt_trail`.
- **Podrobnosti u výrobce:** [Content Logging](https://docs.getbifrost.ai/features/observability/content-logging) (logování obsahu po klíčích a požadavcích), [Latency Breakdown](https://docs.getbifrost.ai/features/observability/latency-breakdown), nativní exporty v Enterprise do [Datadogu](https://docs.getbifrost.ai/features/observability/datadog), [Splunku](https://docs.getbifrost.ai/features/observability/splunk), [Kafky](https://docs.getbifrost.ai/features/observability/kafka) a [BigQuery](https://docs.getbifrost.ai/features/observability/bigquery).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubory `docs/features/observability/default.mdx`, `docs/features/telemetry.mdx`, `docs/features/observability/prometheus.mdx` a `docs/features/observability/otel.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu. Kód je převzatý beze změny. Originály: https://docs.getbifrost.ai/features/observability/default, https://docs.getbifrost.ai/features/telemetry, https://docs.getbifrost.ai/features/observability/prometheus a https://docs.getbifrost.ai/features/observability/otel</sub>
