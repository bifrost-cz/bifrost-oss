<!--
zdroj: maximhq/bifrost docs/quickstart/gateway/multimodal.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
prelozeno: 2026-11-14
-->

# Multimodální AI

*Zpracování různých typů obsahu, včetně obrázků, zvuku a textu, pomocí modelů AI. Bifrost podporuje analýzu obrazu, generování obrázků, syntézu řeči a přepis zvuku u různých poskytovatelů.*

## Vision: analýza obrázků pomocí AI

Posílejte obrázky modelům s podporou vision k analýze, popisu a porozumění. Tento příklad ukazuje analýzu obrázku z URL pomocí GPT-4o se zpracováním ve vysokém detailu pro vyšší přesnost.

```bash
curl --location 'http://localhost:8080/v1/chat/completions' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o",
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "What do you see in this image? Please describe it in detail."
                },
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://pub-cdead89c2f004d8f963fd34010c479d0.r2.dev/Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
                        "detail": "high"
                    }
                }
            ]
        }
    ]
}'
```

**Odpověď obsahuje podrobnou analýzu obrázku:**

```json
{
    "choices": [{
        "message": {
            "role": "assistant",
            "content": "I can see a beautiful wooden boardwalk extending through a natural landscape..."
        }
    }]
}
```

## Generování obrázků pomocí AI

Generujte obrázky z textových promptů pomocí modelů pro generování obrázků kompatibilních s OpenAI.

### Základní generování obrázku

Vygenerování obrázku z textového promptu pomocí `dall-e-3`.

```bash
curl --location 'http://localhost:8080/v1/images/generations' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/dall-e-3",
    "prompt": "A futuristic city skyline at sunset with flying cars",
    "size": "1024x1024",
    "response_format": "url"
}'
```

**Formát odpovědi:**

```json
{
    "created": 1713833628,
    "data": [
        {
            "url": "https://oaidalleapiprodscus.blob.core.windows.net/...",
            "revised_prompt": "A futuristic city skyline at sunset featuring advanced architecture and flying vehicles.",
            "index": 0
        }
    ],
    "background": "opaque",
    "output_format": "png",
    "quality": "standard",
    "size": "1024x1024",
    "usage": {
        "input_tokens": 15,
        "output_tokens": 1,
        "total_tokens": 16
    },
    "extra_fields": {
        "request_type": "image_generation",
        "provider": "openai",
        "model_requested": "dall-e-3",
        "latency": 15265,
        "chunk_index": 0
    }
}
```

## Porozumění zvuku: analýza zvuku pomocí AI

Pokud vaše chatovací aplikace podporuje textový vstup, můžete přidat i zvukový vstup a výstup: stačí zahrnout audio do pole modalities a použít zvukový model, například gpt-4o-audio-preview.

### Zvukový vstup do modelu

```bash
curl --location 'http://localhost:8080/v1/chat/completions' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o-audio-preview",
    "modalities": ["text"],
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "Please analyze this audio recording and summarize what was discussed."
                },
                {
                    "type": "input_audio",
                    "input_audio": {
                        "data": "<base64-encoded audio data containing the word 'Affirmative'>",
                        "format": "wav"
                    }
                }
            ]
        }
    ]
}'
```

### Výstup modelu ke zvuku

```bash
{
    "choices": [
        {
            "index": 0,
            "finish_reason": "stop",
            "message": {
                "role": "assistant",
                "content": "The audio recording captured a brief segment where a speaker simply said \"Affirmative\" in response. There wasn't any detailed discussion or context provided beyond that one-word affirmation. If you have more audio or specific questions, feel free to share!"
            }
        }
    ]
}
```

## Převod textu na řeč

Převádějte text na přirozeně znějící řeč pomocí hlasových modelů AI. Tento příklad ukazuje vygenerování zvukového souboru MP3 z textu hlasem „alloy“. Výsledek se vrací jako binární zvuková data.

```bash
curl --location 'http://localhost:8080/v1/audio/speech' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/tts-1",
    "input": "Hello! This is a sample text that will be converted to speech using Bifrost speech synthesis capabilities. The weather today is wonderful, and I hope you are having a great day!",
    "voice": "alloy",
    "response_format": "mp3"
}' \
--output "output.mp3"
```

