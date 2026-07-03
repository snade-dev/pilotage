# État du projet all-in-one — dernière mise à jour : 2026-07-03

> Suivi centralisé des DEUX repos. Lire ce fichier en premier à chaque session.
> Repos : Frontend (Next.js, All-in-One-Partner-Hub) · Backend (NestJS).

## Vue d'ensemble
| Phase | Repo | État |
|---|---|---|
| 0 Hygiène | front + back | ✅ |
| 1 Sécurité | back | ✅ |
| 2 Tests « argent » | front + back | ✅ |
| Module Statistiques (backend) | back | ✅ mergé |
| Module Statistiques (frontend) | front | ✅ mergé sur main |
| 3 Qualité de type | front | 🟡 Lot 0+1 mergés sur main (build protégé, lib/ 0 any) ; traîne app/+hooks/ restante |
| 4 Robustesse données | back | ✅ **PR #55 mergée** ; migrations déployées ; **e2e-BD Phase 4 verts** (isolation/atomicité/audit, branche `test/phase-4-e2e`) |
| 5 Duplication | front + back | 🟡 **back MERGÉ (`159348c`)** : 4 fusionnés + 1 mort + categories reporté · **front MERGÉ (`8ae73ca`)** : fournisseur (data+hooks) fusionné, 6 autres paires reportées (divergence réelle) |
| 6 Fonctionnalités | front + back | 🟡 stats mergé · **POS offline back MERGÉ** (PR #57, `f50124c`) — reste PR front · **M1 images back MERGÉ** (PR #58, `6de2bfb`) — reste migration à exécuter + vérif front · **M1-b API publique COMPLÈTE** (catalogue + recherche cross-catalogue + toggle visibilité front/back, mergé+poussé `86092e3`/`57562c5`, migration trigram déployée) |

## En cours / en attente
- [x] **M1 (app client) — stockage d'images (backend)** : **PR #58 MERGÉE sur main** (`6de2bfb`).
      Interface abstraite `ImageStorageService` + impl **dossier
      local** (`common/image-storage/`, `@Global`) + miniatures `sharp` (thumb 200² / display
      800px webp) + service statique `/media` (`useStaticAssets`). Bascule des créations produit
      SM + plat resto via un **service commun** `persistImages` (patron Phase 5, zéro dup).
      Tests : **unit 241/241**, **e2e-BD 17/17**.
      **Migration base64→fichiers EXÉCUTÉE sur la base dev `aio` (2026-07-02)** : dump de sauvegarde
      `scripts/backup/aio-full-*.sql` (3,1 Mo) + dry-run + `--confirm` → **9 images converties**
      (4 produits + 1 plat + 4 variantes), 0 base64 restant, 18 fichiers webp dans `uploads/`,
      backup JSON des valeurs d'origine. Validée d'abord sur **copie jetable** `aio_migration_copy`
      (supprimée après). **Correctif runner PR #59 MERGÉE sur main** (`4fbb02a`) : le script plantait
      tel que documenté (`sharp` + `esModuleInterop` off) → `scripts/tsconfig.migrate.json` + script
      `pnpm migrate:images`. **Reste** : vérif visuelle front (URLs relatives `/media`, voir si besoin
      `PUBLIC_MEDIA_BASE_URL` absolue pour l'app desktop/front). Ne pas oublier de sauvegarder
      `uploads/` (backend) au même titre que la base — les images sont désormais des fichiers.
      SESSIONS/2026-07-02-m1-image-storage.md
- [x] **M1-b (app client) — API publique catalogue (backend)** : **MERGÉE + POUSSÉE sur main**
      (`fbb70c3`, commit `2cb82f0`). Surface `@Public()` séparée, préfixe `/public/v1`, throttling
      renforcé (60 req/60 s/IP ; `ThrottlerGuard` n'est pas global). 4 endpoints : liste
      établissements visibles (SM+resto), fiche+catégories, catalogue paginé/filtré, détail item.
      Type inconnu / établissement non visible → **404** (jamais 403). Stock exposé « DISPONIBLE /
      EPUISE » seulement, jamais chiffré. **Double barrière anti-fuite** : `select` Prisma restreint
      + mapper de sortie strict (jamais l'entité brute) ; aucun champ de gestion. Prix public =
      `prixOnline`. Vignettes via convention M1-a (`deriveThumbnailUrl`). **Visibilité opt-in** :
      champ `estVisiblePublic @default(false)` sur Supermarche+Restaurant (migration
      `20260702120000_add_etablissement_visibilite_publique` **appliquée sur `aio`**, poussée ;
      prod = auto au `start`). Écriture du champ via `PATCH /etablissement/:id` (DTO + whitelist
      étendus). Tests : **17 unit + 7 e2e-BD** ; suites **258 unit / 24 e2e-BD** vertes ; API de
      gestion intacte. Toggle « visible publiquement » (Owner Hub) : **mergé+poussé le 2026-07-03**
      (voir bullet recherche + Prochaine action #0). Design doc M1 « fait foi » introuvable dans les
      dépôts (conception reconstituée depuis décisions actées). SESSIONS/2026-07-02-m1-public-catalog.md
- [x] **M1-b — recherche cross-catalogue publique (backend)** : **MERGÉE + POUSSÉE sur `main`**
      (`0941877`, commit `a4b17e2`) ; **migration trigram `20260703120000_add_public_search_trgm`
      DÉPLOYÉE sur `aio`** (extension `pg_trgm` + 2 index GIN vérifiés). Nouvel endpoint
      `GET /public/v1/recherche` (`@Public()`, throttle **renforcé 20 req/60 s/IP**) : cherche un
      produit/plat par nom À TRAVERS tous les établissements visibles (SM+resto) et **regroupe les
      offres par item logique** (clé = EAN si présent, sinon nom normalisé + type ; pas de fusion
      SM/resto). Réponse = { nom, type, prixMin, prixMax, nbOffres, offres:[{établissement, itemId,
      prix, DISPONIBLE/EPUISE, imageThumbnailUrl, distanceKm}] } ; distance (haversine) si lat/lng.
      **Perf** : migration `20260703120000_add_public_search_trgm` (extension `pg_trgm` + index GIN
      trigram sur `produit_magasin.nom` et `plat.nom`) → `ILIKE '%q%'` indexé, pas de scan
      séquentiel. Mêmes garde-fous anti-fuite (select restreint + mapper strict ; `estVisiblePublic`
      par offre). `.env.example` documente `PUBLIC_MEDIA_BASE_URL`. Tests : **13 unit + 6 e2e-BD** ;
      suites **271 unit / 25 · 30 e2e-BD / 9** vertes ; gestion intacte (additif pur).
      SESSIONS/2026-07-03-m1-public-search.md
- [~] **Phase 6 — Mode offline robuste du POS** : CODE + E2E COMPLETS (O0→O2c), en attente de
      merge (PR #57 back + PR front à ouvrir). O3 multi-caisses optionnel. Détail des jalons :
      O0 conception + O1 fait (front).
      Design : `.claude/DESIGN/2026-07-02-pos-offline-sync.md`. Décision cadre actée dans
      DECISIONS.md (2026-07-02). Cadre : file d'opérations idempotentes (opId=UUID caisse →
      `clientRequestId`), stock négatif toléré au rejeu + alerte `ECART_STOCK_OFFLINE`, en
      ligne reste STRICT ; remboursement offline INTERDIT ; supermarché seul en O0 ; aucune
      table nouvelle.
      **O1 FAIT (front, commité `feature/pos-offline-o1` — 322c73b)** : ordre de rejeu =
      chronologique réel (champ `seq` monotone + tri par `createdAt`/`seq`) au lieu de l'ordre
      de clé UUID quelconque ; helpers purs `compareSalesForReplay`/`sortSalesForReplay` +
      5 tests Vitest (28/28 suite), tsc propre. Zéro changement backend.
      **O2a FAIT (backend, commité `feature/pos-offline-o2-stock-seam` — adab2ab)** :
      `negativeStockPolicy` (STRICT défaut / TOLERATE_NEGATIVE) + `ecartNegatif` sur
      `StockMovementService.apply` ; STRICT inchangé, 219 unit verts (dont 3 TOLERATE), tsc
      propre.
      **Découverte** : le POS SM n'applique PAS la vente via le seam mais par un pré-check
      (pos-session.service.ts:1730) + `decrement` atomique inline (~2141/2193). ⇒ O2b devra
      neutraliser ce pré-check dans le contexte de rejeu (endpoint `sync-operations`), pas
      seulement le seam. Bon point : décrément SM déjà atomique (pas de lost-update sur cette
      voie).
      **O2b FAIT (back `feature/pos-offline-o2-stock-seam` 01b937b + front `feature/pos-offline-o1`
      b5d8001)** : `effectuerVente({offlineReplay})` (pré-check relâché + `validateVente
      skipStockCheck`) applique la survente + journalise `ECART_STOCK_OFFLINE`, renvoie
      `stockNegatif` ; endpoint `POST /supermarche/pos-session/sync-operations` (rejeu par lot,
      résultat par op APPLIED/REJECTED/RETRY) ; `/vente` en ligne reste strict. Front : la file
      de sync passe par `sync-operations`. back 226/226 + front 28/28, tsc propre.
      **O2c FAIT (front `feature/pos-offline-o1` ced777f)** : alerte de survente à la sync
      (toast si `stockNegatif`, à régulariser par ajustement) + refus `REJECTED` rendu terminal
      dans la file (`markSaleRejected`, retries=MAX, plus de retry auto) ; transitoires toujours
      retentés. 30/30 front, tsc propre.
      **e2e-BD FAIT (back 46123a5, poussé PR #57)** : exécutés sous Docker → **14/14 verts,
      0 skip**. `pos-offline-stock` (seam TOLERATE/STRICT vrai PG) + `pos-offline-idempotence`
      (clientRequestId P2002) + **`pos-offline-sync` bout-en-bout** (via `syncOperations`) :
      double effet (rejeu ×2 = 1 commande + 1 mouvement), écart (survente → stock négatif +
      `ECART_STOCK_OFFLINE` dans historique), échec partiel (lot [valide, invalide] →
      APPLIED/REJECTED). Fixtures session POS SM ouverte + moyen de paiement ajoutées.
      **PR** : backend **#57 MERGÉE sur main** (`f50124c`) ; front branche `feature/pos-offline-o1`
      poussée, PR à ouvrir à la main (gh non collaborateur du repo front).
      **Reste** : ouvrir + merger PR front ; O3 multi-caisses durci (preuve concurrente).
      SESSIONS/2026-07-02-pos-offline-design.md
- [x] Migrations backend déployées (stats + `phase4_hot_path_indexes` + `phase4_audit_journal`).
- [x] Phase 4 : journal d'audit branché sur remboursements (3), modification de prix
      (produit + plat), changement de permissions (SM + resto), annulation de commande (SM + resto).
- [x] Phase 4 : PR #55 mergée sur main.
- [~] Phase 4 : e2e-BD écrits + verts sur base **jetable Docker** (jamais `aio`). 3 garanties
      prouvées (isolation/atomicité/audit), commande unique `pnpm test:e2e:db`. Branche
      `test/phase-4-e2e` à relire/merger. SESSIONS/2026-07-02-phase-4-e2e.md

## Fait (validé + mergé)
### Backend
- [x] Phase 0 Hygiène — SESSIONS/2026-07-01-phase-0-1.md
- [x] Phase 1 Sécurité (auth globale, CORS, secrets, throttler, Helmet) — SESSIONS/2026-07-01-phase-0-1.md
- [x] Phase 2 Tests « argent » (105 unit + 8 e2e guards) — SESSIONS/2026-07-01-phase-2-money.md
- [x] Module Statistiques agrégation (4 endpoints ; scope établissement/consolidé ; CA net ;
      pagination/tri ; 134 unit + 12 e2e) — SESSIONS/2026-07-01-stats-supermarche.md
- [x] Comparaison des établissements — endpoint `GET /supermarche/statistiques/comparaison`
      (protégé `VOIR_RAPPORTS_CONSOLIDES`, borné au propriétaire) : KPIs par établissement +
      séries de CA net ; factorisé depuis `StatsAgregationService` ; 139 unit + 16 e2e verts —
      **PR #54 mergée sur main**. SESSIONS/2026-07-01-stats-comparaison-etablissements.md
### Frontend
- [x] Phase 2 socle Vitest (3 fichiers, 15 tests : totaux document, promotions, TVA)
- [x] Page Statistiques Supermarché `/statistiques-ventes` (mono-établissement, filtre année
      corrigé, KPIs CA net, courbe, 2 classements, export CSV, Vitest 23 tests) — mergé sur
      main (fff0776). SESSIONS/2026-07-01-stats-supermarche-frontend.md + ...-stats-front-ajustements.md
- [x] Phase 3 Qualité de type Lot 0+1 — `ignoreBuildErrors` retiré (build type-checké vert),
      `lib/` 0 `any` (helper `lib/raw.ts`), Vitest 23/23 — mergé sur main (4d79fc9).
      SESSIONS/2026-07-01-phase-3-types.md
- [x] Page comparaison des établissements `/owner/comparaison` (tableau KPI triable + export CSV,
      graphe multi-courbes CA net, période + granularité) — mergée sur main (637a355 ; merge
      re-vérifié type-clean sous le build type-checké Phase 3). SESSIONS/2026-07-01-stats-comparaison-front.md
- [x] Phase 4 Robustesse données — **PR #55 mergée sur main** (146 unit verts) : B1 isolation
      (faille cross-restaurant POS resto CONFIRMÉE + corrigée : `where` sur relation to-one `plat`
      ignoré par Prisma 5.7), B2 atomicité session caisse resto, B3 index hot paths, B4 journal
      d'audit immuable (`historique`) branché sur remboursements (×3) + prix (produit/plat) +
      permissions (SM/resto) + annulation de commande (SM/resto). Migrations déployées.
      SESSIONS/2026-07-01-phase-4-data-robustness.md
- [x] Journal d'audit — affichage : backend `GET /owner/audit-log` élargi pour remonter les
      traces Phase 4 (scope `etablissementId` + acteur partenaire) — **PR #56 mergée**. Frontend :
      page `/owner/audit-log` (filtre établissement, badges, avant/après lisible) — **mergée sur
      main front (f7f7076)**. Vérif visuelle non faite.
- [x] Phase 5 Duplication (backend) — **mergée sur main (`159348c`, merge --no-ff)**.
- [x] Phase 5 — contrôleur employes (backend) : dédup **écartée** (validation `@Body` NestJS
      exige des classes DTO concrètes, or les DTO des 2 verticales diffèrent → fusion =
      changement de validation). DTO mort `forgot-password.dto` (SM) supprimé (`52e9e1c`, poussé).
- [x] Phase 5 Duplication (frontend) — **mergée sur main front (`8ae73ca`, --no-ff)** : couches
      data + hooks fournisseur factorisées via fabriques + scope (wrappers préservant noms/clés
      de cache), tsc/ESLint propres, Vitest 23/23. Triage : seul suppliers était un swap propre ;
      produits/inventaire/dashboard/orders/statistics/promotions **reportés** (divergence réelle,
      pas de vérif visuelle front possible). SESSIONS/2026-07-02-phase-5-dedup-frontend.md
      Bases communes `common/*` paramétrées par un `Scope`, sous-classes minces (noms de classe
      conservés = tokens d'injection intacts), tests de caractérisation sur les 2 verticales,
      comportement constant. Fusionnés : **taux-tva** (`12ea3a0`), **fournisseur** (`e198453`),
      **employes** (`c461e00`) ; **code-promo** en fusion partielle (`f5b7679`, seules les 6
      méthodes discriminant-only) ; **entite-stock** mort supprimé (`fd306e2`). **categories
      reporté** (divergence métier réelle, voir session). 18 suites / 216 tests verts.
      SESSIONS/2026-07-01-phase-5-dedup-backend.md

## Prochaine action
0. **M1-b API publique — TERMINÉE (recherche + toggle), tout mergé/poussé le 2026-07-03** :
   - **Recherche cross-catalogue** : back `main` (`0941877`) + **migration trigram déployée sur `aio`**.
   - **Toggle « visible publiquement »** : e2e-BD du toggle **rejoué VERT** (Docker de nouveau OK) →
     BACK `feat/etablissement-public-visibility` (`c85a641`) **MERGÉ+POUSSÉ** (`86092e3`) : `PATCH
     /etablissements/:id` (`estVisiblePublic` DTO+whitelist) + `GET /etablissements/:id` owner-scopé ;
     FRONT `feat/public-visibility-toggle` (`5c5b795`) **MERGÉ+POUSSÉ** (`57562c5`) : interrupteur
     page paramètres établissement (Owner Hub). Working tree front **restauré** sur
     `feature/pos-offline-o1`. Suites finales : **271 unit / 33 e2e-BD** vertes ; typecheck back+front VERT.
   - **Ensuite** : M1-c/M1-d (comptes clients + commandes) sur la base de cette API.
1. **M1 images** : PR #58 + #59 mergées ; **migration base64→fichiers EXÉCUTÉE sur `aio`**
   (9 images, backup fait). Reste : vérif visuelle front (URLs `/media` relatives ; trancher
   `PUBLIC_MEDIA_BASE_URL` absolue si le front/desktop charge les images hors proxy).
2. **POS offline** : #57 mergée. Ouvrir + merger la **PR front** (`feature/pos-offline-o1`,
   compare main...feature/pos-offline-o1) — à la main (gh non collaborateur du repo front).
3. Relire + merger `test/phase-4-e2e` (e2e-BD Phase 4). Relancer via `pnpm test:e2e:db` (Docker requis).
3. Vérif visuelle de `/owner/audit-log` et `/owner/comparaison` (backend lancé + login partenaire).
4. Poursuivre la traîne `any` Phase 3 (`hooks/` puis `app/`).
5. (Phase 5) reprise des paires front reportées seulement si une vérif visuelle / e2e front
   est en place. Contrôleur employes : écarté (voir ci-dessus), ne pas re-tenter.
6. (POS offline) O3 multi-caisses durci — optionnel, surtout de la preuve concurrente e2e.

## Décisions ouvertes / à trancher
- Le webhook Stripe existe-t-il ? (à confirmer si pas déjà tranché en Phase 1)

## Pièges connus
- Multi-repos : front et back séparés. Parallèle OK seulement entre repos différents.
- Ne PAS laisser subsister l'ancien `.claude/` du backend → source de vérité unique = ce repo pilotage.
- Migration d'index stats non appliquée → endpoints stats lents tant que non déployée.
- Backend : login via LocalAuthGuard, pas JWT (traité Phase 1).
- Frontend : `ignoreBuildErrors` RETIRÉ (build type-checké). Reste ~233 `any` côté `app/`+`hooks/` (traîne Phase 3).
- Frontend : réponses backend faiblement typées → passer par `lib/raw.ts` (asRecord/asString/asNumber/asArray), pas de `any`.
- Prisma 5.7 : un `where` sur une relation **to-one** dans un `include` est **silencieusement
  ignoré** (pas d'erreur) → tout filtrage d'appartenance écrit ainsi est inopérant. Filtrer au
  niveau requête (`where: { relation: { champ } }`) ou comparer explicitement après chargement.
- Phase 4 : seam `AuditJournalService.record(tx, …)` (module global) pour journaliser une action
  sensible DANS sa transaction. Table = `historique`.
- Phase 5 : `categories` (SM/resto) NE se fusionne PAS — divergence métier réelle (SM plus
  riche : `includeInactive`, `userId`, `reorder`, `toggle` ; signatures + corps différents).
  Reporté volontairement, ne pas re-tenter sans refonte.
- Phase 5 : patron de dédup = base abstraite `common/<module>/` + objet `Scope` (discriminant) +
  sous-classes minces gardant leur nom de classe (token d'injection). Réutiliser ce patron.
- Phase 5 : les CONTRÔLEURS NestJS ne se dédupent pas proprement quand les DTO diffèrent —
  `@Body()` exige une classe DTO concrète pour la validation ; un générique/structurel casse la
  validation. Garder les contrôleurs par verticale.
- Front : même patron via fabriques (`makeXApi`/`makeXHooks`) + wrappers minces préservant les
  noms exportés + clés de cache React Query. Pas de vérif visuelle → s'en tenir aux paires
  réellement identiques (seul `fournisseur` l'était).
- e2e-BD : `pnpm test:e2e:db` (Docker) monte un postgres jetable (port 5434, base `aio_e2e`),
  applique les migrations, tourne les specs `test/e2e-db/*.db-e2e-spec.ts`, détruit le conteneur.
  Garde-fou `setup-env.ts` : refuse de tourner sans `TEST_DATABASE_URL` et refuse `aio`. Ne JAMAIS
  faire écrire les e2e sur la BD dev.
- Règle permanente : aucune trace de Claude/IA dans le git.
- **Incident 2026-07-03 — disque `C:` plein (0 o)** : a fait planter `tsc` en OOM par intermittence
  ET fait passer le **FS interne de Docker en lecture seule** → `docker run/rm` échouent
  (`read-only file system` sur overlay2), un conteneur jetable **`aio-e2e-pg` est resté bloqué**
  sur le port 5434 (non supprimable). Libérer le disque hôte ne suffit PAS : il faut **redémarrer
  Docker Desktop** pour que le FS de sa VM redevienne inscriptible, puis `docker rm -f aio-e2e-pg`
  (nom exact). Après ça, `pnpm test:e2e:db` remarche. `aio-db` (5432) est resté intact.
  Astuce : lancer `tsc` avec `NODE_OPTIONS=--max-old-space-size=2048 --max-semi-space-size=64`
  si la RAM libre est basse.
- **Base dev `aio`** : tourne dans Docker → conteneur **`aio-db`** (`postgres:17-bookworm`, glibc 2.36
  pour matcher le volume, sinon avertissement/blocage de collation), port **5432**, volume
  `70225bc2a0ae…`. Restaurée le 2026-07-02 après qu'un `docker rm` à filtre large a retiré l'ancien
  conteneur (données saines dans le volume). **RÈGLE ABSOLUE** : jamais de `docker rm/prune/volume rm`
  sur filtre large (`name=aio`, `$(docker ps -aq…)`), jamais de suppression conteneur/volume/base
  sans demande explicite. Nettoyage e2e = uniquement le nom exact `aio-e2e-pg`.