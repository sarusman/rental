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

La relance du [run Java 45626227851](https://github.com/sarusman/rental/actions/runs/45626227851) a réussi après le HTTP 429 de Maven Central. Les contrôles Java, Python, Dependency review et Secret detection de la PR #6 sont verts pour le commit `b2ccd0fb65fb50e67fcf322ea78d1b7d7257118f`. Les jobs `supply-chain` sont ignorés sur cet événement de PR, comme prévu. Les corrections locales doivent repasser ces contrôles après leur publication.

Les workflows publient sur `main` et transmettent `steps.build.outputs.digest` au job suivant. Les preuves utilisent toutes cette valeur. Les filtres de chemins sur les pushes `main` ont été supprimés pour que la fusion du compte rendu déclenche aussi les deux pipelines.

## Configuration de la production des preuves

Pour chaque image publiée, le job `supply-chain` attend le job Docker réussi et s’exécute seulement lors d’un push sur `main`.

- Syft, via `anchore/sbom-action`, produit `java-sbom.spdx.json` et `python-sbom.spdx.json` au format SPDX JSON, conservés sept jours.
- Cosign signe le digest avec OIDC, sans clé privée ni secret de signature supplémentaire.
- `actions/attest` atteste la provenance puis le SBOM avec `sbom-path` et publie les attestations au registre.
- `cosign verify` et `gh attestation verify` précèdent Trivy ; le scan utilise le même digest.
- Les permissions du job sont `contents: read`, `packages: write`, `id-token: write`, `attestations: write`, `artifact-metadata: write` et `security-events: write`, pour lire le dépôt, publier l’image et ses preuves, obtenir l’identité OIDC et téléverser le SARIF.

## Preuves d’exécution sur main — en attente

Les étapes de production et de vérification sont configurées, mais leur réussite sur `main` reste à établir. Une exécution de PR ne publie pas d’image et ne fournit pas ces preuves.

| Preuve observée | Java | Python |
| --- | --- | --- |
| Exécution réussie sur main | En attente | En attente |
| Digest vérifié | En attente | En attente |
| Identité du certificat Cosign | En attente | En attente |
| Dépôt dans la provenance | En attente | En attente |
| Commit dans la provenance | En attente | En attente |
| Paquet système du SBOM téléchargé, avec version | En attente | En attente |
| Bibliothèque applicative du SBOM téléchargé, avec version | En attente | En attente |
| Lien vers le rapport Trivy | En attente | En attente |

Le TP reste incomplet tant que ce tableau n’est pas renseigné à partir des preuves réelles et que les deux jobs de chaîne logicielle sur `main` n’ont pas réussi.
