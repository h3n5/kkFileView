# AGENTS.md

Guidance for AI agents working in this repository. kkFileView is a Spring Boot document-preview service: it downloads a file from a URL and renders it in the browser, converting Office/CAD/etc. formats via LibreOffice and other libraries.

## Build & run

- **Java 21** required (set in root `pom.xml`; CI uses Temurin 21). No Maven wrapper is committed — use a system `mvn`.
- Build the whole project from the repo root: `mvn -B package` (CI adds `-Dmaven.test.skip=true`; tests are not run in CI).
- Run locally from an IDE via `cn.keking.ServerMain.main` (module `server`), then open `http://localhost:8012/`.
- The runnable artifact is `server/target/kkFileView-*.jar`; `maven-assembly-plugin` also produces `kkFileView-*.tar.gz` (Linux) and `kkFileView-*.zip` (Windows) bundles containing `bin/` scripts + `config/` + the jar. Distribution descriptors live in `server/src/main/assembly/`.
- Docker: root `Dockerfile` builds on `keking/kkfileview-base:4.4.0` (Ubuntu 24.04 + `libreoffice-nogui` + JDK 21 + Chinese fonts, defined in `docker/kkfileview-base/Dockerfile`). The base image must exist before building the app image.

## Runtime dependencies (important)

- **LibreOffice is mandatory.** `OfficePluginManager` (`@PostConstruct`, `@Order(HIGHEST_PRECEDENCE)`) starts a `LocalOfficeManager` and throws if no office home is found, failing startup. Windows ships a bundled `server/LibreOfficePortable/`; Linux `startup.sh` auto-runs `install.sh` to install LibreOffice if missing; macOS requires manual install. Set `office.home` (or `KK_OFFICE_HOME`) only to override auto-detection.
- ffmpeg/OpenCV (via JavaCV, classifiers for `linux-x86_64` and `windows-x86_64` only) are bundled for video transcoding; other platforms need matching bytedeco classifiers added to `server/pom.xml`.
- Two **system-scoped** jars live in `server/lib/` (`jai_core`, `jai_codec`); `spring-boot-maven-plugin` is configured with `includeSystemScope=true` so they end up in the repacked jar. Do not remove this setting.

## Project layout

- Maven multi-module: parent `pom.xml` (groupId `cn.keking`, version `4.4.0`) with a single `server` module.
- Java sources: `server/src/main/java/cn/keking/` → `config/`, `model/`, `service/` (+ `service/impl/`, `service/cache/impl/`), `utils/`, `web/controller/`, `web/filter/`.
- **Config lives in `server/src/main/config/application.properties`** (NOT `src/main/resources`) and has Maven `filtering=true`. Freemarker templates are in `server/src/main/resources/web/` (suffix `.ftl`, loaded from `classpath:/web/`).
- All config keys support env-var overrides (`KK_*`); see `application.properties` for the full list.

## Architecture: preview dispatch (the central pattern)

Request flow for preview: `OnlinePreviewController.onlinePreview` → `WebUtils.decodeUrl` (Base64 + urlEncode) → `FileHandlerService.getFileAttribute` builds a `FileAttribute` (incl. `FileType`) → `FilePreviewFactory.get(fileAttribute)` looks up a Spring bean **by name** → the bean's `filePreviewHandle` returns a Freemarker view name.

- `FileType` enum (`model/FileType.java`) maps file extensions → a **Spring bean name** (e.g. `OFFICE("officeFilePreviewImpl")`). The mapping table is built in a static block.
- Each `FilePreview` implementation is a `@Service` whose bean name matches the `FileType.instanceName`. Implementations live in `service/impl/` (23 of them, e.g. `OfficeFilePreviewImpl`, `PdfFilePreviewImpl`, `CompressFilePreviewImpl`).
- **To add a new file format**: add the extension(s) to the relevant `*_TYPES` array (or a new enum constant) in `FileType`, ensure the matching `*FilePreviewImpl` bean exists with the right name, and add a Freemarker template if a new view is needed. View-name constants are defined on the `FilePreview` interface.
- `OtherFilePreviewImpl` is the fallback (`FileType.OTHER`) and provides `notSupportedFile(...)` helpers used everywhere.

## Config: static-field pattern

`ConfigConstants` is a `@Component` that holds **static** fields, populated by instance `@Value` setters (plus matching `setXxxValue` static methods for runtime updates). This is intentional — read config via `ConfigConstants.getXxx()` from anywhere. `ConfigRefreshComponent` refreshes some values at runtime. When adding config, follow the same `@Value` + static setter + `setXxxValue` triplet.

## Cache: three implementations selected by `cache.type`

`CacheService` has three `@ConditionalOnExpression` implementations in `service/cache/impl/`:
- `cache.type=default` (unset) → `CacheServiceRocksDBImpl` (embedded RocksDB)
- `cache.type=jdk` → `CacheServiceJDKImpl` (in-memory `ConcurrentLinkedHashMap`)
- `cache.type=redis` → `CacheServiceRedisImpl` (requires `spring.redisson.address` etc.; `RedissonConfig` is also gated on `cache.type=redis`)

`application.properties` defaults `cache.type=jdk`. Only one impl is active at a time.

## Security (do not regress)

- **Since 4.4.0, all external file-preview requests are denied by default** to prevent SSRF. `TrustHostFilter` blocks `/onlinePreview`, `/picturesPreview`, `/getCorsFile` unless `trust.host` (or `KK_TRUST_HOST`) is configured. `*` allows all (test only). `not.trust.host` / `KK_NOT_TRUST_HOST` is a blacklist with higher priority. See `SECURITY_CONFIG.md`.
- The filter chain order is significant and set in `WebConfig`: `ChinesePathFilter`(10) → `BaseUrlFilter`(20) → `UrlCheckFilter`(30); `TrustHostFilter`/`TrustDirFilter`/`AttributeSetFilter` are URL-pattern scoped. Preserve orders when editing.
- URLs are double-encoded (Base64 + urlEncode) end-to-end; use `WebUtils.decodeUrl` / `WebUtils.urlEncoderencode` rather than rolling your own.

## Testing notes

- Tests use JUnit 5 + `@SpringBootTest`, so they boot the full Spring context — **LibreOffice must be available** or context loading fails. This is why CI skips tests.
- `EncodingTests` references test resources with a **Windows path separator** (`"testData\\" + i`); it will not pass on Linux/macOS without changing the path. Fix when touching that test.
- Run a single test class: `mvn -pl server test -Dtest=WebUtilsTests`.

## CI quirks

- `.github/workflows/maven.yml` (GitHub Actions): JDK 21, builds + assembles + builds Docker images.
- `.workflow/MasterPipeline.yml` (Gitee/DevOps pipeline): **stale — declares JDK 8**, which cannot compile this Java-21 codebase. Do not trust it as a source of truth; update or ignore.
