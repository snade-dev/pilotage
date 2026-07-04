# Design — M1-d : commandes de retrait au comptoir (click & collect)

> **Statut : VALIDÉ + IMPLÉMENTÉ** (2026-07-04). Conception approuvée par le
> propriétaire ("utilise commande"), puis implémentée sur la branche backend
> `feat/m1d-commandes-retrait` (non commitée à ce jour — attente relecture).

Repo : `all-in-one-backend` (NestJS + Prisma + PostgreSQL).

## Décisions actées
- Commande possible UNIQUEMENT par un client authentifié (`clientId`).
- Retrait au comptoir uniquement, **paiement au comptoir** (rien encaissé en ligne).
- Couvre supermarché (produits) ET restaurant (plats) via le patron **Scope**.
- Statuts : `RECUE → EN_PREPARATION → PRETE → RETIREE` (+ `REFUSEE`/`ANNULEE` terminaux).
- **Stock décrémenté AU RETRAIT** (pas à la commande) — évite de bloquer du stock.
- Au retrait, stock insuffisant = **même logique qu'offline** : accepté, négatif toléré,
  écart journalisé (`ECART_STOCK_RETRAIT`) via `AuditJournalService`. Jamais de blocage.
- Le retrait alimente les **statistiques comme une vente**.

## Choix structurant : réutiliser `Commande` (pas de table parallèle)
Faits vérifiés dans le code :
1. Une « vente » = une `Commande` avec `statutPaiement = PAYE` (le POS ne fait que
   marquer PAYE ; il n'avance pas `CommandeStatut`).
2. Les stats supermarché/restaurant ne comptent QUE `statutPaiement = PAYE`.
3. Les seams existent déjà : `StockMovementService.apply` (avec `TOLERATE_NEGATIVE`
   + `ecartNegatif`), `AuditJournalService.record`, `EstablishmentAccessService`,
   `ShortIdService`, idempotence `Commande.clientRequestId @unique`.

⇒ Une commande de retrait EST une `Commande` (`sourceCommande = APPLICATION`), non payée
à la création. Au retrait elle devient une vraie vente (`PAYE`) et entre dans les stats via
le pipeline existant — **zéro refactor stats, zéro duplication de lignes**.

Le cycle de vie retrait vit dans un **champ dédié** `statutRetrait` (enum `StatutRetrait`),
distinct de `CommandeStatut` (vocabulaire livraison) pour ne pas polluer la « répartition
par statut ». `statutRetrait` NON NULL ⇔ commande de retrait.

| Étape | `statutRetrait` | `statutPaiement` | stats ? | stock |
|---|---|---|---|---|
| Création | RECUE | EN_ATTENTE | non | intact |
| Accepter | EN_PREPARATION | EN_ATTENTE | non | intact |
| Prête | PRETE | EN_ATTENTE | non | intact |
| **Retirée** | RETIREE | **PAYE** | **oui** | **décrémenté** |
| Refusée/Annulée | REFUSEE/ANNULEE | EN_ATTENTE | non | intact |

## Modèle (migration `20260704120000_add_commande_retrait`, additive)
- Enum `StatutRetrait { RECUE, EN_PREPARATION, PRETE, RETIREE, REFUSEE, ANNULEE }`.
- `Commande` : `statutRetrait StatutRetrait?`, `creneauRetrait DateTime?`, `motifRefus String?`.
- Index : `(supermarcheId, statutRetrait, creneauRetrait)`, idem restaurant, `(clientId, statutRetrait)`.
- **Code de retrait** = dérivé de `Commande.id` via `ShortIdService` (aucune colonne).

## Endpoints
**Client** (`@ClientOnly`, `clientId` via `@CurrentClientId` — point de branchement M1-c) :
- `POST /v1/commandes-retrait` (créer, idempotent `clientRequestId`)
- `GET /v1/commandes-retrait` (ses commandes) · `GET /v1/commandes-retrait/:id` (détail+suivi)
- `PATCH /v1/commandes-retrait/:id/annuler` (tant que < PRETE)

**Commerçant** (`@StaffOnly` + RBAC `MANAGE_ORDERS`, scope SM/resto ; établissement = JWT) :
- `GET .../commande-retrait/file` (kanban), `GET .../:id`
- `PATCH .../:id/accepter | /prete | /retirer | /refuser`
  sur `/supermarche/commande-retrait` ET `/restaurant/commande-retrait`.

## Transition RETIREE (cœur, atomique)
Dans UN `prisma.$transaction` : (1) verrou d'état (double-clic ⇒ 409) ; (2) décrément par
ligne via le seam en `TOLERATE_NEGATIVE` ; (3) si `ecartNegatif`, `ECART_STOCK_RETRAIT`
journalisé dans la même tx ; (4) bascule `statutPaiement=PAYE` + `datePaiement` + `montantPaye`.
Tout ou rien. **Exception assumée au « strict en ligne »** : au comptoir, la marchandise part,
on n'échoue pas pour rupture — on trace l'écart, régularisable par inventaire.

## Scope (SM + resto sans duplication)
Service unique `BaseCommandeRetraitService` paramétré par `CommandeRetraitScope`
(`SUPERMARCHE_RETRAIT_SCOPE` / `RESTAURANT_RETRAIT_SCOPE`). Contrôleurs client (agnostique,
résout le type par l'`etablissementId` + visibilité publique) + 2 contrôleurs commerçant fins.

## Prix / TVA
Le client voit `prixOnline` (prix item, TTC). À la création on fige le snapshot ligne
(`prixUnitaireSnapshot = prixOnline`), on déduit HT/TVA via le taux de l'établissement
(`/…/taux-tva`, sinon global, sinon exonéré 0 %). `montantPaye` = 0 jusqu'au retrait, puis TTC.

## Auth client (M1-c)
`UserType.CLIENT` + JWT client existent déjà. `@CurrentClientId` isole le point unique
d'extraction du `clientId` ; M1-c (vérif téléphone, fidélité) ne changera que le JWT/garde,
pas la signature `(clientId, …)` des services.

## Périmètre volontairement exclu de M1-d
- **Menus-vente** en ligne de commande (le catalogue public M1-b expose produits/plats ;
  les lignes menu décrémenteraient la composition — reporté). Les lignes `menuVenteId` sont
  ignorées au décrément (aucune n'existe en M1-d).
- **Options tarifées** de plats (non exposées publiquement).
- Bucket temporel des stats = `createdAt` (retrait quasi toujours le jour même) — imprécision
  « commande jour N / retrait jour N+1 » notée, à revoir si besoin.

## Tests (verts)
- Unit (`commande-retrait.util.spec.ts`) : transitions valides/invalides + terminaux ; calcul total.
- e2e-BD (`commande-retrait.db-e2e-spec.ts`, 5/5) : cycle nominal SM (retrait décrémente +
  alimente les stats, rien avant) ; écart (négatif toléré + `ECART_STOCK_RETRAIT`) ; scope
  resto ; isolation (commerçant/ client) ; atomicité (échec ⇒ 0 écriture partielle).
- Suites globales : **278/278 unit**, **41/41 e2e-BD**, `tsc` 0 erreur.
