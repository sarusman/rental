# TP 02 — Produire et vérifier les preuves d’une image

## Point de départ et références

La branche `tp/02-supply-chain` part de `checkpoint/01-ci`. La branche `tp/01-ci` reste disponible comme archive. Le relevé ci-dessous porte sur le checkpoint initial : toutes ses références d’actions et d’images de base étaient mutables.

| Emplacement initial | Référence | Classement |
| --- | --- | --- |
| `.github/workflows/ci-car-rental.yml:32` | `actions/checkout@v6` | Mutable |
| `.github/workflows/ci-car-rental.yml:35` | `actions/setup-java@v6` | Mutable |
| `.github/workflows/ci-car-rental.yml:50` | `actions/upload-artifact@v7` | Mutable |
| `.github/workflows/ci-car-rental.yml:66` | `actions/checkout@v6` | Mutable |
| `.github/workflows/ci-car-rental.yml:69` | `actions/setup-java@v6` | Mutable |
| `.github/workflows/ci-car-rental.yml:84` | `actions/upload-artifact@v7` | Mutable |
| `.github/workflows/ci-car-rental.yml:103` | `actions/checkout@v6` | Mutable |
| `.github/workflows/ci-car-rental.yml:106` | `docker/setup-buildx-action@v4` | Mutable |
| `.github/workflows/ci-car-rental.yml:110` | `docker/login-action@v4` | Mutable |
| `.github/workflows/ci-car-rental.yml:118` | `docker/metadata-action@v6` | Mutable |
| `.github/workflows/ci-car-rental.yml:127` | `docker/build-push-action@v7` | Mutable |
| `.github/workflows/ci-car-rental.yml:138` | `aquasecurity/trivy-action@master` | Mutable |
| `.github/workflows/ci-car-rental.yml:146` | `github/codeql-action/upload-sarif@v4` | Mutable |
| `.github/workflows/ci-pricing.yml:32` | `actions/checkout@v6` | Mutable |
| `.github/workflows/ci-pricing.yml:35` | `actions/setup-python@v7` | Mutable |
| `.github/workflows/ci-pricing.yml:53` | `actions/upload-artifact@v7` | Mutable |
| `.github/workflows/ci-pricing.yml:67` | `actions/checkout@v6` | Mutable |
| `.github/workflows/ci-pricing.yml:70` | `actions/setup-python@v7` | Mutable |
| `.github/workflows/ci-pricing.yml:98` | `actions/checkout@v6` | Mutable |
| `.github/workflows/ci-pricing.yml:101` | `docker/setup-buildx-action@v4` | Mutable |
| `.github/workflows/ci-pricing.yml:105` | `docker/login-action@v4` | Mutable |
| `.github/workflows/ci-pricing.yml:113` | `docker/metadata-action@v6` | Mutable |
| `.github/workflows/ci-pricing.yml:122` | `docker/build-push-action@v7` | Mutable |
| `.github/workflows/ci-pricing.yml:133` | `aquasecurity/trivy-action@master` | Mutable |
| `.github/workflows/ci-pricing.yml:141` | `github/codeql-action/upload-sarif@v4` | Mutable |
| `.github/workflows/ci-security.yml:26` | `actions/checkout@v6` | Mutable |
| `.github/workflows/ci-security.yml:31` | `gitleaks/gitleaks-action@v3` | Mutable |
| `apps/car-rental-service/Dockerfile:2` | `gradle:8.14-jdk17` | Mutable |
| `apps/car-rental-service/Dockerfile:8` | `eclipse-temurin:17-jre-jammy` | Mutable |
| `apps/pricing-service/Dockerfile:1` | `python:3.12-slim` | Mutable |

Dans le résultat, chaque action distante utilise un commit de 40 caractères, accompagné du commentaire de version demandé par le TP. Les trois images de base conservent leur tag et un digest SHA-256. Les 15 références d’actions distinctes ont été comparées aux tags de leurs dépôts officiels ; les trois digests ont été résolus dans le registre.

Dependabot suit `github-actions` à la racine et `docker` dans chacun des deux services. Il ouvre des PR ordinaires pour les mises à jour.

## Risques et contrôles

