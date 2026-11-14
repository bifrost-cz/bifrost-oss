<!--
zdroj: maximhq/bifrost docs/quickstart/gateway/provider-configuration.mdx
upstream-commit: 7235fb54f388679e8c4c93ce9687a79ae084700a
prelozeno: 2026-11-12
-->

# Konfigurace poskytovatelů

*Nastavení více poskytovatelů AI s vlastní souběžností, velikostí front, proxy a dalšími volbami.*

## Více poskytovatelů

Nastavte více poskytovatelů, abyste mezi nimi mohli plynule přepínat. Příklad ukazuje nastavení poskytovatelů OpenAI, Anthropic a Mistral.

**Přes webové rozhraní**

![Rozhraní pro konfiguraci poskytovatelů](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-provider-configs.png)

1. Otevřete **http://localhost:8080**.
2. V postranním panelu přejděte na **"Model Providers"**.
3. Vyberte poskytovatele a nastavte klíče.

**Přes API**

```bash
# Add OpenAI provider
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ]
}'

# Add Anthropic provider
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "anthropic",
    "keys": [
        {
            "name": "anthropic-key-1",
            "value": "env.ANTHROPIC_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ]
}'

# Add vLLM (self-hosted OpenAI-compatible server)
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "vllm-local",
    "keys": [
        {
            "name": "vllm-key-1",
            "value": "dummy",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "network_config": {
        "base_url": "http://vllm-endpoint:8000",
        "default_request_timeout_in_seconds": 60
    },
    "custom_provider_config": {
        "base_provider_type": "openai",
        "allowed_requests": {
            "chat_completion": true,
            "chat_completion_stream": true
        }
    }
}'
```

**Přes config.json**

