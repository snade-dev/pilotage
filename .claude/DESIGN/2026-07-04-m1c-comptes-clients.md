# Design — M1-c : comptes clients (auth téléphone vérifiée)

> **Statut : VALIDÉ** (2026-07-04). Conception approuvée par le propriétaire.
> Implémentation sur la branche backend `feat/m1c-comptes-clients` (partant de
> `feat/m1d-commandes-retrait` pour disposer de `@CurrentClientId` + retrait).

Repo : `all-in-one-backend` (NestJS + Prisma + PostgreSQL). Jalon M1-c du chantier
app cliente. Prérequis externe (fournisseur SMS/WhatsApp) **contourné** par une
interface d'envoi + impl dev qui LOGUE le code.

## Décisions actées
- Compte client **grand public**, **séparé** des comptes employés (aucune permission RBAC).
- Auth par **téléphone** + **code de vérification** (SMS/WhatsApp). Pas d'email/mot de passe requis.
- Compte **GLOBAL** (non lié à un établissement) ; c'est la `Commande` qui porte l'établissement.
- Relié à la fiche `ClientFidelite` interne, sans casser la fidélité gérée par le commerçant.
- Catalogue consultable **sans compte** ; compte requis pour **commander** (alimente `@CurrentClientId`).

## Choix structurant : réutiliser `Client` (pas de table parallèle)
Faits vérifiés dans le code :
1. `UserType.CLIENT` existe déjà ; `JwtStrategy.validate` gère la branche client.
2. `@CurrentClientId` (`common/decorators/current-client-id.decorator.ts`) renvoie
   `user.id` pour un `CLIENT` — **le branchement M1-d est déjà en place**.
3. `Commande.clientId`, `Favori`, `Panier`, `AdresseClient` pointent sur `Client`.
4. Paire access/refresh (`signAuthTokens`, `/auth/refresh`) réutilisable telle quelle.

⇒ M1-c **n'introduit pas** un nouveau système d'auth : il **durcit l'obtention du
token client** (téléphone vérifié au lieu de email+mdp). Le décorateur et la
signature `(clientId, …)` des services **ne changent pas**. Le mode email/mdp
existant (`signup-client`/`login-client`, app gestion) **coexiste** sur la même table.

## Modèle (migration additive `add_compte_client_telephone`)
- `Client` : `email` et `motDePasse` passent **nullable** (compte téléphone sans
  email/mdp) ; ajout `telephoneVerifie Boolean @default(false)`,
  `dateVerificationTelephone DateTime?`. `telephone @unique` inchangé (identifiant principal).
- Nouveau `CodeVerificationTelephone` : `telephone`, `codeHash` (bcrypt, jamais en
  clair), `canal` (SMS|WHATSAPP), `expiresAt`, `tentatives`, `maxTentatives`,
  `consommeAt` (single-use), `createdAt`. Index `(telephone, createdAt)`.
- `ClientFidelite` : ajout `clientId String?` + FK optionnelle + index — rempli au
  1er passage du compte dans l'établissement ; nullable ⇒ **ne touche aucune fiche existante**.

Migration 100 % additive / assouplissante (aucune suppression, aucune contrainte durcie).

## Flux & endpoints
Flux **téléphone-first unique** : inscription et connexion indistinguables ⇒ réponses
neutres par construction. Routes OTP `@Public()` + throttle. Préfixe `/v1/auth/client/*`
(aligné sur `/v1/commandes-retrait`).

| Endpoint | Entrée | Sortie | Protections |
|---|---|---|---|
| `POST /v1/auth/client/demarrer` | `{ telephone }` | neutre : « Si ce numéro est valide, un code a été envoyé. » | throttle IP, cooldown 60 s/numéro, cap quotidien |
| `POST /v1/auth/client/renvoyer-code` | `{ telephone }` | idem (même handler) | idem |
| `POST /v1/auth/client/verifier` | `{ telephone, code }` | succès : `{ accessToken, refreshToken, expiresIn, client }` ; échec : `{ code:'CODE_INVALIDE' }` | throttle, `tentatives<max`, single-use |
| `POST /auth/refresh` | `{ refreshToken }` | nouvelle paire | **existant, inchangé** |
| `GET /v1/auth/client/profil` | — (`@ClientOnly`) | profil | Jwt+UserType globaux |
| `PATCH /v1/auth/client/profil` | `{ nom?, prenom?, email?, newsletterOptin? }` | profil MAJ | idem |

