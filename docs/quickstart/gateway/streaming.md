<!--
zdroj: maximhq/bifrost docs/quickstart/gateway/streaming.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
prelozeno: 2026-11-14
-->

# Streamování odpovědí

*Odpovědi AI přijímáte v reálném čase přes Server-Sent Events. Hodí se pro chatovací aplikace, zpracování zvuku a přepis v reálném čase, kde chcete výsledky okamžitě.*

## Streamování textového doplňování

Požádejte o textové doplňování se zapnutým streamováním a průběžně dostávejte části `text`, jak vznikají.

```bash
curl --location 'http://localhost:8080/v1/completions' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o-mini",
    "prompt": "Write a short haiku about the ocean",
    "stream": true
}'
```

**Formát odpovědi (Server-Sent Events):**

```
data: {"choices":[{"text":"Waves whisper soft"}],"model":"gpt-4o-mini"}

data: {"choices":[{"text":" on distant shores, the moon calls"}],"model":"gpt-4o-mini"}

data: {"choices":[{"text":" tides to rise."}],"model":"gpt-4o-mini"}

data: [DONE]
```

## Streamování odpovědí v chatu

Odpovědi AI dostáváte v reálném čase, jak vznikají. Ideální pro chatovací aplikace, kde chcete odpověď zobrazovat průběžně, jako by se psala, a zlepšit tak uživatelský zážitek.

```bash
curl --location 'http://localhost:8080/v1/chat/completions' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o-mini",
    "messages": [
        {"role": "user", "content": "Tell me a story about a robot learning to paint"}
    ],
    "stream": true
}'
```

**Formát odpovědi (Server-Sent Events):**

```
data: {"choices":[{"delta":{"content":"Once"}}],"model":"gpt-4o-mini"}

data: {"choices":[{"delta":{"content":" upon"}}],"model":"gpt-4o-mini"}

data: {"choices":[{"delta":{"content":" a"}}],"model":"gpt-4o-mini"}

data: [DONE]
```

Každý chunk obsahuje část obsahu, kterou můžete připojovat a skládat z ní celou odpověď v reálném čase.

> [!NOTE]
> I pro streamované požadavky platí výchozí timeout nastavený v konfiguraci poskytovatele, který je ve výchozím stavu **30 sekund**.

> [!NOTE]
> Bifrost sjednocuje všechny streamované odpovědi tak, že čerpání (usage) a důvod ukončení (finish reason) posílá jen v posledním chunku a obsah v předchozích.

## Streamování přes Responses API

Streamování Responses API ve stylu OpenAI se SSE založeným na událostech. Obsahuje řádky `event:` a nepoužívá značku `[DONE]`. Stream končí uzavřením spojení.

```bash
curl --location 'http://localhost:8080/v1/responses' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o-mini",
    "input": "Tell me one interesting fact about Mars",
    "stream": true
}'
```

**Formát odpovědi (Server-Sent Events):**

```
event: response.created
data: {"type":"response.created"}

event: response.output_text.delta
data: {"type":"response.output_text.delta","delta": /* partial text delta payload */ }

event: response.output_text.delta
data: {"type":"response.output_text.delta","delta": * more text delta */ }

event: response.completed
data: {"type":"response.completed","response":{ /* usage, finish_reason, etc. */ }}
```

## Streamování převodu textu na řeč: generování zvuku v reálném čase

Generování zvuku streamujte v reálném čase, jak se text převádí na řeč. Ideální pro dlouhé texty nebo když potřebujete zvuk přehrávat okamžitě.

```bash
curl --location 'http://localhost:8080/v1/audio/speech' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o-mini-tts",
    "input": "Hello this is a sample test, respond with hello for my Bifrost",
    "voice": "alloy",
    "stream_format": "sse"
}'
```

**Odpověď:** části zvuku se doručují přes Server-Sent Events. Každý chunk obsahuje zvuková data v base64, která můžete průběžně dekódovat a přehrávat nebo ukládat.

```
data: {"audio":"UklGRigAAABXQVZFZm10IBAAAAABAAEA..."}

data: {"audio":"AKlFQVZFZm10IBAAAAABAAEAq..."}

data: [DONE]
```

**Uložení streamu:** přidejte `> audio_stream.txt` a výstup se přesměruje do souboru.

## Streamování převodu řeči na text: přepis zvuku v reálném čase

Výsledky přepisu zvuku streamujte, jak se zpracovávají. Text dostanete okamžitě, což se hodí pro aplikace v reálném čase i pro dlouhé zvukové soubory.

```bash
curl --location 'http://localhost:8080/v1/audio/transcriptions' \
--form 'file=@"/path/to/your/audio.mp3"' \
--form 'model="openai/gpt-4o-transcribe"' \
--form 'stream="true"' \
--form 'response_format="json"'
```

**Formát odpovědi:**

```
data: {"text":"Hello"}

data: {"text":" this"}

data: {"text":" is"}

data: {"text":" a sample"}

data: [DONE]
```

**Další možnosti:** pro vyšší přesnost přidejte `--form 'language="en"'` nebo `--form 'prompt="context hint"'`.

## Podporované formáty zvuku

**Syntéza řeči:** podporuje `"response_format": "mp3"` (výchozí) a `"response_format": "wav"`.

**Vstup pro přepis:** přijímá MP3, WAV, M4A a další běžné formáty zvuku.

> [!NOTE]
> Možnosti streamování se liší podle poskytovatele a modelu. Konkrétní podporu streamování a omezení najdete v dokumentaci jednotlivých poskytovatelů.

## Další kroky

Když už rozumíte streamování odpovědí, podívejte se na související témata:

### Základní témata

- **[Volání nástrojů](./tool-calling.md)**: modely AI mohou používat externí nástroje a funkce.
- **[Multimodální AI](./multimodal.md)**: zpracování obrázků, zvuku a multimediálního obsahu.
- **[Konfigurace poskytovatelů](./provider-configuration.md)**: více poskytovatelů pro redundanci.
- **[Integrace](./integrations.md)**: přímá kompatibilita se stávajícími SDK.

### Pokročilá témata

- **[Core Features](https://docs.getbifrost.ai/features/)**: pokročilé schopnosti Bifrostu.
- **[Architecture](https://docs.getbifrost.ai/architecture/)**: jak Bifrost funguje uvnitř.
- **[Deployment](https://docs.getbifrost.ai/deployment-guides)**: produkční nasazení a škálování.

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/quickstart/gateway/streaming.mdx`, commit `0fb9342`. Změny: překlad do češtiny, převod z MDX do Markdownu. Originál: https://docs.getbifrost.ai/quickstart/gateway/streaming</sub>