**Uložení zvuku do souboru:**

```bash
# The --output flag saves the binary audio data directly to a file
# File size will vary based on input text length
```

## Převod řeči na text: přepis zvukových souborů

Převádějte zvukové soubory na text pomocí modelů AI pro přepis. Tento příklad ukazuje přepis souboru MP3 modelem Whisper od OpenAI s volitelným kontextovým promptem pro vyšší přesnost.

```bash
curl --location 'http://localhost:8080/v1/audio/transcriptions' \
--form 'file=@"output.mp3"' \
--form 'model="openai/whisper-1"' \
--form 'prompt="This is a sample audio transcription from Bifrost speech synthesis."'
```

**Formát odpovědi:**

```json
{
    "text": "Hello! This is a sample text that will be converted to speech using Bifrost speech synthesis capabilities. The weather today is wonderful, and I hope you are having a great day!"
}
```

## Pokročilé příklady vision

### Více obrázků

Pošlete více obrázků v jednom požadavku k porovnání nebo analýze. Hodí se to pro porovnávání produktů, analýzu změn v čase nebo pochopení vztahů mezi různými vizuálními prvky.

```bash
curl --location 'http://localhost:8080/v1/chat/completions' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o",
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "Compare these two images. What are the differences?"
                },
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://example.com/image1.jpg"
                    }
                },
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://example.com/image2.jpg"
                    }
                }
            ]
        }
    ]
}'
```

### Obrázky v base64

Lokální obrázky zpracujete tak, že je zakódujete jako data URL v base64. Tento postup je ideální, když potřebujete analyzovat obrázky uložené lokálně ve vašem systému, aniž byste je nejdřív nahrávali na externí URL.

```bash
# First, encode your local image to base64
base64_image=$(base64 -i local_image.jpg)
data_url="data:image/jpeg;base64,$base64_image"

curl --location 'http://localhost:8080/v1/chat/completions' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o",
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "Analyze this image and describe what you see."
                },
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "'$data_url'",
                        "detail": "high"
                    }
                }
            ]
        }
    ]
}'
```

## Možnosti nastavení zvuku

### Volba hlasu pro syntézu řeči

OpenAI nabízí šest různých hlasů, každý s jinými vlastnostmi:

- `alloy`: vyvážený, přirozený hlas
- `echo`: hluboký, zvučný hlas
- `fable`: výrazný, vypravěčský hlas
- `onyx`: silný, sebevědomý hlas
- `nova`: jasný, energický hlas
- `shimmer`: jemný, uklidňující hlas

```bash
# Example with different voice
curl --location 'http://localhost:8080/v1/audio/speech' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/tts-1",
    "input": "This is the nova voice speaking.",
    "voice": "nova",
    "response_format": "mp3"
}' \
--output "sample_nova.mp3"
```

### Formáty zvuku

Generujte zvuk v různých formátech podle použití: MP3 pro běžné použití, Opus pro streamování na webu, AAC pro mobilní aplikace a FLAC pro aplikace s vysokou kvalitou zvuku.

```bash
# MP3 format (default)
"response_format": "mp3"

# Opus format for web streaming
"response_format": "opus"

# AAC format for mobile apps
"response_format": "aac"

# FLAC format for high-quality audio
"response_format": "flac"
```

## Možnosti přepisu

### Určení jazyka

Přesnost přepisu zvýšíte určením zdrojového jazyka. Hodí se to hlavně u neanglického zvuku nebo když zvuk obsahuje odborné termíny či slovník konkrétního oboru.

```bash
curl --location 'http://localhost:8080/v1/audio/transcriptions' \
--form 'file=@"spanish_audio.mp3"' \
--form 'model="openai/whisper-1"' \
--form 'language="es"' \
--form 'prompt="This is a Spanish audio recording about technology."'
```

### Formáty odpovědi

Zvolte mezi prostým textovým výstupem a podrobnou odpovědí JSON s časovými značkami. Formát verbose JSON poskytuje časování na úrovni slov a segmentů, což se hodí pro tvorbu titulků nebo analýzu řečových vzorců.

