# Session 2026-07-02 — M1-b : API publique catalogue (backend)

Branche : `feat/m1-public-catalog-api` (repo backend NestJS). **Non commité — en attente de relecture.**

## Objectif
Exposer le catalogue (magasins, produits/plats, détail) à l'app cliente SANS
authentification ni compte. Surface publique séparée de l'API de gestion.

## Conception validée (Étape A)
- Préfixe dédié `/public`, versionné `/v1`, routes `@Public()`.
- Stock affiché en « DISPONIBLE / EPUISE » uniquement — jamais la quantité.
- Aucune donnée de gestion exposée (marges, coûts, prix HT interne, fournisseurs,
  identifiants d'autres établissements, données légales/facturation).
- Visibilité d'un établissement : opt-in (défaut = non exposé).

## Endpoints livrés
| Méthode & chemin | Rôle |
|---|---|
| `GET /public/v1/etablissements` | Liste des établissements visibles (SM + resto), `?type` `?q` + pagination |
| `GET /public/v1/etablissements/:type/:id` | Fiche établissement + ses catégories |
| `GET /public/v1/etablissements/:type/:id/catalogue` | Items paginés/filtrés (`categorieId`, `q`, `disponibleUniquement`, `tri`) |
| `GET /public/v1/etablissements/:type/:id/catalogue/:itemId` | Détail d'un item (variantes, allergènes, régime, images) |

`:type` ∈ `{ supermarche, restaurant }`. Établissement non visible ou type inconnu → 404
(jamais 403 : on ne confirme pas l'existence).

## Fichiers
- `prisma/schema.prisma` : champ `estVisiblePublic Boolean @default(false)` sur `Supermarche`
  et `Restaurant` (+ index). Migration additive `20260702120000_add_etablissement_visibilite_publique`.
- `src/public/` : `public.module.ts`, `public-catalog.controller.ts` (`@Public` + `ThrottlerGuard`
  60 req/60 s/IP), `public-catalog.service.ts` (sélections Prisma explicites — les champs sensibles
  ne quittent jamais la base), `public-catalog.mapper.ts` (projection publique stricte + calcul
  dispo/épuisé), DTO d'entrée (query) et de sortie stricts.
- `src/common/image-storage/image-url.util.ts` : `deriveThumbnailUrl` (convention `_thumb` de M1-a,
  centralisée dans le module images) + `firstImage`.
- `src/app.module.ts` : enregistrement de `PublicModule`. `src/auth/interceptors/trial.interceptor.ts` :
  `/public` ajouté aux préfixes non jaugés (ceinture-bretelles ; aucune session à jauger en anonyme).

## Sécurité (surface publique)
- Double barrière anti-fuite : `select` Prisma restreint + mapper de sortie qui ne recopie jamais
  l'entité brute. Test : la sortie ne contient aucune clé sensible ni stock chiffré.
- Isolation : un item d'un autre établissement n'est ni listé ni accessible (404).
- Throttling explicite (le `ThrottlerGuard` n'est pas global dans l'app).

## Tests
- Unit : `image-url.util.spec.ts` (6) + `public-catalog.mapper.spec.ts` (11) — mapping strict,
  exclusion des champs de gestion, calcul dispo/épuisé, vignettes.
- e2e-BD : `public-catalog.db-e2e-spec.ts` (7) — non visible → 404, produit épuisé badgé « EPUISE »,
  `disponibleUniquement` masque les épuisés, isolation inter-établissements, zéro fuite de champ.
- Résultats : `tsc --noEmit` OK ; **258 unit / 24 suites** verts ; **24 e2e-BD / 8 suites** verts
  (migration appliquée sur base jetable). Aucune régression de l'API de gestion.

## Non-régression
Additif uniquement : nouveau module + routes `/public/v1`, aucun contrôleur/service existant modifié.
Seul changement de schéma = 2 colonnes booléennes défaut `false` (non lues par l'existant).

## Reste à faire
1. Relecture puis commit/merge de la branche.
2. Déployer la migration `20260702120000_add_etablissement_visibilite_publique`.
3. Prévoir côté Owner Hub (front) le toggle « visible publiquement ».
4. Enchaîner M1-c / M1-d (comptes clients, commandes).

## Points ouverts signalés
- Le design doc `.claude/DESIGN/DESIGN-M1-...md` (annoncé « fait foi ») est absent des dépôts :
  conception reconstituée à partir des décisions actées + du code. À archiver si retrouvé.

---

## Suite 2026-07-03 — merge M1-b + toggle de visibilité

- **M1-b** commité (`2cb82f0`) + **mergé** sur `main` (`fbb70c3`) + **poussé** sur `origin`.
  Migration `estVisiblePublic` **appliquée sur `aio`** (`prisma migrate deploy`, la seule en attente).
  Smoke test live OK : `GET /public/v1/etablissements` → 200 (liste vide = opt-in), types inconnus /
  magasins masqués → 404, `/clients` (gestion) → 401 (auth intacte).
- **Toggle Owner Hub** (rend M1-b exploitable) commité sur branches, **non mergé / non poussé** :
  - BACK `feat/etablissement-public-visibility` (`c85a641`) : `PATCH /etablissements/:id` +
    `estVisiblePublic` (DTO+whitelist) ; **nouveau `GET /etablissements/:id`** owner-scopé (corrige
    aussi la page settings qui appelait un endpoint absent).
  - FRONT `feat/public-visibility-toggle` (`5c5b795`, sur `main`) : interrupteur « Visible dans le
    catalogue public » sur la page paramètres établissement. Working tree front restauré sur
    `feature/pos-offline-o1`.
  - typecheck **back + front VERT**. e2e-BD du toggle **écrit mais NON exécuté** (Docker HS).
- **Incident disque plein / Docker read-only** : voir ETAT.md « Pièges connus ». Reste à faire :
  redémarrer Docker Desktop → `docker rm -f aio-e2e-pg` → `pnpm test:e2e:db` (preuve du toggle) →
  merger + pousser les 2 branches.
- **Correctif docs** : la vraie source pilotage est **co-localisée** (`allInOne/allInOne/all-in-one-pilotage`,
  git), pas la copie `Downloads/all-in-one-pilotage` (obsolète) éditée par erreur au début.
