# Phase 6 — Mode offline robuste du POS : conception (jalon O0)

Date : 2026-07-02
Repos concernés : `All-in-One-Partner-Hub` (front) + `all-in-one-backend` (back).
Livrable : **document de conception uniquement**. AUCUN code, AUCUNE migration.

## Objectif
Fixer le modèle de synchronisation offline AVANT toute implémentation, dans le cadre des
décisions déjà arrêtées : file d'opérations idempotentes (UUID caisse), stock négatif toléré
à la réconciliation + alerte d'écart via le seam d'audit, chemin en ligne pouvant rester
strict.

## Ce qui a été fait
1. Lu `ETAT.md`, `ROADMAP.md`, `DECISIONS.md` (pas encore d'entrée « POS offline » dans
   DECISIONS → décisions prises depuis le prompt, à formaliser à la validation).
2. Audité l'existant pour ancrer la conception (réutilisation, pas réinvention) :
   - Front : `pos/lib/offline-pos-store.ts` (IndexedDB `all-in-one-pos`, stores
     catalog/sales/clients, `QueuedSale`), `pos/hooks/useSalesSync.tsx` (rejeu série, verrou
     anti-course, backoff, `MAX_SYNC_RETRIES`, interruption 401/403).
   - Back : idempotence `Commande.clientRequestId @unique` + court-circuit + `catch P2002`
     dans `PosSessionService.effectuerVente` ; seam `StockMovementService.apply(tx, …)`
     (rejette `stockApres < 0`) ; seam `AuditJournalService.record(tx, …)` → `historique` ;
     atomicité Phase 4 (`prisma.$transaction`) ; caisse = `PointDeVente`.
3. Rédigé le design doc : `.claude/DESIGN/2026-07-02-pos-offline-sync.md`.

## Décisions de conception clés (proposées)
- **`opId` (UUID v4 caisse) = clé d'idempotence globale** ; pour la vente, mappé sur le
  `clientRequestId @unique` existant → zéro nouvelle infra pour le cas vente.
- **Généraliser la file « ventes » en file « opérations »** côté IndexedDB (bump DB_VERSION,
  ajout `opId`/`deviceId`/`seq`/`type`).
- **Seam stock : ajouter `negativeStockPolicy` (défaut STRICT)** ; le rejeu offline passe
  `TOLERATE_NEGATIVE` → stock négatif appliqué + `ECART_STOCK_OFFLINE` journalisé via le
  seam audit, dans la même transaction. **Le chemin EN LIGNE reste STRICT** (aucune
  régression).
- **Multi-caisses** : chaque caisse a sa file locale, rejeu séquentiel par caisse ; deltas
  additifs (jamais de « set »), transactions sérialisées par la base → pas d'écrasement ni
  de collision (`opId` distincts).
- **Jalons** : O1 file+rejeu unitaire (0 backend, mais stock encore strict = incomplet vs la
  décision) → O2 tolérance négative + écarts + (option) batch/`OperationIdempotente` → O3
  multi-caisses durci + e2e concurrents.

## Garde-fous rappelés
- Ne pas casser l'atomicité Phase 4 (une transaction par vente, pas de transaction géante).
- Isolation multi-tenant : `supermarcheId` depuis le JWT, pas le payload ; piège Prisma 5.7
  (`where` sur relation to-one dans `include` ignoré).
- Journal immuable : on ajoute, on ne modifie pas.

## Questions tranchées (2026-07-02 — « choisir le plus intelligent »)
Les 8 questions ouvertes ont toutes été tranchées (§8 du doc) :
- **Q1** Périmètre O0 = **supermarché seul** (resto plus tard).
- **Q2** Remboursement offline **INTERDIT** → `VENTE` seul type offline, aucune table
  d'idempotence, front grise le remboursement en `isOffline`.
- **Q3** Constat vérifié : `StockMovementService.apply` = **read-then-write** (lost-update
  possible). Décision : écritures **atomiques** — `increment` en `TOLERATE_NEGATIVE`,
  `updateMany` **conditionnel** (`where stockActuel gte -delta` + throw si `count===0`) en
  `STRICT`. Réalisé en O2, prouvé en O3. Corrige aussi un lost-update latent du chemin en ligne.
