# Configuration & Externalized Settings Inventory

Spring Music is configured through a single multi-document `application.yml` covering six runtime profiles, a Cloud Foundry `manifest.yml`, and a Paketo Buildpacks `project.toml` — with no external config server, no secret store, and no feature flag framework.

## Configuration Sources

| Source | Type | Path/Location | Notes |
|---|---|---|---|
| `application.yml` | Spring Boot YAML (multi-document) | `src/main/resources/application.yml` | Contains base config plus profile-specific sections (`http2`, `mysql`, `postgres`) using `spring.config.activate.on-profile` |
| `manifest.yml` | Cloud Foundry push manifest | `manifest.yml` (project root) | Defines CF app deployment settings, memory, env vars, route strategy |
| `project.toml` | Paketo Buildpacks config | `project.toml` (project root) | Defines build exclusions and disables `spring-cloud-bindings` in favor of CF CfEnv bindings |
| `gradle.properties` | Gradle project properties | `gradle.properties` | Single property: `version=1.0` |
| `gradle-wrapper.properties` | Gradle Wrapper config | `gradle/wrapper/gradle-wrapper.properties` | Pins Gradle distribution to 7.6.2 |
| `albums.json` | Seed data resource | `src/main/resources/albums.json` | JSON array of `Album` objects loaded at startup via `AlbumRepositoryPopulator` |

No external config server (Spring Cloud Config, Consul KV, Azure App Configuration), no bootstrap context, and no `bootstrap.yml` are present.

## Build Profiles

No explicit Gradle build profiles are defined in `build.gradle`. The build is a single Gradle project with no flavors, build types, or Maven-style `<profiles>` blocks.

| Profile | Activation | Purpose | Key Dependencies/Plugins |
|---|---|---|---|
| (default build) | Always | Compiles Java 17 source, assembles Spring Boot fat JAR | `org.springframework.boot:3.1.5`, `io.spring.dependency-management:1.1.3`, `java`, `eclipse-wtp`, `idea` |
| `bootBuildImage` task | Manual (`./gradlew bootBuildImage`) | Builds a Paketo OCI image using `paketobuildpacks/builder-jammy-base:latest` | Configured via `project.toml` (excludes `.git`, `/build`, `/bin`; sets `BP_SPRING_CLOUD_BINDINGS_DISABLED=true`) |

## Runtime Profiles

| Profile | Activation Method | Key Config Overrides |
|---|---|---|
| `(none / default)` | No active profile set | H2 in-memory JPA; Spring Boot auto-configures H2 data source; `spring.jpa.generate-ddl: true` |
| `http2` | `SPRING_PROFILES_ACTIVE=http2` (set in `manifest.yml`) or `-Dspring.profiles.active=http2` | `server.http2.enabled: true` |
| `mysql` | Explicit profile activation; or CfEnv detects a CF service with tag `mysql` | `spring.datasource.url=jdbc:mysql://localhost/music`, `driver-class-name=com.mysql.jdbc.Driver`, `hibernate.dialect=MySQL55Dialect` |
| `postgres` | Explicit profile activation; or CfEnv detects a CF service with tag `postgres` | `spring.datasource.url=jdbc:postgresql://localhost/music`, `driver-class-name=org.postgresql.Driver`, `dialect=ProgressDialect`, `username=postgres` |
| `sqlserver` | CfEnv detects a CF service with tag `sqlserver`; no static YAML section | Connection info injected by `java-cfenv-boot` from VCAP_SERVICES |
| `mongodb` | Explicit profile activation; or CfEnv detects a CF service with tag `mongodb` | MongoDB auto-configuration; `MongoAlbumRepository` bean activated |
| `redis` | Explicit profile activation; or CfEnv detects a CF service with tag `redis` | Redis auto-configuration; `RedisAlbumRepository` and `RedisConfig` bean activated |
| `oracle` | CfEnv detects a CF service with tag `oracle`; no driver/static config included | JDBC driver must be added manually (commented out in `build.gradle`) |
| `cloud` | Added internally by `SpringApplicationContextInitializer` when running on CF | Used to exclude unused auto-configurations (JPA, MongoDB, or Redis) based on detected bound services |

