# Workflow OAuth2 de ce serveur d'autorisation

Notes explicatives sur le fonctionnement de `config/SecurityConfig.java` : les deux chaînes de sécurité, les URLs OAuth2, et un exemple de workflow complet avec les valeurs de ce projet.

## 1. Les « URLs OAuth2 »

Ce sont les URLs que Spring Authorization Server crée automatiquement. On ne les écrit pas soi-même : c'est la ligne `.with(authorizationServerConfigurer, ...)` qui les active. Avec la configuration actuelle (paramètres par défaut et OIDC activé), les principales sont :

| URL | À quoi elle sert |
|---|---|
| `/oauth2/authorize` | Point de départ : l'utilisateur se connecte et donne son accord à l'application |
| `/oauth2/token` | L'application échange le code reçu contre un token |
| `/oauth2/jwks` | Publie la clé publique pour vérifier les tokens |
| `/oauth2/introspect` | « Ce token est-il encore valide ? » |
| `/oauth2/revoke` | Annule un token |
| `/.well-known/openid-configuration` | Fiche descriptive du serveur : la liste de toutes ses URLs |
| `/userinfo` | Renvoie les infos de l'utilisateur connecté (OIDC) |
| `/connect/logout` | Déconnexion OIDC |

Il en existe quelques autres moins utilisées. Pour voir la liste réelle, démarrer le serveur et ouvrir http://localhost:9000/.well-known/openid-configuration.

## 2. URLs OAuth2 et « autres URLs »

- **URLs OAuth2** : la liste fixe ci-dessus, fournie par Spring. Ce sont surtout des **applications** (les clients) qui les appellent.
- **Autres URLs** : tout le reste. Dans ce projet, c'est essentiellement **`/login`**, la page où un **humain** saisit son mot de passe, plus `/logout` et `/error`. Ce projet n'a pas d'API métier.

## 3. Pourquoi deux chaînes de sécurité avec `@Order`

**Chaîne 1 : `authorizationServerSecurityFilterChain`, `@Order(1)`**
- Elle ne règle pas seulement les droits d'accès aux URLs OAuth2 : c'est elle qui **crée** ces URLs. Sans elle, `/oauth2/token` n'existerait pas.
- Grâce à `securityMatcher(...)`, elle ne s'applique **qu'à** ces URLs.
- Elle n'a pas de formulaire de login : un utilisateur non connecté est **redirigé** vers `/login`.

**Chaîne 2 : `defaultSecurityFilterChain`, `@Order(2)`**
- Elle s'applique à toutes les autres URLs.
- Son rôle principal est de fournir la **page de login** (`formLogin`) et de vérifier le mot de passe avec `UserDetailsService`.

**Pourquoi l'ordre est important :** Spring essaie les chaînes dans l'ordre, et la **première qui correspond gagne**. La chaîne 1 ne correspond qu'aux URLs OAuth2, alors que la chaîne 2 n'a pas de filtre et correspond à tout. Si la chaîne 2 passait en premier, elle intercepterait tout, y compris `/oauth2/token`, et le serveur OAuth2 ne fonctionnerait plus.

**En une phrase :** la chaîne 1 s'occupe des **applications** (clients et tokens), la chaîne 2 s'occupe des **humains** (login).

## 4. Exemple de workflow (authorization code)

Les acteurs :
- 👤 l'utilisateur et son navigateur ;
- 🖥️ l'application cliente sur le port 8080 (projet séparé) ;
- 🔐 ce serveur sur le port 9000.

