# App cliente Phase B — Couverture de tests pré-déploiement (2026-07-10)

Repo : `all-in-one-client` (Expo SDK 57, RN 0.86, React 19.2.3, expo-router 57).
Branche : `test/coverage-predeploy`. **Non commité** (attente relecture —
`git status` montré au propriétaire). Périmètre STRICT : fichiers de test +
infra de test (`src/test/`) + `package.json` (devDependencies + bloc `jest` +
script `test`) + `tsconfig.json` (ajout `"types": ["jest"]`, additif, requis
par TypeScript 6 qui n'inclut plus `node_modules/@types` automatiquement) —
**zéro fichier applicatif modifié**, overrides pnpm reanimated/worklets
intouchés. Plan Phase A appliqué : couverture totale US-01→US-14.

## Comptes (avant → après)
- Avant : **0 test** (aucun outillage).
- Après : **20 fichiers / 203 tests** — **199 verts + 4 skips** documentant des
  bugs réels du code applicatif (cf. plus bas). Suite relancée **3×** :
  résultats identiques, zéro flaky (fake timers pour debounce, décomptes,
  auto-refresh et créneaux ; `jest.setSystemTime` fige l'horloge des tests de
  checkout). `npx tsc --noEmit` : **0 erreur**.

## Outillage ajouté
- devDependencies : `jest-expo@57.0.1`, `jest@29.7.0`,
  `@testing-library/react-native@13`, `react-test-renderer@19.2.3` (aligné
  React à l'exact), `@types/jest`.
- Bloc `jest` dans `package.json` : preset `jest-expo` (gère nativement les
  chemins pnpm `.pnpm` et l'alias `@/` du tsconfig), `setupFilesAfterEnv`,
  `clearMocks`, `maxWorkers: 1` (EBUSY aléatoires sous Windows avec plusieurs
  workers), `testTimeout: 30000`.
- `src/test/setup.ts` : `EXPO_PUBLIC_API_BASE_URL` simulée ; doublures
  expo-secure-store (Map), expo-crypto (UUID déterministes `uuid-test-N`),
  expo-location (refus par défaut, surchargeable), expo-image (host
  `ExpoImage` + testID pour tester `onError`), expo-linear-gradient,
  expo-status-bar, expo-font, react-native-webview, AsyncStorage (mock
  officiel), safe-area (mock officiel) ; expo-router → mock manuel pilotable ;
  hygiène par test (cleanup → purge session → purge AsyncStorage → coupure des
  minuteurs réels fuités par le toast).
- `src/test/fetch-mock.ts` : stub `fetch` par ROUTES déclaratives aux formes
  EXACTES du backend (enveloppe `{success,message,data}`, erreur
  `{statusCode,message,path,timestamp}` sans champ `code`, panne réseau,
  timeout par AbortSignal) ; toute requête non déclarée fait échouer le test.
- `src/test/render.tsx` (providers réels Query/Auth/Cart/Toast),
  `src/test/fixtures.ts` (fabriques typées miroir de `src/api/types.ts`),
  `src/test/expo-router-mock.ts` (routeur espion + params par test).

## Couverture par couche
1. **Libs pures** (3 fichiers, node) : `format` (FCFA/NBSP, fourchettes,
   distances), `creneaux` (J/J+1/J+2, marge 30 min, jour fermé → 0 créneau,
   horaires illisibles/inversés → repli 09:00–19:00, ouverture décalée
   arrondie, passage de mois), `statut-retrait` + `statut-livraison`
   (chemins nominaux, annulable, statut actif, tons).
2. **API** : `client.ts` (enveloppe, message d'erreur backend, 429 dédié,
   réponse non-JSON, panne réseau, timeout 15 s, Bearer injecté, 401 →
   refresh single-flight → rejeu, refresh 401/403 → purge + SESSION_EXPIREE,
   panne réseau pendant refresh → tokens CONSERVÉS) ; `session.ts` (trousseau
   illisible → null sans crash, listeners, désabonnement).
3. **Providers** : `cart-provider` (reducer complet : fusion même variante,
   plancher quantité 1, retrait dernière ligne → état vide, REMPLACER, VIDER,
   réhydratation AsyncStorage + stockage corrompu ignoré, sauvegarde après
   hydratation) ; `auth-provider` (chargement → connecté/déconnecté, session
   dormante au trousseau, ouvrirSession amorce le profil, fermerSession purge
   les caches `['client']` en préservant `['public']`, expiration détectée
   hors écran).
4. **Écrans** (13 fichiers) — nominaux ET exceptions, mocks fetch par
   scénario : accueil US-01 (feed/vide/erreur+retry, ajout direct 1 variante →
   toast+panier, N variantes → fiche, épuisé sans bouton), magasins US-02,
   recherche US-03 (< 2 caractères → zéro appel, groupes multi-établissements
   avec fourchette et dépliage, lat/lng seulement si permission accordée,
   distance affichée), catalogue US-04 (filtre catégorie, pagination
   `nextPage`, pastille Épuisé, badges vedette/régimes, placeholder image
   uri null + onError), fiche produit US-05 (présélection 1ʳᵉ variante
   DISPONIBLE, épuisée marquée « — épuisé » non sélectionnable, bouton
   Indisponible, stepper, allergènes), fiche plat US-06 (régimes, ingrédients,
   accompagnements, kcal/min/portion — PAS d'options tarifées, conforme au
   contrat public), connexion US-07 (indicatif +223, `retour` propagé, code
   erroné → champ vidé, renvoi verrouillé 60 s en fake timers, 429), profil
   US-08 (invite/édition PATCH partiel/erreur/déconnexion purge), panier US-09
   (badge Σ quantités, stepper, Vider avec Alert, mono-établissement via
   `useAjoutPanier` : « Vider et remplacer » / « Garder », Commander
   déconnecté → détour téléphone panier INTACT), commande US-10 (créneaux
   selon horaires, jour fermé, créneau sous marge inerte, POST au contrat
   EXACT — items/creneauRetrait ISO/clientRequestId stable entre tentatives —,
   créneau expiré → bannière sans appel, erreur backend → panier conservé),
   suivi US-11 (timeline par statut, code retrait, REFUSEE/ANNULEE + motif,
   annulation + 409, auto-refresh 30 s arrêté au statut terminal, 404/403 →
   erreur propre, déconnecté → aucun appel), historique US-12 (fusion
   retraits+livraisons, pagination `{page,limit,total}`, vide/déconnecté/
   erreur), favoris US-13 (liste/vide/erreur/DELETE au cœur), livraison US-14
   (toggle seulement si `fraisLivraison` exposé, adresse obligatoire, POST
   livraison exact, suivi : code 4 chiffres, carte doublée seulement pendant
   EN_LIVRAISON, position pollée, livreur joignable, REFUSEE, annulation
   + 409, auto-refresh 10 s).

## Bugs réels documentés (tests skippés, code applicatif NON corrigé)
1. `app/commande/nouvelle.tsx` l.53 — le MÊME `clientRequestId` est partagé
   entre les mutations retrait et livraison : un échec retrait suivi d'une
   confirmation livraison peut être dédupliqué à tort côté backend.
   **Majeur** (contrat d'idempotence), probabilité faible.
2. `src/api/client.ts` l.217-224 — panne réseau PENDANT le refresh → message
   « Ta session a expiré » alors que les tokens sont conservés et la session
   valide. **Mineur** (message trompeur, pas de déconnexion réelle).
3. `src/providers/cart-provider.tsx` l.131-145 — un AJOUTER dispatché avant la
   fin de l'hydratation AsyncStorage est écrasé par HYDRATER. **Mineur**
   (fenêtre < 1 s au lancement).
4. `app/(tabs)/panier.tsx` l.26 — `statut !== 'connecte'` inclut `chargement` :
   un client connecté qui appuie sur Commander pendant la relecture du
   trousseau est détourné vers la saisie du téléphone. **Mineur**.

## Non testable automatiquement (→ checklist manuelle Expo Go existante)
Gestes du carrousel d'images (pagination au swipe), clavier réel/autofocus,
carte OSM Leaflet dans la WebView (épingle déplaçable, tuiles), SecureStore
réel (persistance après redémarrage), performances des listes longues,
deep links `retour` de bout en bout à travers la vraie pile expo-router,
appel téléphonique du livreur (`Linking tel:`), toasts animés (rendu vérifié,
animation non).

## Reste à faire
- Relecture + commit de la branche `test/coverage-predeploy` par le
  propriétaire (aucun commit effectué).
- Corriger les 4 bugs ci-dessus puis dé-skipper les 4 tests correspondants.
