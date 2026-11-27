# Tagger og images

Deploy-flyten bruker to typer tagger: git-tagger i GitHub-repoet og image-tagger i Nais sitt container-register. Denne siden forklarer hva de er, hvem som lager dem og hva de brukes til.

## Git-tagger

En git-tag er et navn som peker på en commit.

### `release-<tidspunkt>` i applikasjonsrepoene

Eksempel: `release-20261127-101500`.

- Lages av `deploy.yaml` etter en vellykket deploy til `prod-gcp`, når kalleren sender `source-sha`.
- Peker på commiten som ble deployet.
- Tag-meldingen er image-referansen, med digest.

Den nyeste release-taggen viser hvilken kode som kjører i prod nå. Alle release-taggene til sammen er en historikk over releaser og rollbacks til prod. Nattlige ombygginger er ikke med, fordi de ikke endrer koden. Nattlig ombygging bruker den nyeste release-taggen for å finne koden som skal bygges. Rollback bruker en valgt release-tag for å finne imaget som skal deployes.

| Workflow | Leser fra release-tag | Lager ny release-tag |
|---|---|---|
| Release (merge til `main`) | Ingenting | Ja |
| Nattlig ombygging | Commiten i nyeste tag | Nei |
| Rollback | Commit og image i valgt tag | Ja, på samme commit |

Finn taggene under Tags i GitHub, eller med:

```bash
git ls-remote --tags origin 'release-*'
```

### `v1`, `v1.1.0` osv. i dette repoet

Disse bestemmer hvilken versjon av de felles workflowene applikasjonene bruker. Se [Versjonering](../README.md#versjonering).

## Image-tagger og digest

`nais/docker-build-push` gir hvert image en tag med byggetidspunkt og kort commit-SHA:

```text
europe-north1-docker.pkg.dev/<prosjekt>/team-researchops/reops-innblikk-frontend:2026.11.27-10.15-a1b2c3d
```

Vi bruker ikke image-taggen direkte. En tag er bare et navn, og den kan i prinsippet flyttes til et annet image.

Hvert image har også en digest, `sha256:...`. Digesten regnes ut fra innholdet i imaget, så den kan ikke peke på noe annet. Bygget gir ut en image-referanse som inneholder både taggen og digesten:

```text
...:2026.11.27-10.15-a1b2c3d@sha256:9f8e...
```

Det er denne referansen som sendes videre til deploy, både til dev og prod, og som lagres i release-taggen. Når referansen har digest, henter Kubernetes imaget med digesten og ser bort fra taggen. Taggen er bare med for at mennesker skal kjenne igjen imaget.

Slik kjører prod nøyaktig det samme imaget som ble testet i dev, og rollback deployer nøyaktig det samme imaget som kjørte da release-taggen ble laget.

## Hvordan de henger sammen

```text
git-tag release-20261124-101500  ──►  commit a1b2c3d
  melding: ...:2026.11.24-10.15-a1b2c3d@sha256:aaa...  ──►  image A i registeret
```

Nattlig ombygging bygger samme commit på nytt og får et nytt image, med ny tag og digest. Den lager ingen ny release-tag. Release-taggen peker derfor fortsatt på imaget fra den opprinnelige releasen, ikke nødvendigvis på imaget som kjører i prod. Hvilket image som kjører, ser du i Nais Console.

## Eksempel

| Tidspunkt | Hendelse | Image i prod | Nyeste release-tag |
|---|---|---|---|
| Mandag 10:15 | PR merges. Commit `a1b2c3d` bygges til image A, deployes til dev og så til prod | A | `release-man` → `a1b2c3d`, image A |
| Tirsdag 03:00 | Nattlig ombygging bygger `a1b2c3d` uten cache, med nye base-images og pakker | B | `release-man` → `a1b2c3d`, image A |
| Onsdag 03:00 | Nattlig ombygging. Et nytt base-image gir feil i appen | C | `release-man` → `a1b2c3d`, image A |
| Onsdag 08:00 | Rollback til `release-man` | A | `release-ons` → `a1b2c3d`, image A |
| Torsdag 03:00 | Nattlig ombygging bygger `a1b2c3d` på nytt | D | `release-ons` → `a1b2c3d`, image A |
| Torsdag 14:00 | Ny PR merges. Commit `e4f5a6b` bygges til image E | E | `release-tor` → `e4f5a6b`, image E |

Det er et bevisst valg at rollback gir det opprinnelige imaget. Det har kjørt i prod før, så det er trygt å gå tilbake til når en ombygging har ødelagt noe. Ulempen er at imaget mangler sikkerhetsoppdateringene som ombyggingen tok med. Kjør nattlig ombygging manuelt for å få dem før neste natt.

Rollback fikser ikke årsaken. I eksempelet får image D samme feil som C, med mindre feilen er rettet før natt til torsdag. Rett feilen, eller slå av den nattlige workflowen midlertidig under Actions i GitHub.

## Begrensninger

- Rollback til en gammel tag feiler hvis imaget er slettet fra registeret. Deployen feiler da, og podene som kjører, fortsetter som før.
- Tagger fra den gamle deploy-workflowen (`prod-*` og `dev-*`) har ikke alltid image i meldingen og kan ikke brukes til rollback.