```bash
# Text only response
curl --location 'http://localhost:8080/v1/audio/transcriptions' \
--form 'file=@"audio.mp3"' \
--form 'model="openai/whisper-1"' \
--form 'response_format="text"'

# JSON with timestamps
curl --location 'http://localhost:8080/v1/audio/transcriptions' \
--form 'file=@"audio.mp3"' \
--form 'model="openai/whisper-1"' \
--form 'response_format="verbose_json"' \
--form 'timestamp_granularities[]=word' \
--form 'timestamp_granularities[]=segment'
```

## Multimodální embeddingy

Bifrost podporuje multimodální embeddingy, tedy generování vektorových reprezentací, které zachycují text, obrázky, zvuk, video a dokumenty dohromady. `POST /v1/embeddings` přijímá všechny dosavadní tvary (řetězec, pole řetězců, pole tokenů) a navíc multimodální tvar, ve kterém je každá položka v `input` polem typovaných částí. Všechny části jedné položky vytvoří **jeden** vektor.

```json
{
  "model": "cohere/embed-v4.0",
  "input": [
    [
      { "type": "text",  "text": "a golden retriever in a park" },
      { "type": "image", "image": { "url": "data:image/jpeg;base64,<base64-encoded-jpeg>" } }
    ]
  ]
}
```

### Části obsahu

| Typ | Pole dat | Poznámky |
|------|---------------|-------|
| `text` | `text` | Prostý řetězec. Více textových částí v jedné položce spojí převodník poskytovatele. |
| `image` | `image` | Mediální část. |
| `audio` | `audio` | Mediální část. |
| `video` | `video` | Mediální část a volitelně `video_config` (viz Vertex `multimodalembedding@001`). |
| `file` | `file` | Mediální část pro dokumenty, například PDF. |
| `tokens` | `tokens` | Pole celočíselných ID tokenů. Jen poskytovatelé kompatibilní s OpenAI. V jednom požadavku nejde kombinovat s `text`. |

Každá mediální část je stejný objekt:

| Pole | Typ | Povinné | Popis |
|-------|------|----------|-------------|
| `data` | string | Jedno z `data` / `url` | Obsah vložený jako data URI v base64 (`data:image/png;base64,...`). Nezpracované base64 se přijme, když je nastavené `mime_type`. |
| `url` | string | Jedno z `data` / `url` | Odkaz, který poskytovatel umí přeložit. Které schémata se přijímají, závisí na poskytovateli (viz části pro jednotlivé poskytovatele). |
| `mime_type` | string | Ne | Typ média. Někteří poskytovatelé ho vyžadují, když ho nejde odvodit z `data` nebo URL. |
| `filename` | string | Ne | Zobrazovaný název, předává se tam, kde pro něj má poskytovatel pole. |

Validace proběhne před pluginy a před voláním poskytovatele: každá část musí nastavit právě jednu modalitu, mediální část musí nastavit právě jedno z `data` nebo `url` a prázdná položka vrátí `400`.

### Parametry požadavku

