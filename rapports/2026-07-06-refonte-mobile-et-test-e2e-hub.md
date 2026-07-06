# Rapport — Refonte design app mobile & test bout-en-bout du Hub Partenaire

**Date : 6 juillet 2026**

---

## 1. Refonte design de l'app mobile cliente (`all-in-one-client`)

**Statut : terminé et vérifié** (typecheck ✓, bundle Metro Android ✓). Aucune logique métier touchée.

Direction « **Marché vivant** » : identité verte conservée mais plus saturée, plus chaleureuse, plus vivante.

| Zone | Avant | Après |
|---|---|---|
| Palette | Vert sobre `#318454`, gris | Vert vif `#16A05A` + accent mangue `#F4842C`, ombres teintées forêt |
| Accueil | En-tête texte simple | Hero immersif en dégradé jusqu'à la barre de statut, cercles décoratifs, barre de recherche flottante |
| Tab bar | Standard avec bordure | Coins arrondis, ombre flottante, badge panier orange |
| Cartes produit | Bordures grises | Sans bordure, ombres douces, prix en vert vif, effet d'appui (scale) |
| Barre panier | Rectangle vert | Pill flottante en dégradé avec flèche |
| Compte | Encart fidélité plat | Carte de membre en dégradé forêt avec numéro de téléphone |
| Commandes | Bandeau vert plat | En-tête dégradé à coins arrondis, StatusBar claire |
| Panier / fiche produit | Pieds droits, steppers gris | Pieds arrondis flottants, steppers vert doux, boutons pill |

- Seule dépendance ajoutée : `expo-linear-gradient` (incluse dans Expo Go).
- Pour voir : `pnpm start` dans `all-in-one-client` + scan QR.
- Réglages centralisés dans `src/theme/tokens.ts`.

---

## 2. Test bout-en-bout du logiciel de gestion (Hub Partenaire)

**Statut : le parcours complet d'un nouveau commerçant était cassé — corrigé pendant le test, maintenant fonctionnel de bout en bout.**

### Parcours testé (compte de test neuf)

Inscription → vérification email → connexion → sélection établissement → dashboard (checklist onboarding) → fournisseur → produit → moyen de paiement → point de vente → taux TVA → **vente POS encaissée** : ticket #628cf9, 500 + 90 TVA (18 %) = 590 FCFA, stock 24 → 23 ✓

### Bugs bloquants corrigés (7 fichiers front, typecheck ✓)

1. **Bug systémique de contrat** : le front envoyait `supermarcheId` en body/query, le backend le **rejette** (whitelist NestJS `forbidNonWhitelisted` — il le déduit du JWT `selectedEtablissementId`). Cassait en silence : création produit, liste produits, points de vente, moyens de paiement, taux TVA.
   Fichiers : `supermarche-produits.ts`, `use-pos-locations.ts`, `use-pos-payment-methods.ts`, `restaurant-tva.ts`.
2. **Crash du dialog de paiement POS** (« Maximum update depth exceeded ») — dès qu'un moyen de paiement existait, l'encaissement était impossible. Cause : `paymentMethods` recréé à chaque rendu + `fetchConfigs` instable qui refetchait en boucle. Corrigé avec `useMemo` (`payment-section.tsx`).
3. **Vente refusée** : le front envoyait `codeTVA`, le DTO backend n'accepte que `tauxTVAId`. Corrigé (`useCart.ts`, `pos/page.tsx`, `use-pos-session.ts`).

### Décisions produit à trancher

- ⚠️ **`fournisseurId` obligatoire à la création de produit** (schéma Prisma non-nullable) alors qu'un compte neuf n'a aucun fournisseur → **l'étape 2 de la checklist d'onboarding est impossible** sans passer d'abord par Fournisseurs. Options : migration pour le rendre optionnel, auto-création d'un « Fournisseur général », ou réordonner la checklist.
- **SIRET obligatoire** à l'inscription — incongru pour le marché malien.
- Cosmétique : nom produit dupliqué au POS (« Coca-Cola 33cl Coca-Cola 33cl »), ticket affichant l'UUID du moyen de paiement au lieu de « Espèces », « Prix en ligne » faussement « Auto », échec silencieux (sans toast) à la création produit, bouton dev « Suivant (sans validation) » visible sur le signup.

### Passe de correction complète (même jour, après-midi)

Audit systématique de tous les envois `supermarcheId`/`restaurantId` du front, vérifiés un par un contre les DTO backend. Corrigés (typecheck ✓, non-régression vérifiée au POS : TVA chargée, total correct) :

| Fichier front | Problème | Correctif |
|---|---|---|
| `restaurant-tva.ts` | `restaurantId` rejeté en body à la création (les 2 verticales) | plus aucun ID envoyé (JWT fait foi) |
| `use-pos-locations.ts` (restaurant) | GET liste inexistant + POST `/liste` avec `restaurantId` rejeté → points de vente resto cassés | POST `/liste` direct sans ID |
| `use-pos-payment-methods.ts` | `restaurantId`/`supermarcheId` rejetés sur create config + create personnalisé | IDs retirés des bodies |
| `refund-settings.ts` | param inutile en GET ; body `supermarcheId` en PUT | nettoyé |
| `fournisseur-data.factory.ts` | recherche/filtres en **POST avec body** alors que le backend expose des **GET à query param** → recherche fournisseurs cassée (les 2 verticales) | GET `?recherche=`/`?statut=`/`?type=` |
| `supermarche-inventaire.ts` | ajustement de stock : `supermarcheId`+`commentaire`+`notes` rejetés, `raisonGlobale` requise manquante → **ajustements cassés** | payload aligné sur `AdjustStockDto` |
| `restaurant-produits.ts` | `restaurantId` rejeté à la création de plat ; `fournisseurId` requis non envoyé | aligné sur le mapping supermarché |