Profiles compose: `manifest.yml` always activates `http2`; the data-store profile is added on top by `SpringApplicationContextInitializer` (or set manually via `SPRING_PROFILES_ACTIVE`). At most one data-store profile may be active at a time — the initializer throws `IllegalStateException` if more than one is found.

## Properties Inventory

### spring-music — Base (all profiles)

| Property Key | Default Value | Profile Scope | Source |
|---|---|---|---|
| `spring.jpa.generate-ddl` | `true` | All JPA profiles | `application.yml` |
| `management.endpoints.web.exposure.include` | `*` | All | `application.yml` |
| `management.endpoint.health.show-details` | `always` | All | `application.yml` |

### spring-music — Profile: `http2`

| Property Key | Value | Source |
|---|---|---|
| `server.http2.enabled` | `true` | `application.yml` (http2 section) |

### spring-music — Profile: `mysql`

| Property Key | Value | Source |
|---|---|---|
| `spring.datasource.url` | `jdbc:mysql://localhost/music` | `application.yml` |
| `spring.datasource.driver-class-name` | `com.mysql.jdbc.Driver` | `application.yml` |
| `spring.datasource.username` | (empty) | `application.yml` |
| `spring.datasource.password` | (empty) | `application.yml` |
| `spring.jpa.properties.hibernate.dialect` | `org.hibernate.dialect.MySQL55Dialect` | `application.yml` |

### spring-music — Profile: `postgres`

| Property Key | Value | Source |
|---|---|---|
| `spring.datasource.url` | `jdbc:postgresql://localhost/music` | `application.yml` |
| `spring.datasource.driver-class-name` | `org.postgresql.Driver` | `application.yml` |
| `spring.datasource.username` | `postgres` | `application.yml` |
| `spring.datasource.password` | (empty) | `application.yml` |
| `spring.jpa.properties.hibernate.dialect` | `org.hibernate.dialect.ProgressDialect` | `application.yml` |

### spring-music — Cloud Foundry Deployment (`manifest.yml`)

| Property Key | Value | Source |
|---|---|---|
| `memory` | `1G` | `manifest.yml` |
| `random-route` | `true` | `manifest.yml` |
| `path` | `build/libs/spring-music-1.0.jar` | `manifest.yml` |
| `JBP_CONFIG_SPRING_AUTO_RECONFIGURATION` | `{enabled: false}` | `manifest.yml` env |
| `SPRING_PROFILES_ACTIVE` | `http2` | `manifest.yml` env |
| `JBP_CONFIG_OPEN_JDK_JRE` | `{ jre: { version: 17.+ } }` | `manifest.yml` env |

## Startup Parameters & Resource Requirements

| Service | JVM/Runtime Options | Memory | CPU | Instance Count |
|---|---|---|---|---|
| spring-music (CF) | No explicit `-Xms`/`-Xmx` — managed by JBP (Java Buildpack) based on container memory; JRE 17.+ forced via `JBP_CONFIG_OPEN_JDK_JRE` | 1 GB (set in `manifest.yml`) | Not specified | 1 (default CF) |
| spring-music (local) | No explicit JVM flags in Gradle run configuration | Default JVM heap | Not specified | 1 |
| spring-music (OCI/Paketo) | Heap managed by Paketo Memory Calculator based on container limits | Not specified in `project.toml` | Not specified | Not specified |

No `-Dspring.profiles.active` system properties are set in any startup script. Profile is set via the `SPRING_PROFILES_ACTIVE` environment variable in CF deployments or passed as a command-line argument by the developer.

## Startup Dependency Chain

Spring Music is a single-service application with no external service orchestration:

1. **JVM starts** → Spring Boot context initializes
2. **`SpringApplicationContextInitializer` runs** (before bean creation) → reads `VCAP_SERVICES` via CfEnv, detects bound service tags, activates the appropriate data-store profile, excludes unused auto-configurations
3. **Data source auto-configured** → connection pool established (HikariCP for JPA; Jedis pool via `commons-pool2` for Redis)
4. **`ApplicationReadyEvent` fires** → `AlbumRepositoryPopulator` checks if the repository is empty; if so, loads albums from `albums.json`
5. **HTTP server ready** → REST endpoints and Actuator endpoints become available

No Docker Compose `depends_on`, Kubernetes readiness probes, `dockerize` wait mechanisms, or Spring Cloud Config retry are configured. The application assumes its data store is already running and accessible when it starts.

