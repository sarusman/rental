TP 02 — Chaîne d’approvisionnement

Risques et contrôles :

- Action GitHub : utiliser un commit précis pour éviter l’exécution d’un code modifié.
- Dependabot : gérer les mises à jour des dépendances.
- Image de base : utiliser un digest SHA-256 pour garantir son intégrité.
- Image publiée : utiliser le digest Buildx plutôt que le tag latest.
- Revue des dépendances : détecter les dépendances vulnérables dans les pull requests.
- Trivy : détecter les vulnérabilités dans l’image construite.

Preuves après publication sur main :

- Digest Java et Python : sortie Buildx.
- Identité du certificat : vérification Cosign.
- Dépôt et commit : attestation de provenance.
- Paquets et bibliothèques : SBOM.
- Rapport Trivy : lien vers le rapport SARIF.

Les preuves sont à récupérer après l’exécution des workflows.