Constats backend (à traiter côté backend, pas de correctif front possible) :

- **`PUT /supermarche/remboursement-global/politique` n'existe pas** → la sauvegarde de la politique de remboursement dans Paramètres échouera (404) tant que la route n'est pas créée.
- Les GET `configuration-etablissement`, `categories/root`, `moyen-paiement-pos/configurations` **acceptent** l'ID en query (DTO filtre) — pas touchés, mais l'incohérence de convention entre modules est la cause racine du pattern de bug.

### Suite (même jour, soir) — fait

- ✅ **Route backend `PUT /supermarche/remboursement-global/politique`** créée (upsert, supermarché déduit du JWT) + front aligné sur le DTO. Testée : création puis relecture OK.
- ✅ **Onboarding réorganisé** : nouvelle étape « Ajouter ton premier fournisseur » entre l'établissement et le produit (backend `/onboarding/progress` → 6 étapes, + méta front). Le wizard produit marque désormais le fournisseur **requis** (validation étape 1 + lien vers Fournisseurs si liste vide) — fin des 400 silencieux et de l'impasse d'onboarding.
- ✅ Backend rebuildé/redémarré (`node dist/main` — attention : pas de watch, penser à rebuild après chaque modif backend).
- ✅ Tout commité sur `main` des 4 repos (backend, Hub ×2 commits, app mobile, pilotage).

### Défauts cosmétiques — détail (cause exacte + correctif proposé)

| # | Défaut | Cause (fichier:ligne) | Correctif proposé | Effort |
|---|---|---|---|---|
| 1 | Nom produit dupliqué au POS (« Coca-Cola 33cl Coca-Cola 33cl ») | `pos/page.tsx:240` — `name = p.nom + ' ' + v.nom` sauf si `v.nom === "Standard"` ; or la variante par défaut créée par le backend porte le **nom du produit**, donc concaténation en doublon | Ne pas concaténer si `v.isDefault === true` ou `v.nom === p.nom` | 1 ligne |
| 2 | Ticket : UUID du moyen de paiement (« MODE PAIEMENT 1ff613b9-… FCFA ») | `print-receipt-modal.tsx:159` — `paymentMethodLabels[sale.paymentMethod]` est un mapping statique legacy (`cash`, `card`…) alors que `paymentMethod` est désormais l'**ID de config BD** → fallback sur l'UUID brut ; `ReceiptRow` suffixe en plus la valeur par « FCFA » | Stocker le **label** (`nomPersonnalise`) dans la vente locale au moment de l'encaissement (le paiement mixte le fait déjà via `p.label`) et l'afficher ; ne pas suffixer FCFA pour cette ligne | ~10 lignes |
| 3 | Ticket : « Session: N/A » | `print-receipt-modal.tsx:96-102` — le code tente de **parser `sessionId` comme une date** ; c'est un UUID → toujours N/A | Afficher la référence de session ou l'heure d'ouverture (`sessionDetails.startTime`) | ~5 lignes |
| 4 | « Prix en ligne » affiché « Auto: 500 » mais bloque si vide | `product-wizard-form.tsx:503-508` — la validation de l'étape 2 exige le champ, alors que le placeholder promet l'auto-remplissage et que le mapping (`supermarche-produits.ts`) retombe déjà sur `prixHt` si absent | Supprimer la validation « requis » sur `onlinePrice` (vide → `prixHt`) | 3 lignes |
| 5 | Paramètres → Moyens de paiement : type « Inconnu » | `payment-methods-settings.tsx:346` — `config.type?.nom \|\| "Inconnu"` ; `mapBackendConfig` (`use-pos-payment-methods.ts:39-62`) ne reconstruit `type` que si la réponse contient un champ `type` en string, or le backend renvoie la relation sous un autre nom (`typeMoyenPaiementPOS`) | Mapper aussi `item.typeMoyenPaiementPOS?.nom`/`code` dans `mapBackendConfig` (vérifier la forme réelle de la réponse) | ~5 lignes |
| 6 | Bouton dev « Suivant (sans validation) » visible sur le signup | `partner-signup-form.tsx:222-229` — bouton `forceNextStep` non conditionné | L'entourer de `process.env.NODE_ENV === "development"` (ou le supprimer) | 1 ligne |
| 7 | SIRET obligatoire à l'inscription (incongru au Mali) | `signup-step-two.tsx:190-208` + schéma zod (14 chiffres requis) ; côté backend `siret String` **non-nullable et unique** sur Supermarche/Restaurant (`schema.prisma:613/675`) | Décision produit : renommer « N° registre de commerce (RCCM/NIF) » sans format imposé, ou rendre optionnel (→ **migration Prisma** + contrainte unique à revoir) | front : ~10 lignes ; backend : migration |

Ordre suggéré : #1, #4, #6 (triviaux) → #2, #3, #5 (ticket/labels) → #7 (décision produit).

### Reste à faire

1. Corriger les 7 défauts cosmétiques ci-dessus (détail + effort estimé dans le tableau).
2. Uniformiser la convention backend (ID toujours déduit du JWT, jamais accepté en query) pour éviter que le pattern whitelist revienne.
3. Tester la verticale restaurant de bout en bout (les correctifs restaurant sont vérifiés contre les DTO mais pas exercés en runtime, faute d'établissement restaurant de test).

### Environnement / données de test

- Front dev sur `http://localhost:3000`, backend `:3001`, base Docker `aio-db` (intouchée).
- Compte de test : `test.claude.design@gmail.com` (« Marché Test Claude » + 1 produit, 1 fournisseur, 1 caisse, 1 vente) — réutilisable pour les prochains tests.