## Secrets & Sensitive Configuration

| Secret Reference | Type | Notes |
|---|---|---|
| `spring.datasource.password` (mysql profile) | Database password | Empty string in `application.yml` — expected to be overridden at runtime or injected by CF service binding |
| `spring.datasource.password` (postgres profile) | Database password | Empty string in `application.yml` — expected to be overridden at runtime or injected by CF service binding |
| `VCAP_SERVICES` (CF environment variable) | Platform-injected JSON blob | Contains credentials (host, port, username, password) for all bound CF services; read by `java-cfenv-boot` at runtime |

No encryption (Jasypt, sealed secrets, DPAPI), no HashiCorp Vault, no Azure Key Vault, and no AWS Secrets Manager references are present. Actual secret values are not hardcoded in any configuration file.

### Secrets Provisioning Workflow

When deployed to Cloud Foundry, secrets are provisioned entirely by the CF platform:

1. A CF service instance (e.g., `cf create-service mysql ...`) is created in the CF space.
2. The service is bound to the application (`cf bind-service spring-music <service-instance-name>`).
3. At application startup, CF injects `VCAP_SERVICES` as a JSON environment variable containing the service credentials (host, port, username, password, URI).
4. `java-cfenv-boot` (`io.pivotal.cfenv:java-cfenv-boot:3.1.2`) reads `VCAP_SERVICES`, detects service tags, sets the appropriate Spring profile, and maps credential fields to Spring Boot property keys (e.g., `spring.datasource.url`, `spring.data.mongodb.uri`).

For local development, no secrets workflow exists — credentials are either left empty (connecting to an unauthenticated local database) or set directly as environment variables before starting the application.

## Feature Flags

No feature flag framework (LaunchDarkly, Unleash, Spring Feature Flags) is used. Conditional bean activation is achieved entirely through Spring `@Profile` annotations, which act as binary on/off switches for data-store implementations:

| Conditional | Mechanism | Default State | Controlled By |
|---|---|---|---|
| `JpaAlbumRepository` active | `@Profile({"!mongodb", "!redis"})` | Active (default) | Active profiles |
| `MongoAlbumRepository` active | `@Profile("mongodb")` | Inactive | `mongodb` profile |
| `RedisAlbumRepository` active | `@Profile("redis")` | Inactive | `redis` profile |
| `RedisConfig` active | `@Profile("redis")` | Inactive | `redis` profile |
| HTTP/2 enabled | `spring.config.activate.on-profile: http2` | Disabled | `http2` profile |
| Spring Cloud Bindings | `BP_SPRING_CLOUD_BINDINGS_DISABLED=true` | Disabled in OCI builds | `project.toml` buildpack env var |
| Spring Auto-reconfiguration | `JBP_CONFIG_SPRING_AUTO_RECONFIGURATION={enabled: false}` | Disabled on CF | `manifest.yml` env var |

## Framework & Runtime Versions

| Component | Version | Source |
|---|---|---|
| Java (source and target) | 17 | `build.gradle` (`sourceCompatibility`, `targetCompatibility`) |
| Spring Boot | 3.1.5 | `build.gradle` plugin block |
| Spring Dependency Management Plugin | 1.1.3 | `build.gradle` plugin block |
| Gradle | 7.6.2 | `gradle/wrapper/gradle-wrapper.properties` |
| java-cfenv-boot | 3.1.2 | `build.gradle` (`ext.javaCfEnvVersion`) |
| Paketo Builder | `paketobuildpacks/builder-jammy-base:latest` | `build.gradle` `bootBuildImage` task |
| Spring Data JPA (Hibernate) | Boot-managed (≈ Hibernate 6.2.x for Boot 3.1.5) | Spring Boot BOM |
| Spring Data MongoDB | Boot-managed | Spring Boot BOM |
| Spring Data Redis (Jedis) | Boot-managed | Spring Boot BOM |
| Spring Boot Actuator | Boot-managed | Spring Boot BOM |
| JUnit | 4.x (Boot-managed) | `build.gradle` (`junit:junit` test dependency) |
| AngularJS | 1.2.16 | `build.gradle` (WebJar) |
| Bootstrap | 3.1.1 | `build.gradle` (WebJar) |
| jQuery | 2.1.0-2 | `build.gradle` (WebJar) |
