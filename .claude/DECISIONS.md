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

## 2026-07-02 — POS offline : file d'opérations idempotentes + stock négatif toléré
**Choix :** synchronisation offline du POS par **file d'opérations idempotentes** (UUID
généré côté caisse = `opId`, mappé sur `Commande.clientRequestId @unique` pour la vente),
rejeu à la reconnexion sans double effet. **Conflit de stock : on accepte toutes les ventes**
+ alerte d'écart ; stock négatif **toléré à la réconciliation**, journalisé via
`AuditJournalService.record` (`ECART_STOCK_OFFLINE`). **En ligne = strict** (rejet sous zéro).
**Décisions cadre (2026-07-02) :** supermarché seul en O0 ; **remboursement offline
INTERDIT** (⇒ vente = seul type offline, aucune table nouvelle) ; **seam stock réécrit en
écritures atomiques** (`increment` en mode toléré, `updateMany` conditionnel en strict —
corrige un lost-update latent aussi en ligne) ; session ouverte EN LIGNE avant bascule ; pas
d'expiration temporelle des ops ; numérotation reçus sans objet (`Commande.id` = UUID).
**Pourquoi :** plusieurs caisses vendent en simultané hors-ligne sur un stock partagé ;
refuser une vente au rejeu = perdre de l'argent déjà encaissé. On préfère accepter + tracer
l'écart et régulariser par inventaire.
**Alternatives écartées :** rejet des ventes qui passent sous zéro (perte de CA réel) ;
verrou applicatif / `SELECT FOR UPDATE` (inutile : la base sérialise des deltas atomiques).
**Jalons :** O1 file locale + rejeu unitaire ordonné (fait) → O2 tolérance négative + écarts
→ O3 multi-caisses durci. Design : `.claude/DESIGN/2026-07-02-pos-offline-sync.md`.

## 2026-07-01 — Suivi inter-sessions via `.claude/`
**Choix :** la source de vérité du projet vit dans git, pas dans une session Claude.
**Pourquoi :** les sessions n'ont pas de mémoire fiable ; le git persiste.
**Alternatives écartées :** mémoire de session ou notes externes (divergent du code).
