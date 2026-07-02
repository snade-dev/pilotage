# État du projet all-in-one — dernière mise à jour : 2026-07-01

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
| 4 Robustesse données | back | ✅ **PR #55 mergée sur main** ; migrations déployées ; reste e2e à passer |
| 5 Duplication | front + back | 🟡 **back MERGÉ (`159348c`)** : 4 fusionnés + 1 mort + categories reporté · **front MERGÉ (`8ae73ca`)** : fournisseur (data+hooks) fusionné, 6 autres paires reportées (divergence réelle) |
| 6 Fonctionnalités | front + back | 🟡 stats = 1re (en cours) |

## En cours / en attente
- [x] Migrations backend déployées (stats + `phase4_hot_path_indexes` + `phase4_audit_journal`).
- [x] Phase 4 : journal d'audit branché sur remboursements (3), modification de prix
      (produit + plat), changement de permissions (SM + resto), annulation de commande (SM + resto).
- [x] Phase 4 : PR #55 mergée sur main.
- [ ] Phase 4 : e2e à passer (écrivent sur BD dev).

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
1. Passer les e2e Phase 4 (écrivent sur BD dev) pour valider bout-en-bout.
2. Vérif visuelle de `/owner/audit-log` et `/owner/comparaison` (backend lancé + login partenaire).
3. Poursuivre la traîne `any` Phase 3 (`hooks/` puis `app/`).
4. (Phase 5) reprise des paires front reportées seulement si une vérif visuelle / e2e front
   est en place. Contrôleur employes : écarté (voir ci-dessus), ne pas re-tenter.

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
- Règle permanente : aucune trace de Claude/IA dans le git.