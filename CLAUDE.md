# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Spring Authorization Server (OAuth2 + OpenID Connect 1.0) built for a Udemy course (John Thompson, Spring Framework 7). Spring Boot 4.1.1 (Spring Security 7.1.1), Java 25, Maven. Commits are prefixed with the course section/chapter (e.g. `Sec23_Chap245-XX: ...`). Branches are named after the Spring Boot version they target (e.g. `sb411` = 4.1.1).

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
  - `@Order(1)` — scoped to `OAuth2AuthorizationServerConfigurer.getEndpointsMatcher()` (authorize, token, JWKS, OIDC discovery, userinfo, etc.). OIDC enabled; unauthenticated HTML requests are redirected to `/login`; acts as a JWT resource server for userinfo/client registration.
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