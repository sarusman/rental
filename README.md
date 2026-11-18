# Rental DevOps — TP 01

Ce monorepo est le socle d'une formation DevOps pratique destinée à des
étudiants de master. La branche `tp/01-ci` réalise le TP 01 : tests et qualité
bloquants, construction des images, détection des secrets et rapports exploitables.

Le [compte rendu du TP](docs/TP-01-COMPTE-RENDU.md) rassemble les choix,
les expériences d'échec, les preuves et les étapes restant soumises aux
éléments fournis par l'enseignant.

## Architecture actuelle

| Service | Technologie | Port | Responsabilité |
|---|---|---:|---|
| `car-rental-service` | Java 17, Spring Boot, Gradle | 8080 | Gestion en mémoire des voitures et locations |
| `pricing-service` | Python 3.12, FastAPI | 8000 | Calcul autonome des tarifs |

Les deux services partagent un réseau Docker. Ils ne communiquent pas encore
entre eux.

## Prérequis

- Git ;
- JDK 17 ou supérieur ;
- Python 3.12 ou une version compatible ;
- Docker avec Docker Compose.

Vérification rapide :

```bash
python scripts/course.py prerequisites
```

## Installation des dépendances Python

```bash
cd apps/pricing-service
python -m venv .venv
```

Activation sous Linux ou macOS :

```bash
source .venv/bin/activate
```

Activation sous PowerShell :

```powershell
.venv\Scripts\Activate.ps1
```

Puis :

```bash
python -m pip install -r requirements-dev.txt
cd ../..
```

## Vérification locale

Depuis la racine :

```bash
python scripts/course.py verify
```

Cette commande exécute les tests Gradle et Pytest, puis Checkstyle et Ruff.
Elle s'arrête au premier échec et retourne le code 1.
Pour diagnostiquer une étape séparément, utiliser `test`, `lint`,
`test-java`, `test-python`, `lint-java` ou `lint-python`.

## Intégration continue

Chaque pull request vers `main` déclenche huit contrôles lisibles :

- tests, qualité et construction de l'image pour le service Java ;
- tests, qualité et construction de l'image pour le service Python ;
- détection de secrets sur l'historique Git ;
- revue des dépendances ajoutées ou mises à jour, conservée depuis `main`.

Les rapports de tests et de qualité sont conservés sept jours comme artefacts
GitHub Actions. Les images sont construites uniquement après le succès des
tests et de la qualité. La connexion GHCR, la publication des images et leur
scan Trivy sont réservés aux événements `push` sur `main`.

## Exécution avec Docker Compose

```bash
python scripts/course.py compose-up
python scripts/course.py smoke
```

Endpoints utiles :

```text
GET  http://localhost:8080/health
GET  http://localhost:8080/cars
GET  http://localhost:8000/health
POST http://localhost:8000/quotes
GET  http://localhost:8000/docs
```

Arrêt :

```bash
python scripts/course.py compose-down
```

## Organisation

```text
apps/                 Services applicatifs
archive/              Notes historiques conservées
course/               Présentation Quarto et travaux pratiques
docs/                 État de référence et checkpoints
infra/                Future infrastructure de test
scripts/              Commandes portables du cours
tests/                Futures suites de tests système
.github/workflows/     Pipelines CI par service et sécurité du dépôt
```

## Limites intentionnelles du checkpoint

- stockage Java uniquement en mémoire ;
- aucune communication entre les services ;
- aucune base PostgreSQL ni migration ;
- aucun test de contrat ou test E2E ;
- aucun manifeste Kubernetes ou cluster Kind ;
- dépendances transitives applicatives non verrouillées par hash ;
- aucune vérification de signature à l'admission Kubernetes.

Voir le [compte rendu et les preuves](docs/TP-01-COMPTE-RENDU.md), le
[support de cours](course/slides/01-ci.qmd) et le
[TP 01](course/labs/tp01.qmd).

Le point de départ étudiant reste disponible sous le tag
`checkpoint/00-starter`. Les deux premières étapes validées sont publiées sous
`checkpoint/01-ci` et `checkpoint/02-supply-chain`.

L'état historique antérieur à la refonte reste disponible sous le tag
`legacy/v1-microservices`.