- Action GitHub : déplacer un tag peut substituer du code exécuté avec les permissions du job et exposer son token. Un SHA complet fixe le code exécuté ; les mises à jour passent par une PR Dependabot.
- Image de base : remplacer le manifeste d’un tag peut introduire d’autres paquets ou du code malveillant dans l’image finale. Le digest fixe le contenu de la base utilisée.
- Image produite : déplacer `latest` après le scan peut faire déployer une autre image. La signature, les attestations, le SBOM et Trivy portent sur le digest fourni par Buildx. La vérification Cosign contrôle l’identité du workflow et l’émetteur OIDC ; la provenance rattache l’image au dépôt et au commit.

Dependency review examine les changements de dépendances entre la base et la tête d’une PR et échoue à partir de la gravité `high`. Trivy examine les composants réellement présents dans l’image finale, notamment les paquets système et les bibliothèques applicatives.

## Contrôles effectués

Le 25 novembre 2026, `python scripts/course.py verify` a réussi dans un environnement Python 3.11.5 isolé avec les dépendances déclarées du projet : 34 tests Java, 9 tests Python, Checkstyle, Ruff et politique des références. La CI Python utilise Python 3.12.

La politique a été exécutée avec un `@v7` temporaire puis après restauration du SHA : échec attendu puis succès. Les cas `- uses: actions/checkout@v7`, image sans digest et digest placé seulement dans un commentaire ont également été rejetés. Tous les fichiers ont ensuite été restaurés.

Les alertes de vulnérabilités GitHub sont activées (réponse API HTTP 204). Le graphe de dépendances est accessible : son export SPDX contient 23 paquets. Cet export du dépôt est distinct du SBOM de chaque image demandé ci-dessous.