```
 #  Qui → Où                                 Chaîne   Ce qui se passe
 ── ──────────────────────────────────────── ──────── ─────────────────────────────────────────────
 1  🖥️ Client → 👤 Navigateur                  —      Le client invente un secret PKCE (code_verifier)
                                                       et envoie le navigateur vers le serveur 9000
 2  👤 GET /oauth2/authorize?client_id=...      1      Reçoit aussi le code_challenge (empreinte PKCE).
       &code_challenge=...                             Vérifie le client (RegisteredClientRepository).
                                                       Utilisateur inconnu → mémorise la demande
                                                       et redirige vers /login
 3  👤 GET /login                               2      Affiche le formulaire de login
 4  👤 POST /login  (user + mot de passe)       2      Vérifie le mot de passe (UserDetailsService).
                                                       OK → stocke « user est connecté » dans la
                                                       SESSION (cookie JSESSIONID), puis renvoie
                                                       vers la demande mémorisée à l'étape 2
 5  👤 GET /oauth2/authorize?... (à nouveau)    1      Grâce au cookie, l'utilisateur est reconnu.
                                                       Affiche la page de CONSENTEMENT :
                                                       « oidc-client veut accéder à profile… OK ? »
 6  👤 clique « Submit Consent »                1      Crée un CODE à usage unique, renvoie le
                                                       navigateur vers 127.0.0.1:8080/...?code=XYZ
 7  🖥️ POST /oauth2/token                        1      Le client envoie code + oidc-client/secret
                                                       + code_verifier (le secret PKCE).
                                                       Vérifie le client, le code et PKCE, puis crée les
                                                       tokens, signés avec JWKSource
                                                       → access token (JWT), ID token, refresh token
 8  🖥️ → API (un autre projet)                  —      Le client appelle une API avec le token.
                                                       C'est l'API qui VÉRIFIE le JWT, avec la clé
                                                       publique de /oauth2/jwks
```

### Le passage de la chaîne 1 à la chaîne 2 (et retour)

- **1 → 2 (étape 2)** : la chaîne 1 fait une simple **redirection HTTP** vers `/login`, une URL qui appartient à la chaîne 2.
- **2 → 1 (étape 4)** : après le login, Spring **renvoie automatiquement** le navigateur vers la demande d'origine (`/oauth2/authorize`).
- **Le lien entre les deux, c'est la session** (cookie `JSESSIONID`). La chaîne 2 y écrit « user est connecté », et la chaîne 1 le relit à l'étape 5.

### Se connecter ne suffit pas pour obtenir un token

Il faut quatre choses :
1. **L'utilisateur s'est connecté** (chaîne 2).
2. **L'utilisateur a donné son accord** sur la page de consentement (`requireAuthorizationConsent(true)`).
3. **L'application cliente prouve son identité** avec `oidc-client` / `secret` lors de l'échange du code.
4. **L'application cliente présente le `code_verifier`** qui correspond au `code_challenge` envoyé au début (PKCE, voir section 6).

Le token n'est jamais remis au navigateur : il est remis à **l'application cliente**.

- **Livrer** : seulement à l'étape 7, à l'application cliente, et seulement si elle présente un code valide **et** son secret. Le login de l'utilisateur sert à savoir *pour qui* est le token, pas à le recevoir.
- **Valider** : en général, ce n'est **pas** le serveur d'autorisation qui valide. C'est l'API (le resource server) qui le fait toute seule avec la clé publique, sans rappeler le serveur. Ce serveur ne valide lui-même un token que pour ses propres endpoints, comme `/userinfo` : c'est là qu'intervient `JwtDecoder`.

## 5. Tester soi-même (sans application cliente)

1. Lancer le serveur : `./mvnw spring-boot:run`.
Ce test utilise une paire PKCE toute prête : l'exemple officiel de la RFC 7636 (annexe B).

| Paramètre | Valeur | Envoyé à |
|---|---|---|
| `code_verifier` (le secret) | `dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk` | `/oauth2/token` (étape 5) |
| `code_challenge` (son empreinte SHA-256) | `E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM` | `/oauth2/authorize` (étape 2) |

Une vraie application génère une nouvelle paire à chaque connexion. Pour un test manuel, la paire fixe suffit.

1. Lancer le serveur : `./mvnw spring-boot:run`.
2. Ouvrir dans le navigateur (une seule ligne) :
   ```
   http://localhost:9000/oauth2/authorize?response_type=code&client_id=oidc-client&scope=openid%20profile&redirect_uri=http://127.0.0.1:8080/login/oauth2/code/oidc-client&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=S256
   ```
3. Se connecter avec `user` et son mot de passe, puis cocher `profile` et valider le consentement.
4. Le navigateur affiche une erreur, car rien ne tourne sur le port 8080. C'est normal : copier le `code=...` dans la barre d'adresse.
5. Échanger le code contre les tokens, comme le ferait l'application cliente. Dans PowerShell, utiliser `curl.exe` et non `curl` (une seule ligne) :
   ```
   curl.exe -u oidc-client:secret -d grant_type=authorization_code -d code=COLLER_LE_CODE_ICI -d redirect_uri=http://127.0.0.1:8080/login/oauth2/code/oidc-client -d code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk http://localhost:9000/oauth2/token
   ```
