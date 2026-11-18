# TP 01 — une CI observable et bloquante

Dépôt : [sarusman/rental](https://github.com/sarusman/rental).
Branche conservée : `tp/01-ci`.
PR : [#1](https://github.com/sarusman/rental/pull/1).

## 0. Préparer la branche

La branche part de `checkpoint/00-starter` (`ce7e9cf`). Les commits de
l'expérience initiale étaient déjà présents localement ; ils ont été publiés
successivement dans ce dépôt avant la publication des portes qualité.

`main` contient les étapes avancées du cours. La première PR a donc exécuté
les workflows du résultat de fusion avec `main`. Après ajout des portes du TP,
GitHub signalait des conflits et ne lançait plus la CI. Le commit `bdd0909`
intègre `main` dans la branche de travail et résout ces conflits en conservant
les portes du TP. Les supports du cours et le contrôle de dépendances existants
sont conservés. Aucun commit n'est poussé directement sur `main`.

Lecture des workflows du **starter**, avant modification : les coûts ci-dessous
sont des estimations relatives, pas des mesures d'exécution.

| Job initial | Risque couvert | Entrées | Sortie | Coût estimé |
| --- | --- | --- | --- | --- |
| Java `test` | Régression Java | Sources, tests, JDK 17, dépendances Gradle | Résultat Gradle et XML JUnit | Moyen : compilation et tests Spring |
| Java `code-quality` | Build impossible, secret dans Git | Sources, historique complet, Gradle, Gitleaks | Statut du build et du scan ; aucune règle Checkstyle configurée | Moyen : compilation et scan |
| Java `build-and-push` | Image impossible à construire | Contexte Docker, résultats des deux jobs | Image ; publication et scan sur push main | Élevé : couches Docker et compilation |
| Java `pipeline-status` | Échec des jobs précédents | États des trois jobs | Statut synthétique | Faible : script shell |
| Python `test` | Régression métier ou API | Sources, tests, Python 3.12, dépendances pip | XML Pytest | Faible à moyen : installation puis tests |
| Python `build-and-push` | Image impossible à construire | Contexte Docker, résultat des tests | Image ; publication et scan sur push main | Élevé : récupération et construction des couches |

## 1. Observer un échec

Commits : `472ded9` provoque l'échec ; `09e6e14` rétablit le test.
La PR a été créée en brouillon avant de publier le rétablissement.

Exécution Python : [44327492178](https://github.com/sarusman/rental/actions/runs/44327492178).

- Premier job en échec parmi tous les workflows : `Dependency review`, terminé
  après 9 secondes. Son message indique que le graphe de dépendances du fork
  n'est pas activé. Ce contrôle vient du `main` avancé, pas du starter.
  Le graphe a ensuite été activé et ce contrôle passe.
- Premier job applicatif en échec : `Python quality`, terminé après 15 secondes.
  Ruff signale `B011` sur `tests/test_pricing.py:16:12`, à cause de `assert False`.
  Cette règle supplémentaire venait aussi du résultat de fusion avec `main`.
- `Python tests` termine en échec après 22 secondes. Diagnostic :
  `tests/test_pricing.py:16: AssertionError: Échec volontaire pour le TP 01`.
  Résultat : 1 échec, 8 succès.
- Le rapport structuré `python-test-reports` est disponible malgré l'échec :
  [XML téléchargé](evidence/tp01/initial-failure/pytest.xml),
  [métadonnées de l'exécution](evidence/tp01/initial-failure/run.json).
- Les délais incluent l'attente initiale du runner, depuis la création de
  l'exécution jusqu'à la fin du job. Le workflow Python termine après 23 secondes.

## 2. Ajouter les portes qualité

Checkstyle vérifie les imports inutilisés (`UnusedImports`), les accolades
(`NeedBraces`), les instructions vides (`EmptyStatement`) et la longueur des
lignes (`LineLength`, 120 caractères). Sa version est fixée à 9.3 ; les rapports
XML et HTML sont explicitement activés.

Ruff est fixé à `0.11.13` dans les dépendances de développement. Les familles
`E`, `F` et `I` sont activées, avec une longueur de ligne de 120 caractères.
Le formatage est vérifié séparément avec `ruff format --check app tests`.
Aucune exception locale ni désactivation globale n'a été ajoutée.

## 3. Redessiner le graphe CI

Chaque workflow applicatif a exactement trois jobs : tests et qualité en
parallèle, puis image avec `needs: [tests, quality]`.

- Timeouts : Java tests/qualité 15 min ; Python tests/qualité 10 min ;
  images 25 min ; secrets 10 min.
- Permission du workflow : `contents: read`. Les jobs image ajoutent
  `packages: write` pour GHCR et `security-events: write` pour le scan Trivy
  conservé sur push main. Les autres jobs n'ont pas ces permissions d'écriture.
- Caches Gradle, pip et BuildKit (`type=gha`, un scope par service).
- `concurrency` regroupe les exécutions par workflow et PR/branche ;
  `cancel-in-progress: true` annule les exécutions obsolètes.
- Connexion GHCR et publication uniquement si l'événement est `push`
  **et** la référence `refs/heads/main`. Sur PR, les images sont construites
  sans publication ; le cache n'est pas exporté vers un registre d'images.
- Aucun filtre `paths` ni filtre de branche sur `pull_request` dans les trois
  workflows du TP : leurs sept contrôles existent aussi pour une PR documentaire.
  Avec un filtre `paths`, un workflow requis peut ne jamais démarrer et son
  contrôle rester attendu, bloquant une PR sans rapport avec le service.

Le workflow `ci-dependencies.yml`, déjà présent sur `main`, est conservé.
Son contrôle supplémentaire ne remplace aucun des sept contrôles du TP.

## 4. Produire des preuves

Les téléversements utilisent `if: always()` et une rétention de **7 jours** :

| Artefact | Contenu |
| --- | --- |
| `java-test-results` | XML JUnit et rapport HTML des tests Gradle |
| `java-checkstyle` | Rapports Checkstyle main/test XML et HTML |
| `pricing-test-results` | `build/test-results/pytest.xml` |

Deuxième expérience : `f6dac2b` attend volontairement `201.0` au lieu de `200.0`
dans `test_normalizes_the_category`. `2ee3608` rétablit ensuite l'assertion.

[Exécution 44328125612](https://github.com/sarusman/rental/actions/runs/44328125612) :
`Python quality` réussit ; `Python tests` échoue après 18 secondes, avec
`tests/test_pricing.py:18: AssertionError: assert 200.0 == 201.0`.
L'étape `Upload Pytest report` réussit malgré cet échec et `Python Docker image`
est ignoré, ce qui vérifie la dépendance bloquante.

L'artefact `pricing-test-results` a réellement été téléchargé avant le
rétablissement : [rapport XML](evidence/tp01/artifact-failure/pytest.xml),
[exécution](evidence/tp01/artifact-failure/run.json),
[métadonnées de l'artefact](evidence/tp01/artifact-failure/artifacts.json).
Le XML indique 9 tests, 1 échec, 0 erreur.

## 5. Détecter les secrets

Le workflow dédié est `.github/workflows/ci-security.yml`. L'ancien fichier
`secret.yml` a été regroupé dans ce fichier afin d'avoir un seul contrôle
`Secret detection`. Checkout utilise `fetch-depth: 0` ; Gitleaks `v8.24.2`
analyse l'historique atteignable depuis `HEAD` avec `git --log-opts=HEAD`.
`--redact` masque les valeurs détectées dans les logs.

Le scan sans secret ajouté a réussi :
[exécution 44328125267](https://github.com/sarusman/rental/actions/runs/44328125267).

**Expérience avec valeur factice non réalisée :** ni la valeur fournie par
l'enseignant ni ses instructions de suppression ne figurent dans la consigne
transmise. Aucune valeur n'a été inventée et aucune branche de démonstration
contenant un secret n'a été créée. Cette expérience reste à faire avec ces éléments.

- **Supprimer la ligne** retire la valeur du fichier courant, mais la laisse
  dans les commits précédents : Gitleaks sur l'historique peut encore la détecter.
- **Révoquer un secret exposé** le rend inutilisable chez son fournisseur ;
  c'est nécessaire pour un vrai secret, même après suppression du fichier.
- **Réécrire l'historique** retire la valeur des commits concernés et change
  leurs identifiants. Cela nécessite une coordination des copies et des branches
  et ne révoque pas le secret. Supprimer une branche ne garantit pas l'effacement
  de toutes les copies, des références de PR ni des journaux existants.

## 6. Reproduire en local

Avec les dépendances de `apps/pricing-service/requirements-dev.txt` installées
dans l'interpréteur actif, depuis la racine :

```bash
python scripts/course.py lint-java
python scripts/course.py lint-python
python scripts/course.py lint
python scripts/course.py verify
```

`verify` lance successivement tests Java, tests Python, Checkstyle et Ruff
(lint puis formatage). Le wrapper Gradle est choisi selon le système ; les
commandes Python utilisent `sys.executable`.

La reproduction locale a utilisé le `.venv` déjà présent dans le dépôt
(Python 3.14.3) et Java 21 ; la CI utilise Python 3.12 et Java 17.
Les tests couvrent 34 tests Java et 9 tests Python.
Après rétablissement, `verify` réussit avec le code **0** :
[trace de validation](evidence/tp01/local-success.txt).

Le comportement d'arrêt a aussi été testé avec l'assertion fautive de l'étape 4 :
`verify` renvoie **1**, après les tests Python, et ne lance ni Checkstyle ni Ruff.
[Trace réelle](evidence/tp01/local-failure.txt).

`verify` ne construit pas les images, ne lance pas Gitleaks, ne téléverse pas
les rapports et ne vérifie pas la protection de branche.

Validation distante du code rétabli, commit `2ee3608` :

| Exécution | Contrôles | Résultat |
| --- | --- | --- |
| [44328326308](https://github.com/sarusman/rental/actions/runs/44328326308) | Java tests, Java quality, Java Docker image | Tous réussis |
| [44328326263](https://github.com/sarusman/rental/actions/runs/44328326263) | Python tests, Python quality, Python Docker image | Tous réussis |
| [44328326273](https://github.com/sarusman/rental/actions/runs/44328326273) | Secret detection | Réussi |
| [44328326529](https://github.com/sarusman/rental/actions/runs/44328326529) | Dependency review, contrôle conservé depuis main | Réussi |

Les artefacts `java-test-results`, `java-checkstyle` et `pricing-test-results`
ont été téléchargés. Leurs XML ne contiennent aucun échec ni erreur.
Les [preuves de succès](evidence/tp01/success/) conservent les XML,
les métadonnées d'artefacts et les liens vers chaque contrôle.

## 7. Protéger main

La consigne dit **« Après validation par l'enseignant »**. Cette validation
n'a pas été fournie : la protection n'a donc pas été modifiée.
Le réglage attendu est préparé dans
[tp01-main-protection.json](tp01-main-protection.json), avec ces sept noms exacts :

- `Java tests`, `Java quality`, `Java Docker image` ;
- `Python tests`, `Python quality`, `Python Docker image` ;
- `Secret detection`.

Après validation, appliquer les contrôles requis à `main`, puis constater
qu'une PR non brouillon rouge est bloquée et qu'une PR verte est intégrable.
Ce test de protection **n'est pas revendiqué comme réalisé**. La PR du TP
reste en brouillon pour la revue ; aucune fusion n'a été faite.
