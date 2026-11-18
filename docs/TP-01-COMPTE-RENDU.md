# TP 01 — une CI observable et bloquante

Dépôt : `sarusman/rental`. [PR #1](https://github.com/sarusman/rental/pull/1) :
**base `tp/01-ci-base` ← branche de travail `tp/01-ci`**.
`tp/01-ci-base` pointe sur `checkpoint/00-starter` (`ce7e9cf`).
La branche du TP ne contient aucune fusion de `main` ni les ajouts du TP 02.
Les rapports restent téléchargeables dans Actions ; ils ne sont pas versionnés.

## 0. Préparation et lecture du starter

La branche part du starter et conserve les commits pédagogiques. Les coûts
ci-dessous sont des estimations relatives, pas des mesures.

| Job initial | Risque couvert | Entrées | Sortie | Coût estimé |
| --- | --- | --- | --- | --- |
| Java test | Régression Java | Sources, tests, Gradle, JDK | JUnit et statut | Moyen |
| Java code-quality | Build impossible ou secret | Sources et historique Git | Statut Gradle/Gitleaks | Moyen |
| Java build-and-push | Image non constructible | Contexte Docker et jobs précédents | Image, publication sur push main | Élevé |
| Java pipeline-status | Échec d'un job précédent | États des jobs | Statut synthétique | Faible |
| Python test | Régression API/métier | Sources, tests, dépendances pip | XML Pytest | Faible à moyen |
| Python build-and-push | Image non constructible | Contexte Docker et tests | Image | Élevé |

## 1. Observer un échec

`472ded9` ajoute une assertion volontairement fausse ; `09e6e14` la rétablit.
La PR avait été ouverte en brouillon. La première expérience a été exécutée
avant correction de sa cible, alors qu'elle visait encore `main` :
[exécution Python 44327492178](https://github.com/sarusman/rental/actions/runs/44327492178).

- Premier échec global : `Dependency review`, après 9 s, car le graphe de
  dépendances du fork était désactivé. Ce job venait du `main` avancé.
- Premier échec applicatif : `Python quality`, après 15 s, avec `B011` sur
  `tests/test_pricing.py:16:12` (`assert False`). Cette règle venait de `main`.
- `Python tests` échoue après 22 s : `tests/test_pricing.py:16: AssertionError`.
  Résultat : 1 échec, 8 succès. Les délais partent de la création de l'exécution.
- L'artefact `python-test-reports` reste disponible et a été téléchargé.

La cible de la PR a ensuite été corrigée vers `tp/01-ci-base` et la fusion
erronée de `main` retirée de l'historique de travail.

## 2. Portes qualité

Checkstyle 9.3 couvre `UnusedImports`, `NeedBraces`, `EmptyStatement` et
`LineLength` (120 caractères). Ses rapports XML et HTML sont activés.
Ruff 0.11.13 active `E`, `F`, `I`, puis `ruff format --check app tests` vérifie
le formatage. Aucune règle n'est désactivée pour contourner un échec.

## 3. Graphe CI

Chaque service a trois jobs : tests et qualité en parallèle, puis image avec
`needs: [tests, quality]`. Les noms sont `Java tests`, `Java quality`,
`Java Docker image`, `Python tests`, `Python quality`, `Python Docker image`.

- Timeouts : Java tests/qualité 15 min, Python tests/qualité 10 min,
  images 25 min, secrets 10 min.
- `contents: read` au niveau des workflows ; les jobs image ajoutent
  `packages: write` pour GHCR et `security-events: write` pour Trivy sur main.
- Caches Gradle, pip et BuildKit ; concurrence par workflow et PR/branche,
  avec annulation des exécutions obsolètes.
- Connexion GHCR et publication seulement sur événement `push` vers `main`.
  Les PR construisent les images sans les publier.
- Aucun filtre `paths` sur `pull_request` : les sept contrôles existent aussi
  pour une PR documentaire. Sinon, un contrôle requis peut rester attendu
  indéfiniment lorsqu'un filtre empêche son workflow de démarrer.

## 4. Rapports disponibles en échec

`actions/upload-artifact` utilise `if: always()` et une rétention de 7 jours :
`java-test-results` (JUnit/HTML), `java-checkstyle` (XML/HTML),
`pricing-test-results` (XML Pytest).

Deuxième expérience : `dcb841a` attend `201.0` au lieu de `200.0` ; `ac7897c`
rétablit l'assertion. Ces commits conservent l'expérience initialement publiée
sous `f6dac2b` et `2ee3608`, avant correction de l'historique.
[Exécution 44328125612](https://github.com/sarusman/rental/actions/runs/44328125612) :
qualité verte, tests rouges après 18 s, diagnostic `assert 200.0 == 201.0`
à `tests/test_pricing.py:18`, image ignorée. L'artefact a été téléchargé
avant rétablissement : 9 tests, 1 échec, 0 erreur.

## 5. Détection des secrets

`.github/workflows/secret.yml` fournit `Secret detection` : checkout complet
(`fetch-depth: 0`) et Gitleaks 8.24.2 sur l'historique de `HEAD`, avec `--redact`.

Avec les choix par défaut autorisés par le propriétaire, une valeur entièrement
synthétique a été utilisée dans la [PR jetable #2](https://github.com/sarusman/rental/pull/2).
[Gitleaks détecte une valeur et retourne 1](https://github.com/sarusman/rental/actions/runs/44330157369).
[Après suppression du fichier, il échoue encore](https://github.com/sarusman/rental/actions/runs/44330298407),
car le commit précédent reste analysé. La PR est fermée sans fusion et la branche
`tp/01-ci-secret-demo` est supprimée localement et sur GitHub.

Supprimer une ligne ne nettoie pas les commits précédents. Révoquer un vrai
secret le rend inutilisable chez son fournisseur. Réécrire l'historique retire
la valeur des commits concernés, change leurs identifiants et impose une
coordination des copies ; cela ne remplace pas la révocation.

## 6. Reproduction locale

Après installation de `apps/pricing-service/requirements-dev.txt` dans
l'interpréteur actif, lancer `python scripts/course.py verify` depuis la racine.
Ordre : tests Java, tests Python, Checkstyle, Ruff lint, Ruff format.
Résultat validé : 34 tests Java, 9 Python, qualité verte. Avec l'assertion fautive,
le script retourne 1 après Pytest et ne lance pas la qualité.
Le wrapper Gradle dépend du système ; Python utilise `sys.executable`.
`verify` ne construit pas les images et ne remplace ni Gitleaks, ni les artefacts
Actions, ni la protection de branche.

## 7. Protection et cible d'intégration

Sur autorisation du propriétaire, `main` est protégée par les sept contrôles
du TP, avec branche à jour exigée et administrateurs inclus. Lors de la
vérification initiale contre `main`, la PR #2 était `BLOCKED` avec Gitleaks
seul en échec ; la PR #1 verte était `CLEAN` et `MERGEABLE`.
Aucune PR n'a été fusionnée.

La **cible d'intégration du TP est désormais `tp/01-ci-base`**, conformément
à la correction demandée. `main` conserve son contenu initial.
