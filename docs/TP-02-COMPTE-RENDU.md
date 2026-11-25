# TP 02 — Chaîne d’approvisionnement

## Risques et contrôles

| Élément | Substitution possible et impact | Contrôle ou preuve |
| --- | --- | --- |
| Action GitHub | Un tag d’action déplacé peut exécuter un autre code et exposer les permissions du job. | Les actions distantes sont épinglées à un commit complet. Dependabot propose les changements dans une pull request. |
| Image de base | Un tag déplacé peut changer le système ou les paquets embarqués dans l’image construite. | Chaque image de base conserve son tag lisible et utilise un digest SHA-256. |
| Image publiée | `latest` peut pointer vers une autre image après l’analyse. | Buildx fournit un digest utilisé pour le SBOM, la signature, les attestations, la vérification et Trivy. |

La revue des dépendances analyse les changements de manifestes d’une pull request et peut bloquer l’ajout ou la mise à jour d’une dépendance vulnérable de gravité élevée. Trivy analyse les composants réellement présents dans l’image construite. Ces contrôles couvrent donc des étapes différentes.

## Preuves après publication sur `main`

Ces valeurs doivent être relevées dans une exécution réussie des workflows, sans les déduire du tag `latest`.

| Preuve | Valeur |
| --- | --- |
| Digest Java vérifié | À relever dans la sortie Buildx |
| Digest Python vérifié | À relever dans la sortie Buildx |
| Identité du certificat Cosign | À relever dans la sortie de vérification |
| Dépôt et commit de provenance | À relever dans l’attestation de provenance |
| Paquet système et bibliothèque du SBOM | À relever dans un SBOM téléchargé |
| Rapport Trivy | Lien vers l’analyse SARIF de l’exécution |
