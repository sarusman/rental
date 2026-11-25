# Rental DevOps — checkpoint 00

Le TP 01 est réalisé sur `tp/01-ci`, comparée à `tp/01-ci-base`, qui pointe sur
`checkpoint/00-starter`. Voir le [compte rendu](docs/TP-01-COMPTE-RENDU.md).

Ce monorepo est le socle d'une formation DevOps pratique destinée à des
étudiants de master. Le checkpoint 00 fournit deux services autonomes,
testables et conteneurisés, mais laisse volontairement à construire la
communication distribuée, la persistance et le déploiement Kubernetes.

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

## Tests et qualité

Depuis la racine :

```bash
python scripts/course.py verify
```

Cette commande exécute les tests Gradle, Pytest, Checkstyle puis Ruff
(lint et formatage), et s'arrête au premier échec avec le code 1.
Les commandes `test`, `test-java`, `test-python`, `lint`, `lint-java` et
`lint-python` permettent de lancer les contrôles séparément.

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
course/               Futurs supports Quarto
docs/                 État de référence et checkpoints
infra/                Future infrastructure de test
scripts/              Commandes portables du cours
tests/                Futures suites de tests système
.github/workflows/     CI minimale à améliorer pendant la formation
```

## Limites intentionnelles du checkpoint

- stockage Java uniquement en mémoire ;
- aucune communication entre les services ;
- aucune base PostgreSQL ni migration ;
- aucun test de contrat ou test E2E ;
- aucun manifeste Kubernetes ou cluster Kind.

Voir [la fiche du checkpoint](docs/CHECKPOINT-00.md) pour les objectifs et les
critères de validation.

L'état historique antérieur à la refonte reste disponible sous le tag
`legacy/v1-microservices`.
