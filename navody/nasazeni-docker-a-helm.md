<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/deployment-guides/overview.mdx, docs/deployment-guides/runtime-contract.mdx, docs/deployment-guides/helm.mdx, docs/quickstart/gateway/setting-up.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Nasazení: Docker a Helm

*Co Bifrost potřebuje v produkci a jak ho rozběhnout v Dockeru a na Kubernetes.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis je v dokumentaci výrobce: [Deploy Bifrost](https://docs.getbifrost.ai/deployment-guides/overview), [Deployment Requirements](https://docs.getbifrost.ai/deployment-guides/runtime-contract) a [Helm Quick Start](https://docs.getbifrost.ai/deployment-guides/helm).

## Co nasazujete

Bifrost je linuxový kontejner `docker.io/maximhq/bifrost:<verze>` pro `linux/amd64` a `linux/arm64`. Běží pod uživatelem `1000`, poslouchá na portu `8080` a lokální data má v `/app/data`. API, dashboard, `GET /health` i `GET /metrics` jsou na stejném portu.

V produkci vždy připněte konkrétní verzi (nebo digest image), ať se máte kam vrátit.

## Kde jsou data

| Nasazení | Úložiště |
|---|---|
| Jedna OSS instance se SQLite | SQLite v trvalém svazku `/app/data` (výchozí) |
| Jedna instance s PostgreSQL | PostgreSQL 16 a novější pro konfiguraci i logy, svazek `/app/data` není potřeba |
| Více OSS instancí | Stejný `config.json` (režim jen se souborem) a tajemství na každé instanci, PostgreSQL pro sdílené logy |
| Bifrost Enterprise | PostgreSQL 16 a novější, SQLite není podporovaný |

> [!WARNING]
> Jeden soubor SQLite patří vždy jen jednomu procesu Bifrostu. Nesdílejte ho mezi replikami. Více OSS instancí nad sdílenou databází konfigurace není podporované, protože si stav (rozpočty, spotřebu, klíče) drží každá v paměti. Viz [Rozpočty a limity](./rozpocty-a-limity.md).

## Co si připravit

- **Šifrovací klíč** `BIFROST_ENCRYPTION_KEY`: chrání uložené klíče poskytovatelů. Musí zůstat stejný přes restarty, upgrady, repliky i obnovu ze zálohy, jinak uložené hodnoty nepůjdou přečíst.
- **Klíče poskytovatelů** v proměnných prostředí. V `config.json` se na ně odkazuje přes `env.NAZEV`.
- **`BIFROST_SETUP_TOKEN`**: jednorázový token pro vytvoření prvního administrátora.
- **Reverzní proxy nebo ingress**, který nebufferuje streamované odpovědi, povoluje dlouhé požadavky, podporuje WebSocket (pro Realtime API) a předává hlavičky host a forwarded-proto.

## Docker

```bash
# Pull and run Bifrost HTTP API
docker pull maximhq/bifrost
docker run -p 8080:8080 maximhq/bifrost

# Pull a specific version
docker pull maximhq/bifrost:v2.4.0
docker pull maximhq/bifrost:v2.4.0-amd64
docker pull maximhq/bifrost:v2.4.0-arm64
```

Aby konfigurace přežila restart, připojte svazek do `/app/data`:

```bash
# For configuration persistence across restarts
docker run -p 8080:8080 -v $(pwd)/data:/app/data maximhq/bifrost
```

Další přepínače a ochranu dashboardu popisuje [Nastavení brány](../docs/quickstart/gateway/setting-up.md).

## Kubernetes přes Helm

### 1. Repozitář

```bash
helm repo add bifrost https://maximhq.github.io/bifrost/helm-charts
helm repo update
```

### 2. Instalace

Nejmenší nasazení se SQLite (StatefulSet s 10Gi PVC):

```bash
kubectl create secret generic bifrost-encryption-key \
  --from-literal=encryption-key="$(openssl rand -base64 32)"

helm install bifrost bifrost/bifrost \
  --set image.tag=v1.4.11 \
  --set bifrost.encryptionKeySecret.name="bifrost-encryption-key" \
  --set bifrost.encryptionKeySecret.key="encryption-key"
```

> [!IMPORTANT]
> `image.tag` je povinný, bez něj chart nenastartuje. Ukázky výrobce uvádějí starší verzi, dosaďte aktuální z [Docker Hubu](https://hub.docker.com/r/maximhq/bifrost/tags).

Ekvivalent ve values souboru pro jednu instanci se SQLite:

```yaml
image:
  tag: "<BIFROST_VERSION>"

replicaCount: 1

storage:
  mode: sqlite

bifrost:
  encryptionKeySecret:
    name: bifrost-encryption
    key: encryption-key
```

Jedna instance s externím PostgreSQL:

```yaml
image:
  tag: "<BIFROST_VERSION>"

replicaCount: 1

storage:
  mode: postgres

postgresql:
  enabled: false
  external:
    enabled: true
    host: "<POSTGRES_HOST>"
    port: 5432
    user: bifrost
    database: bifrost
    sslMode: require
    existingSecret: postgres-credentials
    passwordKey: password

bifrost:
  encryptionKeySecret:
    name: bifrost-encryption
    key: encryption-key
```

Chart obsahuje hotové values soubory v `helm-charts/bifrost/values-examples/` (například `sqlite-only.yaml`, `external-postgres.yaml`, `secrets-from-k8s.yaml`, `production-ha.yaml`), [přehled na GitHubu](https://github.com/maximhq/bifrost/tree/main/helm-charts/bifrost/values-examples).

### 3. Ověření

```bash
# Check pods are running
kubectl get pods -l app.kubernetes.io/name=bifrost

# Port forward and hit the health endpoint
kubectl port-forward svc/bifrost 8080:8080
curl http://localhost:8080/health

# Check Prometheus metrics
curl http://localhost:8080/metrics
```

### 4. Upgrade

```bash
helm repo update

# Upgrade reusing all existing values
helm upgrade bifrost bifrost/bifrost --reuse-values

# Upgrade with new values
helm upgrade bifrost bifrost/bifrost -f your-values.yaml

# Upgrade and override a single field
helm upgrade bifrost bifrost/bifrost \
  --reuse-values \
  --set image.tag=v1.4.11
```

## Kontrola po nasazení

1. `/health` vrací `200`.
2. Projde jeden běžný a jeden streamovaný požadavek přes stejnou adresu, kterou budou používat aplikace.
3. Po restartu nebo výměně kontejneru zůstane konfigurace zachovaná.

## Další kroky

- **[Observabilita](./observabilita.md):** logy, metriky a trasování po nasazení.
- **Podrobnosti u výrobce:** [Helm Values](https://docs.getbifrost.ai/deployment-guides/helm/values), [config.json](https://docs.getbifrost.ai/deployment-guides/config-json), [Docker Tuning](https://docs.getbifrost.ai/deployment-guides/docker-tuning), návody pro [EKS](https://docs.getbifrost.ai/deployment-guides/platforms/eks), [GKE](https://docs.getbifrost.ai/deployment-guides/platforms/gke), [AKS](https://docs.getbifrost.ai/deployment-guides/platforms/aks) a [ECS](https://docs.getbifrost.ai/deployment-guides/ecs).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubory `docs/deployment-guides/overview.mdx`, `docs/deployment-guides/runtime-contract.mdx`, `docs/deployment-guides/helm.mdx` a `docs/quickstart/gateway/setting-up.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu. Kód je převzatý beze změny. Originály: https://docs.getbifrost.ai/deployment-guides/overview, https://docs.getbifrost.ai/deployment-guides/runtime-contract a https://docs.getbifrost.ai/deployment-guides/helm</sub>
