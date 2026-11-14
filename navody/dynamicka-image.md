<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/plugins/building-dynamic-binary.mdx, docs/deployment-guides/helm/plugins.mdx, docs/deployment-guides/helm/values.mdx, docs/deployment-guides/helm.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Dynamická image pro vlastní pluginy

*Jak sestavit Bifrost, který vlastní Go plugin načte, a nasadit ho v Dockeru i Kubernetes.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis včetně kompletních Dockerfilů je v dokumentaci výrobce: [Building Dynamically Linked Bifrost Binary](https://docs.getbifrost.ai/plugins/building-dynamic-binary) a [Helm: Plugins](https://docs.getbifrost.ai/deployment-guides/helm/plugins).

## Proč

Go načítá pluginy (`.so`) jen do **dynamicky linkované** binárky. Bifrost se ve výchozím stavu sestavuje staticky, protože tak běží na jakékoli linuxové distribuci bez závislostí. Proto oficiální image `maximhq/bifrost` vlastní plugin nenačte.

| | Statický build (výchozí, oficiální image) | Dynamický build |
|---|---|---|
| Vlastní Go pluginy | ❌ | ✅ |
| Přenositelnost | jakákoli distribuce | jen systém se stejnou libc |
| Kdo ho sestavuje | výrobce | vy |

**Doplněk think private:** v release pipeline výrobce jsme k commitu `0fb9342` dynamickou variantu image nenašli. Počítejte s tím, že si ji sestavíte sami a budete ji znovu sestavovat při každém vydání Bifrostu, stejně jako plugin.

## Postup v kostce

1. Zvolte libc podle cílového prostředí (Alpine, nebo Debian).
2. Naklonujte zdrojový kód Bifrostu ve verzi, kterou chcete provozovat (tag `transports/vX.Y.Z`, viz [Vlastní plugin v Go](./vlastni-plugin-go.md#než-začnete)).
3. Sestavte dynamický Bifrost.
4. Plugin sestavte ve **stejném** prostředí (stejné Go, libc, architektura).
5. Ověřte linkování a načtení pluginu.
6. Nasaďte vlastní image místo oficiální.

## 1. Volba libc

| Cílové prostředí | Sestavujte s | Základní image pro build |
|---|---|---|
| Alpine kontejnery, Kubernetes s Alpine | musl | `golang:1.27.3-alpine3.24` |
| Debian a Ubuntu kontejnery i servery, Kubernetes s Debianem | glibc | `golang:1.27.3-bookworm` |
| RHEL a CentOS servery | glibc | nativní build nebo glibc kontejner |

> [!WARNING]
> Binárka sestavená s musl nepoběží na glibc a naopak. Plugin a Bifrost musí mít stejnou libc.

## 2. Sestavení binárky

Ve zdrojovém kódu Bifrostu:

```bash
# Build dynamically linked binary for your current platform
make build DYNAMIC=1

# With version tag
make build DYNAMIC=1 VERSION=1.3.30
```

Přepínač `DYNAMIC=1` odstraní statické linkování (`-extldflags "-static"` a tag `sqlite_static`) a nechá zapnuté CGO, které potřebují SQLite i pluginy.

## 3. Sestavení image

V repozitáři výrobce je `transports/Dockerfile` pro statický build. Pro dynamický ho upravte podle dokumentace výrobce, která uvádí kompletní Dockerfile pro Alpine i Debian. Rozdíl proti statickému je hlavně v příkazu `go build` bez `-extldflags '-static'` a bez `-tags "sqlite_static"`. Sestavení a spuštění:

```bash
# Build the image
docker build -f transports/Dockerfile -t bifrost:dynamic-alpine .

# Run the container
docker run -p 8080:8080 -v ./plugins:/app/data/plugins bifrost:dynamic-alpine
```

Varianta s Debianem:

```bash
# Build the image
docker build -f transports/Dockerfile.debian -t bifrost:dynamic-debian .

# Run the container
docker run -p 8080:8080 -v ./plugins:/app/data/plugins bifrost:dynamic-debian
```

## Plugin sestavte ve stejném prostředí

Nejjednodušší je sestavit plugin v kontejneru se stejným základním image jako Bifrost:

```bash
# If Bifrost was built with Alpine/musl
docker run --rm \
  -v "$PWD:/work" \
  -w /work \
  golang:1.27.3-alpine3.24 \
  sh -c "apk add --no-cache gcc musl-dev && \
         go build -buildmode=plugin -o myplugin.so main.go"

# If Bifrost was built with Debian/glibc  
docker run --rm \
  -v "$PWD:/work" \
  -w /work \
  golang:1.27.3-bookworm \
  sh -c "apt-get update && apt-get install -y gcc && \
         go build -buildmode=plugin -o myplugin.so main.go"
```

Ještě spolehlivější je sestavit plugin i Bifrost v jednom vícestupňovém Dockerfilu. Pak mají jistě identické prostředí a plugin je rovnou součástí image:

```dockerfile
# Build plugin
FROM golang:1.27.3-alpine3.24 AS plugin-builder
WORKDIR /plugin
COPY plugins/myplugin/ .
RUN apk add --no-cache gcc musl-dev && \
    go build -buildmode=plugin -o myplugin.so main.go

# Build Bifrost
FROM golang:1.27.3-alpine3.24 AS bifrost-builder
# ... (bifrost build steps)

# Runtime
FROM alpine:3.23
COPY --from=bifrost-builder /app/main .
COPY --from=plugin-builder /plugin/myplugin.so /app/plugins/
```

## 4. Ověření

```bash
# Check binary dependencies
ldd tmp/bifrost-http

# Expected output (musl):
linux-vdso.so.1
libc.musl-x86_64.so.1 => /lib/ld-musl-x86_64.so.1

# Expected output (glibc):
linux-vdso.so.1
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0
```

Pokud `ldd` vypíše `statically linked`, binárka pluginy nenačte. Shodu verzí pluginu a binárky ověříte přes `go version -m`, viz [Vlastní plugin v Go](./vlastni-plugin-go.md#když-se-plugin-nenačte).

## 5. Kubernetes přes Helm

Chart ve výchozím stavu používá oficiální image `docker.io/maximhq/bifrost` (`image.repository`). **Doplněk think private:** dokumentace Helmu u vlastních pluginů neříká, že je potřeba image vyměnit. Bez toho se plugin nenačte. Ve values nastavte svou dynamickou image:

<!-- kód: think private -->
```yaml
image:
  repository: registry.example.com/bifrost-dynamic
  tag: "v2.5.1"
imagePullSecrets:
  - name: registry-credentials
```

Plugin zapnete v `bifrost.plugins.custom`:

```yaml
bifrost:
  plugins:
    custom:
      - name: "my-custom-plugin"
        enabled: true
        path: "/plugins/my-plugin.so"
        config:
          api_endpoint: "https://my-service.example.com"
          timeout: 5000
```

Soubor `.so` musí být v kontejneru na cestě z `path`. Dokumentace výrobce ukazuje připojení přes svazek:

```yaml
volumes:
  - name: custom-plugins
    configMap:
      name: bifrost-custom-plugins

volumeMounts:
  - name: custom-plugins
    mountPath: /plugins
```

> [!TIP]
> **Doplněk think private:** ConfigMap má v Kubernetes limit 1 MiB a Go plugin bývá větší. Spolehlivější je mít plugin přímo ve vlastní image (vícestupňový Dockerfile výše), případně ho stáhnout init kontejnerem, jak popisuje [dokumentace výrobce](https://docs.getbifrost.ai/deployment-guides/helm/plugins). Verze pluginu a image pak zůstávají svázané.

## Bifrost Enterprise

Zákazníci Enterprise dostávají vlastní image v privátním registru výrobce. Než budete vlastní plugin nasazovat na Enterprise, ověřte s podporou výrobce, jak má být image pro pluginy sestavená. Rádi to zjistíme za vás.

## Další kroky

- **[Vlastní plugin v Go](./vlastni-plugin-go.md):** kostra pluginu a řešení chyb při načtení.
- **[Nasazení: Docker a Helm](./nasazeni-docker-a-helm.md):** úložiště, tajemství a kontrola po nasazení.
- **Podrobnosti u výrobce:** [Building Dynamically Linked Bifrost Binary](https://docs.getbifrost.ai/plugins/building-dynamic-binary) (kompletní Dockerfily, závislosti modulů), [Helm: Plugins](https://docs.getbifrost.ai/deployment-guides/helm/plugins), [Helm Values](https://docs.getbifrost.ai/deployment-guides/helm/values).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubory `docs/plugins/building-dynamic-binary.mdx`, `docs/deployment-guides/helm/plugins.mdx`, `docs/deployment-guides/helm/values.mdx` a `docs/deployment-guides/helm.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu, doplňky think private (označené). Kód výrobce je převzatý beze změny. Originály: https://docs.getbifrost.ai/plugins/building-dynamic-binary a https://docs.getbifrost.ai/deployment-guides/helm/plugins</sub>
