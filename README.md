# spring-7-auth-server-jt

Hands-on from John Thompson's (JT) Udemy course on Spring Framework: an **OAuth2 / OpenID Connect Authorization Server**, re-implemented with Spring Boot 4 and Spring Security 7.

- JT's original course code (Spring Boot 3.4.0): [springframeworkguru/spring-6-auth-server](https://github.com/springframeworkguru/spring-6-auth-server)
- On my branches, the authorization server follows the official Spring Security guide for the matching Spring Boot version: [Spring Authorization Server – Getting Started](https://docs.spring.io/spring-security/reference/servlet/oauth2/authorization-server/getting-started.html)

## Branches

This repo holds two kinds of branches.

### JT's branches (unchanged)

All branches of JT's repo (`1-initial-project` … `10-k8s-issuer-name`, `8-server-settings`, `junie-init`, `main`) are copied here **as is**, for reference. They use Spring Boot 3.4.0 and are never modified. They are licensed under JT's GPL v3 license, included in each of them.

### My branches: `<JT branch>-sb<version>`

Each of my branches re-implements one of JT's branches on a **specific Spring Boot version**. The name is JT's branch name followed by the compacted Spring Boot version.

| My branch | JT branch | Course | Spring Boot | Role |
|---|---|---|---|---|
| `8-package-refactor-sb403` | `8-package-refactor` | Section 23, Chapter 245 | 4.0.3 | **Reference branch**: my starting point on Spring Boot 4 |
| `8-package-refactor-sb411` | `8-package-refactor` | Section 23, Chapter 245 | 4.1.1 | Upgraded to Spring Boot 4.1.1 (Spring Security 7.1.1) and aligned with the official docs for that version |

The goal is to **see what changes from one version to the next** by comparing branches. Branches are **never merged**, so this repo has no pull requests.

## Comparing branches

Between two of my versions (works on GitHub too):

- GitHub: [`8-package-refactor-sb403...8-package-refactor-sb411`](https://github.com/PierreSQS/spring-7-auth-server-jt/compare/8-package-refactor-sb403...8-package-refactor-sb411)
- Locally: `git diff origin/8-package-refactor-sb403 origin/8-package-refactor-sb411`

Between JT's code and mine (local only: GitHub can't compare branches with unrelated histories):

```
git diff -M origin/8-package-refactor origin/8-package-refactor-sb411
```

(After a `git clone`, all branches are available as `origin/<branch>`.) The `-M` option shows the `spring6authserver` → `spring7authserver` package change as file renames.

## Running

Requires Java 25.

```
./mvnw spring-boot:run
```

The server listens on port **9000**. Its OpenID configuration is at http://localhost:9000/.well-known/openid-configuration.
