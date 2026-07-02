# Feuille de route — all-in-one (front + back)

> Plan global. Change rarement. Le curseur « où on en est » vit dans ETAT.md.

## Principe directeur
D'abord ne rien casser ni fuiter (sécurité + tests sur l'argent), ensuite consolider
(qualité, dette), enfin faire grandir (fonctionnalités). On ne démarre pas la Phase 6
avant d'avoir fini les Phases 1 et 2. Deux chantiers ne se mènent en parallèle QUE s'ils
sont dans des repos différents et sans dépendance d'ordre.

| Phase | Thème | Repo | État |
|---|---|---|---|
| 0 | Hygiène | front + back | ✅ |
| 1 | Sécurité backend | back | ✅ |
| 2 | Tests « argent » | front + back | ✅ |
| 3 | Qualité de type | front | ⬜ |
| 4 | Robustesse données | back | ✅ PR #55 mergée ; migrations déployées |
| 5 | Dette de duplication | front + back | 🟡 back mergé (4 fusionnés + 1 mort + categories reporté) ; front mergé (fournisseur fusionné, 6 paires reportées) |
| 6 | Fonctionnalités | front + back | 🔗 stats en cours |

## PHASE 3 — Qualité de type (frontend)
**But :** réactiver la protection du compilateur, sans masquer (pas de `as any`/`@ts-ignore`).
- [ ] Mesurer (`tsc --noEmit`), résorber les `any` (`lib/` et `types/` d'abord).
- [ ] Retirer `ignoreBuildErrors: true` de next.config une fois à zéro erreur.
**Critère de sortie :** `next build` échoue sur une erreur de type ; Vitest reste vert.

## PHASE 4 — Robustesse des données (backend)
**But :** intégrité (pas de demi-vente) + zéro fuite entre clients.
- [x] Transactions Prisma sur les opérations multi-étapes (vente, remboursement, stock) — déjà en
      place ; ajout de l'atomicité ouverture/fermeture de session caisse resto.
- [x] Audit d'isolation multi-tenant — modèle sain (scope depuis le JWT) ; 1 faille cross-restaurant
      (POS resto) corrigée.
- [x] Index DB manquants — migration créée (non déployée).
- [x] Journal d'audit immuable — `historique` + seam ; branché sur remboursements, prix
      (produit/plat), permissions (SM/resto), annulation de commande (SM/resto).
- [x] PR #55 mergée sur main ; migrations déployées. Reste : e2e à passer.
**Critère de sortie :** opérations atomiques ; isolation vérifiée ; requêtes indexées ; audit en place. ✅

## PHASE 5 — Dette de duplication (continu)
**But :** réduire le coût de maintenance Supermarché/Restaurant (front ET back).
- [x] Backend : extraire la logique commune + variantes par type via base `common/*` + `Scope`.
      Fusionnés : taux-tva, fournisseur, employes, code-promo (partiel). Mort supprimé :
      entite-stock. Reporté (divergence réelle) : categories. **Mergé sur main (`159348c`).**
- [ ] Backend : dédup contrôleur employes (asymétrie `forgot-password.dto`).
- [x] Frontend : fabriques data+hooks + variantes par scope. Fait : fournisseur (mergé `8ae73ca`).
      Reporté (divergence réelle) : produits, inventaire, dashboard, orders, statistics, promotions.
- [ ] Préparer i18n (next-intl) si expansion multi-langue.
**Critère de sortie :** une correction métier ne se fait plus en double.

## PHASE 6 — Fonctionnalités (croissance)
**But :** faire grandir le produit sur un socle fiable. Chaque feature arrive avec ses tests.
- [🔗] Module Statistiques (backend fait, frontend en cours).
- [ ] Mode offline robuste du POS (file de synchro + résolution de conflits).
- [ ] Dashboard propriétaire consolidé multi-établissements.
- [ ] Alertes intelligentes (ruptures, péremptions, écarts d'inventaire).
- [ ] Programme de fidélité client.
- [ ] Export comptable conforme aux pays cibles.
- [ ] App mobile vendeur.
