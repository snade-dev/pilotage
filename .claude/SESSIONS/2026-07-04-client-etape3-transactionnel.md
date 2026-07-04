# App cliente — Étape 3 : écrans transactionnels (2026-07-04)

Repos touchés : `all-in-one-client` (branche `feat/etape3-transactionnel`, NON commité),
`all-in-one-backend` (main, feed public NON commité), `All-in-One-Partner-Hub` (main,
kanban NON commité). Tout attend relecture + test Expo Go / navigateur.

## Périmètre livré (app cliente, Expo)

### Bloc 1 — COMPTE (M1-c)
- Session : `src/api/session.ts` (tokens dans **expo-secure-store**, jamais en clair),
  `client.ts` étendu (`getJsonAuth`/`postJsonAuth`/`patchJsonAuth`) : sur 401 → **refresh
  silencieux unique** (single-flight) via `/auth/refresh` puis rejeu ; refresh KO → purge +
  déconnexion propre ; panne réseau pendant refresh ne détruit PAS la session.
- `AuthProvider` (statut chargement/connecté/déconnecté, réagit à l'expiration en plein vol).
- Écrans : onglet Compte (invite hero OU profil avatar/initiales + édition nom/prénom/email +
  fidélité + déconnexion), `compte/telephone` (indicatif **+223** par défaut, champ unifié),
  `compte/verification` (6 cases OTP, auto-validation, renvoi minuté 60 s = cooldown serveur,
  encart dev « code dans les logs backend », param `retour` pour reprise de parcours).

### Bloc 2 — PANIER (local)
- `CartProvider` : reducer + **AsyncStorage** (réhydraté au lancement), règle
  **mono-établissement** (ajout depuis un autre magasin → alerte « Vider et remplacer »).
- Onglet Panier avec badge, stepper ±, suppression, vider, total, mention paiement comptoir.
- Ajout depuis fiche (choix variante + quantité, barre collante) et depuis les cartes
  (bouton + ; 1 variante dispo = ajout direct, sinon fiche). **Toast** de confirmation.
- Barre panier persistante sur catalogue + accueil (maquette 2b).

### Bloc 3 — COMMANDE (M1-d)
- `commande/nouvelle` (maquette 2g) : pastilles Jour (3 j), grille Heures générée des
  **horaires publics** de l'établissement (passés grisés, marge 30 min, fermé gér��, repli
  9h–19h si horaires illisibles), encart magasin, récapitulatif, CTA « Confirmer · … ».
- Création `POST /v1/commandes-retrait` avec **clientRequestId idempotent** (expo-crypto),
  détour connexion (panier → téléphone → code → retour créneau), re-vérification
  « créneau expiré » à la soumission, erreurs backend en bannière.

### Bloc 4 — SUIVI
- `commande/[id]` (maquette 2h) : en-tête **vert forêt** + **code de retrait** en évidence,
  timeline reçue → en préparation → prête → retirée (échec = étape rouge ✕ + motif),
  auto-refresh 30 s tant que vivante + pull-to-refresh, annulation client (RECUE/EN_PREP).
- Onglet **Commandes** (maquette 2i) : en-tête vert plein, cartes #code · magasin + pastille
  statut + créneau/date + total, pagination infinie, états vide/déconnecté/erreur.
- Onglets finaux : Accueil · Recherche · Panier (badge) · Commandes · Compte
  (« Magasins » devient un écran poussé depuis l'accueil).

## Hors périmètre initial, ajoutés à la demande
- **Accueil produits cross-magasins** : nouvel endpoint public `GET /public/v1/feed`
  (backend, patron RAW_CAP + mapper public réutilisé, throttle 60/min) + écran Accueil
  (salutation, barre recherche, rail magasins, grille Découvrir avec ajout direct).
- **Partner Hub — « Commandes en ligne »** (maquette 2j) : kanban Reçues → En préparation →
  Prêtes, refus avec motif, « Retirée & encaissée » (stock + vente), refresh auto 30 s,
  nav SM + resto gated `GERER_COMMANDES`. Fichiers : `types/commandes-retrait.ts`,
  `lib/commandes-retrait-data.ts`, `hooks/use-commandes-retrait.ts`,
  `components/commandes-retrait/`, pages `/commandes-retrait(-restaurant)`.
- **Refonte design** guidée par la maquette Claude Design importée (MCP) : tokens `forest`,
  `mist`, `dangerBorder`, ombres `shadows.card/raised`, cases OTP, timeline, pastilles.

## Contrats vérifiés en RÉEL (appels curl, session A)
- M1-c : demarrer (réponse neutre SANS `data`), verifier (session 24h/30j + client),
  profil GET/PATCH, refresh OK avec token client. OTP : 6 chiffres, 5 min, 5 essais,
  5 envois/24 h, cooldown 60 s. ⚠️ Le filtre d'exceptions global AVALE le champ `code`
  des erreurs (`{statusCode,message,path,timestamp}`) → le front branche sur statut+message.
- M1-d client : créer (idempotent vérifié), lister (pagination {page,limit,total}),
  détail, annuler (transitions 409). `codeRetrait` court dans chaque réponse.
- Migration `add_compte_client_telephone` APPLIQUÉE en local (deploy, additive).
- ⚠️ Backend ne valide pas que `creneauRetrait` est futur → validation côté app.
- ⚠️ La commande client ne porte pas le NOM de l'établissement (résolu côté app via la
  liste publique en cache) — champ à envisager côté backend.

## Dépendances ajoutées (app cliente)
`expo-secure-store` (+ plugin app.json), `@react-native-async-storage/async-storage`,
`expo-crypto` (~57.0.0, aligné SDK).

## Vérifié / non vérifié
- ✅ `pnpm typecheck` vert sur les 3 repos ; `expo export` (bundle Android) passe.
- ✅ Feed public testé en réel ; commande de test #00DkLEY créée (kanban).
- ❌ NON testé sur téléphone : rendu visuel, SecureStore réel, clavier, parcours complet
  Expo Go ↔ Partner Hub. À faire par l'utilisateur avant tout commit.

## Reste à faire (étape 3)
- Tests manuels bout-en-bout (les 4 blocs + kanban commerçant).
- Relecture utilisateur → commits (3 repos, messages sans trace IA).
- Sourcing fournisseur SMS/WhatsApp (prod du `VerificationCodeSender`).
