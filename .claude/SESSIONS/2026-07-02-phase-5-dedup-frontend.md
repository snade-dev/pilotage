# Phase 5 — Déduplication frontend (Supermarché ↔ Restaurant)

Date : 2026-07-02
Repo : `All-in-One-Partner-Hub` (Next.js)
Branche : `refactor/phase-5-dedup-frontend` → **mergée sur main (`8ae73ca`, --no-ff)**.

## Méthode
Même patron qu'au backend, adapté au front : fabrique commune paramétrée par un
`scope`, et les fichiers existants deviennent de **minces wrappers ré-exportant
les noms d'origine** (fonctions, hooks) → consommateurs, types et clés de cache
React Query inchangés. Garde-fous : `tsc --noEmit`, ESLint, Vitest (pas de vérif
visuelle possible → périmètre volontairement restreint aux couches data/hooks).

## Fait
- **fournisseur (data + hooks)** ✅ — `lib/fournisseur-data.factory.ts`
  (`makeFournisseurApi`, générique sur le type de réponse, scope = basePath +
  logLabel) et `hooks/use-fournisseur.factory.ts` (`makeFournisseurHooks`, scope =
  API + clés de cache). `supplier-data` / `restaurant-supplier-data` et
  `use-supplier` / `use-restaurant-supplier` réduits à des wrappers. 6 fichiers,
  net **+597 / -762**. tsc 0 erreur, ESLint propre, Vitest 23/23. Commit `7d11788`.

## Non fusionné — reporté (mesuré, divergence réelle)
Triage des paires restantes (diff insensible aux espaces) — seul suppliers était
un swap propre (domaine identique). Les autres divergent en signature, options,
forme de réponse ou DTO :
- **produits** — la liste SM prend `(supermarcheId?, enabled)` + `staleTime`, la
  resto aucune option ; DTO différents (variantes, champs magasin vs plat).
- **inventaire** — SM 130 l vs resto 67 l (SM bien plus fourni).
- **dashboard** (~97% de lignes divergentes) / **orders** (~95%) — KPIs/flux
  différents.
- **statistics** (~78%) — agrégations différentes.
- **promotions** — la plus proche des restantes mais formes de promo différentes.

Décision (validée) : **finaliser sur suppliers**, reporter le reste. Sans vérif
visuelle front, forcer des abstractions sur du code réellement divergent = risque
de régression non détectable pour un gain faible.

## Bilan
1 paire fusionnée (data + hooks) + patron documenté et réutilisable. Les paires
divergentes restent candidates à une refonte ultérieure (idéalement quand une
vérif visuelle / e2e front est en place).
