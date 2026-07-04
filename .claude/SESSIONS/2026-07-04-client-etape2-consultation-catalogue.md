# App cliente — Étape 2 : consultation du catalogue (2026-07-04)

Repo : `all-in-one-client` (Expo SDK 57, expo-router, TanStack Query, TS strict).
Branche : `feat/etape2-consultation-catalogue`. **Non commité** (attente relecture).
Consomme UNIQUEMENT l'API publique `/public/v1`.

## Périmètre livré (lecture seule, sans compte)
- Navigation à **onglets bas** : « Magasins » + « Recherche » (groupe `app/(tabs)/`).
  Le catalogue et le détail item sont poussés au-dessus des onglets (Stack racine).
- **Catalogue d'un établissement** (`app/etablissement/[id]/index.tsx`) : en-tête (nom +
  badge type + adresse), chips de catégories (filtre), grille 2 colonnes virtualisée
  (`FlatList numColumns=2`), défilement infini (`useInfiniteQuery`, pagination maison),
  prix FCFA + pastille DISPONIBLE/EPUISE, mini-badges régime + temps de prépa pour les plats.
- **Détail item** (`app/etablissement/[id]/[itemId].tsx`), SM produit ET resto plat :
  carrousel d'images (placeholder si aucune), prix, dispo, description, variantes (puces,
  SANS prix propre — l'API n'expose pas de prix par variante), allergènes ; pour un plat :
  régimes, ingrédients, accompagnements, calories/poids/temps de prépa. Détection resto =
  présence des champs spécifiques dans la réponse.
- **Recherche cross-catalogue** (`app/(tabs)/recherche.tsx`) : barre + debounce 350 ms,
  seuil ≥ 2 caractères (contrainte backend), résultats groupés par item logique
  (fourchette de prix, nb d'offres), dépliables en offres multi-établissements
  (logo, prix, dispo, **distance** si dispo). Tap sur une offre → détail item.
  Distance activée via **expo-location** (permission foreground demandée à l'ouverture
  de l'onglet ; refus/indisponible → `distanceKm` absent, recherche fonctionne sans).
- **États** partout : squelettes (liste / grille / détail), vide, erreur réseau + réessayer,
  invite de recherche vs aucun résultat, pied de liste (page suivante / relance).

## Points de contrat importants (relevés dans le backend, à retenir)
- **Les variantes publiques n'ont pas de prix** (`PublicVariante` = nom/couleur/image/dispo).
  Le prix est unique par item → affichage d'un seul prix, variantes en puces d'info.
- **Ni « options » tarifées, ni menus-vente** ne sont exposés par l'API publique pour les
  plats. Seuls `ingredients` / `accompagnements` (listes de chaînes, sans prix) sortent →
  affichés en listes informatives, **aucun calcul de prix inventé**.
- **Distance** seulement si l'app envoie `lat`/`lng` à `/recherche` (sinon `null`).
- `imageThumbnailUrl` peut être `null` OU relative (correctif d'URL absolue backend en cours) :
  composant `ItemThumbnail` gère `null` ET l'échec de chargement par le même placeholder.

## Technique
- Types calqués sur `public-output.dto.ts` (ajoutés à `src/api/types.ts`).
- Modules API : `src/api/catalogue.ts`, `src/api/recherche.ts` (réutilisent `getJson`).
- Hooks : `use-etablissement-detail`, `use-catalogue` (infinie), `use-catalogue-item`,
  `use-recherche` (infinie, activée si terme ≥ 2), `use-device-location`, `use-debounced-value`.
- Composants réutilisables : `TypeBadge`, `DisponibiliteBadge`, `ItemThumbnail`,
  `CatalogueItemCard`, `CategoryChips`, `ScreenHeader`, `TabIcon` (icônes pur `View`,
  aucune dépendance d'icônes), `RechercheResultCard`. `screen-states` généralisé (skeletons
  + `ScreenMessage` + `ListFooter`), exports existants conservés.
- Dépendance ajoutée : **expo-location ~57.0.2** (+ plugin dans `app.json`).
- **`pnpm typecheck` VERT** (TS strict, `noUncheckedIndexedAccess`, zéro `any`).

## Non vérifié / à faire côté utilisateur
- **Exécution sur Expo Go (téléphone physique) non faite ici** : rendu réel, permission de
  localisation, chargement des images, filtres et pagination à valider sur l'appareil.
  Rappel : `EXPO_PUBLIC_API_BASE_URL` = IP locale du PC ; CORS `/public/v1` doit autoriser
  cette origine.
- Les types de routes expo-router (`.expo/types`) se régénèrent au `pnpm start`.
- Détail visuel mineur : une grille 2 colonnes avec un dernier item seul l'affiche pleine
  largeur (comportement `flex` accepté pour l'instant).

## État
🟡 Code complet, typecheck vert, **en attente de relecture + test Expo Go**. Non commité.