Tato pole na nejvyšší úrovni platí pro každou položku, pokud je položka nepřepíše (viz [Parametry pro jednotlivé položky](#parametry-pro-jednotlivé-položky)):

| Pole | Typ | Popis |
|-------|------|-------------|
| `encoding_format` | string | `float` (výchozí), `base64` nebo u modelů z rodiny Cohere `int8`, `uint8`, `binary`, `ubinary`. |
| `dimensions` | integer | Velikost výstupního vektoru. Mapuje se na vlastní název pole u každého poskytovatele. |
| `task_type` | string | Zamýšlené použití vektoru, např. `RETRIEVAL_QUERY` / `RETRIEVAL_DOCUMENT` u Gemini a Vertex nebo hodnota `embeddingPurpose` u Nova. |
| `title` | string | Název dokumentu. Jen textové modely Gemini a Vertex. |
| `auto_truncate` | boolean | Vstupy přesahující limit modelu oříznout, místo aby požadavek selhal. Gemini a Vertex. |

Jakékoli jiné pole na nejvyšší úrovni (například `input_type` u Cohere, `embeddingMode` u Nova nebo `embedContentConfig` u Gemini) se poskytovateli předá beze změny.

`encoding_format: "base64"` funguje u všech poskytovatelů. Kde má model nativní výstup v base64 (OpenAI, Cohere Embed v4), předá se. Všude jinde Bifrost zakóduje vektor float jako base64 z little-endian float32, ve stejném rozložení, jaké používá OpenAI.

### Pole odpovědi

Každá položka v `data` má `index`, `object` a `embedding` a navíc tři volitelná pole, která se objeví jen tehdy, když je poskytovatel potřebuje:

| Pole | Popis |
|-------|-------------|
| `modality` | `text`, `image`, `audio`, `video` nebo `audio_video`. Nastaví se, když poskytovatel vrací samostatné vektory pro každou modalitu (Vertex `multimodalembedding@001`, Nova). |
| `video_segment` | `{ "start_offset_sec", "end_offset_sec" }` u vektorů videa, které pokrývají jen část klipu. |
| `encoding_format` | Kterou reprezentaci vektor představuje, když poskytovatel pro jeden vstup vrací několik (Cohere `embedding_types`). |

### Podpora u poskytovatelů

| Poskytovatel / model | Text | Obrázek | Zvuk | Video | Soubor | Dávka (více položek) | Parametry pro položky | Zdroje médií |
|------------------|------|-------|-------|-------|------|--------------------------|-----------------|---------------|
| Gemini `gemini-embedding-2-preview` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | vložené, URI z Files API |
| Gemini `gemini-embedding-001` | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | - |
| Vertex `gemini-embedding-2-preview` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | vložené, `gs://`, `https://` |
| Vertex `multimodalembedding@001` | ✅ | ✅ (max. 1) | ❌ | ✅ (max. 1) | ❌ | ❌ | ❌ | vložené, `gs://` |
| Vertex `text-embedding-005` a další textové modely | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | - |
| Cohere `embed-v4.0` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | vložené |
| Cohere `embed-*-v3.0` | ✅ | ✅ (položky jen s obrázkem) | ❌ | ❌ | ❌ | ✅ | ❌ | vložené |
| Bedrock `amazon.titan-embed-image-v1` | ✅ | ✅ (max. 1) | ❌ | ❌ | ❌ | ❌ | ❌ | vložené |
| Bedrock `amazon.titan-embed-text-v1` / `v2:0` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | - |
| Bedrock `cohere.embed-*` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | vložené |
| Bedrock `amazon.nova-2-multimodal-embeddings-v1:0` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | vložené, `s3://` |
| OpenAI, Azure, Mistral, Parasail a další kompatibilní s OpenAI | ✅ (+ tokeny) | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | - |
| HuggingFace | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | - |

Část, kterou cílový model nedokáže přenést, vrátí `400` s uvedením modality. Bifrost nikdy žádnou část potichu nezahodí.

### Textové embeddingy (všichni poskytovatelé)

Nejjednodušší tvar: předejte řetězec nebo pole řetězců přímo v `input`:

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "cohere/embed-v4.0",
    "input": ["a golden retriever playing fetch"],
    "input_type": "search_document"
  }'
```

### Gemini

`gemini-embedding-2-preview` přijímá v jedné položce části text, obrázek, zvuk, video a `file` (PDF) v libovolné kombinaci. `gemini-embedding-001` je jen textový.

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini/gemini-embedding-2-preview",
    "input": [
      [
        { "type": "text",  "text": "transcript of the audio" },
        { "type": "audio", "audio": { "data": "data:audio/mpeg;base64,<base64-encoded-mp3>" } }
      ]
    ]
  }'
```

Pro PDF použijte typ části `file`:

```json
"input": [
  [
    { "type": "file", "file": { "data": "data:application/pdf;base64,<base64-encoded-pdf>", "mime_type": "application/pdf" } }
  ]
]
```

Jak Gemini přijímá média:

- Vložená `data` se posílají jako `inlineData`.
- `url` se posílá jako `fileData.fileUri`. Gemini API tam umí přeložit jen URI z [Files API](https://ai.google.dev/gemini-api/docs/files) (`https://generativelanguage.googleapis.com/v1beta/files/...`), takže velká média nejdřív nahrajte, nebo je pošlete vložená. URI `gs://` poskytovatel Gemini odmítne s `400`. Pro média z Cloud Storage použijte Vertex.
- Více položek se pošle jako jedno volání `:batchEmbedContents` a `task_type` / `title` se uplatní na každou položku. Je to také jediný poskytovatel, který respektuje `params` pro jednotlivé položky.
- `documentOcr` a `audioTrackExtraction` patří pod `embedContentConfig`, jediné místo, kde je Gemini přijímá:

```json
{
  "model": "gemini/gemini-embedding-2-preview",
  "input": [[ { "type": "file", "file": { "data": "data:application/pdf;base64,<...>" } } ]],
  "embedContentConfig": { "documentOcr": true, "audioTrackExtraction": true }
}
```

Spotřeba tokenů pochází z `usageMetadata` od Gemini.

### Vertex AI

Vertex má tři API pro embeddingy a Bifrost podle názvu modelu zvolí to správné. Projekt a region pocházejí z konfigurace klíče Vertex, ne z těla požadavku.

| Model | Endpoint Vertex | Vstup |
|-------|-----------------|-------|
| `text-embedding-005`, `text-multilingual-embedding-002` a další textové modely | `:predict` | Jen text. Každá položka se stane jednou instancí. `dimensions` → `outputDimensionality`; `task_type` a `title` se nastavují pro každou instanci; `auto_truncate` je ve výchozím stavu `true`. |
| `gemini-embedding-2-preview` | `:embedContent` | Stejné části jako u Gemini výše, **jedna položka na požadavek** (endpoint nemá dávkový tvar, druhá položka vrátí `400`). |
| `multimodalembedding@001` | `:predict` (multimodální) | Jedna položka s libovolným počtem textových částí, nejvýše jedním obrázkem a nejvýše jedním videem. Části zvuku a `file` se odmítnou. |

#### `gemini-embedding-2-preview` na Vertex

Platí vše z části o Gemini s jedním doplňkem: u částí s obrázkem, zvukem, videem a `file` se přijímají URI `gs://`, protože Vertex Cloud Storage překládá sám. URL `https://` fungují, když je Vertex umí načíst. U čehokoli velkého dejte přednost `gs://`.

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "vertex/gemini-embedding-2-preview",
    "input": [
      [
        { "type": "video", "video": { "url": "gs://my-bucket/clip.mp4", "mime_type": "video/mp4" } }
      ]
    ],
    "task_type": "RETRIEVAL_DOCUMENT"
  }'
```

#### `multimodalembedding@001`

Tento model vrací **samostatný vektor o 1408 dimenzích pro každou modalitu**, všechny ve stejném prostoru embeddingů. URL obrázků a videí musí být `gs://`. Přijímají se i vložené data URI v base64.

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "vertex/multimodalembedding@001",
    "input": [
      [
        { "type": "text",  "text": "product photo" },
        { "type": "image", "image": { "url": "gs://my-bucket/photo.jpg" } }
      ]
    ]
  }'
