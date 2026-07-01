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
| Module Statistiques (frontend) | front | 🟡 en relecture (branche non mergée) |
| 3 Qualité de type | front | ⬜ à venir (après merge du front stats) |
| 4 Robustesse données | back | ⬜ prompt prêt, à lancer |
| 5 Duplication | front + back | ⬜ |
| 6 Fonctionnalités | front + back | 🟡 stats = 1re (en cours) |

## En cours / en attente
- [~] **Page Statistiques Supermarché (front)** — branche `feat/stats-supermarche-frontend`,
      en attente de relecture. Route `/statistiques-ventes` : KPIs CA net + évolution, courbe
      recharts, 2 classements complets triables/paginés, export CSV. **Mono-établissement**
      (bascule consolidé retirée → sera traitée dans le Hub Propriétaire). Filtre par année
      corrigé (sélecteur d'année quand granularité=année). Typé strict, Vitest vert (23 tests).
      Voir SESSIONS/2026-07-01-stats-supermarche-frontend.md + 2026-07-01-stats-front-ajustements.md.
- [ ] ⚠️ **Migration d'index stats backend NON appliquée en base** (`prisma migrate deploy`).

## Fait (validé + mergé)
### Backend
- [x] Phase 0 Hygiène — SESSIONS/2026-07-01-phase-0-1.md
- [x] Phase 1 Sécurité (auth globale, CORS, secrets, throttler, Helmet) — SESSIONS/2026-07-01-phase-0-1.md
- [x] Phase 2 Tests « argent » (105 unit + 8 e2e guards) — SESSIONS/2026-07-01-phase-2-money.md
- [x] Module Statistiques agrégation (4 endpoints ; scope établissement/consolidé ; CA net ;
      pagination/tri ; 134 unit + 12 e2e) — SESSIONS/2026-07-01-stats-supermarche.md
### Frontend
- [x] Phase 2 socle Vitest (3 fichiers, 15 tests : totaux document, promotions, TVA)

## Prochaine action
1. Relire + merger la page stats frontend (`feat/stats-supermarche-frontend`).
2. Appliquer la migration d'index stats backend en base.
3. Lancer la Phase 4 backend (chantier parallèle possible pendant la relecture front).
4. Une fois le front stats mergé → démarrer la Phase 3 (qualité de type) sur le repo front.

## Décisions ouvertes / à trancher
- Le webhook Stripe existe-t-il ? (à confirmer si pas déjà tranché en Phase 1)

## Pièges connus
- Multi-repos : front et back séparés. Parallèle OK seulement entre repos différents.
- Ne PAS laisser subsister l'ancien `.claude/` du backend → source de vérité unique = ce repo pilotage.
- Migration d'index stats non appliquée → endpoints stats lents tant que non déployée.
- Backend : login via LocalAuthGuard, pas JWT (traité Phase 1).
- Frontend : `ignoreBuildErrors: true` encore actif + ~247 `any` (chantier Phase 3).
- Règle permanente : aucune trace de Claude/IA dans le git.