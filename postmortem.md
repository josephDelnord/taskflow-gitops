# Postmortem — La 2.1.0 renvoie 30 % d'erreurs et est avortée automatiquement

> Sans reproche : on cherche ce qui a permis l'erreur, pas qui l'a faite.

| Champ | Valeur |
| --- | --- |
| Date et heure | 15/10/2026, à 10h45 |
| Version en cause | taskflow:2.1.0 (révision 8) |
| PR à l'origine | PR "feat: deploy TaskFlow 2.1.0- #31" |
| Durée d'exposition | Environ 1 minute (du démarrage du pod canary à son arrêt) |
| Part du trafic touché | Jusqu'à 25 % (1 pod sur 4) |
| Détecté par | Test de charge k6 (AnalysisRun) |
| Résolu par | Git revert de la PR 2.1.0 |

## Chronologie

| Heure | Événement |
| --- | --- |
| ~10:44 | Merge de la PR 2.1.0, le rollout passe à la révision 8 et crée 1 pod canary |
| ~10:44 | Étape 1/7 : setWeight 25, la 2.1.0 reçoit une part du trafic |
| 10:45:02 | Après la pause de 20 s (étape 2/7), l'AnalysisRun lance le test k6 |
| 10:45:39 | k6 échoue (p95 = 313 ms, 30,16 % d'erreurs) |
| 10:45:39 | Abort automatique : la révision 8 ne passe pas, la 2.0.0 repasse de 3 à 4 pods |
| 11:10:00 | Revert de la PR 2.1.0, Git repasse à 2.0.0 |

## Composant défaillant et cause racine

- Composant défaillant : l'application TaskFlow en 2.1.0.
  Preuve : les logs k6 donnent `http_req_failed` à 30,16 % (seuil < 2 %) et p95 = 313 ms (seuil < 250 ms).
  `kubectl argo rollouts get rollout taskflow -n taskflow` affiche `RolloutAborted ... failed (1) > failureLimit (0)`.
- Pourquoi les probes Kubernetes ne l'ont-elles pas vu ? La probe de readiness interroge seulement `/health` (port 8000).
  Dans les événements, son seul échec est un `connection refused` au démarrage du pod, après quoi le pod a été considéré prêt et a reçu du trafic.
  `/health` vérifie que le processus répond, pas que les vraies requêtes réussissent.
- Cause racine : l'image 2.1.0 a été construite avec FAILURE_RATE=0.3 et LATENCY_MS=300
  (la 2.0.0 : 0 et 0), ce qui fait échouer environ 30 % des requêtes et ajoute 300 ms de latence.

## Ce qui a bien fonctionné

- L'analyse automatique a détecté le problème en moins d'une minute.
- L'abort s'est fait sans intervention humaine, et la 2.0.0 est restée stable.
- L'exposition a été limitée à un palier de 25 % grâce au canary.

## Actions correctives

| Action | Responsable | Échéance |
| --- | --- | --- |
| Reconstruire l'image avec FAILURE_RATE=0 et LATENCY_MS=0, publier la 2.2.0 | Yani & Joseph | Aujourd'hui |
| Déployer la 2.2.0 par PR jusqu'à 100 % | Yani & Joseph | Aujourd'hui |
| Reverter la PR 2.1.0 (aligner Git sur le cluster) | Yani & Joseph | Aujourd'hui |
| Ajouter un test de charge en CI avant publication d'une image | Yani & Joseph | Prochaine release |
| Faire tester `/` par la probe de readiness, pas seulement `/health` | Yani & Joseph | Prochaine release |
