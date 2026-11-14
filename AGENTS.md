# AGENTS.md — Spring PetClinic (AngularJS + Spring Boot)

## What this app is
Sample veterinary-clinic app (Spring PetClinic, AngularJS fork). Clinic staff list/search owners,
view an owner with pets and visits, create/edit owners and pets, add visits, and list vets with
specialties. A Spring Boot REST API serves JSON; an AngularJS 1.6 single-page app (packaged as a
webjar) is the only UI. Default data: in-memory HSQLDB seeded at startup.

## Archetype
`A-boot-spa` — Spring Boot 2.1.3 backend with REST controllers + AngularJS SPA built with Bower/Gulp
(see `devin-modernization-kit/catalog/archetypes.md`).

## Layout
| Path | Purpose |
|---|---|
| `pom.xml` | Parent POM; parent `spring-boot-starter-parent` 2.1.3.RELEASE; modules client, server |
| `spring-petclinic-client/` | AngularJS 1.6.10 SPA: `src/scripts/` with one folder per screen (module, controller, component, template), `src/less`, `src/css`; `bower.json`, `package.json`, `gulpfile.js`; built into a jar with assets under `public/` |
| `spring-petclinic-server/` | Spring Boot REST API: `web/*Resource.java` (11 endpoints), `service/`, `repository/` (Spring Data JPA), `model/`, `config/` (Ehcache via JCache, web config); `templates/index.html` (Thymeleaf host page loading the SPA) |
| `spring-petclinic-server/src/main/resources/db/{hsqldb,mysql}/` | `schema.sql` + `data.sql` per vendor (no migration tool) |
| `docker-compose.yml`, `spring-petclinic-server/src/main/docker/Dockerfile` | Legacy MySQL 5.7 + Docker Hub image setup (not used by the build) |
| `MODERNIZATION_PLAN.md`, `inventory.json` | Modernization plan and machine-readable inventory |

## Commands
Run from `~/repos/spring-petclinic-angularjs`. All verified in the assessment session.

| Purpose | Command |
|---|---|
| Build | `JAVA_HOME=$JAVA_HOME_8 ./mvnw -B clean install` |
| Test | `JAVA_HOME=$JAVA_HOME_8 ./mvnw -B verify` (baseline: 3 tests, all pass) |
| Run | `cd spring-petclinic-server && JAVA_HOME=$JAVA_HOME_8 ../mvnw -B spring-boot:run` → `http://localhost:8080` |
| UI smoke | Not present yet — added by the characterization-tests task (T3, Playwright in `e2e/`) |

Current JDK: `8` — select with `JAVA_HOME=$JAVA_HOME_8`.

Run needs the build first: the server resolves `spring-petclinic-client` from `~/.m2`.

## Database
- Default profiles `hsqldb,prod` (`application.properties:15`): in-memory HSQLDB `jdbc:hsqldb:mem:petclinic`,
  schema/data from `db/hsqldb/schema.sql` + `data.sql` via `spring.datasource.schema/data`, `ddl-auto=none`.
- Optional `mysql` profile (`application-mysql.properties`) points at host `mysql-petclinic` (docker-compose).
- Target: Postgres with Flyway migrations (tasks T19/T20).

## Baseline tests
- `VetResourceTests`, `PetResourceTests` (`@WebMvcTest`, JUnit 4), `ValidatorTests`: 3 tests, pass on JDK 8.
- `AbstractClinicServiceTests` (10 `@Test` methods) is abstract with no subclass → never runs.
- No API, e2e, or UI tests; coverage not measured (no JaCoCo).

## Modernization
- Plan: `MODERNIZATION_PLAN.md`. Tasks are GitHub issues labelled `modernization`.
- Procedures: skills in `devin-modernization-kit` (`@skills:` + the skill named in the issue, e.g. `@skills:baseline-build-ci 8`).
- Definition of Done per task type: `devin-modernization-kit/catalog/task-catalog.md`.

## Guardrails
- Run the Test command (and UI smoke once it exists) before opening a PR; paste commands and results in the PR.
- One issue per PR. Do not bundle unrelated changes or drive-by refactors.
- Never delete, `@Disabled`/`@Ignore`, skip, or weaken tests or assertions to make a build pass. If a test is genuinely obsolete, explain why in the PR and ask.
- Never merge PRs. Never force-push to the default branch.
- Do not change public API contracts (URLs, JSON shapes, DB schema) unless the issue says so.
- Do not commit secrets, credentials, or generated build output.
- If blocked for more than ~3 attempts on the same error, stop and report what you tried.

## Known gotchas
- Use `install`, not `package`, from the root. `./mvnw -B -pl spring-petclinic-server verify` without a prior
  install fails with `Could not find artifact org.springframework.samples:spring-petclinic-client:jar:2.1.3`.
- The client build downloads its own Node v4.4.3 + npm 2.15.1 into `spring-petclinic-client/target/node`
  (frontend-maven-plugin 1.3); system Node is not used. `node_modules/` and `bower_components/` are created
  in `spring-petclinic-client/` and are gitignored.
- `pom.xml:63-74` declares `repo.spring.io/milestone`; Maven queries it first for every artifact — slow but
  harmless, and not needed for `.RELEASE` versions (`pom.xml:62`); removed in T1.
- Static assets are served from the client jar's `public/` folder. The `dev` profile
  (`application-dev.properties:1`) instead serves `spring-petclinic-client/src` and `bower_components` from disk,
  relative to the working directory, so start from the repo root when using `dev`.
- UI routes use ui-router with `hashPrefix('!')`: URLs look like `http://localhost:8080/#!/owners`.
- REST endpoints sit at the root (no `/api` prefix); actuator is at `/manage` (`management.endpoints.web.base-path`),
  e.g. `/manage/health`, with all endpoints exposed (`exposure.include=*`).
- `GET /owners/*/pets/{petId}` uses a wildcard segment; keep it in mind for Boot 3 path-matching changes.
- `git-commit-id-plugin` runs during the server build and needs the `.git` directory.
