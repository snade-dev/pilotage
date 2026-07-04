# État All-in-One — 2026-07-04 (soir)

> Curseur du projet. Ne contient que le PRÉSENT et le futur immédiat.
> Historique des jalons terminés → ROADMAP.md (coché) + SESSIONS/ (détail).
> Repos : pilotage · back (NestJS) · front-gestion (Next.js) · app-cliente (Expo).

## Vue d'ensemble
| Phase / Chantier | Repo | État |
|---|---|---|
| 0–2 (hygiène, sécurité, tests) | front+back | ✅ |
| 3 Qualité de type | front | 🟡 traîne `any` hooks/+app/ |
| 4 Robustesse données | back | ✅ mergé + e2e |
| 5 Duplication (ciblée) | front+back | ✅ |
| 6 · POS offline | back+front | ✅ mergé |
| 6 · App cliente M1-a/b (images, API publique, catalogue) | back | ✅ |
| 6 · App cliente — front Expo (Étape 1+2 consultation) | app | ✅ commité sur main |
| 6 · M1-c (comptes tél.) + M1-d (commande retrait) — BACKEND | back | ✅ mergés main, migration appliquée en local |
| 6 · App cliente Étape 3 (compte, panier, commande, suivi) | app | ✅ testé Expo Go, **mergé main** (479222f) |
| 6 · Feed public `/public/v1/feed` (accueil produits) | back | ✅ mergé main (a5bbf63) |
| 6 · Kanban « Commandes en ligne » (maquette 2j) | front-gestion | ✅ mergé main (06e2a8c) |

## En cours / en attente
- [ ] **Push des 4 repos** vers leurs remotes (merges locaux seulement — risque disque).
- [ ] Hub : diff local NON commité sur `src/lib/api.ts` (assainissement messages d'erreur
      HTML dans les toasts, session antérieure) — à relire puis commiter ou jeter.
- [ ] Migration `add_compte_client_telephone` : appliquée en LOCAL, à déployer en prod
      avec le prochain déploiement backend.
- [ ] Traîne `any` Phase 3 (front) — tâche de fond.

## Prochaine action
1. Pousser les 4 repos.
2. Sourcer le fournisseur SMS/WhatsApp (impl prod `VerificationCodeSender`) + démarches
   Wave/Orange (bloque M2).
3. Backend (améliorations relevées) : exposer le `code` d'erreur (filtre global l'avale),
   valider `creneauRetrait` futur, porter le nom d'établissement sur la commande client.

## Décisions ouvertes
- Majoration « application mobile » : le prix public inclut-il la majoration, ou le prix brut ?

## Pièges ACTIFS
- App Expo : URLs d'images à consommer en ABSOLU (dépend du fix images backend mergé).
- Erreurs backend côté app : PAS de champ `code` (avalé par le filtre) → statut + message.
- Multi-repos (4) : parallèle seulement entre repos différents.
- Prisma 5.7 : `where` sur relation to-one dans `include` silencieusement ignoré.
- Règle permanente : aucune trace de Claude/IA dans le git.
- docker rm/prune : jamais de filtre large (incident conteneur passé).