```

**Odpověď** obsahuje jeden vektor pro každou modalitu:

```json
{
  "data": [
    { "index": 0, "modality": "text",  "embedding": [0.12, -0.34, ...] },
    { "index": 1, "modality": "image", "embedding": [0.56, -0.78, ...] }
  ],
  "usage": null
}
```

Video se vkládá jako **posloupnost segmentů**. K části s videem přidejte `video_config`:

| Pole | Popis |
|-------|-------------|
| `start_offset_sec` | Kde vzorkování začíná. |
| `end_offset_sec` | Kde vzorkování končí. |
| `interval_sec` | Rozestup mezi embeddingy segmentů. Minimálně 4 sekundy. |

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "vertex/multimodalembedding@001",
    "input": [
      [
        { "type": "text",  "text": "product demo" },
        { "type": "video", "video": { "url": "gs://my-bucket/demo.mp4" },
          "video_config": { "start_offset_sec": 0, "end_offset_sec": 16, "interval_sec": 8 } }
      ]
    ]
  }'
```

Každý segment přijde jako samostatná položka s `video_segment`:

```json
{
  "data": [
    { "index": 0, "modality": "text",  "embedding": [ ... ] },
    { "index": 1, "modality": "video", "video_segment": { "start_offset_sec": 0, "end_offset_sec": 8 },  "embedding": [ ... ] },
    { "index": 2, "modality": "video", "video_segment": { "start_offset_sec": 8, "end_offset_sec": 16 }, "embedding": [ ... ] }
  ]
}
```

