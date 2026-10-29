# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of jkutner's Heroku WildFly buildpack, customised to run the **myWMS / LOS warehouse-management EAR** on Heroku. It is a classic (v2) Heroku buildpack: three bash scripts under `bin/` plus a WildFly `standalone.xml` template and a PostgreSQL JDBC driver that get copied into the WildFly install at build time. There is no build system, no tests, and no linter; "testing" means pushing to a Heroku app that uses this buildpack and reading the build log.

## Runtime flow (`bin/compile`)

`bin/detect` always matches (prints "Wildfly"), `bin/release` emits an empty config. All real work happens in `bin/compile`, which runs with `set -e`:

1. Does **not** read any DB config at build time. Database credentials are resolved on dyno start (see step 5).
2. Fetches the `heroku/jvm-common` helper tarball and calls `install_java_with_overlay` to install a JDK into the build dir.
3. Fetches **WildFly 16.0.0.Final** (no checksum verification) and extracts it to `$BUILD_DIR/.jboss/wildfly-16.0.0.Final` (`JBOSS_HOME`). The tarball is stored in Heroku's build cache (`$CACHE_DIR/wildfly-<version>.tar.gz`) and only downloaded on the first build or after `heroku builds:cache:purge`; the source URL can be overridden with a `WILDFLY_URL` config var (e.g. an S3 mirror).
4. Deploys `05_myWMS-ear/target/*.ear` from the app's build dir (not `target/*.war` as the README still says) and the PostgreSQL JDBC jar from this repo into `standalone/deployments/`, renamed to the version-less `postgresql.jar`.
5. Copies this repo's `standalone.xml` unchanged into `standalone/configuration/`. The datasource uses WildFly expressions `${env.JDBC_DATABASE_URL}`, `${env.JDBC_DATABASE_USERNAME}`, `${env.JDBC_DATABASE_PASSWORD}`. Those are exported at dyno start by `.profile.d/jdbc.sh`, which jvm-common installs in step 2; it parses Heroku's `DATABASE_URL` (adding `sslmode=require` to the JDBC URL). Rotating the Postgres credentials therefore only needs a dyno restart, not a rebuild, and the password never reaches the build log or the slug.
6. Writes a `Procfile` (unless the app has one) that starts `standalone.sh -b=0.0.0.0 -Djboss.http.port=$PORT`, and a `.profile.d/jboss.sh` that exports `JBOSS_HOME`.

`standalone.conf` is **not** used by `bin/compile`; it is kept for reference / future use.

## Files and their relationships

- `standalone.xml` — the live WildFly 16 (`urn:jboss:domain:10.0`) config template. Key customisations relative to a stock profile:
  - Datasource `java:/losDS` (pool `PostgreSQLPool`) using the deployment-style driver name `postgresql.jar` (the name the jar gets in `standalone/deployments/`, independent of the versioned file in this repo); it is also the EE `default-bindings` datasource. Connection URL and credentials are `${env.JDBC_DATABASE_*}` expressions, resolved by WildFly at boot (see runtime flow step 5).
  - Legacy PicketBox `security` subsystem with the `los-login` domain: `DatabaseServerLoginModule` against `mywms_user` / `mywms_group…` tables via `java:/losDS`, MD5-hex hashes. `los-login` is the EJB3 default security domain; `los-system-realm` wraps it as a JAAS realm.
  - Messaging (ActiveMQ) subsystem and MDB resource adapter are enabled (the EAR uses JMS).
  - Logging: root level `TRACE`, verbose `io.undertow.*` loggers, `DEBUG` for `org.mywms`, `de.linogistix`, `de.melt`. Console/file pattern includes MDC keys `sessionUUID`, `userThreadID`, `userPrincipalName` (`%-4X{sessionUUID} … %C{1}.%M`).
- `postgresql-<version>.jar` — committed binary, deployed as a WildFly deployment (not a module). `bin/compile` copies it to `standalone/deployments/postgresql.jar`, so bumping it only means swapping the file and updating the source name in `bin/compile`; the `<driver>` element in `standalone.xml` stays `postgresql.jar`.

## Conventions / gotchas

- Changing the WildFly version means updating `WILDFLY_VERSION` (the download URL and cache file name derive from it) and supplying a `standalone.xml` whose subsystem namespace versions match that release. A jump to WildFly 25+ also requires migrating the `los-login` domain from PicketBox to Elytron; an earlier attempt at that (`standalone-25_0_1_Final.xml`, removed in git history) stalled on exactly this.
- The build log is the only debugging tool, which is why `bin/compile` is full of `echo`/`ls`/`cat` statements. Keep that in mind before "cleaning them up".
- History is almost entirely `Update compile` / `Update standalone.xml` commits made via the GitHub web UI; there is no branching workflow. `origin` is `ehrlich/heroku-buildpack-wildfly`; a second remote `247` points at `24Sevent-Connect/wildfly-heroku-buildpack`.
- The `.idea/` directory and `.DS_Store` are untracked and there is no `.gitignore`.

## Local sanity checks

There is no local runner, but the pieces can be checked without Heroku:

```sh
xmllint --noout standalone.xml
bash -n bin/compile   # syntax check only
```
