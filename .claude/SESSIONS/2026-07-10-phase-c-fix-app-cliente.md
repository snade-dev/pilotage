# App cliente Phase C — Correction des 4 bugs consignés en Phase B (2026-07-10)

Repo : `all-in-one-client` (Expo SDK 57, RN 0.86, React 19.2.3).
Branche : `test/coverage-predeploy` (par-dessus le commit Phase B `b3c10f3`).
**Non commité** (attente relecture — `git status` montré au propriétaire).
Périmètre STRICT : les 4 correctifs ci-dessous + dé-skip des 4 tests associés.
Aucun refactor opportuniste, overrides pnpm reanimated/worklets intouchés.

## Comptes
- Avant : 203 tests (199 verts + 4 skips documentant les bugs).
- Après : **204 tests, 204 verts, 0 skip** (les 4 skips réactivés + 1 test
  ajouté sur le refresh en erreur serveur 500). Suite relancée **2×** :
  résultats identiques, zéro flaky. `npx tsc --noEmit` : **0 erreur**.

## Bug 1 (majeur) — `clientRequestId` partagé retrait/livraison
`app/commande/nouvelle.tsx` : le même UUID (une seule `useRef`) servait aux
mutations retrait ET livraison — un POST retrait abouti côté serveur avec
timeout client, suivi d'une bascule en livraison, pouvait être dédupliqué à
tort sur la commande de retrait.
- Correctif : **deux refs distinctes** (`requestIdRetrait`, `requestIdLivraison`),
  chacune figée à l'ouverture de l'écran. La clé reste STABLE entre les
  tentatives d'un même mode (comportement d'idempotence anti double-tap
  conservé, testé), mais n'est plus partagée entre types de commande.
- Choix : deux refs plutôt qu'une régénération au changement de mode — plus
  simple, et une bascule aller-retour retrait → livraison → retrait garde la
  même clé de retrait (retry protégé même après hésitation).
- Test réactivé, déplacé dans le bloc US-14 (il a besoin de la position
  simulée + du nettoyage des mocks expo-location) et renforcé : 2 tentatives
  de retrait (clé identique entre elles) puis livraison (clé différente).

## Bug 2 (mineur) — « session expirée » à tort sur panne réseau
`src/api/client.ts` : quand le refresh échouait pour cause RÉSEAU, les tokens
étaient conservés (correct) mais l'appelant recevait « Ta session a expiré.
Reconnecte-toi. ».
- Correctif : `rafraichirSession()` renvoie désormais une issue typée
  (`ok` | `session-invalide` | `indisponible` + `ApiError` porteuse de la
  cause) au lieu d'un booléen. `requeteAuth` :
  - `session-invalide` (refresh 401/403 → purge) → message « session expirée »
    inchangé (comportement correct conservé) ;
  - `indisponible` (panne réseau, timeout, erreur serveur passagère) → tokens
    conservés, l'erreur réseau/serveur d'origine est remontée telle quelle
    (« Connexion au serveur impossible… », « Le serveur ne répond pas… »,
    ou le message serveur pour un 5xx).
- Usages vérifiés : aucun écran ne teste le code `SESSION_EXPIREE` ni le
  message — les écrans affichent `error.message` générique (ErrorBanner) et la
  déconnexion visuelle passe par le listener de session (purge des tokens),
  inchangé. Rien à adapter côté écrans.
- Tests : skip réactivé (message réseau + code ≠ SESSION_EXPIREE + tokens
  conservés) ; le test existant « tokens conservés sur panne réseau » reste
  vert ; ajout d'un test refresh en 500 (tokens conservés, message serveur).

## Bug 3 (mineur) — course d'hydratation du panier
`src/providers/cart-provider.tsx` : un AJOUTER dispatché avant la résolution
de `AsyncStorage.getItem` était écrasé par HYDRATER (remplacement complet de
l'état).
- Correctif minimal : ref `touche` levée à la première action utilisateur
  (dispatch enveloppé `dispatchUtilisateur`) ; si l'hydratation arrive après,
  elle est **ignorée** — l'action de l'utilisateur prime. Pas de fusion
  hasardeuse entre un panier abandonné stocké et le panier en cours.
- Persistance gardée cohérente : le drapeau `hydrate` passe de ref à **état**,
  si bien que la fin d'hydratation (appliquée OU ignorée) déclenche la
  sauvegarde de l'état courant — le panier touché avant hydratation est bien
  écrit dans AsyncStorage (assertion ajoutée au test réactivé).

## Bug 4 (mineur) — tap « Commander » pendant le chargement de session
`app/(tabs)/panier.tsx` : `statut !== 'connecte'` englobait `chargement` — un
client connecté qui tapait très tôt partait vers la saisie du téléphone.
- Correctif : pendant `chargement`, le bouton est **désactivé**
  (`PrimaryButton disabled`, style existant) ET `commander()` ignore l'appui
  (ceinture et bretelles). `deconnecte` → détour téléphone inchangé.
- Test réactivé et rendu déterministe : l'ancien skip n'était pas exécutable
  tel quel (panier pas encore hydraté à l'appui, et les deux chargements se
  résolvaient dans la même fenêtre de microtâches). La relecture du trousseau
  est figée derrière une barrière (mock `SecureStore.getItemAsync` restauré en
  `finally`) : appui pendant `chargement` → aucune navigation ; après
  résolution → navigation vers le choix du créneau ; jamais de détour
  téléphone.

## Fichiers modifiés
- `app/commande/nouvelle.tsx` (bug 1)
- `src/api/client.ts` (bug 2)
- `src/providers/cart-provider.tsx` (bug 3)
- `app/(tabs)/panier.tsx` (bug 4)
- Tests : `src/__tests__/ecrans/commande-nouvelle.test.tsx`,
  `src/__tests__/api/client.test.ts`,
  `src/__tests__/providers/cart-provider.test.tsx`,
  `src/__tests__/ecrans/panier.test.tsx`

## Reste à faire
- Relecture + commit de la branche `test/coverage-predeploy` (Phase B + C)
  par le propriétaire — aucun commit effectué.
- Tests manuels Expo Go de la checklist existante (inchangée).
