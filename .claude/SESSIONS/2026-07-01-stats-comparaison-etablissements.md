# Session 2026-07-01 — Comparaison des établissements (backend)

Chantier 2 du module Statistiques. Branche `feat/stats-comparaison-etablissements`.

## Objectif
Comparer les supermarchés d'un même propriétaire : détail PAR établissement (group by),
pas la somme du scope `consolide` existant. Alimente un tableau + un graphique multi-courbes
dans le Hub Propriétaire (frontend traité dans un chantier suivant).

## Réalisé
- 1 endpoint `GET /supermarche/statistiques/comparaison?from&to&granularite`, protégé par
  `VOIR_RAPPORTS_CONSOLIDES` (les partenaires/super-admins passent nativement le guard).
  Bornage strict aux établissements du propriétaire (`partenaireId`).
- Réponse unique : `etablissements[]` (KPIs : caNet, caBrut, remboursements, nbVentes,
  panierMoyen, évolution % vs période précédente) + `series[]` (par établissement,
  points `{periode, caNet}` alignés sur les mêmes buckets).
- Factorisation : extraction de `calculerKpisPeriode` (cœur CA net + comparaison) réutilisé
  par `getResume` (comportement inchangé) et par la comparaison ; séries réutilisent
  `getChiffreAffaires`. Aucun calcul dupliqué.
- Granularité `mois | annee` confirmée correcte : le filtrage se fait uniquement sur
  `from/to` ; la granularité n'affecte que le bucketing d'affichage. Le backend stats
  n'est donc PAS la cause du bug « filtre par année » signalé côté produit (à chercher
  ailleurs, hors de ce module).

## Tests
- Unitaires : détail par établissement, CA net après remboursement, évolution vs période
  précédente, cas limites (établissement sans vente, un seul établissement, granularité année),
  isolation tenant. Suite stats : 34 unit.
- Intégration (e2e) : refus sans `VOIR_RAPPORTS_CONSOLIDES` (403), 200 avec la permission,
  200 partenaire, 401 sans token.
- Non-régression : 139 unit + 16 e2e verts.

## Reste à faire
- Frontend Hub Propriétaire (tableau + graphique multi-courbes).
- Migration d'index stats toujours NON appliquée en base (rappel du chantier précédent).