La relance du [run Java 45626227851](https://github.com/sarusman/rental/actions/runs/45626227851) a réussi après le HTTP 429 de Maven Central. Les contrôles Java, Python, Dependency review et Secret detection de la PR #6 sont verts pour le commit `b2ccd0fb65fb50e67fcf322ea78d1b7d7257118f`. Les jobs `supply-chain` sont ignorés sur cet événement de PR, comme prévu. Après publication du commit `d939ba87c4a2dfd77ba22a9c1bd85a8798f2f52d`, tous les nouveaux contrôles de la PR #6 ont réussi. La PR a été fusionnée au commit `348b6108b8ef24ec0e9fbd765df86d7818308284`.

Les workflows publient sur `main` et transmettent `steps.build.outputs.digest` au job suivant. Les preuves utilisent toutes cette valeur. Les filtres de chemins sur les pushes `main` ont été supprimés pour que la fusion du compte rendu déclenche aussi les deux pipelines.

## Configuration de la production des preuves

Pour chaque image publiée, le job `supply-chain` attend le job Docker réussi et s’exécute seulement lors d’un push sur `main`.

- Syft, via `anchore/sbom-action`, produit `java-sbom.spdx.json` et `python-sbom.spdx.json` au format SPDX JSON, conservés sept jours.
- Cosign signe le digest avec OIDC, sans clé privée ni secret de signature supplémentaire.
- `actions/attest` atteste la provenance puis le SBOM avec `sbom-path` et publie les attestations au registre.
- `cosign verify` et `gh attestation verify` précèdent Trivy ; le scan utilise le même digest.
- Les permissions du job sont `contents: read`, `packages: write`, `id-token: write`, `attestations: write`, `artifact-metadata: write` et `security-events: write`, pour lire le dépôt, publier l’image et ses preuves, obtenir l’identité OIDC et téléverser le SARIF.

## Preuves d’exécution sur main

Les deux exécutions du 25 novembre 2026 ont réussi sur le commit `348b6108b8ef24ec0e9fbd765df86d7818308284`. Les journaux, attestations, SBOM et rapports SARIF ont été téléchargés et recoupés les 25 et 26 novembre. Ces preuves décrivent ce commit précis, indépendamment des déplacements ultérieurs de `latest`.

| Preuve observée | Java | Python |
| --- | --- | --- |
| Exécution réussie | [45631150548](https://github.com/sarusman/rental/actions/runs/45631150548) | [45631148316](https://github.com/sarusman/rental/actions/runs/45631148316) |
| Image | `ghcr.io/sarusman/rental/car-rental-service` | `ghcr.io/sarusman/rental/pricing-service` |
| Digest vérifié | `sha256:5e8875cd9fd2b75f9d0645944df3603c15ddc144a64ea4732ba192c1c5d8a545` | `sha256:66ae1d2a0a7ae716c5686d3467b38d04d3fa0478247b270f3cbd5e7f6154011c` |
| Identité du certificat Cosign | `https://github.com/sarusman/rental/.github/workflows/ci-car-rental.yml@refs/heads/main` | `https://github.com/sarusman/rental/.github/workflows/ci-pricing.yml@refs/heads/main` |
| Émetteur OIDC vérifié | `https://token.actions.githubusercontent.com` | `https://token.actions.githubusercontent.com` |
| Dépôt dans la provenance | `https://github.com/sarusman/rental` | `https://github.com/sarusman/rental` |
| Commit dans la provenance | `348b6108b8ef24ec0e9fbd765df86d7818308284` | `348b6108b8ef24ec0e9fbd765df86d7818308284` |
| SBOM téléchargé | [java-sbom.spdx.json](https://github.com/sarusman/rental/actions/runs/45631150548/artifacts/15029299977) | [python-sbom.spdx.json](https://github.com/sarusman/rental/actions/runs/45631148316/artifacts/15030328517) |
| Format observé | SPDX-2.3 JSON | SPDX-2.3 JSON |
| Paquet système | `libc6 2.35-0ubuntu3.15` | `libc6 2.41-12+deb13u4` |
| Bibliothèque applicative | `spring-boot 3.4.0` | `fastapi 0.162.7` |
| Analyse Trivy | [2248635713](https://api.github.com/repos/sarusman/rental/code-scanning/analyses/2248635713), catégorie `trivy-car-rental` | [2248621344](https://api.github.com/repos/sarusman/rental/code-scanning/analyses/2248621344), catégorie `trivy-pricing` |
| Journal du scan et de son téléversement | [Java supply chain](https://github.com/sarusman/rental/actions/runs/45631150548/job/139265870815) | [Python supply chain](https://github.com/sarusman/rental/actions/runs/45631148316/job/139265072864) |
| Résultats dans le SARIF téléchargé | 234 | 261 |

Les rapports sont également consultables dans [GitHub Code scanning, outil Trivy sur main](https://github.com/sarusman/rental/security/code-scanning?query=tool%3ATrivy%20ref%3Arefs%2Fheads%2Fmain). Les identifiants d’analyse ci-dessus désignent les rapports exacts de ce commit. Leur SARIF est téléchargeable avec l’API GitHub en utilisant l’en-tête `Accept: application/sarif+json`.

Les artefacts SBOM ont été créés le 25 novembre et expirent le 2 décembre 2026, conformément aux sept jours demandés. Les versions de composants ci-dessus proviennent de ces fichiers téléchargés.

## Analyse des preuves

Les étapes `Verify signature and attestations` ont réussi pour les deux images avant leur scan Trivy. Cosign a accepté l’identité exacte du workflow sur `main` avec l’émetteur OIDC GitHub. `gh attestation verify` a accepté les attestations associées au digest.

Les sujets des attestations de provenance et de SBOM contiennent chacun le digest indiqué dans le tableau. La provenance SLSA contient le dépôt dans `buildDefinition.externalParameters.workflow.repository` et le commit dans `buildDefinition.resolvedDependencies[].digest.gitCommit`.

Pour chaque image, le document SPDX téléchargé a été comparé au prédicat de son attestation SBOM : les documents JSON sont identiques. Les paquets système et les bibliothèques applicatives ont donc été relevés dans le SBOM attesté de l’image vérifiée.

Ces preuves rattachent le contenu immuable à un dépôt, un commit et une identité de workflow. Elles ne prouvent pas l’absence de vulnérabilités : les rapports Trivy contiennent respectivement 234 et 261 résultats. Le TP exige le scan et ses preuves ; ce workflow ne configure pas un échec automatique sur ces résultats.

La validation locale a réussi, les contrôles de la PR #6 ont réussi et les deux jobs de chaîne logicielle sur `main` ont réussi. Les preuves ci-dessus couvrent les deux images.