- **Q4** **Sans objet** : `Commande.id` = UUID, pas de séquence de numéro de reçu.
- **Q5** Rejeu **unitaire en O1** (0 backend), **batch en O2**.
- **Q6** **Pas d'expiration** temporelle ; terminal (`REJECTED`) seulement sur cause métier
  définitive (session clôturée, entité supprimée).
- **Q7** Session **ouverte en ligne** avant bascule ; pas d'ouverture/clôture offline en O0.
- **Q8** Après sync : recharge catalogue (`onSynced`) **+ écran d'écarts** listant les
  variantes négatives pour régularisation.

Conséquence : **aucune nouvelle table en O0** ; la seule évolution backend structurante est
la réécriture du seam stock en écritures atomiques.

## O1 — réalisé (front, 2026-07-02, NON commité)
Décision cadre actée dans `DECISIONS.md` (2026-07-02) après feu vert propriétaire.

**Constat déclencheur** : `localId === clientRequestId` était déjà câblé (page.tsx:527) →
l'idempotence par UUID existait déjà. Le vrai trou : `listQueuedSales` itère par clé
(`localId` = UUID **aléatoire**) → les ventes offline étaient rejouées dans un ordre
**quelconque**, pas chronologique.

**Fait** (repo `All-in-One-Partner-Hub`, `src/app/(app)/pos/`) :
- `lib/offline-pos-store.ts` : champ `seq` (rang monotone par caisse, calculé DANS la
  transaction d'écriture de `enqueueSale`) ; helpers purs `compareSalesForReplay` +
  `sortSalesForReplay` (tri `createdAt` puis `seq`, legacy sans `seq` → fallback `createdAt`).
- `hooks/useSalesSync.tsx` : la file est triée par `sortSalesForReplay` avant rejeu → les
  ventes plus anciennes partent d'abord (décréments de stock cohérents avec la chronologie).
- `lib/offline-pos-store.spec.ts` : 5 tests Vitest (ordre chronologique, départage `seq`,
  legacy, non-mutation, cohérence du comparateur). **28/28 suite verte, `tsc --noEmit` propre.**
- **Zéro changement backend** (rejeu sur `POST …/vente` avec `clientRequestId=opId`).

## O1 — commité
Branche front `feature/pos-offline-o1` (322c73b), non mergée, non poussée. Message sans
trace IA.

## O2a — réalisé (backend, 2026-07-02, commité)
Branche `feature/pos-offline-o2-stock-seam` (adab2ab), non mergée, non poussée.
- `StockMovementService.apply` : param `negativeStockPolicy` (`STRICT` défaut /
  `TOLERATE_NEGATIVE`) + `ecartNegatif` en retour ; la variante par défaut suit sous zéro en
  mode toléré (invariant préservé). STRICT strictement inchangé → **aucun appelant en ligne
  impacté**.
- Spec étendu (+3 tests TOLERATE). **219/219 unit verts, tsc propre.**

### Découverte de code (impacte O2b)
Le POS **supermarché** n'applique PAS la vente via le seam : `effectuerVente` fait un
**pré-check** (`pos-session.service.ts:1730`) + un **`decrement` atomique inline** (~2141 /
2193) de la variante + variante par défaut. Donc :
- O2b devra **neutraliser le pré-check 1730** et lire `stockApres` après décrément, **dans le
  contexte de rejeu uniquement** (endpoint `sync-operations`), + `ECART_STOCK_OFFLINE`.
- Le décrément SM étant déjà atomique, **pas de lost-update** sur la voie de vente SM (le
  durcissement atomique de §8-Q3 ne vise que les autres appelants du seam / le resto).

## O2b — réalisé (backend + front, 2026-07-02, commité)
Back `feature/pos-offline-o2-stock-seam` (01b937b) ; front `feature/pos-offline-o1` (b5d8001).
- **Back** : `effectuerVente({ offlineReplay })` → `validateVente({ skipStockCheck })` +
  pré-check 1730 relâché ; applique la vente sous zéro et journalise `ECART_STOCK_OFFLINE`
  (variante spécifique + par défaut) dans la transaction ; drapeau `stockNegatif` remonté.
  Endpoint **`POST /supermarche/pos-session/sync-operations`** (rejeu par lot ; un résultat par
  op : `APPLIED`/`REJECTED`/`RETRY` via `classifyReplayError` + `extractReplayError`). DTO +
  interface ajoutés. `/vente` en ligne **reste strict** (mode tolérant non exposé).
- **Front** : `syncOfflineOperations` (hook `use-pos-session`) ; `useSalesSync` rejoue via
  `sync-operations` au lieu de `/vente` et interprète le résultat (APPLIED → supprimée ;
  REJECTED/RETRY → échec/retry).
- Vérifs : back **226/226** unit (+ `replay-error.spec`), front **28/28**, tsc propre des 2 côtés.
- Découverte confirmée : le POS SM applique la vente par décrément atomique inline (pas le
  seam) → tolérance câblée directement dans `effectuerVente`.

## O2c — réalisé (front, 2026-07-02, commité)
Branche `feature/pos-offline-o1` (ced777f).
- **Alerte d'écart opérateur** : le drapeau `stockNegatif` du rejeu remonte jusqu'à
  `useSalesSync` → toast « N vente(s) en survente » (à régulariser par ajustement). Ferme la
  boucle « accepter + alerter » de la décision.
- **REJECTED terminal** : `markSaleRejected` (status failed, retries=MAX) → plus de retry auto,
  toast dédié. Les échecs **transitoires** restent retentés (backoff).
- Tests : +2 (vente rejetée non ré-éligible / transitoire ré-éligible), **30/30** front, tsc propre.

## e2e-BD — squelettes ajoutés (back, commit f43337f)
Le harnais e2e-db (Phase 4) est bien sur `main`. Ajouté sous `test/e2e-db/` :
- Fixtures supermarché (`createSupermarche`, `createProduitMagasin`, `createVarianteProduit`…).
- `pos-offline-stock.db-e2e-spec.ts` : seam `TOLERATE_NEGATIVE` (stock < 0 + `ecartNegatif`,
  invariant défaut) vs `STRICT` (rollback réel) contre vrai PostgreSQL.
- `pos-offline-idempotence.db-e2e-spec.ts` : `clientRequestId @unique` → double rejeu = 1
  commande (P2002). + `describe.skip` du flux complet `sync-operations` (session+paiement+TVA)
  à finaliser sous Docker.
- **Typecheck OK** contre le schéma Prisma (via tsconfig temporaire). **Non exécutés** :
  Docker Desktop indisponible dans la session (`docker version` échoue). Lancer `pnpm
  test:e2e:db` quand Docker est up.

## PR ouvertes (2026-07-02)
- **Backend** : `Nagtic-co/all-in-one-backend#57` (branche `feature/pos-offline-o2-stock-seam`
  = O2a + O2b + e2e). Créée via gh.
- **Front** : branche `feature/pos-offline-o1` (= O1 + O2b-front + O2c) **poussée** sur
  `snade-dev/all-in-one`, mais `gh pr create` refusé (compte non collaborateur du repo front).
  Ouvrir à la main : https://github.com/snade-dev/all-in-one/compare/main...feature/pos-offline-o1?expand=1

## En attente
- **Exécuter les e2e-BD** (Docker) : les 2 specs actifs + finaliser les 3 scénarios skippés.
- **Ouvrir le PR front** manuellement (permissions gh).
- **O3** multi-caisses durci : concurrence (décréments SM déjà atomiques ; prouver par e2e
  concurrents), tableau d'écarts par caisse. (périmètre resto, remboursement offline, delta
  atomique, numérotation reçus, unitaire vs batch, expiration d'op, session offline,
  réalignement catalogue).

## Prochaine action
1. Faire relire le doc, trancher les questions ouvertes.
2. À validation : ajouter l'entrée « POS offline » dans `DECISIONS.md`, puis démarrer O1.
3. Aucune trace IA dans le git (respecté).
