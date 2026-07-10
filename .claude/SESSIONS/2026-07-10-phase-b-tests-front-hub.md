# Front Hub Phase B — Couverture de tests pré-déploiement (2026-07-10)

Repo : `All-in-One-Partner-Hub`. Branche : `test/coverage-predeploy`.
**Non commité** (attente relecture — `git status` montré au propriétaire).
Périmètre STRICT : fichiers de test + config de test + devDependencies de test —
**zéro fichier applicatif modifié**. Plan Phase A appliqué : Cercle 1 intégral +
Cercle 2 ; Cercle 3 listé sans écriture. Option A confirmée : pas d'extraction
des pages POS, les payloads d'encaissement sont testés en composant lourd
(vue remplacée par une sonde qui capture les props).

## Comptes (avant → après)
- Avant : 5 fichiers / **23 tests** (logique pure, env node).
- Après : 24 fichiers / **191 tests** — **188 verts + 3 skips** documentant des
  bugs réels du code applicatif (cf. plus bas). Suite relancée **3×** :
  résultats identiques, zéro flaky. `pnpm typecheck` : **0 erreur**.

## Outillage ajouté
- devDependencies : `jsdom`, `@testing-library/react`, `@testing-library/dom`,
  `@testing-library/user-event`, `fake-indexeddb`.
- `vitest.config.ts` : include élargi aux `.tsx` + `setupFiles`. Environnement
  par défaut inchangé (node) ; les tests composants/hooks déclarent
  `// @vitest-environment jsdom` par fichier — les 23 tests historiques
  tournent à l'identique.
- `vitest.setup.ts` : cleanup Testing Library explicite (pas de `globals: true`)
  + stub `ResizeObserver` (requis par Radix sous jsdom).
- Piège d'environnement résolu : `pnpm add` lancé depuis un chemin à la casse
  divergente (`allinone` vs `allInOne`) crée des jonctions pnpm à cible
  minuscule → **deux instances de React** chargées (dispatcher null dans les
  tests). Corrigé par réinstallation complète de `node_modules` depuis
  `C:\dev\allInOne\…`. À retenir : toujours travailler avec la casse canonique.
- `pnpm` en mode workspace plantait (`manifestsByPath[rootDir]` undefined) :
  utiliser `--ignore-workspace` dans ce repo.

## Couverture ajoutée (Cercle 1 — argent & stock)
1. `useCart` (14 tests) : plafonnement au stock, remises ligne %/fixe bornées,
   remise globale, TVA calculée APRÈS remise globale, TVA off, taux par défaut
   BD, `vatRateId` exposé, déduplication des taux par valeur.
2. Encaissement POS supermarché — `page.payment.spec.tsx` (7 tests, composant
   avec `PosPageView` sondé) : payload complet (lignes, paiement simple,
   `tauxTVAId`, `clientRequestId`), TVA off → pas de `tauxTVAId`, paiement
   mixte (`paiements[]`, somme exacte), idempotence (même clé après échec
   réseau, clé fraîche après succès), branche offline (enqueue avec
   `localId === clientRequestId`, décrément stock local, panier vidé, 2 ventes
   offline = 2 clés distinctes).
3. `offline-pos-store` (24 tests) : fonctions pures (backoff 1s→60s,
   `isSaleDueForSync` avec plafond MAX_SYNC_RETRIES=5 = état terminal,
   `isSaleRetryable`) + CRUD réel sur fake-indexeddb (FIFO, écrasement par
   `localId` = idempotence locale, `markSaleFailed/Pending/Syncing`,
   `countPendingSales` incluant les failed).
4. `useSalesSync` (8 tests, file réelle fake-indexeddb) : rejeu FIFO +
   suppression au succès, échec métier → failed + continue, 401/403 → remise en
   pending SANS incrément retries + arrêt de boucle, verrou `syncLock` (2
   déclenchements simultanés = 1 seule boucle, concurrence max mesurée = 1),
   passage offline en cours → arrêt, état terminal non rejoué, compteur.
5. `use-pos-session` (19 tests) : basePath RESTAURANT/SUPERMARCHE via
   `localStorage.selectedEtablissementType` (défaut supermarché),
   NO_ACTIVE_SESSION → null sans throw, `ouvrir`, `cloturer/:id` avec
   `montantCompteEspeces`, historique tolérant aux 4 formes de réponse,
   `processVente`.
