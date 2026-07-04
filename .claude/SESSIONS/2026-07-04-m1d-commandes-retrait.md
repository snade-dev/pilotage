# Backend M1-d — Commandes de retrait (click & collect) (2026-07-04)

Repo : `all-in-one-backend`. Branche : `feat/m1d-commandes-retrait`.
**Non commité** (attente relecture — `git status` montré au propriétaire).
Conception validée en Étape A ("utilise commande"), puis implémentée (Étape B).
Design complet : `.claude/DESIGN/2026-07-04-m1d-commandes-retrait.md`.

## Périmètre livré
- **Modèle** : enum `StatutRetrait` + champs `Commande` (`statutRetrait`, `creneauRetrait`,
  `motifRefus`) + 3 index. Migration `20260704120000_add_commande_retrait` (additive,
  appliquée en local sur la base de dev).
- **Service profond** `common/commande-retrait/base-commande-retrait.service.ts` (patron
  Scope SM+resto) : création (snapshots + total + idempotence `clientRequestId`), transitions,
  transaction atomique de retrait (seam stock `TOLERATE_NEGATIVE` + `ECART_STOCK_RETRAIT` +
  bascule `PAYE`), isolation stricte, résolution TVA + visibilité publique.
- **Endpoints** : client `/v1/commandes-retrait` (`@ClientOnly` + `@CurrentClientId`) ;
  commerçant `/supermarche|restaurant/commande-retrait` (file kanban + accepter/prete/
  retirer/refuser, RBAC `MANAGE_ORDERS`). 3 modules câblés dans `app.module`.
- **Décorateur** `@CurrentClientId` = point de branchement unique de l'auth client (M1-c).

## Réutilisation (aucune duplication)
Une commande de retrait EST une `Commande` (`sourceCommande=APPLICATION`), non payée à la
création, qui devient une vente (`statutPaiement=PAYE`) au retrait ⇒ entre dans les stats via
le pipeline existant. Seams réutilisés : StockMovement, AuditJournal, EstablishmentAccess,
ShortId, idempotence `clientRequestId`.

## Décisions clés
- **Stock au retrait**, pas à la commande. Retrait = transaction atomique tout-ou-rien.
- **Écart au retrait** : négatif toléré + journalisé (comme l'offline) — exception assumée
  au « strict en ligne » (au comptoir, la marchandise part ; on trace, on ne bloque pas).
- Cycle retrait dans un champ dédié `statutRetrait`, séparé de `CommandeStatut` (livraison).

## Exclusions volontaires M1-d
Menus-vente en ligne de commande, options tarifées de plats, bucket stats sur `datePaiement`
(reste `createdAt`). Documentés dans le design.

## Tests — verts
- Unit : `commande-retrait.util.spec.ts` (transitions + total). Suite globale **278/278**.
- e2e-BD : `commande-retrait.db-e2e-spec.ts` **5/5** (cycle nominal SM stock+stats ; écart
  négatif toléré ; scope resto ; isolation commerçant/client ; atomicité). Suite **41/41**.
- `tsc --noEmit` : 0 erreur. Lint propre sur les fichiers `src` ajoutés.

## Reste / suite
- Relecture + commit (attente propriétaire). Déployer la migration en prod le moment venu.
- App cliente : écrans commande/suivi À CONSTRUIRE APRÈS ces endpoints (piège ETAT respecté).
- Extensions futures : menus en ligne de commande, options tarifées, bucket stats `datePaiement`.
