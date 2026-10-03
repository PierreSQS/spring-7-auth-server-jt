# spring-7-auth-server-jt

Hands-on from John Thompson's Udemy course on Spring Framework 7: an **OAuth2 / OpenID Connect Authorization Server** built with Spring Boot 4 and Spring Security 7.

On each branch, the authorization server follows the official Spring Security guide for the matching Spring Boot version:
[Spring Authorization Server – Getting Started](https://docs.spring.io/spring-security/reference/servlet/oauth2/authorization-server/getting-started.html)

## Branches: one per Spring Boot version

Each branch carries the same course content on a **specific Spring Boot version**. The branch name is the compacted version number.

| Branch | Spring Boot | Role |
|---|---|---|
| `master` | 4.0.3 | **Reference branch.** Starting point of this repo; it is never updated with newer versions. |
| `sb411` | 4.1.1 | Code upgraded to Spring Boot 4.1.1 (Spring Security 7.1.1) and aligned with the official docs for that version. |

Further branches follow the same pattern, e.g. `sb420` for Spring Boot 4.2.0.

The goal is to **see what changes from one Spring Boot version to the next**, by comparing branches:

- On GitHub: [`master...sb411`](https://github.com/PierreSQS/spring-7-auth-server-jt/compare/master...sb411)
- Locally: `git diff master sb411`

Branches are **never merged**, so this repo has no pull requests: `master` stays as a fixed reference.

## Running

Requires Java 25.

```
./mvnw spring-boot:run
```

The server listens on port **9000**. Its OpenID configuration is at http://localhost:9000/.well-known/openid-configuration.