**demarrer** : normaliser tél. (E.164) → invalider codes actifs → générer 6 chiffres →
`codeHash`+`expiresAt` (5 min) → `VerificationCodeSender.envoyer` (dev = log) → réponse neutre.
**verifier** : dernier code non consommé → vérifier expiration, `tentatives<max`,
`bcrypt.compare` → échec : `tentatives++`, erreur neutre → succès : `consommeAt=now`,
find-or-create `Client` par `telephone` (`telephoneVerifie=true`), `signAuthTokens({ id, userType:CLIENT })`.

## Session / token client & `@CurrentClientId`
- Même infra JWT que le staff, discriminée par `userType:CLIENT`. **Secret unique**
  (userType discrimine), cohérent avec la séparation partenaire/utilisateur existante
  et le refresh/proxy Next. Pas de second secret.
- `@CurrentClientId` renvoie déjà `user.id` ⇒ **aucune modification**. Le « branchement »
  = un client vérifié reçoit désormais un token, que la route retrait consomme.
- Séparation client/employé : `@ClientOnly` vs `@StaffOnly` via `UserTypeGuard` global ;
  `permissions:[]` ⇒ `PermissionsGuard` refuse toute route à permission. Token staff sur
  route client → 403 et inversement.

## VerificationCodeSender (interface + dev + garde-fou prod)
- `interface VerificationCodeSender { envoyer(telephone, code): Promise<void> }` + token Symbol.
- Impl dev `LogVerificationCodeSender` : LOGue le code. **Garde-fou dur** : throw si
  `NODE_ENV === 'production'` (constructeur + `envoyer`).
- Sélection par env : en prod sans provider SMS configuré ⇒ **échec au bootstrap**
  (fail-fast), jamais de fallback log silencieux. L'impl réelle s'injecte plus tard
  sous le même Symbol, sans toucher au flux.

## Rattachement fidélité (`ClientFidelite`, par établissement)
Fiche = clé `(établissement, telephone)`, un compte global = un téléphone ⇒ 1-à-N.
**Pas de fiche à l'inscription** (aucun établissement). Rattachement **à la commande**
(dans le service retrait M1-d) : `findFirst(établissement, telephone)` →
**trouvée** : relier (`clientId` si null), champs commerçant **préservés** →
**absente** : créer fiche minimale (`établissement, telephone, nom/prenom, clientId`) →
poser `Commande.clientFideliteId`. Match téléphone exact, jamais de doublon, jamais d'écrasement.

## Anti-abus
- Throttle : `demarrer`/`renvoyer` strict (IP) + cooldown 60 s/numéro + cap quotidien
  (coût SMS) ; `verifier` ~5/min.
- Code : 6 chiffres, TTL 5 min, max 5 essais, single-use, anciens invalidés à chaque envoi.
- `codeHash` bcrypt au repos. Réponses neutres (demarrer ne révèle jamais l'existence ;
  verifier ne distingue pas faux/expiré). Normalisation E.164.

## Tests
- Unit : génération/hash/vérif de code (succès, faux, expiré, essais épuisés, single-use) ;
  find-or-create + rattachement fidélité (crée/relie, ne casse pas les champs commerçant).
- e2e-BD : `demarrer` → code (log) → `verifier` → paire de tokens → route client protégée
  (`/v1/commandes-retrait`) **accepte** le token client et **refuse** un token employé, et
  inversement ; throttle actif ; réponse neutre si numéro déjà connu ; garde-fou prod.

## Périmètre exclu (reporté)
- Impl réelle SMS/WhatsApp (fournisseur non sourcé) — se branche sous le même Symbol.
- Fusion des deux modes d'auth client (email/mdp vs téléphone) : ils coexistent.