6. La réponse est un JSON avec `access_token`, `id_token` et `refresh_token`. Coller l'`access_token` sur https://jwt.io pour voir son contenu.

Erreurs fréquentes :
- `OAuth 2.0 Parameter: code_challenge` à l'étape 2 : le `code_challenge` manque dans l'URL.
- `{"error":"invalid_grant"}` à l'étape 5 : le `code_verifier` manque ou est faux, ou le code a déjà été utilisé ou a expiré. Le code est à usage unique et ne vit que quelques minutes : recommencer à l'étape 2.

Comme le logging de sécurité est en `trace`, la console montre pour chaque requête laquelle des deux chaînes la prend en charge.

## 6. PKCE et versions de Spring Boot

### Ce qu'est PKCE

PKCE (Proof Key for Code Exchange, RFC 7636) ne **remplace** pas le flux authorization code : il s'y **ajoute**.

- Au début, le client invente un secret aléatoire, le `code_verifier`. Il envoie seulement son empreinte, le `code_challenge`, à `/oauth2/authorize`.
- À la fin, il envoie le secret lui-même à `/oauth2/token`. Le serveur vérifie que l'empreinte correspond.
- Si quelqu'un vole le code pendant la redirection, il ne peut pas l'utiliser : il lui manque le `code_verifier`, qui n'est jamais passé par le navigateur.

Les flux `client_credentials` et `refresh_token` ne sont pas concernés : il n'y a ni navigateur, ni code.

### Côté serveur (ce projet)

Avec Spring Boot 4.1.1 (Spring Security 7.1.1), `ClientSettings.builder()` met `requireProofKey(true)` par défaut. Ce projet n'ayant pas changé ce réglage, **PKCE est obligatoire** pour `oidc-client`. Une demande sans `code_challenge` est refusée.

### Côté client : ça dépend de la version

Le point clé est le type de client :
- **Client public** (sans secret, ex. application mobile ou SPA) : PKCE automatique, dans toutes les versions récentes.
- **Client confidentiel** (avec secret, comme notre `oidc-client` / `secret`) : le comportement dépend de la version.

| Client | Spring Security | PKCE pour un client confidentiel |
|---|---|---|
| Spring Boot 4.x | 7.x | ✅ Activé par défaut. La doc 7.1.1 indique qu'il faut le **désactiver** (`ClientRegistration.clientSettings.requireProofKey = false`) si le serveur ne le supporte pas. |
| Spring Boot 3.x | 6.x | ❌ Pas envoyé par défaut. Il faut l'activer soi-même. |

Version 3.x exacte à partir de laquelle un réglage `requireProofKey` existe aussi côté client : non vérifiée (peut-être 3.5).

**Symptôme** : un client Spring Boot 3.x configuré avec `oidc-client` / `secret`, sans réglage spécial, est refusé par ce serveur avec l'erreur `OAuth 2.0 Parameter: code_challenge`.

### Solutions pour un client Spring Boot 3.x

**1. Activer PKCE côté client (recommandé)**

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http,
        ClientRegistrationRepository clientRegistrationRepository) throws Exception {

    DefaultOAuth2AuthorizationRequestResolver resolver =
            new DefaultOAuth2AuthorizationRequestResolver(
                    clientRegistrationRepository, "/oauth2/authorization");
    resolver.setAuthorizationRequestCustomizer(
            OAuth2AuthorizationRequestCustomizers.withPkce());   // ajoute PKCE

    http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .oauth2Login(login -> login
            .authorizationEndpoint(endpoint ->
                endpoint.authorizationRequestResolver(resolver)));
    return http.build();
}
```

**2. Désactiver PKCE côté serveur, pour ce client seulement (déconseillé)**

Dans `SecurityConfig.java` de ce projet :

```java
.clientSettings(ClientSettings.builder()
        .requireAuthorizationConsent(true)
        .requireProofKey(false)      // accepte les clients sans PKCE
        .build())
```

Pratique pour un vieux client qu'on ne peut pas modifier, mais moins sûr.
