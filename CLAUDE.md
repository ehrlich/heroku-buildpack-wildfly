# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of jkutner's Heroku WildFly buildpack, customised to run the **myWMS / LOS warehouse-management EAR** on Heroku. It is a classic (v2) Heroku buildpack: three bash scripts under `bin/` plus a WildFly `standalone.xml` template and a PostgreSQL JDBC driver that get copied into the WildFly install at build time. There is no build system, no tests, and no linter; "testing" means pushing to a Heroku app that uses this buildpack and reading the build log.

## Runtime flow (`bin/compile`)

`bin/detect` always matches (prints "Wildfly"), `bin/release` emits an empty config. All real work happens in `bin/compile`, which runs with `set -e`:

1. Reads `DATABASE_URL` from the Heroku env dir (`$ENV_DIR/DATABASE_URL`) and splits it with bash parameter expansion into `DB_USER_ENV`, `DB_PASSWORD_ENV` and `DB_URL_ENV` (the latter is `jdbc:postgresql://` + the `HOST:PORT/DBNAME` part). The file is `cat`ed unconditionally, so **`DATABASE_URL` must be set on the Heroku app** (it is, whenever a Heroku Postgres add-on is attached) or the build aborts. All three derived values are echoed to the build log.
2. Fetches the `heroku/jvm-common` helper tarball and calls `install_java_with_overlay` to install a JDK into the build dir.
3. Downloads **WildFly 16.0.0.Final** from download.jboss.org (SHA1 verification is commented out), extracts it to `$BUILD_DIR/.jboss/wildfly-16.0.0.Final` (`JBOSS_HOME`).
4. Deploys `05_myWMS-ear/target/*.ear` from the app's build dir (not `target/*.war` as the README still says) and `postgresql-42.7.2.jar` from this repo into `standalone/deployments/`.
5. Renders `standalone.xml`: copies this repo's `standalone.xml`, runs `sed` to replace the literal placeholders `DB_CONNECTION_URL`, `DB_USER_NAME`, `DB_USER_PASSWORD` with the env values, and installs the result as `standalone/configuration/standalone.xml`. The rendered XML (including the password) is echoed to the build log twice.
6. Writes a `Procfile` (unless the app has one) that starts `standalone.sh -b=0.0.0.0 -Djboss.http.port=$PORT`, and a `.profile.d/jboss.sh` that exports `JBOSS_HOME`.

`standalone.conf` is **not** copied (the `cp` is commented out); it is kept for reference / future use. The same is true of `standalone-25_0_1_Final.xml`.

## Files and their relationships

- `standalone.xml` — the live WildFly 16 (`urn:jboss:domain:10.0`) config template. Key customisations relative to a stock profile:
  - Datasource `java:/losDS` (pool `PostgreSQLPool`) using the deployment-style driver name `postgresql-42.7.2.jar`; it is also the EE `default-bindings` datasource. Credentials are the three `sed` placeholders above — keep them literal and unique in the file, since the substitution is plain `sed s,…,…,` with comma as delimiter (so a DB URL/password containing a comma will break it).
  - Legacy PicketBox `security` subsystem with the `los-login` domain: `DatabaseServerLoginModule` against `mywms_user` / `mywms_group…` tables via `java:/losDS`, MD5-hex hashes. `los-login` is the EJB3 default security domain; `los-system-realm` wraps it as a JAAS realm.
  - Messaging (ActiveMQ) subsystem and MDB resource adapter are enabled (the EAR uses JMS).
  - Logging: root level `TRACE`, verbose `io.undertow.*` loggers, `DEBUG` for `org.mywms`, `de.linogistix`, `de.melt`. Console/file pattern includes MDC keys `sessionUUID`, `userThreadID`, `userPrincipalName` (`%-4X{sessionUUID} … %C{1}.%M`).
- `standalone-25_0_1_Final.xml` — an older attempt at WildFly 25.0.1 (`urn:jboss:domain:18.0`, Elytron-only, no legacy security subsystem). Unused; the WildFly 25 download/copy lines in `bin/compile` are commented out. It contains hard-coded (stale) Heroku Postgres credentials rather than placeholders.
- `postgresql-42.7.2.jar` — committed binary, deployed as a WildFly deployment (not a module). If you bump it, update both the file name in `bin/compile` and the `<driver>` element in `standalone.xml`.

## Conventions / gotchas

- Changing the WildFly version means updating `WILDFLY_VERSION`, the hard-coded `wget` URL, and supplying a `standalone.xml` whose subsystem namespace versions match that release. The 16→25 jump also requires migrating the `los-login` domain from PicketBox to Elytron, which is why that migration stalled.
- The build log is the only debugging tool, which is why `bin/compile` is full of `echo`/`ls`/`cat` statements. Keep that in mind before "cleaning them up".
- History is almost entirely `Update compile` / `Update standalone.xml` commits made via the GitHub web UI; there is no branching workflow. `origin` is `ehrlich/heroku-buildpack-wildfly`; a second remote `247` points at `24Sevent-Connect/wildfly-heroku-buildpack`.
- The `.idea/` directory and `.DS_Store` are untracked and there is no `.gitignore`.

## Local sanity checks

There is no local runner, but the `sed` rendering step can be exercised without Heroku:

```sh
sed -e 's,DB_CONNECTION_URL,jdbc:postgresql://host:5432/db,' \
    -e 's,DB_USER_NAME,user,' \
    -e 's,DB_USER_PASSWORD,secret,' standalone.xml | xmllint --noout -
bash -n bin/compile   # syntax check only
```
