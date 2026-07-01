# Journal des décisions

> Le « pourquoi » des choix structurants, pour ne pas les re-débattre.

## 2026-07-01 — Repo de pilotage centralisé (Option B)
**Choix :** créer un dépôt dédié `all-in-one-pilotage` contenant uniquement `.claude/`,
source de vérité unique du suivi pour les deux repos de code.
**Pourquoi :** le projet est sur deux repos ; un `.claude/` par repo fait diverger l'état.
Le déclencheur : passage à deux chantiers menés en parallèle (front stats + back Phase 4).
**Alternatives écartées :** un `.claude/` par repo (Option A) — état éclaté, charge mentale
de savoir lequel dit quoi.
**À faire en conséquence :** retirer/geler l'ancien `.claude/` du repo backend pour éviter
deux sources de vérité concurrentes (voir ETAT.md).

## 2026-07-01 — Sécurité « sûre par défaut » (Phase 1)
**Choix :** auth et permissions globales (APP_GUARD) + décorateur `@Public()` pour les rares
routes ouvertes.
**Pourquoi :** le modèle « opt-in » laissait la majorité des controllers sans garde granulaire.
**Alternatives écartées :** guard controller par controller (trop d'occasions d'oubli).

## 2026-07-01 — Statistiques : agrégation à la volée + couche isolée (V1)
**Choix :** requêtes à la volée bien indexées, logique isolée dans un StatistiquesService
basculable en pré-agrégé plus tard, sans changer le contrat d'API.
**Pourquoi :** le pré-agrégé introduit un risque de totaux faux après remboursement ; on ne
paie cette complexité que si la lenteur est mesurée. CA toujours NET des remboursements.
**Alternatives écartées :** pré-agrégat dès la V1 (complexité + bugs d'invalidation prématurés).

## 2026-07-01 — Suivi inter-sessions via `.claude/`
**Choix :** la source de vérité du projet vit dans git, pas dans une session Claude.
**Pourquoi :** les sessions n'ont pas de mémoire fiable ; le git persiste.
**Alternatives écartées :** mémoire de session ou notes externes (divergent du code).
