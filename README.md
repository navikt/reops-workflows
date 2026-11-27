# reops-workflows

Felles GitHub Actions-workflows for Team ResearchOps. Brukes foreløpig av `innblikk-frontend` og `innblikk-backend`.

Repoet inneholder byggeklosser. Hver applikasjon har egne, korte workflows som bestemmer når det bygges og deployes.

Repoet må være offentlig. Offentlige repoer, som `innblikk-frontend` og `innblikk-backend`, kan bare bruke workflows fra offentlige repoer ([GitHub-dokumentasjon](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations#access-to-reusable-workflows)). Ikke legg hemmeligheter her.

## Workflows

### `build.yaml`

Bygger et image med `nais/docker-build-push` og pusher det til Nais-registeret.

| Input | Beskrivelse |
|---|---|
| `ref` | Branch, tag eller commit som skal bygges. Standard er commiten som startet workflowen |
| `push` | Push imaget. Sett til `false` i CI for bare å sjekke at imaget bygger. Standard er `true` |
| `runner` | Runner for bygget. Standard er `ubuntu-latest` |
| `fresh` | Bygg uten lagcache, slik at oppdaterte base-images og pakker kommer med |
| `build-args` | Ekstra build-args, én per linje. `GIT_SHA` settes automatisk |

Outputs: `image` (med digest når imaget er pushet) og `source-sha`.

`NODE_AUTH_TOKEN` sendes alltid som build-secret, slik at private `@navikt`-pakker kan installeres. Dockerfiler som ikke bruker den, ignorerer den.

Kalleren må gi `contents: read`, `packages: read` og `id-token: write`, også når `push` er `false`, fordi Nais-innloggingen alltid kjøres.

### `deploy.yaml`

Deployer et image med `nais/deploy` og venter til de nye podene er klare.

| Input | Beskrivelse |
|---|---|
| `cluster` | `dev-gcp` eller `prod-gcp`. Brukes også som navn på GitHub-environment |
| `manifest` | Sti til Nais-manifestet |
| `ref` | Commiten manifestet hentes fra. Standard er commiten som startet workflowen |
| `image` | Image som skal deployes |
| `source-sha` | Commiten imaget er bygget fra. Settes den ved deploy til `prod-gcp`, lages en release-tag |

Etter en vellykket deploy til `prod-gcp` med `source-sha` lages en annotert tag, `release-<tidspunkt>`, på kildecommiten. Tag-meldingen er image-referansen. Taggene viser hvilken kode som kjører i prod. Nattlig ombygging bygger den nyeste release-taggen på nytt uten å sende `source-sha`, så den lager ingen ny tag. Rollback deployer imaget i en valgt release-tag og lager en ny tag, slik at nattlig ombygging bygger koden det ble rullet tilbake til.

Kalleren må gi `id-token: write`, og `contents: write` ved deploy til `prod-gcp`.

`nais/deploy` v3 deployer med `nais apply`, som ignorerer `metadata.annotations` i manifestet ([CLI-dokumentasjon](https://cli.nais.io/nais_apply.html)). Deployen feiler ikke, men annotasjonene blir ikke brukt. Har manifestet annotasjoner, finn en annen løsning før appen tar i bruk `deploy.yaml`.

## Bruk

```yaml
jobs:
  build:
    uses: navikt/reops-workflows/.github/workflows/build.yaml@v1
    permissions:
      contents: read
      packages: read
      id-token: write
    with:
      ref: ${{ github.sha }}

  deploy-dev:
    needs: build
    uses: navikt/reops-workflows/.github/workflows/deploy.yaml@v1
    permissions:
      contents: read
      id-token: write
    with:
      cluster: dev-gcp
      manifest: .nais/nais-dev.yaml
      ref: ${{ needs.build.outputs.source-sha }}
      image: ${{ needs.build.outputs.image }}
```

## Versjonering

Applikasjonene bruker `@v1`. Taggen `v1` flyttes til nyeste kompatible versjon, så endringer når alle applikasjonene uten en PR i hvert repo. Lag en ny hovedversjon (`v2`) ved endringer som krever endringer hos kallerne.

Etter at en endring er merget til `main`:

```bash
git tag -a v1.2.0 -m "Kort beskrivelse"
git tag -f v1 "v1.2.0^{commit}"
git push origin v1.2.0
git push -f origin v1
```

## Actions er låst til commit-SHA

Actions som workflowene bruker, refereres med full commit-SHA og versjonen som kommentar:

```yaml
uses: nais/deploy/actions/deploy@427e29c94d911626a023a76f402f6dcae1f840b7 # v3
```

Nais gjør det samme i for eksempel [`nais/docker-build-push`](https://github.com/nais/docker-build-push/blob/main/action.yml).

En versjonstag som `@v2` kan flyttes av den som eier actionen, slik at koden bak endres uten at vi merker det. Det har skjedd i praksis: i mars 2025 ble taggene til `tj-actions/changed-files` flyttet til en ondsinnet commit som lekket hemmeligheter. En commit-SHA kan ikke endres ([GitHub-dokumentasjon](https://docs.github.com/en/actions/reference/security/secure-use#using-third-party-actions)).

Det er ekstra viktig her, fordi workflowene har rettigheter til å logge inn i Nais (`id-token: write`) og lage release-tagger (`contents: write`).

Dependabot oppdaterer SHA-en og versjonskommentaren sammen. Den venter tre dager etter en ny versjon før den lager PR, slik at kompromitterte versjoner som regel blir oppdaget og trukket først.

Når du legger til en ny action, slå opp commit-SHA-en for versjonen du vil bruke. Står det en linje som slutter på `^{}`, er det den SHA-en som er commiten:

```bash
git ls-remote --tags https://github.com/<eier>/<action> 'v2*'
```
