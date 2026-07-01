# Phase 5 — Déduplication ciblée backend (Supermarché ↔ Restaurant)

Date : 2026-07-01
Branche : `refactor/phase-5-dedup-backend`
Périmètre autorisé : 6 modules jumeaux mesurés très proches — `entite-stock`,
`employes`, `taux-tva`, `fournisseur`, `code-promo`, `categories`.

## Méthode
Refactoring **à comportement constant** : aucune modification d'API, d'entrées/sorties
ni de permissions. Pour chaque module fusionné :
1. Diff exhaustif des deux versions pour isoler les vraies différences des simples
   swaps de discriminant (établissement + clés de portée).
2. Base commune abstraite extraite **verbatim** depuis la version supermarché,
   paramétrée par un objet `Scope` capturant toutes les différences.
3. Services des deux verticales réduits à de minces sous-classes — **noms de classe
   conservés** (tokens d'injection NestJS inchangés).
4. Tests de caractérisation `describe.each` sur les DEUX verticales, verrouillant le
   comportement ET la forme de réponse (clé de portée / entité liée). Suite complète
   verte après chaque module.

## Résultats par module
- **taux-tva** ✅ fusionné — `common/taux-tva/` (base + scope + spec) ; 2 services 1170→15 l.
  Commit `12ea3a0`.
- **entite-stock** ✅ supprimé — code mort (routes commentées, service jamais appelé) ;
  retiré des modules et de `produit-magasin`. Commit `fd306e2`.
- **fournisseur** ✅ fusionné — `common/fournisseur/` ; scope inclut l'entité vendable
  liée (produitMagasin/plat) pour findOne + garde de suppression ; 2 services 926→17 l.
  Commit `e198453`.
- **employes** ✅ fusionné — `common/employes/` ; scope = établissement + type de rôle ;
  2 services 604→17 l. Commit `c461e00`.
- **code-promo** ✅ fusion **partielle** — `common/code-promo/` : seules les 6 méthodes
  strictement identiques (accès, findOne, activer/désactiver, supprimer, génération)
  sont partagées. Les 5 méthodes au comportement métier divergent (create, findAll,
  valider, utiliserSurCommande, getStatistiques) + `utiliser` (SM seul) restent par
  verticale. Commit `f5b7679`.
- **categories** ⬜ **reporté** (voir ci-dessous).

## Non fusionné — à signaler
- **categories** : divergence réelle et généralisée, PAS un swap de discriminant.
  SM = 880 l / resto = 479 l ; 579 lignes divergentes. SM possède 2 méthodes en plus
  (`reorderCategories`, `toggleCategoryStatus`) et des signatures différentes sur
  `findTree` / `findOne` / `incrementPopularity` (support `includeInactive`, tracking
  `userId`). Corps divergents jusque dans `findAll` (filtrage de type, comptage
  `produits`+`plats` vs `plats`, gestion d'erreur) et `searchByName` (222 l vs 109 l).
  Aucun sous-ensemble discriminant-only propre → fusion = risque de régression élevé
  pour gain faible. **Décision : reporter, ne pas toucher.**
- Asymétrie **contrôleur** (hors périmètre service) sur `employes` : le supermarché a un
  `dto/forgot-password.dto.ts` et un contrôleur plus étoffé que le restaurant, alors que
  les deux **services** sont identiques. Dédup contrôleur possible séparément.

## Vérifications finales
- `tsc --noEmit` propre.
- Suite complète : **18 suites, 216 tests verts** (dont caractérisation des 4 bases :
  taux-tva, fournisseur, employes, code-promo).

## Bilan
4 modules fusionnés + 1 mort supprimé + 1 reporté motivé. ~ -5 400 lignes nettes de
duplication de services. Une correction métier sur ces modules ne se fait plus en double.