`dimensions` se mapuje na `parameters.dimension` modelu. Tento model spotřebu tokenů nehlásí, takže `usage` je `null`.

### Cohere

API Cohere má tři vzájemně se vylučující vstupní pole a Bifrost podle obsahu zvolí jedno z nich:

| Obsah | Pole v Cohere | Modely |
|---------|--------------|--------|
| Každá položka je jedna textová část | `texts[]` | v3, v4 |
| Každá položka je jedna obrázková část | `images[]` | v3, v4 |
| Cokoli jiného (text + obrázek v jedné položce, položky s více částmi) | bloky obsahu `inputs[]` | jen v4 |

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "cohere/embed-v4.0",
    "input": [
      [
        { "type": "text",  "text": "a golden retriever in a park" },
        { "type": "image", "image": { "url": "data:image/jpeg;base64,<base64-encoded-jpeg>" } }
      ]
    ],
    "input_type": "search_document"
  }'
```

- Obrázky musí být vložené. Předejte data URI v `url` nebo `data`, případně nezpracované base64 v `data` s nastaveným `mime_type`. Cohere vzdálené URL nenačítá.
- `input_type` je ve výchozím stavu `search_document` a předává se, když je nastavené.
- `encoding_format` se mapuje na `embedding_types` v Cohere, pokud `embedding_types` nepředáte sami. Když Cohere pro jeden vstup vrátí několik reprezentací, každá je samostatnou položkou `data` označenou svým `encoding_format`.
- Obrázkové vstupy se účtují na obrázkový čítač Cohere, zobrazený jako `usage.prompt_tokens_details.image_tokens` a započítaný do `total_tokens`, ne do `prompt_tokens`.

Přímá náhradní cesta `/cohere/v2/embed` přijímá nativní tělo Cohere s `texts` / `images` / `inputs` pro jakéhokoli poskytovatele a vrací nativní obálku Cohere (`embeddings`, `response_type: "embeddings_by_type"`, `meta.billed_units`).

### Bedrock

Modely embeddingů v Bedrocku se směrují podle rodiny a každá rodina má vlastní formát přenosu a omezení.

#### Titan

`amazon.titan-embed-image-v1` přijímá jednu položku s textem a nejvýše jedním obrázkem. Obrázek musí být vložený v poli `data` (data URI nebo nezpracované base64). Titan neumí načítat URL, takže `url` vrátí `400`.

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "bedrock/amazon.titan-embed-image-v1",
    "input": [
      [
        { "type": "text",  "text": "a red square" },
        { "type": "image", "image": { "data": "data:image/png;base64,<base64-encoded-png>" } }
      ]
    ],
    "dimensions": 384
  }'
```

- `dimensions` (256, 384 nebo 1024) se mapuje na `embeddingConfig.outputEmbeddingLength`.
- Textové modely `amazon.titan-embed-text-v1` a `v2:0` přijímají jednu položku na požadavek. Dávka vrátí `400`, místo aby se sloučila do jednoho vektoru. `encoding_format: "binary"` se u v2 předá jako `embeddingTypes` v Titanu.

#### Cohere na Bedrocku

