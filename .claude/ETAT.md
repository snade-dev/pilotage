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
| 4 Robustesse données | back | 🟡 lots isolation/tx/index/audit faits + audit branché (remb./prix/perms/annulation) ; reste : déployer 2 migrations + PR/merge + e2e |
| 5 Duplication | front + back | ⬜ |
| 6 Fonctionnalités | front + back | 🟡 stats = 1re (en cours) |

## En cours / en attente
- [ ] ⚠️ **3 migrations backend NON appliquées en base** (`prisma migrate deploy`), à déployer ensemble :
      `..._add_stats_supermarche_indexes`, `20260701140000_phase4_hot_path_indexes`,
      `20260701150000_phase4_audit_journal` (cette dernière fait un DROP/ADD de la FK
      `historique_utilisateurId_fkey` — additif, aucune donnée perdue).
- [x] Phase 4 : journal d'audit branché sur remboursements (3), modification de prix
      (produit + plat), changement de permissions (SM + resto), annulation de commande (SM + resto).
- [ ] Phase 4 : PR + merge de `feat/phase-4-data-robustness` ; e2e à passer (écrivent sur BD dev).

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
- [x] Phase 4 Robustesse données — 4 lots sur `feat/phase-4-data-robustness` (146 unit verts) :
      B1 isolation (faille cross-restaurant POS resto CONFIRMÉE + corrigée : `where` sur relation
      to-one `plat` ignoré par Prisma 5.7), B2 atomicité session caisse resto, B3 index hot paths,
      B4 journal d'audit immuable (`historique`) branché sur les 3 remboursements.
      SESSIONS/2026-07-01-phase-4-data-robustness.md

## Prochaine action
1. Déployer les 3 migrations backend en attente (`prisma migrate deploy`).
2. Terminer Phase 4 : brancher le journal d'audit sur prix + permissions ; PR + merge ; e2e.
3. Vérif visuelle de la page `/owner/comparaison` (backend lancé + login partenaire) — jamais faite.
4. Optionnel : poursuivre la traîne `any` Phase 3 (`hooks/` puis `app/`).

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
- Règle permanente : aucune trace de Claude/IA dans le git.