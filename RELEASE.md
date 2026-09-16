# MicroProfile Release Process

This document describes the release workflow for MicroProfile specifications published through the
Eclipse Foundation, using the Eclipse-managed Nexus instance at `repo3.eclipse.org`.

## Background

Eclipse Foundation helpdesk issue [#7213](https://gitlab.eclipse.org/eclipsefdn/helpdesk/-/work_items/7213)
requested a dedicated staging repository for MicroProfile so that release candidates can be staged,
verified by the community during the ballot period, and only then promoted to Maven Central — without
re-deploying, thereby preserving artifact hashes for certification requests.

The staging repository allocated for this project is:

```
https://repo3.eclipse.org/repository/microprofile-maven2-staging/
```

This is fixed as the default value of the `nexus.staging.repository` property in `pom.xml` and
does not need to be set on the command line.

---

## Workflow Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. Development  │  version: X.Y.Z-SNAPSHOT                         │
│                  │  mvn clean deploy  (snapshots → repo.eclipse.org) │
└──────────────────┴──────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  2. Stage RC     │  version: X.Y.Z                                   │
│                  │  mvn -Pstage-release clean deploy                 │
│                  │  → deploys signed artifacts to repo3.eclipse.org  │
│                  │  → consumers validate with -Pstaged               │
└──────────────────┴──────────────────────────────────────────────────┘
          │
          ▼  (ballot passes / community approves)
┌─────────────────────────────────────────────────────────────────────┐
│  3. Promote      │  version: X.Y.Z                                   │
│                  │  mvn -Ppromote-stage clean deploy                 │
│                  │  → syncs from repo3.eclipse.org → Maven Central   │
│                  │  → hashes preserved for EF certification          │
└──────────────────┴──────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  4. Next cycle   │  mvn versions:set -DnewVersion=X.Y.(Z+1)-SNAPSHOT│
└──────────────────┴──────────────────────────────────────────────────┘
```

---

## Maven Profiles Reference

### `stage-release` — Deploy to Eclipse Nexus staging

Deploys signed, release artifacts to `https://repo3.eclipse.org/repository/microprofile-maven2-staging/`.

**Requirements:**
- A GPG key must be available for signing.
- Your Maven `settings.xml` must have credentials for `repo3.eclipse.org` (see below).
- Snapshot versions are rejected — the deploy step skips any `*-SNAPSHOT` version.
- Maven ≥ 3.2.5 (enforced).

**Command:**

```bash
mvn -Pstage-release clean deploy
```

**What it does:**
1. Enforces Maven ≥ 3.2.5.
2. Attaches sources and Javadoc JARs.
3. Signs all artifacts with GPG (`--pinentry-mode loopback`).
4. Deploys to the staging repository via `altReleaseDeploymentRepository`
   (bypasses `distributionManagement`).
5. Uses `deployAtEnd=true` so nothing is uploaded if the build fails mid-way.

---

### `staged` — Consume staged artifacts (downstream consumers)

Adds `repo3.eclipse.org` as a repository so downstream projects can resolve staged
release candidates before they are promoted to Maven Central.

**Command:**

```bash
mvn -Pstaged clean verify
```

---

### `promote-stage` — Promote staging → Maven Central

Reads the staged artifacts from `repo3.eclipse.org` and syncs them to Maven Central using the
Eclipse CBI `central-staging-plugins:rc-sync` goal, **preserving the original hash values**.
This is required for Eclipse Foundation certification requests.

**Requirements:**
- JDK 17 or newer (enforced — `central-staging-plugins` requires JDK 17+).
- Eclipse Foundation credentials in `settings.xml` for the staging repo.

**Command:**

```bash
mvn -Ppromote-stage clean deploy
```

**What it does:**
1. Enforces JDK ≥ 17.
2. Skips the standard `maven-deploy-plugin` (nothing new is deployed from local).
3. Invokes `central-staging-plugins:rc-sync` to sync the already-staged artifacts to
   Maven Central with auto-publish and drop-after-publish enabled.

---

### `release` — Legacy direct release (deprecated)

> ⚠️ **This profile is kept for backwards compatibility only.**
> Prefer `-Pstage-release` + `-Ppromote-stage` for all new releases.

Attaches sources, signs artifacts with GPG, and deploys directly to
`https://repo.eclipse.org/content/repositories/microprofile-maven2-releases/`
(no staging, no hash-preservation guarantee).

**Command:**

```bash
mvn -Prelease clean deploy
```

---

### `staging` — Consume from legacy `repo.eclipse.org` staging (pre-existing)

Adds the old `https://repo.eclipse.org/content/repositories/microprofile-maven2-staging/` repository.
Activated by passing `-Dstaging` on the command line.

```bash
mvn -Dstaging clean verify
```

---

## `settings.xml` Configuration

You need server entries for the Eclipse Nexus instances.

```xml
<settings>
  <servers>

    <!-- Eclipse Foundation Nexus (repo3) — for stage-release and promote-stage -->
    <server>
      <id>repo3.eclipse.org</id>
      <username>your-eclipse-committer-id</username>
      <password>your-eclipse-password</password>
    </server>

    <!-- Legacy Eclipse Nexus — for snapshots / legacy release profile -->
    <server>
      <id>repo.eclipse.org</id>
      <username>your-eclipse-committer-id</username>
      <password>your-eclipse-password</password>
    </server>

  </servers>
</settings>
```

---

## Step-by-Step: Staging a Release Candidate

```bash
# 1. Prepare the release version (remove -SNAPSHOT)
mvn versions:set -DnewVersion=3.5 -DgenerateBackupPoms=false
git add pom.xml tck-bom/pom.xml
git commit -m "Prepare release 3.5"
git tag 3.5

# 2. Stage to repo3.eclipse.org (requires GPG key)
mvn -Pstage-release clean deploy -Drelease.revision=Final

# 3. Ask community to validate — consumers build against the staging repo with:
#    mvn -Pstaged ...

# 4. Once the ballot passes — promote to Maven Central
mvn -Ppromote-stage clean deploy

# 5. Bump to next snapshot
mvn versions:set -DnewVersion=3.6-SNAPSHOT -DgenerateBackupPoms=false
git add pom.xml tck-bom/pom.xml
git commit -m "Prepare for next development iteration"
git push && git push --tags
```

---

## Key Properties

| Property | Value | Description |
|---|---|---|
| `nexus.staging.repository` | `microprofile-maven2-staging` | Fixed name of the MicroProfile staging repo on `repo3.eclipse.org`. Provisioned via [helpdesk#7213](https://gitlab.eclipse.org/eclipsefdn/helpdesk/-/work_items/7213). |
| `release.revision` | `Draft` (default) | Set to `Final` to activate EF license (`efsl-1.1`) and TCK license (`eftckl-1.1`) profiles. |
| `version.eclipse.staging` | `1.4.3` | Version of `org.eclipse.cbi.central:central-staging-plugins` used by `-Ppromote-stage`. |

---

## Repository URLs Summary

| Purpose | URL |
|---|---|
| Stage deploy / consumer resolution | `https://repo3.eclipse.org/repository/microprofile-maven2-staging/` |
| Legacy releases | `https://repo.eclipse.org/content/repositories/microprofile-maven2-releases/` |
| Legacy snapshots | `https://repo.eclipse.org/content/repositories/microprofile-maven2-snapshots/` |
| Legacy staging (consumer) | `https://repo.eclipse.org/content/repositories/microprofile-maven2-staging/` |

---