`cohere.embed-english-v3`, `cohere.embed-multilingual-v3` a `cohere.embed-v4` se řídí stejným výběrem `texts[]` / `images[]` / `inputs[]` jako přímé Cohere, s jedním pravidlem specifickým pro Bedrock: Embed v3 přijímá `images[]` jen tehdy, když je `input_type` `image`. Požadavky jen s obrázky proto odcházejí jako `images[]` s výchozím `input_type` `image`, což přijímá v3 i v4. `input_type` zadaný volajícím se vždy zachová. Položky kombinující text a obrázek používají `inputs[]` a potřebují Embed v4.

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "bedrock/cohere.embed-english-v3",
    "input": [
      [ { "type": "image", "image": { "data": "data:image/png;base64,<base64-encoded-png>" } } ]
    ]
  }'
```

#### Amazon Nova

`amazon.nova-2-multimodal-embeddings-v1:0` vytváří embeddingy textu, obrázku, zvuku nebo videa, **jedna modalita na položku a jedna položka na požadavek**. Kombinace modalit vrátí `400`.

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "bedrock/amazon.nova-2-multimodal-embeddings-v1:0",
    "input": [
      [ { "type": "image", "image": { "url": "s3://my-bucket/photos/cat.png" } } ]
    ],
    "task_type": "IMAGE_RETRIEVAL",
    "detailLevel": "STANDARD_IMAGE"
  }'
```

- **Zdroje:** vložená `data` nebo URL `s3://bucket/key`, které Nova čte přímo. Jakákoli jiná URL vrátí `400`.
- **Formát:** Nova vyžaduje u každé mediální části deklarovaný formát a kontroluje ho proti bajtům. Bifrost ho odvodí z `mime_type` nebo z přípony klíče S3 a když to nejde, vrátí `400`. Podporované: obrázky `png`, `jpeg`, `webp`, `gif`; zvuk `mp3`, `wav`, `ogg`; video `mp4`, `mov`, `mkv`, `webm`, `flv`, `mpeg`, `wmv`, `3gp` (do AWS se posílá jako `three_gp`).
- **`task_type`** se mapuje na `embeddingPurpose`: `GENERIC_INDEX` (výchozí), `GENERIC_RETRIEVAL`, `TEXT_RETRIEVAL`, `IMAGE_RETRIEVAL`, `VIDEO_RETRIEVAL`, `AUDIO_RETRIEVAL`, `DOCUMENT_RETRIEVAL`, `CLASSIFICATION`, `CLUSTERING`.
- **`dimensions`:** 256, 384, 1024 nebo 3072 (výchozí).
- **Pole specifická pro Nova na nejvyšší úrovni:** `truncationMode` (výchozí `END`, `START`, `NONE`), `detailLevel` (výchozí `STANDARD_IMAGE`, `DOCUMENT_IMAGE` pro stránky s velkým množstvím textu; jen obrázkové části), `embeddingMode` (jen video, viz níže).

Video ve výchozím stavu vrací jeden vektor `audio_video`. Nastavením `embeddingMode` na `AUDIO_VIDEO_SEPARATE` získáte z jednoho vstupu vektor `audio` a vektor `video`:

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "bedrock/amazon.nova-2-multimodal-embeddings-v1:0",
    "input": [
      [ { "type": "video", "video": { "url": "s3://my-bucket/clips/demo.mp4" } } ]
    ],
    "embeddingMode": "AUDIO_VIDEO_SEPARATE"
  }'
