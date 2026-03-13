# Copilot Instructions — eux-parent-pom

## What This Is

A Maven parent POM for the EUX/EESSI platform at NAV. It contains no source code — only centralized dependency versions and plugin configurations inherited by child projects across the `no.nav.eux` group.

## Build

```sh
mvn clean install --settings ./.github/settings.xml --no-transfer-progress -B
```

The `--settings ./.github/settings.xml` flag is required — it configures GitHub Packages repositories under `navikt`.

## Versioning and Release

- Versions are managed automatically by the custom `eux-versions-maven-plugin` on merge to `main`.
- The deploy workflow runs `mvn eux-versions:set-next -DmajorVersion=2` to bump the version, then deploys to GitHub Packages and creates a GitHub release via release-drafter.
- Do not manually set versions in `pom.xml` — leave it as `1.0-SNAPSHOT`.

## Key Conventions

- **Technology stack**: Kotlin 2, Spring Boot 4, JDK 21.
- **NAV token-support**: `token-validation-spring` and `token-client-spring` from `no.nav.security` are the standard for OAuth2/token handling in child projects.
- **Logging**: `eux-logging` + `logstash-logback-encoder` for structured JSON logging.
- **Testing**: JUnit Jupiter + Kotest assertions. OkHttp `mockwebserver` for HTTP mocking.
- When updating dependency versions, update only the property in `<properties>` — never override versions inline on individual `<dependency>` entries.
