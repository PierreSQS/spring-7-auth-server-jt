# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Spring Authorization Server (OAuth2 + OpenID Connect 1.0) built for a Udemy course (John Thompson, Spring Framework 7). Spring Boot 4.0.3, Java 25, Maven. Commits are prefixed with the course section/chapter (e.g. `Sec23_Chap245-XX: ...`).

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
- **Users**: `InMemoryUserDetailsManager` with `user` / `password` (bcrypt-encoded).
- **Clients**: `InMemoryRegisteredClientRepository` with one client `oidc-client` / `secret` (`{noop}`), `client_secret_basic`, grants `authorization_code`, `refresh_token`, `client_credentials`; scopes `openid`, `profile`, `message.read`, `message.write`; consent required. Redirect URI `http://127.0.0.1:8080/login/oauth2/code/oidc-client` (the client app is expected on port 8080).
- **Signing keys**: a fresh RSA 2048 key pair is generated at every startup (`jwkSource`), so issued tokens become invalid after a restart.
- `AuthorizationServerSettings` uses defaults (standard `/oauth2/*` and `/.well-known/*` endpoint paths).

H2 and `spring-boot-starter-jdbc` are on the classpath but nothing is persisted yet — users, clients and authorizations are all in-memory.

`application.properties` sets `org.springframework.security` logging to `trace`, so console output is very verbose.