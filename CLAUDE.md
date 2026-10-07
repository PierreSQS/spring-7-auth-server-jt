# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Spring Authorization Server (OAuth2 + OpenID Connect 1.0) built for a Udemy course (John Thompson, Spring Framework 7). Spring Boot 4.1.1 (Spring Security 7.1.1), Java 25, Maven. Commits are prefixed with the course section/chapter (e.g. `Sec23_Chap245-XX: ...`): `-XX` marks a personal trial or variant that is not part of the lecture, `-NOK` a commit known not to work yet (fixed in a later commit).

Branches:
- My branches are named `<JT branch>-sb<version>`: they re-implement one branch of JT's original repo ([springframeworkguru/spring-6-auth-server](https://github.com/springframeworkguru/spring-6-auth-server), Spring Boot 3.4.0) on a given Spring Boot version. Current ones: `8-package-refactor-sb403` (reference, SB 4.0.3) and `8-package-refactor-sb411` (SB 4.1.1, default branch on GitHub). Both correspond to Section 23, Chapter 245.
- JT's branches (`1-initial-project` … `10-k8s-issuer-name`, `8-server-settings`, `junie-init`, `main`) are copied unchanged into `origin` and must never be modified. The local remote `jt` points to JT's repo (fetch only, push disabled).
- Branches are compared, **never merged**: no pull requests. A file that must exist on every one of my branches (like `README.md`) is committed on the reference branch and cherry-picked onto the others, so it never shows up in comparisons.
- Compare JT vs mine locally with `git diff -M origin/8-package-refactor origin/8-package-refactor-sb411` (GitHub can't compare unrelated histories).
- Tooling files (`CLAUDE.md`, `docs/`, `.claude/skills/`) belong on **every** version branch. A new version branch is always created from the latest one, so it inherits them automatically. **Exception, on purpose:** `8-package-refactor-sb403` is obsolete, kept only temporarily for comparison, and doesn't get them.
- Branches of older Spring Boot versions are temporary: once superseded and no longer needed for comparison, they are deleted.

## Claude Code skills

- `.claude/skills/explain-code/`: explains code with an everyday analogy, an ASCII diagram, a step-by-step walkthrough and a common gotcha. Copied unchanged from [PierreSQS/eazybytes-spring-ai](https://github.com/PierreSQS/eazybytes-spring-ai) (`section09/springai/.claude/skills/explain-code/`, branch `hands-on-sb3.5.x`).

## Commands

Use the Maven wrapper (`mvnw.cmd` on Windows/PowerShell, `./mvnw` in Bash):

- Build: `./mvnw clean package`
- Run: `./mvnw spring-boot:run` (listens on port **9000**)
- All tests: `./mvnw test`
- Single test class: `./mvnw test -Dtest=Spring7AuthServerApplicationTests`
- Single test method: `./mvnw test -Dtest=Spring7AuthServerApplicationTests#contextLoads`

## Architecture

All authorization-server wiring lives in `config/SecurityConfig.java`:

- **Two ordered `SecurityFilterChain`s**:
  - `@Order(1)` — enabled with `http.oauth2AuthorizationServer(...)` and scoped to its `getEndpointsMatcher()` (authorize, token, JWKS, OIDC discovery, userinfo, etc.). OIDC enabled; unauthenticated HTML requests are redirected to `/login`. Spring Security 7.1 adds JWT bearer token support for `/userinfo` automatically when OIDC is on, so the explicit `.oauth2ResourceServer(jwt)` call is commented out (see `docs/oauth2-workflow.md`, section 7).
  - `@Order(2)` — everything else, with default form login (which serves the `/login` page the first chain redirects to).
- **Users**: `InMemoryUserDetailsManager` with one user `user` (password stored as a bcrypt hash). Never write the plain-text password in code comments or docs.
- **Clients**: `InMemoryRegisteredClientRepository` with one client `oidc-client` / `secret` (`{noop}`), `client_secret_basic`, grants `authorization_code`, `refresh_token`, `client_credentials`; scopes `openid`, `profile`, `message.read`, `message.write`; consent required. Redirect URI `http://127.0.0.1:8080/login/oauth2/code/oidc-client` (the client app is expected on port 8080). PKCE is **required** for `authorization_code`: `ClientSettings.builder()` defaults to `requireProofKey(true)` in Spring Security 7.1, and this project doesn't override it. Spring Boot 4.x clients send PKCE automatically, even confidential ones; Spring Boot 3.x confidential clients don't, and get `OAuth 2.0 Parameter: code_challenge` unless they add `OAuth2AuthorizationRequestCustomizers.withPkce()`. Details in `docs/oauth2-workflow.md`, section 6.
- **Signing keys**: a fresh RSA 2048 key pair is generated at every startup (`jwkSource`), so issued tokens become invalid after a restart.
- `AuthorizationServerSettings` uses defaults (standard `/oauth2/*` and `/.well-known/*` endpoint paths).

### Component diagram

```
                                                      HTTP request
                                                            │
                      OAuth2 URLs ┌─────────────────────────┴────────────────────────┐ other URLs
                                  ▼                                                  ▼
┌────────────────────────────────────────────────────────────────────┐    ┌─────────────────────┐
│ authorizationServerSecurityFilterChain                             │    │ defaultSecurity     │
│ @Order(1) · /oauth2/*, /.well-known/*, /userinfo                   │    │ FilterChain         │
│ not logged in → redirect to /login                                 │───►│ @Order(2) · /login  │
└───────┬───────────────────┬─────────────────┬───────────────┬──────┘    └──────────┬──────────┘
        │ endpoint URLs     │ check client    │ verify token  │ sign tokens          │ check user
        ▼                   ▼                 ▼               ▼                      ▼
┌────────────────┐ ┌──────────────────┐ ┌────────────┐  ┌────────────┐    ┌─────────────────────┐
│ Authorization  │ │ RegisteredClient │ │ JwtDecoder │─►│ JWKSource  │    │ UserDetailsService  │
│ ServerSettings │ │ Repository       │ │            │  │ RSA keys   │    │ user (bcrypt)       │
└────────────────┘ └──────────────────┘ └────────────┘  └────────────┘    └─────────────────────┘
```

### Component roles

| Component | Role (in simple words) | In this project |
|---|---|---|
| `authorizationServerSecurityFilterChain` | Creates and protects the OAuth2 URLs. Redirects to `/login` if the user isn't logged in. | `@Order(1)`: checked first |
| `defaultSecurityFilterChain` | Protects all other URLs and shows the login page. | `@Order(2)`: default Spring login page |
| `UserDetailsService` | The **user directory**: "Does this user exist, and what is their password?" | One user in memory: `user` (bcrypt password) |
| `RegisteredClientRepository` | The **list of apps** allowed to ask for tokens, with their secret, allowed URLs and scopes. | One app: `oidc-client` / `secret` |
| `JWKSource` | The **key ring**. The private key signs every token; the public key is published at `/oauth2/jwks`. | New RSA key at each startup, so old tokens become invalid |
| `JwtDecoder` | **Reads and checks** a token, using the public key from `JWKSource`. | Used when an app calls `/userinfo` with a token |
| `AuthorizationServerSettings` | The server's **address book**: the URLs of all OAuth2 endpoints and the issuer name written into tokens. | Default values |

A step-by-step authorization code workflow (how requests move between the two chains, plus a manual test with the browser and curl) is in `docs/oauth2-workflow.md` (French).

H2 and `spring-boot-starter-jdbc` are on the classpath but nothing is persisted yet — users, clients and authorizations are all in-memory.

`application.properties` sets `org.springframework.security` logging to `trace`, so console output is very verbose.