# Configuration & Externalized Settings Inventory

Spring Music uses a small set of configuration sources centered on Spring Boot YAML, Cloud Foundry deployment metadata, and Gradle build settings. Runtime behavior changes mostly through Spring profiles, service bindings, and environment variables rather than through a large property catalog.

## Configuration Sources

| Source | Type | Path/Location | Notes |
| --- | --- | --- | --- |
| Spring Boot application config | YAML | `src/main/resources/application.yml` | Defines default JPA behavior, actuator exposure, HTTP/2 toggle, and profile-specific datasource settings |
| Build definition | Gradle | `build.gradle` | Declares plugins, Java compatibility, dependencies, boot image builder, and runtime drivers |
| Build settings | Gradle | `settings.gradle`, `gradle.properties`, `gradle/wrapper/gradle-wrapper.properties` | Defines project name, version, and wrapper version |
| Cloud Foundry manifest | YAML | `manifest.yml` | Sets app name, memory, route policy, deployment path, and environment variables |
| Paketo buildpack config | TOML | `project.toml` | Disables Spring Cloud Bindings support and excludes build artifacts from image context |
| Cloud Foundry service bindings | Environment | `VCAP_SERVICES` and related platform variables | Read through Java CFEnv to select runtime profile and connection settings |
| Seed data | JSON | `src/main/resources/albums.json` | Provides startup album catalog bootstrap data |

## Build Profiles

| Profile | Activation | Purpose | Key Dependencies/Plugins |
| --- | --- | --- | --- |
| `default` | Automatic | Builds the executable Spring Boot jar | Spring Boot plugin `3.1.5`, dependency management `1.1.3`, Java plugin |
| `bootBuildImage` | Manual Gradle task | Produces a container image with Paketo buildpacks | Builder `paketobuildpacks/builder-jammy-base:latest` |

No custom Gradle flavor matrix or environment-specific build profiles were defined.

## Runtime Profiles

| Profile | Activation Method | Config Files | Key Overrides |
| --- | --- | --- | --- |
| `default` | No explicit profile | `application.yml` | Enables JPA DDL generation and uses the default relational path |
| `http2` | `SPRING_PROFILES_ACTIVE=http2` or `-Dspring.profiles.active=http2` | `application.yml` | Sets `server.http2.enabled=true` |
| `mysql` | Active profile or Cloud Foundry service tag detection | `application.yml` | Configures MySQL JDBC URL, driver, and Hibernate dialect |
| `postgres` | Active profile or Cloud Foundry service tag detection | `application.yml` | Configures PostgreSQL JDBC URL, driver, and Hibernate dialect |
| `mongodb` | Active profile or Cloud Foundry service tag detection | Code-based `@Profile("mongodb")` beans | Activates Mongo repository and excludes JDBC and Redis auto-configuration |
| `redis` | Active profile or Cloud Foundry service tag detection | Code-based `@Profile("redis")` beans | Activates Redis repository and excludes JDBC and Mongo auto-configuration |
| `oracle`, `sqlserver` | Cloud Foundry service tag detection | No profile-specific file | Use default JPA path with service binding-derived datasource settings |

Only one persistence-oriented profile may be active at a time; the initializer throws an exception if multiple storage profiles are enabled together.

## Properties Inventory