```

```json
{
  "data": [
    { "index": 0, "modality": "audio", "embedding": [ ... ] },
    { "index": 1, "modality": "video", "embedding": [ ... ] }
  ]
}
```

Nova a Cohere na Bedrocku hlásí spotřebu tokenů jen v hlavičce odpovědi, kterou Bifrost načte do `usage`.

Nativní cesta `/bedrock/model/{modelId}/invoke` přijímá těla pro Titan (`inputText` + `inputImage`), Nova (`SINGLE_EMBEDDING`) a Cohere (`texts` / `images` / `inputs`) a vrací vlastní obálku každé rodiny.

### Poskytovatelé kompatibilní s OpenAI

OpenAI, Azure OpenAI, Mistral, Parasail a všichni další poskytovatelé, kteří používají dialekt embeddingů OpenAI, přijímají jen části `text` a `tokens` a v jednom požadavku je nejde kombinovat. Přenášejí se jen `encoding_format` a `dimensions`. `task_type`, `title` a `auto_truncate` nejsou součástí formátu OpenAI a nepředávají se.

### Parametry pro jednotlivé položky

Každá položka v `input` může být místo toho objekt s vlastními `params`, které pro tuto položku přepíšou parametry na úrovni požadavku. Použijte to, když položky potřebují *různé* parametry, například dotaz s `RETRIEVAL_QUERY` spolu s dokumenty s `RETRIEVAL_DOCUMENT`. Běžné položky a položky s parametry lze libovolně kombinovat.

```bash
curl -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini/gemini-embedding-001",
    "dimensions": 768,
    "input": [
      { "content": [{ "type": "text", "text": "how do I reset my password?" }],
        "params": { "task_type": "RETRIEVAL_QUERY", "dimensions": 768 } },
      { "content": [{ "type": "text", "text": "Password reset guide" }],
        "params": { "task_type": "RETRIEVAL_DOCUMENT", "dimensions": 256 } },
      [ { "type": "text", "text": "a plain entry, using the request-level params" } ]
    ]
  }'
```

`params` položky parametry na úrovni požadavku **nahrazují**, neslučují se pole po poli. Zopakujte proto každou hodnotu z úrovně požadavku, kterou chcete zachovat (jako je výše zopakované `dimensions`).

> [!WARNING]
> Respektovat to mohou jen poskytovatelé, jejichž formát přenosu má pro každou položku jeden blok parametrů, aktuálně Gemini (`:batchEmbedContents`). Všichni ostatní vrátí 400 s indexem problematické položky, místo aby přepsání potichu ignorovali a vrátili vektor, který vypadá správně, ale správný není.

Požadavky na náhradních cestách GenAI (`:embedContent`, `:batchEmbedContents`) mají opačné zacházení: když každá položka `requests[]` nese stejné `taskType` / `outputDimensionality`, přesunou se na úroveň požadavku, aby volání fungovalo i proti jiným poskytovatelům než Gemini.

### Co je dobré vědět

- **Spotřeba tokenů se u multimodálních modelů liší.** Vertex `multimodalembedding@001` vrací `usage: null`, vstupy jen s obrázkem u Titanu hlásí nula tokenů a obrázkové tokeny Cohere jsou v `prompt_tokens_details`. Rozpočty a náklady měřené podle `prompt_tokens` tyto vstupy nevidí.
- Klíče **sémantické cache** zahrnují `task_type`, `title` a `auto_truncate`, takže požadavek s `RETRIEVAL_QUERY` nikdy nedostane z cache vektor pro `RETRIEVAL_DOCUMENT`.
- **Logy** ukládají celý `input` do vyhrazeného sloupce `embedding_input` a zobrazují ho v detailu logu. Vložená média nad prahem velkých dat se redigují.

> [!NOTE]
> Další informace o multimodálních schopnostech jednotlivých poskytovatelů najdete na stránce [Supported Providers](https://docs.getbifrost.ai/providers/supported-providers/overview).

## Další kroky

Když už rozumíte multimodálním schopnostem, podívejte se na související témata:

### Základní témata

- **[Streamování odpovědí](./streaming.md)**: multimodální zpracování v reálném čase.
- **[Volání nástrojů](./tool-calling.md)**: kombinace s externími nástroji.
- **[Konfigurace poskytovatelů](./provider-configuration.md)**: více poskytovatelů pro různé schopnosti.
- **[Integrace](./integrations.md)**: přímá kompatibilita se stávajícími SDK.

### Pokročilá témata

- **[Core Features](https://docs.getbifrost.ai/features/)**: pokročilé schopnosti Bifrostu.
- **[Architecture](https://docs.getbifrost.ai/architecture/)**: jak Bifrost funguje uvnitř.
- **[Deployment](https://docs.getbifrost.ai/deployment-guides)**: produkční nasazení a škálování.

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/quickstart/gateway/multimodal.mdx`, commit `0fb9342`. Změny: překlad do češtiny, převod z MDX do Markdownu. Originál: https://docs.getbifrost.ai/quickstart/gateway/multimodal</sub>