6. Remboursements : `useRefunds` supermarché (11 tests + 1 skip bug #3) et
   `useRestaurantRefunds` (7 tests) : fallback moyen de paiement
   (standards[0] → personnalises[0] → erreur), payload
   `{commandeId, lignes[{ligneCommandeId, quantite, remettreEnStock}],
   moyenPaiementId}`, statuts refunded_total/partial, réincrément stock local
   si restock, erreurs propagées sans mutation locale.
7. Ajustements stock : `supermarche-inventaire` (8 tests — body whitelisté SANS
   `supermarcheId`/`commentaire`/`notes`, cascade de fallback `raisonGlobale`,
   comptage, retrait-lot, transfert) + `restaurant-inventory` (4 tests —
   POST `/restaurant/plat/adjust-stock` payload intact).
8. POS restaurant `table-grid-tab` (9 tests + 2 skips bugs #1/#2, composant
   sondé) : totaux UI (options `priceModifier`, remises ligne/globale, TVA),
   payload (plats, menus, remise globale, paiements mixtes, idempotence
   `clientRequestId`), branche offline (enqueue, `onSaleQueued`, décrément
   stock plats).

## Couverture ajoutée (Cercle 2)
9. Kanban commandes retrait (13 tests, api mockée + react-query réel) :
   actions par colonne (RECUE = Accepter/Refuser seulement ; EN_PREPARATION =
   Prête/Refuser ; PRETE = Retirée & encaissée seule), endpoints PATCH
   `/:scope/commande-retrait/:id/{accepter,prete,retirer,refuser}`, dialog de
   retrait affichant `totalHt + totalTva`, toast d'écart si
   `stockNegatif: true`, refus avec motif optionnel, scope restaurant.
10. Toggle `estVisiblePublic` (7 tests, page settings owner) : GET
    `/etablissements/:id`, mapping strict `=== true` (undefined → false),
    PATCH `{nom, adresse, estVisiblePublic}`, nom édité transmis, toast
    d'erreur backend.
11. Création produit avec photo : mapping DTO supermarché (10 tests —
    `supplementPrix` = prix variante − prixHt, code-barres numérique only,
    `prixOnline` fallback, conditionnement ignoré si ≤ 1, `images` =
    data-URLs, pas de `supermarcheId`) + restaurant (6 tests) + wizard en
    composant (6 tests — MIME whitelisté, > 2 Mo rejeté, lot mixte, data-URLs
    dans la soumission finale, validation étape 1).
12. Stats CA net : fetchers `stats-supermarche` (5 tests — endpoints + params,
    payload à nu sans enveloppe) + cartes KPI (6 tests — CA NET affiché et pas
    le brut, évolutions signées ±, remboursements).
13. Comparaison établissements : fetcher `/supermarche/statistiques/comparaison`
    → `{etablissements[], series[]}` + tableau (4 tests — KPIs par ligne,
    évolutions signées +/−/0, tri, export désactivé à vide).

## Bugs réels documentés (skips — code applicatif NON modifié)
1. **BLOQUANT** — `pos-restaurant/components/table-grid-tab.tsx` : options
   (`priceModifier`) et remises de ligne comptées dans le total UI encaissé
   (l.472-509) mais ABSENTES du payload (l.547-552 : `{varianteId, quantite}`
   seulement) → écart entre le montant affiché/encaissé et la commande serveur.
2. **MAJEUR** — même fichier l.107 : TVA codée en dur `[18, 10, 5.5, 0]`, pas
   de `tauxTVAId` dans le payload (le fix BD existe côté supermarché, à
   reporter côté resto).
3. **MAJEUR** — `pos/hooks/useRefunds.ts` l.73 : fallback
   `ligneCommandeId: item.lineId || saleId` envoie un ID de commande comme ID
   de ligne quand `lineId` manque (ventes locales sans historique API).
4. Pour mémoire (non testé, code mort) : `pos/hooks/usePayments.ts`.

## Cercle 3 — recensé, rien écrit
api.ts utils, livraison board, dine-in/carte QR, menus resto, promotions,
purchase-orders + PDF, rayons, clients/fidélité, équipes, daily-report,
billing, owner users/audit, RBAC nav + route-guard, auth hooks, import/export
CSV, dashboards, settings.

## Non testable automatiquement (checklist manuelle)
Impression réelle (tickets caisse/cuisine, PDF), scan caméra code-barres,
comportement offline réel du navigateur (service worker/réseau), rendu visuel
des toasts/receipts, périphériques (tiroir-caisse).