| Property Key | Default | Profiles | Source |
| --- | --- | --- | --- |
| `spring.jpa.generate-ddl` | `true` | default and inherited unless overridden externally | `application.yml` |
| `management.endpoints.web.exposure.include` | `*` | all | `application.yml` |
| `management.endpoint.health.show-details` | `always` | all | `application.yml` |
| `server.http2.enabled` | `true` | `http2` | `application.yml` |
| `spring.datasource.url` | `jdbc:mysql://localhost/music` | `mysql` | `application.yml` |
| `spring.datasource.driver-class-name` | `com.mysql.jdbc.Driver` | `mysql` | `application.yml` |
| `spring.datasource.username` | empty | `mysql` | `application.yml` |
| `spring.datasource.password` | empty | `mysql` | `application.yml` |
| `spring.jpa.properties.hibernate.dialect` | `org.hibernate.dialect.MySQL55Dialect` | `mysql` | `application.yml` |
| `spring.datasource.url` | `jdbc:postgresql://localhost/music` | `postgres` | `application.yml` |
| `spring.datasource.driver-class-name` | `org.postgresql.Driver` | `postgres` | `application.yml` |
| `spring.datasource.username` | `postgres` | `postgres` | `application.yml` |
| `spring.datasource.password` | empty | `postgres` | `application.yml` |
| `spring.jpa.properties.hibernate.dialect` | `org.hibernate.dialect.ProgressDialect` | `postgres` | `application.yml` |
| `spring.autoconfigure.exclude` | computed at startup | `mongodb`, `redis`, default SQL path | Injected by `SpringApplicationContextInitializer` |
| `SPRING_PROFILES_ACTIVE` | `http2` in Cloud Foundry manifest | deployment-specific | `manifest.yml` environment |
| `JBP_CONFIG_SPRING_AUTO_RECONFIGURATION` | `{enabled: false}` | deployment-specific | `manifest.yml` environment |
| `JBP_CONFIG_OPEN_JDK_JRE` | `{ jre: { version: 17.+ } }` | deployment-specific | `manifest.yml` environment |
| `BP_SPRING_CLOUD_BINDINGS_DISABLED` | `true` | buildpack image builds | `project.toml` |

## Startup Parameters & Resource Requirements

| Service | JVM/Runtime Options | Memory | Instance Count |
| --- | --- | --- | --- |
| `spring-music` | `-Dspring.profiles.active=<profile>` supported for local runs; `SPRING_PROFILES_ACTIVE=http2` in Cloud Foundry manifest; Java 17 JRE requested via buildpack config | `1G` in `manifest.yml` | Not specified |

No explicit `-Xms`, `-Xmx`, CPU limits, or horizontal scaling settings were found in the repository.

## Startup Dependency Chain

1. `Application` starts the Spring Boot app and registers `SpringApplicationContextInitializer` before the context refresh.
2. `SpringApplicationContextInitializer` reads Cloud Foundry services through Java CFEnv, validates that at most one persistence profile is active, and injects `spring.autoconfigure.exclude` to disable unused data auto-configuration.
3. Spring Boot completes auto-configuration for the selected persistence path and exposes the REST controllers and actuator endpoints.
4. `AlbumRepositoryPopulator` runs on `ApplicationReadyEvent` and loads `albums.json` only when the selected repository is empty.

There is no explicit container wait script or external readiness dependency beyond the selected datastore being reachable through Spring Boot or Cloud Foundry service bindings.

## Secrets & Sensitive Configuration

| Secret Reference | Type | Storage (masked) |
| --- | --- | --- |
| `spring.datasource.username` | Database username | `[MASKED]` or service binding value |
| `spring.datasource.password` | Database password | `[MASKED]` or service binding value |
| Cloud Foundry service credentials | Connection URI / credentials | `[MASKED]` in `VCAP_SERVICES` |

### Secrets Provisioning Workflow

Secrets are expected to come from environment-bound services rather than from source-controlled files. In local profile examples, usernames and passwords are left blank or defaulted in `application.yml`, while Cloud Foundry deployments rely on bound service credentials that Java CFEnv reads at startup to derive datasource or service connection details. No dedicated Vault, Key Vault, AWS Secrets Manager, or encrypted property framework was detected.

## Feature Flags

| Flag Name | Default | Controlled By |
| --- | --- | --- |
| None detected | N/A | Runtime profile selection is used instead of feature flags |

## Framework & Runtime Versions

| Component | Version | Source |
| --- | --- | --- |
| Java source and target compatibility | 17 | `build.gradle` |
| Spring Boot plugin | 3.1.5 | `build.gradle` |
| Spring dependency management plugin | 1.1.3 | `build.gradle` |
| Java CFEnv | 3.1.2 | `build.gradle` |
| Gradle wrapper | 7.6.2 | `gradle/wrapper/gradle-wrapper.properties` |
| Bootstrap WebJar | 3.1.1 | `build.gradle` |
| AngularJS WebJar | 1.2.16 | `build.gradle` |
| Angular UI WebJar | 0.4.0-2 | `build.gradle` |
| Angular UI Bootstrap WebJar | 0.10.0-1 | `build.gradle` |
| jQuery WebJar | 2.1.0-2 | `build.gradle` |
| Paketo builder | `paketobuildpacks/builder-jammy-base:latest` | `build.gradle` |