> [!NOTE]
> Každý klíč u poskytovatele musí mít jedinečný název.

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ]
    },
    "anthropic": {
      "keys": [
        {
          "name": "anthropic-key",
          "value": "env.ANTHROPIC_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ]
    },
    "vllm-local": {
      "keys": [
        {
          "name": "vllm-key",
          "value": "dummy",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "network_config": {
        "base_url": "http://vllm-endpoint:8000",
        "default_request_timeout_in_seconds": 60
      },
      "custom_provider_config": {
        "base_provider_type": "openai",
        "allowed_requests": {
          "chat_completion": true,
          "chat_completion_stream": true
        }
      }
    }
  }
}
```

> [!TIP]
> **DNS v Kubernetes (jen pro vlastní endpoint):** v Kubernetes používejte pro vlastní endpointy v jiném namespace plně kvalifikované doménové názvy (FQDN), například `http://<service>.<namespace>.svc.cluster.local:8000`. Krátké názvy jako `http://<service>:8000` fungují jen v rámci stejného namespace.

> [!TIP]
> **Prostředí bez internetu nebo certifikáty podepsané sami sebou:** pokud váš vlastní poskytovatel používá HTTPS s certifikátem podepsaným sám sebou nebo interní certifikační autoritou, přidejte do `network_config` `"insecure_skip_verify": true` nebo `"ca_cert_pem": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"`. Podrobnosti najdete v části [Custom Providers - TLS](https://docs.getbifrost.ai/providers/custom-providers#tls-for-self-signed-or-internal-certificates).

## Odesílání požadavků

Jakmile máte poskytovatele nastavené, můžete posílat požadavky kterémukoli z nich. Příklad posílá požadavek přímo na model GPT-4o Mini od OpenAI. Formátování API pro konkrétního poskytovatele zajistí Bifrost automaticky.

```bash
curl --location 'http://localhost:8080/v1/chat/completions' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o-mini",
    "messages": [
        {"role": "user", "content": "Hello!"}
    ]
}'
```

## Proměnné prostředí

Nastavte API klíče poskytovatelů, které chcete používat. Bifrost podporuje přímé hodnoty klíčů i odkazy na proměnné prostředí s prefixem `env.`:

```bash
export OPENAI_API_KEY="your-openai-api-key"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export MISTRAL_API_KEY="your-mistral-api-key"
export CEREBRAS_API_KEY="your-cerebras-api-key"
export DEEPSEEK_API_KEY="your-deepseek-api-key"
export GROQ_API_KEY="your-groq-api-key"
export COHERE_API_KEY="your-cohere-api-key"
```

**Práce s proměnnými prostředí:**

- `"value": "env.VARIABLE_NAME"` odkazuje na proměnnou prostředí.
- `"value": "sk-proj-xxxxxxxxx"` předá klíč přímo.
- Všechny citlivé údaje se v odpovědích na požadavky GET a v rozhraní automaticky skrývají.

## Pokročilá konfigurace

### Vážené vyvažování zátěže

Rozdělujte požadavky mezi více API klíčů nebo poskytovatelů podle vlastních vah. Příklad rozděluje provoz mezi dva klíče OpenAI v poměru 70/30, což se hodí pro řízení limitů požadavků nebo nákladů napříč různými účty.

**Přes webové rozhraní**

![Rozhraní pro vážené vyvažování zátěže](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-multi-key-for-models.png)

1. Přejděte na **"Model Providers"** → **"Configurations"** → **"OpenAI"**.
2. Kliknutím na **"Add Key"** přidejte více klíčů.
3. Nastavte váhy (0.7 a 0.3).
4. Uložte konfiguraci.

**Přes API**

```bash
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY_1",
            "models": ["*"],
            "weight": 0.7
        },
        {
            "name": "openai-key-2",
            "value": "env.OPENAI_API_KEY_2",
            "models": ["*"],
            "weight": 0.3
        }
    ]
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY_1",
          "models": ["*"],
          "weight": 0.7
        },
        {
          "name": "openai-key-2",
          "value": "env.OPENAI_API_KEY_2",
          "models": ["*"],
          "weight": 0.3
        }
      ]
    }
  }
}
```

### Klíče pro konkrétní modely

Pro konkrétní modely používejte různé API klíče a řiďte tak přístupy i účtování zvlášť. Příklad používá prémiový klíč pro pokročilé reasoning modely (o1-preview, o1-mini) a standardní klíč pro běžné modely GPT.

**Přes webové rozhraní**

![Rozhraní pro klíče ke konkrétním modelům](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-multi-key-for-models.png)

1. Přejděte na **"Model Providers"** → **"Configurations"** → **"OpenAI"**.
2. Přidejte první klíč s modely `["gpt-4o", "gpt-4o-mini"]`.
3. Přidejte prémiový klíč s modely `["o1-preview", "o1-mini"]`.
4. Uložte konfiguraci.

**Přes API**

```bash
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["gpt-4o", "gpt-4o-mini"],
            "weight": 1.0
        },
        {
            "name": "openai-key-2",
            "value": "env.OPENAI_API_KEY_PREMIUM",
            "models": ["o1-preview", "o1-mini"],
            "weight": 1.0
        }
    ]
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["gpt-4o", "gpt-4o-mini"],
          "weight": 1.0
        },
        {
          "name": "openai-key-2",
          "value": "env.OPENAI_API_KEY_PREMIUM",
          "models": ["o1-preview", "o1-mini"],
          "weight": 1.0
        }
      ]
    }
  }
}
```

### Vlastní základní URL

Přepište výchozí endpoint API poskytovatele. Hodí se pro připojení k vlastním modelům, lokálním vývojovým serverům nebo API kompatibilním s OpenAI, jako jsou vLLM, Ollama nebo LiteLLM.

**Přes webové rozhraní**

[▶ Video: nastavení vlastní základní URL](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/custom-base-url.mp4)

1. Přejděte na **"Model Providers"** → **"Configurations"** → **"OpenAI"** → **"Provider level configuration"** → **"Network config"**.
2. Nastavte **Base URL**: `http://localhost:8000`.
3. Uložte konfiguraci.

**Přes API**

```bash
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "network_config": {
        "base_url": "http://localhost:8000"
    }
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "network_config": {
        "base_url": "http://localhost:8000"
      }
    }
  }
}
```

> [!NOTE]
> U vlastních poskytovatelů jako Ollama a SGL je `base_url` povinné. U standardních poskytovatelů je nepovinné a přepisuje výchozí endpoint.

### Přístup do privátní sítě

Pokud váš poskytovatel běží na privátní IP adrese (například lokální instance vLLM nebo Ollama na `192.168.x.x` nebo `10.x.x.x`), nastavte v `network_config` `allow_private_network: true`. Adresy link-local (`169.254.x.x`) zůstávají blokované bez ohledu na toto nastavení.

**Přes API**

```bash
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "ollama-local",
    "keys": [
        {
            "name": "ollama-key",
            "value": "dummy",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "network_config": {
        "base_url": "http://192.168.1.195:8000",
        "allow_private_network": true
    }
}'
```

**Přes config.json**

```json
{
  "providers": {
    "ollama-local": {
      "keys": [
        {
          "name": "ollama-key",
          "value": "dummy",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "network_config": {
        "base_url": "http://192.168.1.195:8000",
        "allow_private_network": true
      }
    }
  }
}
```

### Správa opakování

Nastavte chování při opakování pro dočasná selhání a limity požadavků. Příklad nastavuje exponenciální backoff s až 5 opakováními, začíná na 1 ms a končí na 10 sekundách. Ideální pro přechodné potíže se sítí.

> [!NOTE]
> Úplné vysvětlení, jak opakování fungují, jak se při limitech požadavků střídají klíče a jak opakování navazují na fallbacky, najdete v části [Retries & Fallbacks](https://docs.getbifrost.ai/features/retries-and-fallbacks).

**Přes webové rozhraní**

![Rozhraní pro nastavení opakování](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-concurrency-timeout.png)

1. Přejděte na **"Model Providers"** → **"Configurations"** → **"OpenAI"** → **"Provider level configuration"** → **"Network config"**.
2. Nastavte **Max Retries**: `5`.
3. Nastavte **Initial Backoff**: `1` ms.
4. Nastavte **Max Backoff**: `10000` ms.
5. Uložte konfiguraci.

**Přes API**

```bash
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "network_config": {
        "max_retries": 5,
        "retry_backoff_initial": 1,
        "retry_backoff_max": 10000
    }
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "network_config": {
        "max_retries": 5,
        "retry_backoff_initial": 1,
        "retry_backoff_max": 10000
      }
    }
  }
}
```

### Vlastní souběžnost a velikost fronty

Vyladěním počtu workerů a velikosti fronty pro jednotlivé poskytovatele doladíte výkon (výchozí hodnoty jsou 1000 workerů a fronta 5000). Příklad dává OpenAI vyšší limity (100 workerů, fronta 500) pro vysokou propustnost, zatímco Anthropic dostane konzervativní limity, aby respektoval jeho limity požadavků.

**Přes webové rozhraní**

![Rozhraní pro nastavení souběžnosti](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-concurrency-buffer-size.png)

1. Přejděte na **"Model Providers"** → **"Configurations"** → **{Provider}** → **"Provider level configuration"** → **"Performance tuning"**.
2. Nastavte **Concurrency**: počet workerů (100 pro OpenAI, 25 pro Anthropic).
3. Nastavte **Buffer Size**: velikost fronty (500 pro OpenAI, 100 pro Anthropic).
4. Uložte konfiguraci.

**Přes API**

```bash
# OpenAI with high throughput settings
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "concurrency_and_buffer_size": {
        "concurrency": 100,
        "buffer_size": 500
    }
}'

# Anthropic with conservative settings
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "anthropic",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.ANTHROPIC_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "concurrency_and_buffer_size": {
        "concurrency": 25,
        "buffer_size": 100
    }
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "concurrency_and_buffer_size": {
        "concurrency": 100,
        "buffer_size": 500
      }
    },
    "anthropic": {
      "keys": [
        {
          "name": "anthropic-key-1",
          "value": "env.ANTHROPIC_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "concurrency_and_buffer_size": {
        "concurrency": 25,
        "buffer_size": 100
      }
    }
  }
}
```

### Vlastní hlavičky

Bifrost umí přidávat vlastní hlavičky do požadavků na poskytovatele dvěma způsoby: **statické hlavičky** nastavené na úrovni poskytovatele a **dynamické hlavičky** předávané s jednotlivými požadavky.

#### Statické hlavičky (na úrovni poskytovatele)

Nastavte hlavičky, které se automaticky přidají do každého požadavku na konkrétního poskytovatele. Hodí se pro požadavky konkrétního poskytovatele, verzování API nebo metadata organizace.

**Přes webové rozhraní**

![Rozhraní pro nastavení dalších hlaviček](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-extra-headers.png)

1. Přejděte na **"Model Providers"** → **"Configurations"** → **"OpenAI"** → **"Provider level configuration"** → **"Network config"**.
2. Přidejte hlavičky v části **"Extra Headers"**.
3. Uložte konfiguraci.

**Přes API**

```bash
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "network_config": {
        "extra_headers": {
            "x-custom-org": "my-organization",
            "x-environment": "production",
            "x-client-secret": "env.UPSTREAM_CLIENT_SECRET"
        }
    }
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "network_config": {
        "extra_headers": {
          "x-custom-org": "my-organization",
          "x-environment": "production",
          "x-client-secret": "env.UPSTREAM_CLIENT_SECRET"
        }
      }
    }
  }
}
```

#### Dynamické hlavičky (pro jednotlivé požadavky)

S jednotlivými požadavky posílejte vlastní hlavičky s prefixem `x-bf-eh-*`. Po odstranění prefixu se hlavičky automaticky předají poskytovateli. Hodí se pro metadata konkrétního požadavku, identifikaci uživatele nebo vlastní sledování.

```bash
curl --location 'http://localhost:8080/v1/chat/completions' \
--header 'Content-Type: application/json' \
--header 'x-bf-eh-user-id: user-123' \
--header 'x-bf-eh-tracking-id: trace-456' \
--data '{
    "model": "openai/gpt-4o-mini",
    "messages": [
        {"role": "user", "content": "Hello!"}
    ]
}'
```

Prefix `x-bf-eh-` se před přeposláním odstraní, takže z `x-bf-eh-user-id` se v požadavku na poskytovatele stane `user-id`.

**Příklady použití:**

- Identifikace uživatele: `x-bf-eh-user-id`, `x-bf-eh-tenant-id`
- Sledování požadavků: `x-bf-eh-correlation-id`, `x-bf-eh-trace-id`
- Vlastní metadata: `x-bf-eh-department`, `x-bf-eh-cost-center`
- A/B testování: `x-bf-eh-experiment-id`, `x-bf-eh-variant`

#### Bezpečnostní seznam zakázaných hlaviček

Bifrost udržuje bezpečnostní seznam hlaviček, které se poskytovatelům nikdy nepředávají, bez ohledu na konfiguraci:

```go
denylist := map[string]bool{
    "proxy-authorization": true,
    "cookie":              true,
    "host":                true,
    "content-length":      true,
    "connection":          true,
    "transfer-encoding":   true,

    // prevent auth/key overrides via x-bf-eh-*
    "x-api-key":      true,
    "x-goog-api-key": true,
    "x-bf-api-key":   true,
    "x-bf-vk":        true,
}
```

Seznam se uplatňuje na statické i dynamické hlavičky, aby nevznikly bezpečnostní zranitelnosti.

#### Přepsání beta hlaviček

Pro poskytovatele z rodiny Anthropic (Anthropic, Vertex, Bedrock, Azure) udržuje Bifrost výchozí tabulku podpory hlaviček `anthropic-beta`. Tyto výchozí hodnoty můžete pro jednotlivé poskytovatele přepsat, když se podpora u poskytovatele změní dřív, než Bifrost aktualizuje své výchozí hodnoty.

TypeSafe, vlastní poskytovatelé postavení na něm a Databricks mají jednu položku, `typesafe-danger-zone-images`: hlavičku `TypeSafe-Danger-Zone-Images`, kterou Bifrost přidává k požadavkům na obrázky a box a která je ve výchozím stavu podporovaná. Viz [TypeSafe: Images and Box](https://docs.getbifrost.ai/providers/supported-providers/typesafe#images-and-box) a [Databricks: System One models](https://docs.getbifrost.ai/providers/supported-providers/databricks#system-one-models).

**Přes webové rozhraní**

1. Přejděte na **"Model Providers"** → **"Configurations"** → vyberte poskytovatele → záložka **"Beta Headers"**.
2. U každé známé beta hlavičky je vidět její výchozí stav podpory.
3. V rozbalovacím seznamu pro přepsání nastavte **Enabled**, **Disabled** nebo **Default** (použít vestavěné výchozí hodnoty).
4. Uložte konfiguraci.

**Přes API**

```bash
curl --location --request PUT 'http://localhost:8080/api/providers/vertex' \
--header 'Content-Type: application/json' \
--data '{
    "network_config": {
        "beta_header_overrides": {
            "redact-thinking-": true,
            "fast-mode-": false
        }
    }
}'
```

**Přes config.json**

```json
{
  "providers": {
    "vertex": {
      "network_config": {
        "beta_header_overrides": {
          "redact-thinking-": true,
          "fast-mode-": false
        }
      }
    }
  }
}
```

Klíče pro přepsání jsou **prefixy** beta hlaviček (například `"redact-thinking-"`), ne úplné hodnoty hlaviček. Přepsání tak platí bez ohledu na změny verzí hlaviček podle data. Úplnou tabulku podpory najdete v [dokumentaci poskytovatele Anthropic](https://docs.getbifrost.ai/providers/supported-providers/anthropic#beta-headers).

### Nastavení proxy

Směrujte požadavky přes proxy kvůli compliance, bezpečnosti nebo geografickým požadavkům. Příklad ukazuje HTTP proxy pro OpenAI a SOCKS5 proxy s ověřováním pro Anthropic, což se hodí ve firemních prostředích nebo pro regionální přístup.

**Přes webové rozhraní**

![Rozhraní pro nastavení proxy](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-proxy-setup.png)

1. Přejděte na **"Model Providers"** → **"Configurations"** → **{Provider}** → **"Provider level configuration"** → **"Proxy config"**.
2. Vyberte **Proxy Type**: HTTP nebo SOCKS5.
3. Nastavte **Proxy URL**: `http://localhost:8000`.
4. V případě potřeby přidejte přihlašovací údaje (uživatelské jméno a heslo).
5. Uložte konfiguraci.

**Přes API**

```bash
# HTTP proxy for OpenAI
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "proxy_config": {
        "type": "http",
        "url": "http://localhost:8000"
    }
}'

# SOCKS5 proxy with authentication for Anthropic
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "anthropic",
    "keys": [
        {
            "name": "anthropic-key-1",
            "value": "env.ANTHROPIC_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "proxy_config": {
        "type": "socks5",
        "url": "http://localhost:8000",
        "username": "user",
        "password": "password"
    }
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "proxy_config": {
        "type": "http",
        "url": "http://localhost:8000"
      }
    },
    "anthropic": {
      "keys": [
        {
          "name": "anthropic-key-1",
          "value": "env.ANTHROPIC_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "proxy_config": {
        "type": "socks5",
        "url": "http://localhost:8000",
        "username": "user",
        "password": "password"
      }
    }
  }
}
```

### Vracení původní odpovědi

Spolu se standardizovanou odpovědí Bifrostu vrací i původní odpověď poskytovatele. Hodí se pro ladění a přístup k metadatům konkrétního poskytovatele.

> [!NOTE]
> Pro jednotlivý požadavek to můžete přepsat hlavičkou `x-bf-send-back-raw-response` (`"true"` nebo `"false"`), bez ohledu na konfiguraci poskytovatele. Podrobnosti najdete v části [Request Options](https://docs.getbifrost.ai/providers/request-options#send-back-raw-response).

**Přes webové rozhraní**

![Rozhraní pro nastavení původní odpovědi](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-raw-response.png)

1. Přejděte na **"Model Providers"** → **"Configurations"** → **{Provider}** → **"Provider level configuration"** → **"Performance tuning"**.
2. Zapněte **"Include Raw Response"**.
3. Uložte konfiguraci.

**Přes API**

```bash
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "send_back_raw_response": true
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "send_back_raw_response": true
    }
  }
}
```

Po zapnutí se původní odpověď poskytovatele objeví v `extra_fields.raw_response`:

```json
{
    "choices": [...],
    "usage": {...},
    "extra_fields": {
        "provider": "openai",
        "raw_response": {
            // Original OpenAI response here
        }
    }
}
```

### Vracení původního požadavku

Spolu s odpovědí Bifrostu vrací i původní požadavek odeslaný poskytovateli. Hodí se pro ladění transformací požadavků a ověření, co se poskytovateli skutečně poslalo.

> [!NOTE]
> Pro jednotlivý požadavek to můžete přepsat hlavičkou `x-bf-send-back-raw-request` (`"true"` nebo `"false"`), bez ohledu na konfiguraci poskytovatele. Podrobnosti najdete v části [Request Options](https://docs.getbifrost.ai/providers/request-options#send-back-raw-request).

**Přes webové rozhraní**

![Rozhraní pro nastavení původního požadavku](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-raw-request.png)

1. Přejděte na **"Model Providers"** → **"Configurations"** → **{Provider}** → **"Provider level configuration"** → **"Performance tuning"**.
2. Zapněte **"Include Raw Request"**.
3. Uložte konfiguraci.

**Přes API**

```bash
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "send_back_raw_request": true
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "send_back_raw_request": true
    }
  }
}
```

Po zapnutí se původní požadavek na poskytovatele objeví v `extra_fields.raw_request`:

```json
{
    "choices": [...],
    "usage": {...},
    "extra_fields": {
        "provider": "openai",
        "raw_request": {
            // Original request sent to OpenAI here
        }
    }
}
```

> [!TIP]
> `send_back_raw_request` a `send_back_raw_response` můžete zapnout zároveň a při ladění uvidíte celý cyklus požadavku a odpovědi.

### Ukládání původního požadavku a odpovědi

Ukládá původní požadavek a odpověď poskytovatele do záznamu v logu. Je to nezávislé na `send_back_raw_request` a `send_back_raw_response`: zapnutí neovlivní, zda se původní data objeví v odpovědi API, a zapnutí vracení automaticky neukládá původní data do logů. Pokud chcete obojí, zapněte obojí.

**Přes webové rozhraní**

1. Přejděte na **"Model Providers"** → **"Configurations"** → **{Provider}** → **"Provider level configuration"** → **"Performance tuning"**.
2. Zapněte **"Store Raw Request/Response"**.
3. Uložte konfiguraci.

**Přes API**

```bash
curl --location 'http://localhost:8080/api/providers' \
--header 'Content-Type: application/json' \
--data '{
    "provider": "openai",
    "keys": [
        {
            "name": "openai-key-1",
            "value": "env.OPENAI_API_KEY",
            "models": ["*"],
            "weight": 1.0
        }
    ],
    "store_raw_request_response": true
}'
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "store_raw_request_response": true
    }
  }
}
```

> [!NOTE]
> `store_raw_request_response` má účinek, jen když je aktivní plugin pro logování: původní data do záznamu v logu zapisuje právě on. Bez něj se při zapnutí data zachytí, ale nic je neuloží.
>
> Pro jednotlivý požadavek to můžete přepsat hlavičkou `x-bf-store-raw-request-response` (`"true"` nebo `"false"`), bez ohledu na konfiguraci poskytovatele. Podrobnosti najdete v části [Request Options](https://docs.getbifrost.ai/providers/request-options#store-raw-requestresponse).

### Ignorování ceny hlášené poskytovatelem

Někteří poskytovatelé kompatibilní s OpenAI vracejí v odpovědi vlastní `usage.cost`. Když je tato hodnota vyšší než nula, Bifrost ji použije jako cenu požadavku a vlastní ocenění přeskočí, takže přepsání cen na tohoto poskytovatele nemají vliv. Hlášená cena nula propadne na ceny z katalogu a přepsání. Hodnota nemusí být vždy v USD. Například Cortecs hlásí kredity, kde 1 EUR je 1 000 000 kreditů, takže požadavek, který stál 0,001067 EUR, se v logech a rozpočtech objeví jako 1067 USD.

Zapněte `ignore_provider_cost` a Bifrost tuto hodnotu zahodí. Požadavek pak ocení podle cen modelu a případných [přepsání cen](https://docs.getbifrost.ai/providers/custom-pricing) a tuto cenu použijí logy, rozpočty i telemetrie. Odpověď vracená vašemu klientovi nese `usage.cost` od poskytovatele beze změny. Pokud model nemá cenu v katalogu ani přepsání, samotné volání modelu žádnou cenu nedostane. Požadavek přesto může zalogovat cenu a započítat se do rozpočtů, pokud obsahuje samostatně oceňovaná volání soudce guardrails nebo klasifikace pro směrování.

Akce **Recalculate costs** v LLM Logs toto nastavení také respektuje, takže můžete znovu ocenit požadavky zalogované před jeho zapnutím.

**Přes webové rozhraní**

1. Přejděte na **"Model Providers"** → **"Configurations"** → **{Provider}** → **"Provider level configuration"** → **"Pricing"**.
2. Zapněte **"Ignore Provider-Reported Cost"**.
3. Uložte konfiguraci.

**Přes API**

`PUT /api/providers/{provider}` nahradí nastavení sítě a souběžnosti poskytovatele tím, co pošlete. Proto načtěte aktuální konfiguraci, nastavte příznak a pošlete ji zpět.

```bash
curl -s 'http://localhost:8080/api/providers/openai' \
  | jq 'del(.name, .keys, .provider_status, .status, .description, .config_hash) | .ignore_provider_cost = true' \
  | curl --location --request PUT 'http://localhost:8080/api/providers/openai' \
      --header 'Content-Type: application/json' \
      --data @-
```

**Přes config.json**

```json
{
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["*"],
          "weight": 1.0
        }
      ],
      "ignore_provider_cost": true
    }
  }
}
```

### Předávání dalších parametrů

Zapne režim předávání dalších parametrů. Po zapnutí se všechny parametry v poli `extra_params` (nebo v polích pro další parametry konkrétního poskytovatele) sloučí přímo do požadavku odesílaného poskytovateli a obejdou filtrování parametrů v Bifrostu.

```bash
curl --location 'http://localhost:8080/v1/chat/completions' \
--header 'Content-Type: application/json' \
--header 'x-bf-passthrough-extra-params: true' \
--data '{
    "model": "openai/gpt-4o-mini",
    "messages": [
        {"role": "user", "content": "Hello!"}
    ],
    "extra_params": {
        "custom_param": "value",
        "another_param": 123,
        "nested_param": {
            "nested_key": "nested_value"
        }
    }
}'
```

Po zapnutí se další parametry sloučí do těla JSON požadavku odesílaného poskytovateli. Můžete tak předávat parametry konkrétních poskytovatelů, které Bifrost nativně nepodporuje.

> [!NOTE]
> - Funguje jen pro požadavky JSON, ne pro požadavky multipart/form-data.
> - Parametry, které Bifrost už zpracovává (například `addWatermark`, `enhancePrompt`), se neduplikují, objeví se na svém správném místě.
> - Vnořené parametry (například `parameters.custom_field`) se rekurzivně sloučí se stávajícími vnořenými strukturami.
> - Úplný seznam hlaviček Bifrostu najdete v části [Supported Headers](https://docs.getbifrost.ai/providers/supported-headers).

## Další kroky

Teď, když rozumíte konfiguraci poskytovatelů, pokračujte souvisejícími tématy:

### Základní témata

- **[Streaming Responses](./streaming.md)**: generování odpovědí v reálném čase.
- **[Tool Calling](./tool-calling.md)**: umožněte AI používat externí funkce.
- **[Multimodal AI](./multimodal.md)**: zpracování obrázků, zvuku a textu.
- **[Integrace](./integrations.md)**: přímá kompatibilita se stávajícími SDK.

### Pokročilá témata

- **[Core Features](https://docs.getbifrost.ai/features/)**: pokročilé funkce Bifrostu.
- **[Architecture](https://docs.getbifrost.ai/architecture/)**: jak Bifrost funguje uvnitř.
- **[Deployment](https://docs.getbifrost.ai/deployment-guides)**: produkční nasazení a škálování.

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/quickstart/gateway/provider-configuration.mdx`, commit `7235fb5`. Změny: překlad do češtiny, převod z MDX do Markdownu (záložky jako podnadpisy). Originál: https://docs.getbifrost.ai/quickstart/gateway/provider-configuration</sub>
