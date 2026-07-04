# Session 2026-07-04 — Fix : catalogue public « Établissement introuvable » (backend)

Branche : `fix/public-catalogue-etablissement-introuvable` (repo backend).
**Non commité — en attente de relecture.**

## Contexte / bug
Test manuel : `GET /public/v1/etablissements` renvoie bien un établissement visible
(« fenetre », SUPERMARCHE), mais `GET /public/v1/etablissements/{id}/catalogue` répond
**404 « Établissement introuvable »** pour ce même id. Incohérence liste ↔ catalogue.

## Diagnostic (Étape A, validé)
Cause = **incohérence de contrat de résolution**, PAS une requête Prisma cassée :
- La **liste** identifie un établissement par son **id seul** (balaie les 2 tables
  supermarché+restaurant, filtre `{ estVisiblePublic, estActif }`) ; le type n'est qu'une
  donnée de sortie.
- Le **catalogue/fiche/item** étaient routés `/:type/:id[/catalogue[/:itemId]]` et
  appelaient `normaliseType(type)` EN TÊTE. L'URL `/etablissements/{id}/catalogue` (2 seg.)
  ne matche pas la route catalogue (3 seg.) mais la route fiche `:type/:id`, avec
  `type = {id}` → `normaliseType` jette 404 « Établissement introuvable ». Un établissement
  VISIBLE devenait inatteignable par son catalogue.
- **Piège Prisma 5.7 écarté** : la visibilité était résolue par `findFirst` à plat sur la
  table concrète (`where` racine), pas via un `include` sur relation to-one. Non concerné.
- Même défaut sur **fiche** et **item** (mêmes `:type/:id`). **Recherche** OK (pas de type
  dans l'URL) mais renvoie le type en donnée → même friction côté client, sans 404.

## Correctif (Étape B — approche « résolution par id seul » validée)
- **Service** `public-catalog.service.ts` : `normaliseType` + `assertEtablissementVisible`
  remplacés par `resolveEtablissementVisible(id)` — cherche l'id dans supermarché PUIS
  restaurant avec le même filtre `{ estVisiblePublic, estActif }` que la liste, renvoie le
  type découvert, sinon **404 (jamais 403)**. `getEtablissementDetail(id)`,
  `listCatalogue(id, query)`, `getCatalogueItem(id, itemId)` : signatures sans `type`.
- **Contrôleur** : routes adressées par id seul — `@Get(':id')`, `@Get(':id/catalogue')`,
  `@Get(':id/catalogue/:itemId')` (1/2/3 segments, non ambigus).
- Espaces d'id disjoints (`uuid` sur Supermarche et Restaurant) → résolution par id sûre.
- **Garde-fous M1-b intacts** : double barrière anti-fuite (select restreint + mapper
  strict), stock DISPONIBLE/EPUISE, opt-in `estVisiblePublic`, 404 pour non-visible/inexistant.

## Tests
`test/e2e-db/public-catalog.db-e2e-spec.ts` :
- SM VISIBLE → catalogue + fiche par **id seul** (200, pas 404) ; produit épuisé badgé ;
  non-visible → 404 ; **id inexistant → 404** ; isolation inter-établissements.
- Nouveau bloc **RESTAURANT** : resto visible expose catalogue/fiche/item par id ;
  resto masqué → 404. Couvre les DEUX verticales.

Résultats : **e2e-BD 9 suites / 36 tests verts** (`pnpm test:e2e:db`, PG jetable 5434 ;
`aio-db` intact). **Unit public 24/24**. `tsc --noEmit` vert.

## Reste
- Relecture + commit/merge de la branche.
- Note client : la navigation vers un item se fait désormais par `id` seul (le `type`
  renvoyé par liste/recherche n'est plus nécessaire dans l'URL).
