# Feuille de route — All-in-One (front + back)

> Plan global. Change rarement. Le curseur « où on en est » vit dans ETAT.md.
> Dernière mise à jour : 2026-07-02

## Principe directeur
D'abord ne rien casser ni fuiter (sécurité + tests sur l'argent), ensuite consolider
(qualité, dette), enfin faire grandir (fonctionnalités). Deux chantiers ne se mènent en
parallèle QUE s'ils sont dans des repos différents et sans dépendance d'ordre.

| Phase / Chantier | Repo | État |
|---|---|---|
| 0 Hygiène | front + back | ✅ |
| 1 Sécurité backend | back | ✅ |
| 2 Tests « argent » | front + back | ✅ |
| Module Statistiques (back + front) | front + back | ✅ mergé |
| Comparaison établissements (back + front) | front + back | ✅ mergé |
| 3 Qualité de type | front | 🟡 Lot 0+1 mergés ; traîne `hooks/`+`app/` |
| 4 Robustesse données | back | ✅ mergé + e2e-BD verts |
| 5 Dette de duplication | front + back | ✅ mergé (ciblé ; reste reporté volontairement) |
| 6 Fonctionnalités | front + back | 🟡 en cours |
| ↳ POS offline (O0→O2c) | back + front | ✅ mergé (O3 multi-caisses optionnel) |
| ↳ App de commande client | front + back | 🟡 M1-a fait ; M1-b→M4 à venir |

---

## PHASE 3 — Qualité de type (frontend)
**But :** réactiver la protection du compilateur, sans masquer (pas de `as any`/`@ts-ignore`).
- [x] `ignoreBuildErrors` retiré (build type-checké), `lib/` 0 `any` (helper `lib/raw.ts`).
- [ ] Traîne restante : ~233 `any` côté `hooks/` puis `app/`.
**Critère de sortie :** `next build` échoue sur une erreur de type ; Vitest reste vert.

## PHASE 4 — Robustesse des données (backend) — ✅
Transactions atomiques, isolation multi-tenant (faille B1 corrigée), index, journal d'audit
(`historique`). e2e-BD verts (isolation/atomicité/audit). **Prérequis de l'app client : levé.**

## PHASE 5 — Dette de duplication — ✅ (ciblée)
Fusion des modules jumeaux via patron `common/<module>` + `Scope`. taux-tva, fournisseur,
employes, code-promo (partiel) fusionnés ; categories + contrôleurs + 6 paires front **reportés
volontairement** (divergence réelle). Ne pas re-tenter sans refonte.

## PHASE 6 — Fonctionnalités (croissance)
**But :** faire grandir le produit sur un socle fiable. Chaque feature arrive avec ses tests.
- [x] Module Statistiques (back + front).
- [x] Comparaison des établissements (Hub Propriétaire).
- [x] **POS offline** (O0→O2c) mergé et prouvé (e2e-BD 14/14). O3 multi-caisses = optionnel.
- [ ] **App de commande client** — voir chantier dédié ci-dessous.
- [ ] Alertes intelligentes (ruptures, péremptions, écarts d'inventaire — réutilise `ECART_STOCK_*`).
- [ ] Export comptable conforme aux pays cibles.

---

## CHANTIER — App de commande client (All-in-One)

> Le client final commande depuis son téléphone. Direction visuelle retenue : **1b (Frais épuré,
> vert)**. Design doc : `.claude/DESIGN/DESIGN-M1-api-publique-comptes-clients.md`.
> **Périmètre M1 (acté 2026-07-02) : Supermarchés ET restaurants, recherche cross-catalogue,
> retrait au comptoir. Livraison repoussée en M3.**

### Prérequis
- ✅ Phase 4 mergée + prouvée (atomicité/isolation) — prérequis levé.
- 🕒 **Démarches externes à lancer EN PARALLÈLE dès maintenant** (délais hors code) :
  compte marchand + API **Wave / Orange Money** (pour M2) ; fournisseur **SMS / WhatsApp**
  (pour la vérification téléphone de M1-c).
- ✅ Stockage d'images (M1-a) fait.

### Décisions actées
- Catalogue **consultable sans compte** ; compte requis seulement pour commander.
- **Compte client grand public** (auth par **téléphone** + vérification), séparé des employés,
  relié à la fiche fidélité interne. Compte GLOBAL (non lié à un magasin) ; la commande porte l'établissement.
- API publique **séparée et versionnée** (`/v1`), routes catalogue `@Public()`, throttling renforcé.
- Stock affiché **Disponible / Épuisé** (jamais la quantité exacte).
- **Décrément du stock au RETRAIT** (pas à la commande).
- **URLs d'images ABSOLUES** (`PUBLIC_MEDIA_BASE_URL`) : l'app mobile appelle depuis un autre
  domaine, les URLs `/media` relatives ne fonctionneraient pas.
- M1 couvre **SM (produits) ET resto (plats avec options/menus)** — factoriser via patron Scope.

### Jalons

**M1 — Catalogue SM+resto, recherche, commande retrait, paiement comptoir**
- [x] **M1-a Stockage d'images** — `ImageStorageService` (dossier local) + miniatures `sharp`
      + `/media` ; bascule création produit SM + plat resto ; migration base64→fichiers exécutée.
      *(Reste : vérif visuelle front, à plier dans M1-b avec les URLs absolues.)*
- [ ] **M1-b API publique catalogue** — endpoints publics versionnés (magasins visibles,
      produits/plats d'un établissement, détail), **recherche cross-catalogue** orientée produit
      (tous les établissements visibles), URLs d'images **absolues**. Couvre SM + resto.
      ⚠️ Recherche = index texte (pg_trgm/tsvector), PAS un `LIKE` non indexé (scalabilité).
- [ ] **M1-c Comptes clients** — modèle compte client, auth par téléphone + vérification
      (SMS/WhatsApp), rattachement fiche fidélité.
- [~] **M1-d Commande retrait** — **BACKEND fait** (branche `feat/m1d-commandes-retrait`,
      tests verts, non commité) : création de commande (SM + resto) côté client + file
      commerçant (RECUE→EN_PREPARATION→PRETE→RETIREE, +REFUSEE/ANNULEE), décrément stock +
      impact stats au retrait, écart négatif toléré (`ECART_STOCK_RETRAIT`). Réutilise `Commande`.
      Design : `.claude/DESIGN/2026-07-04-m1d-commandes-retrait.md`.
      **Reste** : écrans app (après M1-c) ; menus/options en ligne de commande (reporté) ;
      écrans d'exception (article épuisé, refus, panier vide, créneau expiré).

**M2 — Paiement en ligne** (Wave / Orange Money ; le client choisit payer maintenant ou au retrait).

**M3 — Livraison** *(jalon LOURD, pas une simple option — cadrer par son propre design doc)*
- [ ] Adresses de livraison (adressage informel ouest-africain : repères / GPS).
- [ ] Zones de livraison par établissement + frais de livraison.
- [ ] Statut « en livraison » dans le suivi.
- [ ] **Livreurs DÉDIÉS** : affectation, suivi de course, statuts — potentiellement une app
      livreur séparée. Sous-système à part entière.

**M4 — Fidélité & finitions** — lien fidélité approfondi, historique, notifications.

### Séquencement
M1-b → M1-c → M1-d (M1-c et M1-a sont partageables ; M1-d touche stock/stats → sous garanties Phase 4).
Ne pas mener de front deux gros tracks (ex. app client et O3 offline). Traîne `any` Phase 3 =
tâche de fond parallélisable côté front.