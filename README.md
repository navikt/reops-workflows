# reops-workflows

Felles GitHub Actions-workflows for Team ResearchOps. Brukes foreløpig av `innblikk-frontend` og `innblikk-backend`.

Repoet inneholder byggeklosser. Hver applikasjon har egne, korte workflows som bestemmer når det bygges og deployes.

Repoet må være offentlig. Offentlige repoer, som `innblikk-frontend` og `innblikk-backend`, kan bare bruke workflows fra offentlige repoer ([GitHub-dokumentasjon](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations#access-to-reusable-workflows)). Ikke legg hemmeligheter her.

## Workflows

### `build.yaml`

Bygger og pusher et image med `nais/docker-build-push`.

| Input | Beskrivelse |
|---|---|
| `ref` | Branch, tag eller commit som skal bygges (påkrevd) |
| `runner` | Runner for bygget. Standard er `ubuntu-latest` |
| `fresh` | Bygg uten lagcache, slik at oppdaterte base-images og pakker kommer med |
| `build-args` | Ekstra build-args, én per linje. `GIT_SHA` settes automatisk |

Outputs: `image` (med digest) og `source-sha`.

`NODE_AUTH_TOKEN` sendes alltid som build-secret, slik at private `@navikt`-pakker kan installeres. Dockerfiler som ikke bruker den, ignorerer den.

Kalleren må gi `contents: read`, `packages: read` og `id-token: write`.

### `deploy.yaml`

Deployer et image med `nais/deploy` og venter til de nye podene er klare.

| Input | Beskrivelse |
|---|---|
| `cluster` | `dev-gcp` eller `prod-gcp`. Brukes også som navn på GitHub-environment |
| `manifest` | Sti til Nais-manifestet |
| `ref` | Commiten manifestet hentes fra. Standard er commiten som startet workflowen |
| `image` | Image som skal deployes |
| `source-sha` | Commiten imaget er bygget fra. Påkrevd for `prod-gcp` |

Etter en vellykket deploy til `prod-gcp` lages en annotert tag, `release-<tidspunkt>`, på kildecommiten. Tag-meldingen er image-referansen. Nattlig ombygging bygger den nyeste release-taggen på nytt, og rollback deployer imaget i en valgt release-tag.

Kalleren må gi `id-token: write`, og `contents: write` ved deploy til `prod-gcp`.

## Bruk

```yaml
jobs:
  build:
    uses: navikt/reops-workflows/.github/workflows/build.yaml@<commit-sha>
    permissions:
      contents: read
      packages: read
      id-token: write
    with:
      ref: ${{ github.sha }}

  deploy-dev:
    needs: build
    uses: navikt/reops-workflows/.github/workflows/deploy.yaml@<commit-sha>
    permissions:
      contents: read
      id-token: write
    with:
      cluster: dev-gcp
      manifest: .nais/nais-dev.yaml
      ref: ${{ needs.build.outputs.source-sha }}
      image: ${{ needs.build.outputs.image }}
```

Pin til en commit-SHA, slik at endringer her ikke endrer pipelines i applikasjonene uten en egen PR.
